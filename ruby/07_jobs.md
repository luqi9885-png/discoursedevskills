# Background Jobs 规范

## 概述
Discourse 使用 Sidekiq 处理后台任务，封装在 `Jobs` 命名空间下。分为三类：Regular（触发执行）、Scheduled（定时执行）、Onceoff（仅执行一次）。

## 来源验证

- `app/jobs/base.rb` — Jobs::Base 基类，`execute` 方法约定，`Jobs.enqueue` API
- `app/jobs/regular/post_alert.rb` — 最简 Regular Job
- `app/jobs/regular/close_topic.rb` — 含业务逻辑的 Regular Job
- `app/jobs/regular/push_notification.rb` — 含 HTTP 调用的 Regular Job
- `app/jobs/scheduled/heartbeat.rb` — 最简 Scheduled Job（`every N.hours`）
- `app/jobs/scheduled/badge_grant.rb` — Scheduled Job 触发 Regular Job
- `app/jobs/scheduled/enqueue_digest_emails.rb` — 含复杂查询的 Scheduled Job

---

## 核心规范

### 1. 三类 Job 及其基类

```ruby
# Regular Job：由业务代码手动触发
class Jobs::PostAlert < ::Jobs::Base
  def execute(args)
    # ...
  end
end

# Scheduled Job：由 MiniScheduler 按频率自动触发
class Jobs::Heartbeat < ::Jobs::Scheduled
  every 24.hours   # 调度频率声明

  def execute(args)
    # ...
  end
end

# Onceoff Job：服务器启动或 deploy 后执行一次
# 继承 ::Jobs::Onceoff
```

### 2. Regular Job 标准骨架

```ruby
# frozen_string_literal: true

module Jobs
  class PostAlert < ::Jobs::Base
    def execute(args)
      # args 是 Hash（with_indifferent_access），键名用 symbol 或 string 均可
      post = Post.find_by(id: args[:post_id])
      return unless post&.topic && post.raw.present?

      PostAlerter.new.after_save_post(post, true)
    end
  end
end
```

**约定**：
- 必须实现 `execute(args)` 方法
- 对象找不到时直接 `return`（不要 raise）
- `args[:key]` 和 `args["key"]` 等价

### 3. 触发 Job：Jobs.enqueue

```ruby
# 立即入队（异步）
Jobs.enqueue(:post_alert, post_id: post.id)

# 延迟 N 秒后执行
Jobs.enqueue_in(5.minutes, :user_email, type: "digest", user_id: user.id)

# 在指定时间执行
Jobs.enqueue_at(1.hour.from_now, :close_topic, topic_id: topic.id)
```

**命名规则**：`:post_alert` 对应 `Jobs::PostAlert`（自动 camelcase 转换）。

**参数约定**：只传可序列化为 JSON 的类型：string、number、boolean、nil、Array、Hash。不要传 Ruby 对象或 ActiveRecord 实例。

### 4. Scheduled Job 频率声明

```ruby
class Jobs::BadgeGrant < ::Jobs::Scheduled
  every 1.day          # 每天一次

  def execute(args)
    # ...
  end
end

class Jobs::EnqueueDigestEmails < ::Jobs::Scheduled
  every 30.minutes

  def execute(args)
    # ...
  end
end
```

`every` 使用 ActiveSupport 时间辅助方法，常见值：`1.minute`、`30.minutes`、`1.hour`、`1.day`。

### 5. Scheduled Job 触发 Regular Job（扇出模式）

当定时任务需要对多条数据执行操作时，Scheduled Job 只负责查询和分发，将每条数据的处理委托给 Regular Job：

```ruby
# ✅ 正确：Scheduled 只负责分发
class Jobs::BadgeGrant < ::Jobs::Scheduled
  every 1.day

  def execute(args)
    Badge.enabled.pluck(:id).each do |badge_id|
      Jobs.enqueue(:backfill_badge, badge_id: badge_id)   # 每个 badge 单独 Job
    end
  end
end

# ❌ 错误：Scheduled Job 直接处理所有数据（超时风险）
class Jobs::BadgeGrant < ::Jobs::Scheduled
  every 1.day

  def execute(args)
    Badge.enabled.each { |badge| badge.recalculate! }    # 可能执行数小时
  end
end
```

### 6. 文件目录结构

```
app/jobs/
├── base.rb             ← Jobs::Base / Jobs::Scheduled 定义
├── regular/            ← 手动触发的 Job（动词_名词.rb）
│   ├── post_alert.rb
│   └── close_topic.rb
├── scheduled/          ← 定时 Job（动词_名词.rb 或 名词.rb）
│   ├── heartbeat.rb
│   └── badge_grant.rb
└── onceoff/            ← 单次执行 Job
```

### 7. 幂等性设计

Job 可能因网络错误而重试，必须保证重复执行安全：

```ruby
def execute(args)
  topic = Topic.find_by(id: args[:topic_id])
  return if topic.nil?
  return if topic.closed?          # 已关闭则幂等返回，不报错

  topic.update_status("closed", true, user)
end
```

### 8. 条件守卫（SiteSetting 检查）

Scheduled Job 通常在第一行检查开关设置：

```ruby
def execute(args)
  return unless SiteSetting.enable_badges          # Feature flag 守卫
  return if SiteSetting.disable_emails == "yes"   # 全局开关守卫
  # ...
end
```

---

## 反模式（避免这样做）

```ruby
# ❌ 传 ActiveRecord 对象作为参数
Jobs.enqueue(:process_post, post: post)   # post 无法被 JSON 序列化

# ✅ 只传 id
Jobs.enqueue(:process_post, post_id: post.id)

# ❌ 在 Job 里 raise 已知错误（会导致 Sidekiq 无限重试）
def execute(args)
  post = Post.find(args[:post_id])   # 找不到时 raise ActiveRecord::RecordNotFound
end

# ✅ 用 find_by 并 return
def execute(args)
  post = Post.find_by(id: args[:post_id])
  return unless post
end

# ❌ Scheduled Job 直接处理大量数据
class Jobs::ProcessAllPosts < ::Jobs::Scheduled
  every 1.hour

  def execute(args)
    Post.all.each { |p| p.rebake! }   # 可能跑几小时，阻塞其他 Job
  end
end
```

---

## 关联规范

- `ruby/01_service_objects.md` — 复杂业务逻辑委托给 Service，Job 只是触发点
- `ruby/05_rspec_testing.md` — Job 测试用 `Jobs::Base.jobs.last` / `expect_enqueued_with`
