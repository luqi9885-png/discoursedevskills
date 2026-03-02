# Ruby 10 — RateLimiter 限流模式

> 来源：`lib/rate_limiter.rb`、`lib/rate_limiter/on_create_record.rb`、
> `app/controllers/session_controller.rb`、
> `plugins/discourse-solved/app/controllers/discourse_solved/answer_controller.rb`

---

## 概述

RateLimiter 是基于 Redis 的滑动窗口限流器，用于防止接口被滥用。架构：

- 底层：Redis List + Lua 脚本原子操作，线程安全
- Staff 用户默认豁免（可配置）
- 测试环境和 profile 模式自动禁用
- 两种使用方式：Controller 手动调用 / Model 层 `rate_limit` DSL

---

## 1. 构造参数

```ruby
RateLimiter.new(
  user,          # User 对象（匿名传 nil，key 包含 user.id）
  type,          # 字符串 key（如 "accept-hr-123"，需唯一）
  max,           # 窗口内最大次数
  secs,          # 窗口大小（秒）
  global: false,           # true = 跨站点共享（不加命名空间前缀）
  aggressive: false,       # true = 超限后仍记录请求（更严格）
  error_code: nil,         # 自定义错误码（会在异常中传出）
  apply_limit_to_staff: false,  # true = staff 也受限
  staff_limit: { max: nil, secs: nil }  # staff 专属宽松限制
)
```

---

## 2. 核心方法

```ruby
limiter = RateLimiter.new(user, "my-action", 10, 1.hour)

# 执行：在限额内返回 true，超限 raise RateLimiter::LimitExceeded
limiter.performed!

# 执行：超限时不抛异常，返回 false
limiter.performed!(raise_error: false)

# 判断是否在限额内（不消耗次数）
limiter.can_perform?

# 剩余次数
limiter.remaining

# 等待秒数（超限时需等多久）
limiter.seconds_to_wait

# 回滚：撤销最后一次 performed!（事务回滚时用）
limiter.rollback!

# 清除所有记录（测试 / 管理用）
limiter.clear!
```

---

## 3. Controller 中手动限流

### 3.1 最简模式

```ruby
def accept
  RateLimiter.new(current_user, "accept-hr-#{current_user.id}", 20, 1.hour).performed!
  RateLimiter.new(current_user, "accept-min-#{current_user.id}", 4, 30.seconds).performed!
  # ...业务逻辑...
rescue RateLimiter::LimitExceeded
  render_json_error(I18n.t("rate_limiter.slow_down"), status: 429)
end
```

### 3.2 Staff 豁免模式（Plugin 常用）

```ruby
def limit_accepts
  return if current_user.staff?   # staff 豁免，直接返回

  RateLimiter.new(nil, "accept-hr-#{current_user.id}", 20, 1.hour).performed!
  RateLimiter.new(nil, "accept-min-#{current_user.id}", 4, 30.seconds).performed!
end

def accept
  limit_accepts   # 在 before 或 action 开头调用
  # ...
end
```

注意：第一个参数传 `nil` 而非 `current_user`，因为 key 已包含 user.id，同时确保 key 格式一致。

### 3.3 基于 IP 限流（匿名用户）

```ruby
# session_controller.rb 中的密码重置限流
RateLimiter.new(nil, "forgot-password-hr-#{request.remote_ip}", 6, 1.hour).performed!
RateLimiter.new(nil, "forgot-password-min-#{request.remote_ip}", 3, 1.minute).performed!
```

**规律**：用 `request.remote_ip` 作为 key 的一部分，覆盖匿名场景。

### 3.4 多层限流（双窗口保护）

```ruby
# 同时设置小窗口（防爆发）和大窗口（防持续滥用）
RateLimiter.new(nil, "action-min-#{user.id}", 3, 1.minute).performed!   # 1分钟最多3次
RateLimiter.new(nil, "action-hr-#{user.id}", 20, 1.hour).performed!     # 1小时最多20次
```

两个限流器都 pass 才继续，任一超限即触发异常。

---

## 4. Model 层：rate_limit DSL

对创建记录的限流，通过 `RateLimiter::OnCreateRecord` mixin + `rate_limit` DSL 自动接入：

