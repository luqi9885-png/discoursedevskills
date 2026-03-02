# Patterns 03 — DiscourseEvent 事件系统

> 来源：`lib/discourse_event.rb`、`lib/plugin/instance.rb`（on/on_enabled_change）、
> `plugins/discourse-solved/plugin.rb`（实际使用案例）

---

## 概述

DiscourseEvent 是 Discourse 的发布-订阅（pub/sub）系统，用于核心代码与插件之间的解耦通信。核心代码 `trigger` 事件，插件 `on` 订阅，两者无直接依赖。

```
核心代码                          插件
────────                          ────
DiscourseEvent.trigger(:user_created, user)
                                  on(:user_created) { |user| do_something(user) }
```

---

## 1. 底层实现

```ruby
class DiscourseEvent
  def self.events
    @events ||= Hash.new { |hash, key| hash[key] = Set.new }
  end

  def self.trigger(event_name, *args, **kwargs)
    events[event_name].each { |event| event.call(*args, **kwargs) }
  end

  def self.on(event_name, &block)
    events[event_name] << block
  end

  def self.off(event_name, &block)
    events[event_name].delete(block)
  end

  def self.all_off(event_name)
    events.delete(event_name)
  end
end
```

**数据结构**：`Hash<Symbol, Set<Proc>>`。同一事件可有多个监听器，触发顺序为注册顺序。

---

## 2. Plugin 中的 on()（推荐用法）

Plugin 内部**必须用 `on()`** 而非 `DiscourseEvent.on()`。区别：

```ruby
# plugin.rb 内
on(:post_destroyed) { |post| DiscourseSolved.unaccept_answer!(post) }

# 底层实现（plugin/instance.rb）：
def on(event_name, &block)
  DiscourseEvent.on(event_name) do |*args, **kwargs|
    block.call(*args, **kwargs) if enabled?   # ← 自动 guard enabled?
  end
end
```

**关键差异**：Plugin 的 `on()` 会在 `enabled?` 为 false 时**静默跳过**，`DiscourseEvent.on()` 没有这个保护。

---

## 3. 常用事件一览

### 用户类

| 事件 | 触发时机 | 参数 |
|------|----------|------|
| `:user_created` | 用户注册 | `user` |
| `:user_logged_in` | 用户登录 | `user` |
| `:user_logged_out` | 用户登出 | `user` |
| `:user_updated` | 用户资料更新 | `user` |
| `:user_confirmed_email` | 邮件确认 | `user` |
| `:user_approved` | 用户被审批 | `user` |
| `:user_destroyed` | 用户被删除 | `user` |
| `:username_changed` | 用户名变更 | `user, old_username, new_username` |
| `:user_badge_granted` | 徽章授予 | `badge_id, user_id` |
| `:user_badge_revoked` | 徽章撤销 | `badge_id, user_id` |
| `:user_added_to_group` | 加入 group | `user, group` |
| `:user_removed_from_group` | 退出 group | `user, group` |

### 话题 / 帖子类

| 事件 | 触发时机 | 参数 |
|------|----------|------|
| `:topic_trashed` | 话题软删除 | `topic` |
| `:topic_recovered` | 话题恢复 | `topic` |
| `:topic_status_updated` | 状态变更（关闭/归档等） | `topic, status, enabled` |
| `:topic_merged` | 话题合并 | `orig, destination` |
| `:post_destroyed` | 帖子软删除 | `post, opts, user` |
| `:posts_destroyed` | 批量软删除 | `posts, opts, user` |
| `:post_moved` | 帖子移动 | `post, original_topic_id` |
| `:post_owner_changed` | 帖子归属变更 | `post` |

### 审核类

| 事件 | 触发时机 | 参数 |
|------|----------|------|
| `:flag_agreed` | Flag 被同意 | `post_action, post, user` |
| `:flag_disagreed` | Flag 被驳回 | `post_action, post, user` |
| `:flag_deferred` | Flag 被推迟 | `post_action, post, user` |
| `:queued_post_created` | 帖子进入审核队列 | `reviewable` |
| `:approved_post` | 帖子通过审核 | `reviewable, reviewable_perform_result` |
| `:rejected_post` | 帖子被拒绝 | `reviewable, reviewable_perform_result` |

