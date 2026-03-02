# Ruby 09 — Notification 系统

> 来源：`app/models/notification.rb`、`plugins/discourse-solved/plugin.rb`（Notification.create!）、
> `lib/plugin/instance.rb`（registerNotificationTypeRenderer 前端侧）

---

## 概述

Notification 是发给用户的站内消息（铃铛图标）。Plugin 主要使用 `Notification.types[:custom]` 创建自定义通知，同时在前端注册渲染器控制显示样式。

---

## 1. 数据结构

```
notifications 表：
  notification_type  integer  (Notification.types 枚举)
  user_id            integer
  topic_id           integer  (可选，点击通知跳转到该话题)
  post_number        integer  (可选，跳转到该楼)
  data               string   (JSON，最大 1000 字符，存显示所需数据)
  read               boolean  (默认 false)
  high_priority      boolean  (true 时会在菜单顶部显示)
```

---

## 2. 内置 Notification Types

```ruby
Notification.types
# 插件常用的：
# :custom           (14) — Plugin 自定义通知
# :mentioned        (1)  — @提及
# :replied          (2)  — 回复
# :private_message  (6)  — 私信
# :granted_badge    (12) — 授予徽章
# :watching_first_post (17) — 监看分类的首帖
```

**Plugin 几乎只用 `:custom`**，通过 `data.message` 指向 i18n key 定制显示文本。

---

## 3. 创建通知（Notification.create!）

### 3.1 标准模式

```ruby
Notification.create!(
  notification_type: Notification.types[:custom],
  user_id: recipient.id,
  topic_id: post.topic_id,
  post_number: post.post_number,
  data: {
    message: "solved.accepted_notification",  # i18n key（前端显示文本）
    display_username: acting_user.username,   # 显示在通知中的操作者
    topic_title: topic.title,                 # 话题标题
    title: "solved.notification.title",       # 可选：通知标题 i18n key
  }.to_json,
)
```

### 3.2 data 字段约定（JSON，max 1000 字符）

| 键 | 用途 |
|----|------|
| `message` | i18n key，前端渲染通知文本（必填） |
| `display_username` | 通知中显示的操作者用户名 |
| `topic_title` | 话题标题（显示在通知文本中） |
| `title` | 通知标题 i18n key（可选） |
| `original_username` | 备选操作者用户名字段 |
| `badge_id` / `badge_name` | 徽章通知用 |

Discourse 会用 `data_hash[:display_username]` 等字段自动填充 acting_user 信息。

### 3.3 安全检查（避免重复/无意义通知）

```ruby
# discourse-solved 的完整模式
def self.accept_answer!(post, acting_user, topic: nil)
  topic ||= post.topic

  # 1. 通知帖子作者（不通知自己接受自己的答案）
  if acting_user.id != post.user_id && notify_solved?(recipient: post.user, acting_user:)
    Notification.create!(
      notification_type: Notification.types[:custom],
      user_id: post.user_id,
      topic_id: post.topic_id,
      post_number: post.post_number,
      data: {
        message: "solved.accepted_notification",
        display_username: acting_user.username,
        topic_title: topic.title,
        title: "solved.notification.title",
      }.to_json,
    )
  end

  # 2. 通知话题作者（条件不同：staff accept + 配置开启 + 不通知自己）
  if SiteSetting.notify_on_staff_accept_solved &&
       acting_user.id != topic.user_id &&
       notify_solved?(recipient: topic.user, acting_user:)
    Notification.create!(
      notification_type: Notification.types[:custom],
      user_id: topic.user_id,
      topic_id: post.topic_id,
      post_number: post.post_number,
      data: { ... }.to_json,
    )
  end
end

def self.notify_solved?(recipient:, acting_user:)
  return false if recipient.blank?
  return false if acting_user.blank?
  # 不通知自己
  return false if recipient.id == acting_user.id
  # 不通知忽略了对方的用户
  return false if recipient.ignored_user_ids.include?(acting_user.id)
  true
end
```

### 3.4 consolidate_or_create!（合并通知，防垃圾）

```ruby
# 对于可能大量产生的通知（如 liked），用合并创建避免刷屏
Notification.consolidate_or_create!(
  notification_type: Notification.types[:liked],
  user_id: user.id,
  data: { ... }.to_json,
)
```

底层调用 `Notifications::ConsolidationPlanner`，根据已注册的合并规则决定是否合并现有通知。

---

## 4. High Priority 通知

```ruby
# high_priority 通知会置顶显示，且有 badge 标记
Notification.create!(
  notification_type: Notification.types[:private_message],
  user_id: user.id,
  high_priority: true,   # 明确设置；private_message 等类型会自动设为 true
  data: { ... }.to_json,
)

# 内置 high_priority 类型（自动设为 true）：
Notification.high_priority_types
# => [Notification.types[:private_message], Notification.types[:bookmark_reminder]]
```

---

## 5. 前端：注册自定义通知渲染器

```js
// initializers/extend-for-solved.gjs
import { withPluginApi } from "discourse/lib/plugin-api";

withPluginApi((api) => {
  // 替换铃铛图标
  api.replaceIcon(
    "notification.solved.accepted_notification",  // notification.{message key}
    "square-check"                                // SVG icon name
  );

  // 如需完全自定义渲染（高级）：
  // api.registerNotificationTypeRenderer("custom", (NotificationItemBase) =>
  //   class extends NotificationItemBase { ... }
  // );
});
```

**图标替换规则**：`notification.{data.message}` → SVG icon name。

---

## 6. i18n 配置

```yaml
# config/locales/client.en.yml（plugin 内）
en:
  js:
    notification_types:
      solved:
        accepted_notification: "accepted your post as the solution to <a href='%{topicUrl}'>%{topicTitle}</a>"

    solved:
      notification:
        title: "Solution accepted"
```

```yaml
# config/locales/server.en.yml（邮件模板等）
en:
  solved:
    accepted_notification: "Your post was accepted as the solution"
```

---

## 7. 查询通知（后端）

```ruby
# 用户未读通知数
user.unread_notifications           # 普通未读数
user.unread_high_priority_notifications  # 高优先级未读数

# 查询特定类型
Notification.where(
  user_id: user.id,
  notification_type: Notification.types[:custom],
  read: false,
)

# 用户最近通知（含话题预加载）
Notification.for_user_menu(user.id, limit: 20)

# 标记已读
Notification.mark_posts_read(user, topic_id, [post_number])
Notification.read(user, [notification_id1, notification_id2])
```

---

## 8. 触发 MessageBus 推送（自动）

`Notification.create!` 后，`after_commit :refresh_notification_count` 会自动调用：

```ruby
user.publish_notifications_state
# → MessageBus.publish("/notification/#{user.id}", { unread_notifications: n, ... })
```

前端订阅了 `/notification/{user_id}` 频道，收到后自动更新铃铛数字。Plugin **不需要**手动 publish。

---

## 快速参考

```ruby
# 创建自定义通知（Plugin 标准模式）
Notification.create!(
  notification_type: Notification.types[:custom],
  user_id: recipient.id,
  topic_id: topic.id,
  post_number: post.post_number,
  data: {
    message: "my_plugin.notification_type",    # i18n key
    display_username: actor.username,
    topic_title: topic.title,
  }.to_json,
)

# 安全检查
return if recipient.id == actor.id                        # 不通知自己
return if recipient.ignored_user_ids.include?(actor.id)   # 不通知已忽略的

# 合并通知（防刷屏）
Notification.consolidate_or_create!(notification_type: ..., user_id: ..., data: ...)

# 前端图标
api.replaceIcon("notification.my_plugin.notification_type", "icon-name")
```
