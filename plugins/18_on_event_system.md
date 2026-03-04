# Plugin 18 — on() 事件系统完整规范

> 来源验证：
> - `plugins/discourse-reactions/plugin.rb`（`on(:first_post_moved)`、`on(:site_setting_changed)`）
> - `plugins/discourse-assign/plugin.rb`（`on(:post_created/edited/destroyed/recovered/moved)`、`on(:topic_status_updated)`、`on(:user_added_to_group)`、`on(:group_destroyed)`、`on(:move_to_inbox/archive_message)`）
> - `plugins/discourse-solved/plugin.rb`（`on(:post_destroyed)`、`on(:filter_auto_bump_topics)`、`on(:before_post_publish_changes)`）
> - `plugins/discourse-ai/lib/ai_helper/entry_point.rb`（`plugin.on(:chat_message_created)`）

## 概述

`on(:event_name) { |args| }` 是 Discourse 插件订阅核心事件的机制，底层是 `DiscourseEvent`。与 `register_modifier` 的区别：**`on()` 是副作用（fire-and-forget），不影响返回值；`register_modifier` 是数据变换，返回值被使用**。

---

## 核心规范

### 1. 基本用法

```ruby
# plugin.rb — 始终在 after_initialize 内注册
after_initialize do
  on(:post_created) do |post|
    next unless SiteSetting.my_plugin_enabled?   # 1. SiteSetting 守卫（最先检查）
    next if post.topic.private_message?           # 2. 业务条件守卫
    MyPlugin::Processor.process(post)             # 3. 执行副作用
  end
end
```

**守卫条件顺序**（从最廉价到最昂贵）：
1. `SiteSetting.xxx?` — 最廉价，内存读取
2. `opts` 参数检查（如 `next if opts[:silent]`）
3. 对象状态检查（`post.topic.private_message?`）
4. 数据库查询（尽量避免，或放在最后）

### 2. EntryPoint 中使用 on()

大型插件将 `on()` 注册放在 EntryPoint 模块：

```ruby
# lib/my_plugin/feature/entry_point.rb
module MyPlugin
  module Feature
    class EntryPoint
      def inject_into(plugin)
        plugin.on(:post_created) do |post|
          next unless SiteSetting.my_plugin_enabled?
          MyPlugin::Feature::PostHandler.process(post)
        end

        plugin.on(:chat_message_created) do |message, channel, user, extra|
          next unless SiteSetting.my_plugin_enabled?
          MyPlugin::Feature::ChatHandler.process(message, user)
        end
      end
    end
  end
end

# plugin.rb
after_initialize do
  MyPlugin::Feature::EntryPoint.new.inject_into(self)
end
```

### 3. 数据迁移事件 — 关联数据随内容移动

当帖子被移动到其他话题时，插件关联数据必须同步迁移（来自 discourse-reactions 真实代码）：

```ruby
# on(:first_post_moved)：话题第一帖被移走（话题合并/拆分场景）
on(:first_post_moved) do |target_post, original_post|
  id_map = {}

  ActiveRecord::Base.transaction do
    reactions = MyPlugin::Reaction.where(post_id: original_post.id)
    next unless reactions.any?

    # 批量插入到新 post_id（用 insert_all 避免逐条）
    reactions_attributes =
      reactions.map { |r| r.attributes.except("id").merge(post_id: target_post.id) }
    MyPlugin::Reaction
      .insert_all(reactions_attributes)
      .each_with_index { |entry, i| id_map[reactions[i].id] = entry["id"] }

    # 迁移关联的子记录（需要更新外键映射）
    reaction_users = MyPlugin::ReactionUser.where(post_id: original_post.id)
    next unless reaction_users.any?

    reaction_users_attributes =
      reaction_users.map do |ru|
        ru.attributes.except("id").merge(
          post_id: target_post.id,
          reaction_id: id_map[ru.reaction_id],  # 用映射表更新关联 ID
        )
      end
    MyPlugin::ReactionUser.insert_all(reaction_users_attributes)
  end
end

# on(:post_moved)：普通帖子被移到其他话题（discourse-assign 真实代码）
on(:post_moved) do |post, original_topic_id|
  assignment =
    Assignment.where(topic_id: original_topic_id, target_type: "Post", target_id: post.id).first
  next unless assignment

  if post.is_first_post?
    # 第一帖移走意味着话题本身被移走，需要把 target 改成 Topic
    assignment.update!(topic_id: post.topic_id, target_type: "Topic", target_id: post.topic_id)
  else
    assignment.update!(topic_id: post.topic_id)
  end
end
```