### 系统类

| 事件 | 触发时机 | 参数 |
|------|----------|------|
| `:notification_created` | 通知创建 | `notification` |
| `:site_setting_changed` | SiteSetting 变更 | `setting_name, old_value, new_value` |
| `:category_updated` | 分类更新 | `category` |

### 邮件 / 通知推送类

| 事件 | 触发时机 | 参数 |
|------|----------|------|
| `:before_create_notification` | 通知创建前 | `user, type, post, opts` |
| `:before_create_notifications_for_users` | 批量通知前 | `users, type, post, opts` |
| `:push_notification` | 推送通知 | `user, payload` |

### Plugin 扩展点（filter/hook 类）

| 事件 | 用途 | 参数 |
|------|------|------|
| `:filter_auto_bump_topics` | 过滤自动 bump 话题列表 | `category, filters` |
| `:before_post_publish_changes` | 发布帖子变更前 | `post_changes, topic_changes, options` |
| `:notify_mailing_list_subscribers` | 邮件列表通知前 | `topic, post, subscribers` |
| `:post_alerter_before_post` | Post 通知前 | `post, new_record, notified` |

---

## 4. 完整 Plugin 用法示例

```ruby
# plugin.rb
after_initialize do
  # 基本用法：帖子删除时取消接受答案
  on(:post_destroyed) do |post|
    DiscourseSolved.unaccept_answer!(post)
  end

  # 带参数过滤：修改 auto bump 逻辑
  on(:filter_auto_bump_topics) do |_category, filters|
    filters.push(
      ->(r) do
        r.where("NOT EXISTS (SELECT 1 FROM solved_topics WHERE topic_id = topics.id)")
      end,
    )
  end

  # 帖子发布变更前：检查分类变化
  on(:before_post_publish_changes) do |post_changes, topic_changes, options|
    category_id_changes = topic_changes.diff["category_id"].to_a
    if category_id_changes.present?
      # 分类变了，做相应处理
    end
  end
end
```

---

## 5. 插件自定义事件（trigger）

Plugin 也可以触发自定义事件供其他插件监听：

```ruby
# plugin A 触发
DiscourseEvent.trigger(:my_plugin_something_happened, resource, acting_user)

# plugin B 监听
on(:my_plugin_something_happened) do |resource, acting_user|
  # 响应 plugin A 的事件
end
```

**命名规范**：用 `snake_case`，加 plugin 前缀避免冲突，如 `:discourse_solved_answer_accepted`。

---

## 6. on_enabled_change（插件开关监听）

```ruby
# plugin.rb
on_enabled_change do |old_value, new_value|
  if new_value
    # 插件被启用：初始化缓存、注册定时任务等
    MyPlugin.setup!
  else
    # 插件被禁用：清理资源
    MyPlugin.teardown!
  end
end
```

底层监听 `:site_setting_changed` 事件，但只关注 `enabled_site_setting` 对应的设置项。

---

## 7. 测试中使用 DiscourseEvent

```ruby
# spec 中验证事件触发
it "triggers event when answer is accepted" do
  events = DiscourseEvent.track(:post_destroyed) do
    post.trash!(user)
  end
  expect(events).to include(post)
end

# spec 中监听并记录
called_with = nil
DiscourseEvent.on(:user_created) { |u| called_with = u }
User.create!(...)
expect(called_with).to eq(user)
```

---

## 快速参考

```ruby
# Plugin 内（推荐）：自动 enabled? 保护
on(:event_name) { |*args| handle(args) }
on_enabled_change { |old, new| manage_lifecycle }

# 全局（慎用）：无 enabled? 保护
DiscourseEvent.on(:event_name) { |*args| ... }
DiscourseEvent.trigger(:event_name, arg1, arg2)
DiscourseEvent.off(:event_name, &block)

# 常用 hook 事件
on(:user_created)         { |user| }
on(:post_destroyed)       { |post| }
on(:topic_status_updated) { |topic, status, enabled| }
on(:site_setting_changed) { |name, old_val, new_val| }
on(:filter_auto_bump_topics) { |category, filters| filters.push(lambda) }
```
