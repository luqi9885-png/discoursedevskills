# Ruby Service Objects — Service::Base 模式

## 概述
Discourse 使用 `Service::Base` 将业务逻辑从 Controller 中提取出来，
以声明式 DSL 描述流程步骤，自动处理依赖注入、验证、权限和事务。

## 来源验证（真实代码，多处比对）
- `app/services/flags/create_flag.rb`
- `app/services/flags/update_flag.rb`
- `app/services/flags/destroy_flag.rb`
- `app/services/flags/toggle_flag.rb`
- `app/services/user/silence.rb`

---

## 核心规范

### 1. 基本结构

```ruby
# frozen_string_literal: true

class Flags::CreateFlag          # 命名空间 + 动词+名词
  include Service::Base          # 引入 DSL

  policy :invalid_access         # 权限检查（最早失败）

  params do                      # 入参声明 + 验证
    attribute :name, :string
    validates :name, presence: true
  end

  model :flag, :instantiate_flag # 加载/实例化模型（方法名可选）

  transaction do                 # 事务包裹写操作
    step :create
    step :log
  end

  private

  def invalid_access(guardian:)  # 方法参数通过关键字注入
    guardian.can_create_flag?
  end

  def instantiate_flag(params:)  # model 的自定义获取方法
    Flag.new(params.merge(notify_type: true))
  end

  def create(flag:)
    flag.save!
  end

  def log(guardian:, flag:)
    StaffActionLogger.new(guardian.user).log_custom("create_flag", { ... })
  end
end
```

### 2. DSL 步骤类型

| DSL | 用途 | 失败行为 |
|-----|------|---------|
| `policy :name` | 权限 / 前置条件检查，返回 bool | 返回 false 则中止 |
| `params do...end` | 输入参数声明与验证 | validates 失败则中止 |
| `model :name` | 加载 ActiveRecord 对象 | 返回 nil 则报 404 |
| `step :name` | 普通执行步骤 | 一般不中止流程 |
| `transaction do...end` | 包裹写操作事务 | 内部异常触发回滚 |

### 3. 方法命名约定

- `model :flag` → 框架自动调用 `fetch_flag(params:)` 方法
- `model :flag, :instantiate_flag` → 使用自定义方法名 `instantiate_flag`
- `policy :invalid_access` → 调用同名方法 `invalid_access(...)`
- `step :create` → 调用同名方法 `create(...)`

### 4. 依赖注入

所有 private 方法通过**关键字参数**接收依赖，由框架自动注入：

```ruby
# 可用的注入对象（按声明顺序积累）
def my_step(guardian:, params:, flag:, users:)
  # guardian: 当前用户权限对象
  # params:   经验证的参数对象
  # flag:     model 步骤加载的对象（名与 model :xxx 一致）
end
```

### 5. policy vs step 的区别

```ruby
# policy：返回 bool，false 则中止整个 service
policy :not_system   # → def not_system(flag:) = !flag.system?

# step：执行操作，不检查返回值（除非 step 内部 raise）
step :destroy        # → def destroy(flag:) = flag.destroy!
```

### 6. 可选 model

```ruby
model :post, optional: true    # 找不到时不中止，post 为 nil
```

见：`app/services/user/silence.rb` 第 35 行

### 7. policy 复用外部 Policy 类

```ruby
policy :not_silenced_already, class_name: User::Policy::NotAlreadySilenced
```

当 policy 逻辑复杂或跨 service 复用时，提取到独立 Policy 类。

### 8. 写入上下文

```ruby
def silence(guardian:, users:, params:)
  context[:full_reason] = User::Action::SilenceAll.call(...)
  # context 是 service 内部共享的 hash，后续步骤可读取
end
```

---

## 文件位置规范

```
app/services/
├── flags/
│   ├── create_flag.rb     # 命名：动词_名词.rb
│   └── destroy_flag.rb
├── user/
│   ├── silence.rb
│   └── policy/            # 独立 Policy 类放 policy/ 子目录
└── base.rb                # Service::Base 定义
```

---

## 反模式（避免这样做）

```ruby
# ❌ 在 Controller 里直接写业务逻辑
def create
  flag = Flag.new(params)
  if guardian.can_create_flag?
    flag.save!
    StaffActionLogger.new(current_user).log_custom(...)
  end
end

# ✅ 应该：Controller 只调用 Service
def create
  Flags::CreateFlag.call(params: flag_params, guardian: guardian)
end
```

---

## 关联规范
- `ruby/03_controllers.md` — Controller 如何调用 Service
- `ruby/05_rspec_testing.md` — Service 的测试方式
