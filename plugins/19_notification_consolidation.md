# Plugin 19 — 通知合并规范（register_notification_consolidation_plan）

> 来源验证：`plugins/discourse-reactions/plugin.rb:370~474`（完整真实代码）

## 概述

`register_notification_consolidation_plan` 让插件注册通知合并策略，避免同类通知堆积。有两种策略：
- **`ConsolidateNotifications`**：阈值式，当数量超过阈值时，把多条通知合并为一条
- **`DeletePreviousNotifications`**：替换式，新通知包含前一条的用户信息（"A 和 B 都反应了"）

**注册顺序**：具体策略先注册，通用策略后注册——第一个匹配的策略生效，后续不再检查。

---

## 核心规范

### 1. DeletePreviousNotifications — 替换式（两用户信息合并）

当第 2 个用户触发同类通知时，删除前一条，创建包含两个用户信息的新通知（来自 discourse-reactions 真实代码）：

```ruby
reacted_by_two_users =
  Notifications::DeletePreviousNotifications
    .new(
      type: Notification.types[:reaction],         # 要处理的通知类型
      previous_query_blk:
        Proc.new do |notifications, data|
          notifications.where(id: data[:previous_notification_id])  # 找到要删除的前一条
        end,
    )
    .set_mutations(
      set_data_blk:
        Proc.new do |notification|
          # 查找同类型的上一条通知
          existing =
            Notification
              .where(user: notification.user)
              .order("notifications.id DESC")
              .where(
                topic_id: notification.topic_id,
                post_number: notification.post_number,
                notification_type: notification.notification_type,
              )
              .where("created_at > ?", 1.day.ago)
              .first

          data = notification.data_hash

          if existing
            same_type_data = existing.data_hash
            # 合并两个用户的信息到 data hash
            data.merge(
              previous_notification_id: existing.id,
              username2: same_type_data[:display_username],
              name2: same_type_data[:display_name],
              count: (same_type_data[:count] || 1).to_i + 1,
            )
          else
            data  # 没有前一条，返回原始 data
          end
        end,
    )
    .set_precondition(
      precondition_blk:
        Proc.new do |data, notification|
          # 只有用户设置了"总是"接收通知，且有前一条通知时才生效
          always_freq = UserOption.like_notification_frequency_type[:always]
          notification.user&.user_option&.like_notification_frequency == always_freq &&
            data[:previous_notification_id].present?
        end,
    )
```

### 2. ConsolidateNotifications — 阈值式（超过阈值合并为一条）

当同类通知超过阈值数量时，销毁所有单条通知，创建一条合并通知（来自 discourse-reactions 真实代码）：

```ruby
field_key = "display_username"

consolidated_reactions =
  Notifications::ConsolidateNotifications
    .new(
      from: Notification.types[:reaction],   # 源通知类型
      to:   Notification.types[:reaction],   # 合并后的通知类型（通常相同）
      threshold: -> { SiteSetting.notification_consolidation_threshold },  # 用 lambda 读实时配置
      consolidation_window: SiteSetting.likes_notification_consolidation_window_mins.minutes,

      # 查找未合并的通知（排除已有 username2 或 consolidated 标记的）
      unconsolidated_query_blk:
        Proc.new do |notifications, data|
          notifications
            .where("data::json ->> 'username2' IS NULL AND data::json ->> 'consolidated' IS NULL")
            .where("data::json ->> '#{field_key}' = ?", data[field_key.to_sym].to_s)
        end,

      # 查找已合并的通知（有 consolidated 标记的）
      consolidated_query_blk:
        Proc.new do |notifications, data|
          notifications
            .where("(data::json ->> 'consolidated')::bool")
            .where("data::json ->> '#{field_key}' = ?", data[field_key.to_sym].to_s)
        end,
    )
    .set_mutations(
      set_data_blk:
        Proc.new do |notification|
          data = notification.data_hash
          # 合并通知的 data：添加 consolidated 标记，统一用 display_username 显示
          data.merge(
            username: data[:display_username],
            name: data[:display_name],
            consolidated: true,
          )
        end,
    )
    .set_precondition(
      # 只在单用户通知时触发（有 username2 说明已经是两用户合并通知）
      precondition_blk: Proc.new { |data| data[:username2].blank? }
    )

# 注册前置回调：合并前/更新前修改 data
consolidated_reactions.before_consolidation_callbacks(
  before_consolidation_blk:
    Proc.new do |notifications, data|
      # 如果多条通知的 reaction_icon 不一致，则移除 icon（显示通用图标）
      new_icon = data[:reaction_icon]
      if new_icon
        icons = notifications.pluck("data::json ->> 'reaction_icon'")
        data.delete(:reaction_icon) if icons.any? { |i| i != new_icon }
      end
    end,
  before_update_blk:
    Proc.new do |consolidated, updated_data, notification|
      # 更新已有合并通知时，icon 不一致则移除
      if consolidated.data_hash[:reaction_icon] != notification.data_hash[:reaction_icon]
        updated_data.delete(:reaction_icon)
      end
    end,
)
```

