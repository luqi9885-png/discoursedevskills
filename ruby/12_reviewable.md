# Ruby 12 — Reviewable 审核系统

> 来源：`app/models/reviewable.rb`、`app/models/reviewable_flagged_post.rb`、
> `app/models/reviewable_queued_post.rb`、`lib/reviewable/`

---

## 概述

Reviewable 是 Discourse 的统一审核/flag 框架。代替以前的 PostAction/SpamReport，所有需要 Moderator 审核的内容（被举报的帖子、排队等待的帖子、待批准的用户）都抽象为 Reviewable 记录。

---

## 来源验证

- `app/models/reviewable.rb`（973 行，基类）
- `app/models/reviewable_flagged_post.rb`（被举报帖子）
- `app/models/reviewable_queued_post.rb`（排队帖子）
- `app/models/reviewable_user.rb`（待批准用户）
- `lib/reviewable/` 目录（Actions / PerformResult 等）

---

## 核心规范

### 1. 内置 Reviewable 类型

| 类 | 触发场景 |
|----|---------|
| `ReviewableFlaggedPost` | 被 Flag 的帖子，score 累积超过阈值 |
| `ReviewableQueuedPost` | 新用户帖子需要排队等审核 |
| `ReviewableUser` | 站点需要人工批准注册 |
| `ReviewablePost` | Plugin 使用的通用帖子审核 |

### 2. 状态机

```ruby
enum :status, { pending: 0, approved: 1, rejected: 2, ignored: 3, deleted: 4 }

# pending  → 等待审核（默认）
# approved → 通过
# rejected → 拒绝
# ignored  → 忽略（不处理）
# deleted  → 目标已删除
```

### 3. 创建 Reviewable（needs_review!）

```ruby
# ❌ 不要用 Reviewable.create — 用 needs_review!
# ✅ needs_review! 会处理重复 target 的情况（upsert 回 pending 状态）

reviewable = ReviewableFlaggedPost.needs_review!(
  target: post,
  topic: post.topic,
  created_by: flagger,
  reviewable_by_moderator: false,  # true = 只有 moderator 可处理
  potential_spam: true,
)

# Plugin 创建自定义审核
reviewable = ReviewablePost.needs_review!(
  target: post,
  topic: post.topic,
  created_by: Discourse.system_user,
  payload: { reason: "custom_reason" },
)
```

**needs_review! 逻辑**：
- 如果 (type, target_id) 还没有 reviewable → 直接 `save!`
- 如果已存在 → SQL `UPDATE status = pending`（原子操作，并发安全）
- 如果从 reviewed 状态恢复 → `recalculate_score` + `log_history(:transitioned)`

### 4. Score（评分）系统

Score 决定审核优先级和自动动作触发：

```ruby
# 添加 score（每次用户举报时调用）
reviewable.add_score(
  user,                          # 举报用户
  reviewable_score_type,         # PostActionType ID（spam/inappropriate 等）
  reason: :manual_flag,
  take_action: false,            # true 时 +5 分 bonus
  force_review: false,
)

# score 计算
sub_total = user_trust_level_bonus + type_bonus + take_action_bonus
# user_accuracy_bonus = 该用户历史举报准确率加成

# 关键阈值（SiteSetting）
Reviewable.score_required_to_hide_post      # 帖子被隐藏
Reviewable.score_to_auto_close_topic        # 话题自动关闭
Reviewable.spam_score_to_silence_new_user   # 新用户被静默
```

### 5. 执行审核动作（perform）

```ruby
# Guardian 检查后执行动作
result = reviewable.perform(
  current_user,
  :approve_post,          # 动作 ID（对应 perform_approve_post 方法）
  version: params[:version],
  # 可传额外 params 给 perform_ 方法
)

if result.success?
  # result.created_post   → 若动作创建了帖子
  # result.remove_reviewable → 是否移除这个 reviewable
else
  render_json_error(result)
end
```

Reviewable 子类必须实现 `build_actions` 和 `perform_{action_id}` 方法：

```ruby
class ReviewableFlaggedPost < Reviewable
  def build_actions(actions, guardian, args)
    return [] unless pending?

    # 普通动作
    actions.add(:agree_and_keep)   { |a| a.label = "agree_and_keep" }
    actions.add(:disagree)         { |a| a.label = "disagree" }

    # Bundle（分组）
    agree = actions.add_bundle("agree")
    actions.add(:agree_and_delete, bundle: agree)
    actions.add(:agree_and_silence, bundle: agree)
  end

  def perform_agree_and_keep(performed_by, args)
    agree(performed_by, args) do
      # 什么都不做，保留帖子
    end
  end

  def perform_disagree(performed_by, args)
    disagree(performed_by, args)
    ignore(performed_by, args)
  end

  private

  def agree(performed_by, args, &block)
    yield block
    create_result(:success, :approved)
  end
end
```