```ruby
# app/models/topic.rb
include RateLimiter::OnCreateRecord

rate_limit :default_rate_limiter          # 使用默认限流器（按 SiteSetting）
rate_limit :limit_topics_per_day          # 使用自定义限流方法
rate_limit :limit_private_messages_per_day

# 自定义限流方法
def limit_topics_per_day
  return unless regular?
  apply_per_day_rate_limit_for("topics", :max_topics_per_day)
end
```

**default_rate_limiter 自动逻辑**：

```ruby
def default_rate_limiter
  limit_key = "create_#{self.class.name.underscore}"  # → "create_topic"

  max_setting =
    if user&.new_user? && SiteSetting.has_setting?("rate_limit_new_user_#{limit_key}")
      SiteSetting.get("rate_limit_new_user_#{limit_key}")  # 新用户专属限制（更严）
    else
      SiteSetting.get("rate_limit_#{limit_key}")           # 普通限制
    end

  RateLimiter.new(user, limit_key, 1, max_setting)
end
```

**自动钩子**（`rate_limit` DSL 注入）：

```ruby
after_create  → limiter.performed!    # 创建时扣一次
after_destroy → limiter.rollback!     # 删除时回滚
after_rollback → limiter.rollback!   # 事务回滚时恢复
```

---

## 5. 底层原理（Redis Lua 脚本）

```lua
-- PERFORM_LUA（非 aggressive 模式）
local key = KEYS[1]
local now = ARGV[1]
local secs = ARGV[2]
local max = ARGV[3]

-- 列表长度 < max，OR 最老的记录已超过时间窗口 → 允许
if (LLEN(key) < max) or (now - LRANGE(key, -1, -1)[1] >= secs) then
  LPUSH(key, now)     -- 记录本次时间戳
  LTRIM(key, 0, max-1) -- 只保留最近 max 条
  EXPIRE(key, secs*2)  -- TTL = 窗口×2，自动清理
  return 1             -- 允许
else
  return 0             -- 拒绝
end
```

本质是**滑动窗口**：维护最近 N 次请求的时间戳列表，判断最老的请求是否已过期。

---

## 6. Key 设计规范

```ruby
# 好的 key 设计（唯一且有意义）
"accept-hr-#{current_user.id}"          # 用户维度 + 操作 + 时间粒度
"forgot-password-hr-#{request.remote_ip}" # IP 维度（匿名）
"email-hr-#{request.remote_ip}"          # 组合：操作 + 时间 + 维度

# 避免
"action"                    # 太泛，所有用户共享同一个 key（通常不是想要的）
"action-#{user.id}"         # 缺少时间粒度标识，同一 action 不同窗口 key 会冲突
```

**约定**：`{操作}-{时间粒度}-{用户id或ip}`，如 `accept-hr-123`（每小时）、`accept-min-123`（每分钟）。

---

## 7. 全局限流

```ruby
# global: true → key 不加 namespace 前缀，跨多个 Discourse 站点共享
RateLimiter.new(user, "global-action", 100, 1.hour, global: true).performed!

# 清除所有全局限流记录（管理命令）
RateLimiter.clear_all_global!
```

---

## 快速参考

```ruby
# 最常见：Controller 手动限流
def my_action
  return if current_user.staff?
  RateLimiter.new(nil, "my-action-hr-#{current_user.id}", 10, 1.hour).performed!
  RateLimiter.new(nil, "my-action-min-#{current_user.id}", 2, 30.seconds).performed!
  # ... 业务 ...
rescue RateLimiter::LimitExceeded
  render_json_error(I18n.t("rate_limiter.slow_down"), status: 429)
end

# IP 限流（匿名）
RateLimiter.new(nil, "action-#{request.remote_ip}", 5, 1.hour).performed!

# Model 层（SiteSetting 驱动）
include RateLimiter::OnCreateRecord
rate_limit   # 自动读 SiteSetting.rate_limit_create_{model}

# 检查剩余而不消耗
limiter = RateLimiter.new(user, "key", 5, 1.hour)
limiter.can_perform?    # bool
limiter.remaining       # 剩余次数
limiter.seconds_to_wait # 需等待秒数
```