### 3. 注册（顺序关键）

```ruby
# 具体策略先注册，通用策略后注册（第一个匹配就停止）
register_notification_consolidation_plan(reacted_by_two_users)   # 先：两用户替换式
register_notification_consolidation_plan(consolidated_reactions)  # 后：多用户阈值式
```

### 4. 通知 data hash 三种形态

```ruby
# 单用户通知（初始）
{
  display_username: "alice",
  display_name: "Alice",
  reaction_icon: "heart",
}

# 两用户通知（DeletePreviousNotifications 后）
{
  display_username: "bob",
  display_name: "Bob",
  reaction_icon: "heart",
  previous_notification_id: 123,
  username2: "alice",
  name2: "Alice",
  count: 2,
}

# 合并通知（ConsolidateNotifications 后）
{
  display_username: "carol",
  display_name: "Carol",
  username: "carol",      # 额外添加，供前端显示
  name: "Carol",
  consolidated: true,
  count: 5,
}
```

---

## API 速查

| 方法 | 参数 | 说明 |
|------|------|------|
| `ConsolidateNotifications.new(...)` | `from:, to:, threshold:, consolidation_window:, unconsolidated_query_blk:, consolidated_query_blk:` | 阈值式 |
| `DeletePreviousNotifications.new(...)` | `type:, previous_query_blk:` | 替换式 |
| `.set_mutations(set_data_blk:)` | Proc，参数为 `notification` | 修改 data hash |
| `.set_precondition(precondition_blk:)` | Proc，参数为 `data[, notification]` | 控制是否触发 |
| `.before_consolidation_callbacks(before_consolidation_blk:, before_update_blk:)` | 两个 Proc | 合并前/更新前钩子 |
| `register_notification_consolidation_plan(plan)` | plan 实例 | 注册（在 after_initialize 内） |

---

## 反模式

```ruby
# 错误：threshold 用固定值（无法响应管理员改设置）
threshold: 5
# 正确：用 lambda 读实时配置
threshold: -> { SiteSetting.notification_consolidation_threshold }

# 错误：注册顺序错误（通用策略先注册，先匹配就不走具体策略）
register_notification_consolidation_plan(consolidated_reactions)  # 先注册通用
register_notification_consolidation_plan(reacted_by_two_users)   # 后注册具体 → 永远不会触发！
# 正确：具体策略先，通用策略后

# 错误：unconsolidated_query_blk 没有过滤用户（把所有用户的通知都算进去）
unconsolidated_query_blk: Proc.new { |notifications, data| notifications }
# 正确：用 data 中的用户标识过滤
unconsolidated_query_blk:
  Proc.new { |notifications, data|
    notifications.where("data::json ->> 'display_username' = ?", data[:display_username].to_s)
  }
```