### 6. PerformResult

```ruby
# 在 perform_ 方法中创建结果
def perform_approve_post(performed_by, args)
  post = target
  post.recover! if post.deleted?

  # 成功结果
  create_result(:success, :approved) do |result|
    result.created_post = post  # 可附加额外数据
  end
end

# create_result(status, transition_to)
# status: :success | :invalid_parameters | :error
# transition_to: :approved / :rejected / :ignored / :deleted (nil = 不改状态)
```

### 7. Plugin 创建自定义 Reviewable

```ruby
# plugin.rb
after_initialize do
  # 注册自定义 Reviewable 类型
  DiscoursePluginRegistry.register_reviewable_type(
    MyPlugin::ReviewableCustom,
    plugin: self
  )
end

# app/models/my_plugin/reviewable_custom.rb
class MyPlugin::ReviewableCustom < Reviewable
  def build_actions(actions, guardian, args)
    return unless pending?
    actions.add(:approve) { |a| a.label = "approve" }
    actions.add(:reject)  { |a| a.label = "reject" }
  end

  def perform_approve(performed_by, args)
    # 业务逻辑
    target.approve!
    create_result(:success, :approved)
  end

  def perform_reject(performed_by, args)
    target.destroy!
    create_result(:success, :rejected)
  end

  def build_editable_fields(fields, guardian, args)
    # 可在审核界面让 mod 编辑字段
    fields.add(:category_id, :category)
  end
end
```

### 8. 自动化与事件

```ruby
# after_commit 触发 Job（创建或更新时）
after_commit(on: %i[create update]) do
  Jobs.enqueue(:notify_reviewable, reviewable_id: self.id) if pending?
end

# DiscourseEvent
after_commit(on: :create) { DiscourseEvent.trigger(:reviewable_created, self) }
DiscourseEvent.trigger(:reviewable_score_updated, self)  # add_score 后触发

# Plugin 可监听
on(:reviewable_created) { |reviewable| do_something(reviewable) }
```

### 9. 自定义过滤器（Plugin 扩展搜索）

```ruby
# plugin.rb
after_initialize do
  Reviewable.add_custom_filter(
    ->(results, filter_params) do
      if filter_params[:only_solved]
        results.joins(:topic).where.not(topics: { solved: nil })
      else
        results
      end
    end
  )
end
```

### 10. 查询 Reviewable

```ruby
# 获取待处理列表（带权限过滤）
Reviewable.viewable_by(guardian.user, order: :score)
          .where(status: :pending)
          .includes(:target, :created_by)

# 按类型过滤
Reviewable.where(type: "ReviewableFlaggedPost").pending

# 判断帖子是否有待处理审核
ReviewableFlaggedPost.pending.find_by(target: post)

# Queueable（排队审核）
ReviewableQueuedPost.needs_review!(
  target: post,
  topic: post.topic,
  created_by: Discourse.system_user,
)
```

---

## 反模式（避免这样做）

```ruby
# ❌ 直接 create 而非 needs_review!
Reviewable.create!(type: "ReviewableFlaggedPost", target: post, ...)

# ❌ 直接改 status 字段
reviewable.update!(status: :approved)

# ✅ 通过 perform 方法走状态机
reviewable.perform(current_user, :approve, version: reviewable.version)
```

---

## 关联规范

- `ruby/08_guardian.md` — `can_review?` / `can_perform_action?` 权限检查
- `ruby/07_jobs.md` — `NotifyReviewable` Job
- `patterns/03_discourse_event.md` — `reviewable_created` 事件

---

## 快速参考

```ruby
# 创建审核（幂等）
ReviewableFlaggedPost.needs_review!(target: post, topic: topic, created_by: user)

# 加 score
reviewable.add_score(flagger, PostActionType.types[:spam], reason: :flagged)

# 执行动作
result = reviewable.perform(mod_user, :agree_and_keep, version: version)
result.success?   # => true
result.transition_to  # => :approved

# 阈值检查
Reviewable.score_required_to_hide_post  # 根据 SiteSetting 动态计算
Reviewable.min_score_for_priority(:high)

# Plugin 注册类型
DiscoursePluginRegistry.register_reviewable_type(MyReviewable, plugin: self)
```