### 4. 清理事件 — 级联删除插件数据

```ruby
# on(:post_destroyed)：帖子删除时清理关联（discourse-solved 真实代码）
on(:post_destroyed) { |post| MyPlugin::Manager.cleanup_for_post(post) }

# on(:post_destroyed)：复杂清理逻辑（discourse-assign 真实代码）
on(:post_destroyed) do |post|
  if Assignment.active.exists?(target: post)
    post.assignment.deactivate!
    MessageBus.publish("/topic/#{post.topic_id}", reload_topic: true, refresh_stream: true)
  end

  # 清理关联的 small_action 帖（避免孤儿记录）
  PostCustomField
    .where(name: "action_code_post_id", value: post.id)
    .find_each do |pcf|
      next if pcf.post.nil?
      next unless [Post.types[:small_action], Post.types[:whisper]].include?(pcf.post.post_type)
      pcf.post.destroy
    end
end

# on(:post_recovered)：帖子恢复时重新激活（discourse-assign 真实代码）
on(:post_recovered) do |post|
  if SiteSetting.reassign_on_open && Assignment.inactive.exists?(target: post)
    post.assignment.reactivate!
    MessageBus.publish("/topic/#{post.topic_id}", reload_topic: true, refresh_stream: true)
  end
end

# on(:group_destroyed)：组删除时清理（discourse-assign 真实代码）
on(:group_destroyed) do |group, user_ids|
  User.where(id: user_ids).find_each do |user|
    user.notifications.for_assignment(group.assignments.select(:id)).destroy_all
  end
  Assignment.active_for_group(group).destroy_all
end
```

### 5. 状态变更事件

```ruby
# on(:topic_status_updated)：话题关闭/开启时联动（discourse-assign 真实代码）
on(:topic_status_updated) do |topic, status, enabled|
  if SiteSetting.unassign_on_close &&
     (status == "closed" || status == "autoclosed") &&
     enabled &&
     Assignment.active.exists?(topic: topic)

    assigner = ::Assigner.new(topic, Discourse.system_user)
    assigner.unassign(silent: true, deactivate: true)
    MessageBus.publish("/topic/#{topic.id}", reload_topic: true, refresh_stream: true)
  end

  if SiteSetting.reassign_on_open &&
     (status == "closed" || status == "autoclosed") &&
     !enabled &&
     Assignment.inactive.exists?(topic: topic)

    Assignment.reactivate!(topic: topic)
    MessageBus.publish("/topic/#{topic.id}", reload_topic: true, refresh_stream: true)
  end
end

# on(:site_setting_changed)：设置变更时重新调度 Job（discourse-reactions 真实代码）
on(:site_setting_changed) do |name, old_value, new_value|
  if name == :my_plugin_excluded_setting && SiteSetting.my_plugin_sync_enabled
    ::Jobs.cancel_scheduled_job(Jobs::MyPlugin::Synchronizer)
    ::Jobs.enqueue_at(5.minutes.from_now, Jobs::MyPlugin::Synchronizer)
  end
end
```

### 6. 过滤事件 — 修改查询集合

```ruby
# on(:filter_auto_bump_topics)：排除已解答话题（discourse-solved 真实代码）
on(:filter_auto_bump_topics) do |_category, filters|
  filters.push(
    ->(r) do
      r.where(
        "NOT EXISTS (
          SELECT 1 FROM my_plugin_solved_topics
          WHERE my_plugin_solved_topics.topic_id = topics.id
        )",
      )
    end,
  )
end
```

### 7. 前置修改事件 — 在核心逻辑前介入

```ruby
# on(:before_post_publish_changes)：帖子编辑发布前修改选项（discourse-solved 真实代码）
on(:before_post_publish_changes) do |post_changes, topic_changes, options|
  category_id_changes = topic_changes.diff["category_id"].to_a
  tag_changes = topic_changes.diff["tags"].to_a

  old_allowed = Guardian.new.allow_accepted_answers?(category_id_changes[0], tag_changes[0])
  new_allowed = Guardian.new.allow_accepted_answers?(category_id_changes[1], tag_changes[1])

  # 分类/标签变更影响功能可用性时，强制刷新 stream
  options[:refresh_stream] = true if old_allowed != new_allowed
end
```

### 8. 用户/群组事件

```ruby
# on(:user_added_to_group)：加入群组时补发通知（discourse-assign 真实代码）
on(:user_added_to_group) do |user, group, automatic:|
  group.assignments.active.find_each do |assignment|
    Jobs.enqueue(:assign_notification, assignment_id: assignment.id)
  end
end

# on(:user_removed_from_group)：离开群组时清理通知
on(:user_removed_from_group) do |user, group|
  user.notifications.for_assignment(group.assignments.select(:id)).destroy_all
end

# on(:move_to_inbox) / on(:archive_message)：私信状态变化（discourse-assign 真实代码）
on(:move_to_inbox) do |info|
  topic = info[:topic]
  TopicTrackingState.publish_assigned_private_message(topic, topic.assignment.assigned_to) if topic.assignment
  next unless SiteSetting.unassign_on_group_archive && info[:group]
  Assignment.reactivate!(topic: topic)
end

on(:archive_message) do |info|
  topic = info[:topic]
  next unless topic.assignment
  TopicTrackingState.publish_assigned_private_message(topic, topic.assignment.assigned_to)
  next unless SiteSetting.unassign_on_group_archive && info[:group]
  Assignment.deactivate!(topic: topic)
end
```

---

## 常用事件速查表

| 事件 | 参数 | 典型用途 |
|------|------|---------|
| `post_created` | `post` | 自动处理新帖 |
| `post_edited` | `post, topic_changed` | 编辑后重新处理 |
| `post_destroyed` | `post` | 级联清理关联数据 |
| `post_recovered` | `post` | 恢复后重新激活 |
| `post_moved` | `post, original_topic_id` | 迁移关联数据 |
| `first_post_moved` | `target_post, original_post` | 话题合并时迁移 |
| `before_post_publish_changes` | `post_changes, topic_changes, options` | 编辑发布前修改选项 |
| `topic_created` | `topic` | 新话题处理 |
| `topic_destroyed` | `topic` | 清理话题关联 |
| `topic_status_updated` | `topic, status, enabled` | 关闭/归档联动 |
| `filter_auto_bump_topics` | `category, filters` | 修改自动置顶查询 |
| `site_setting_changed` | `name, old_value, new_value` | 设置变更时重调度 |
| `user_created` | `user` | 新用户初始化 |
| `user_destroyed` | `user` | 清理用户数据 |
| `user_added_to_group` | `user, group, automatic:` | 加组联动 |
| `user_removed_from_group` | `user, group` | 离组清理 |
| `group_destroyed` | `group, user_ids` | 组删除清理 |
| `move_to_inbox` | `info (topic:, group:)` | 私信移入收件箱 |
| `archive_message` | `info (topic:, group:)` | 私信归档 |
| `chat_message_created` | `message, channel, user, extra` | Chat 消息处理 |
| `assign_topic` | `topic, user, assigning_user, force` | 自定义指派触发 |
| `unassign_topic` | `topic, unassigning_user` | 自定义取消指派 |

---

## on() vs register_modifier 选择规则

| 场景 | 用 `on()` | 用 `register_modifier` |
|------|-----------|----------------------|
| 执行副作用（发通知、写 DB、清缓存） | ✓ | — |
| 返回值被调用方使用 | — | ✓ |
| 修改查询集合/数组（filters.push） | ✓ | — |
| 修改字符串/Hash 返回值 | — | ✓ |
| 监听核心事件触发 | ✓ | — |

---

## 反模式

```ruby
# 错误：on() 放在 after_initialize 外
on(:post_created) { |post| ... }  # 插件加载时 Discourse 可能未就绪
# 正确：始终在 after_initialize 内
after_initialize { on(:post_created) { |post| ... } }

# 错误：无守卫条件（插件禁用时仍执行）
on(:post_created) do |post|
  MyPlugin::Processor.process(post)  # 插件禁用也会跑！
end
# 正确：第一行检查 SiteSetting
on(:post_created) do |post|
  next unless SiteSetting.my_plugin_enabled?
  MyPlugin::Processor.process(post)
end

# 错误：在 on() 中调用同步外部 API（阻塞请求线程）
on(:post_created) do |post|
  ExternalApi.call(post.id)  # 如果 API 慢，整个 post 创建流程被卡住
end
# 正确：用 Job 异步处理
on(:post_created) do |post|
  next unless SiteSetting.my_plugin_enabled?
  Jobs.enqueue(:my_plugin_process_post, post_id: post.id)
end

# 错误：期望 on() 的返回值被使用（on 是 fire-and-forget）
on(:post_created) { |post| "some_value" }  # 返回值被丢弃
```
