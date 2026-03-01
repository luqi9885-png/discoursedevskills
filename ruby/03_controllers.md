# Controllers 规范

## 概述
Discourse Controller 遵循 Rails 约定，叠加一套访问控制、Guardian 授权和统一响应格式的规范。

## 来源验证

- `app/controllers/application_controller.rb` — 基类，定义全局行为
- `app/controllers/topics_controller.rb` — 最复杂 Controller 的典范
- `app/controllers/categories_controller.rb` — 标准 CRUD Controller

---

## 核心规范

### 1. Controller 基础结构

```ruby
# frozen_string_literal: true

class CategoriesController < ApplicationController
  include TopicQueryParams   # include 领域相关 module

  # 登录控制（白名单：requires_login + except，或黑名单）
  requires_login except: %i[index show find_by_slug]

  # before_action 精准绑定（用 only:）
  before_action :fetch_category, only: %i[show update destroy]
  before_action :initialize_staff_action_logger, only: %i[create update destroy]

  # 跳过全局 before_action（谨慎使用）
  skip_before_action :check_xhr, only: %i[index redirect]

  def index
    # ...
  end
end
```

### 2. 两种登录控制模式

```ruby
# 模式 A: requires_login + except（大多数 action 需登录，少数公开）
requires_login except: %i[index show]

# 模式 B: requires_login + only（大多数公开，少数需登录）
requires_login only: %i[
  timings update destroy status invite
  move_posts merge_topic
]
```

来源：`topics_controller.rb` 用 `only:`，`categories_controller.rb` 用 `except:`

### 3. Guardian 授权（必须）

每个写操作都必须调用 guardian 授权，**不能跳过**：

```ruby
def create
  guardian.ensure_can_create!(Category)   # 抛出 Discourse::InvalidAccess
  # ...
end

def update
  guardian.ensure_can_edit!(@category)   # 抛出 Discourse::InvalidAccess
  # ...
end

def destroy
  guardian.ensure_can_delete!(@category)  # 抛出 Discourse::InvalidAccess
  # ...
end

def show
  guardian.ensure_can_see!(@category)     # 抛出 Discourse::InvalidAccess
  # ...
end
```

常用 guardian 方法：
- `ensure_can_see!(obj)` — 可见性
- `ensure_can_create!(klass)` — 创建权限
- `ensure_can_edit!(obj)` — 编辑权限
- `ensure_can_delete!(obj)` — 删除权限
- `can_see?(obj)` — 返回布尔值（不抛异常）

### 4. 响应格式规范

```ruby
# JSON 响应：用 render_serialized（推荐）
render_serialized(@category, CategorySerializer)

# 成功响应（无数据）
render json: success_json

# 失败响应
render json: { errors: [e.message] }, status: :unprocessable_entity

# HTML + JSON 双格式
respond_to do |format|
  format.html { render }
  format.json { render_serialized(@category_list, CategoryListSerializer) }
end
```

### 5. 参数处理

```ruby
# 必须的参数用 require（缺失时 400）
params.require("category_id")
params.require(:mapping)

# 强参数用 permit（防止批量赋值攻击）
category_params = params.require(:category).permit(
  :name, :color, :text_color, :slug, :description
)
```

### 6. 错误处理

```ruby
# 资源不存在
raise Discourse::NotFound unless topic

# 权限不足
raise Discourse::InvalidAccess

# 参数非法
raise Discourse::InvalidParameters.new("message")
```

这些异常由 `ApplicationController` 的全局 `rescue_from` 统一处理，转为标准 JSON 错误响应。

### 7. before_action 提取公共逻辑

```ruby
before_action :fetch_category, only: %i[show update destroy]

private

def fetch_category
  @category = Category.find_by(slug: params[:id]) || Category.find_by(id: params[:id].to_i)
  raise Discourse::NotFound if @category.nil?
end

def initialize_staff_action_logger
  @staff_action_logger = StaffActionLogger.new(current_user)
end
```

## 反模式（避免这样做）

```ruby
# ❌ 直接在 action 里做授权判断
def update
  if current_user.admin?
    # ...
  end
end

# ✅ 用 guardian
def update
  guardian.ensure_can_edit!(@category)
  # ...
end

# ❌ render json: @category.to_json（直接序列化，会泄露敏感字段）
render json: @category.to_json

# ✅ 用 Serializer 控制输出字段
render_serialized(@category, CategorySerializer)

# ❌ 用宽泛的 before_action（没有 only: 限制）
before_action :fetch_category

# ✅ 精准绑定
before_action :fetch_category, only: %i[show update destroy]
```

## 关联规范

- `ruby/04_serializers.md` — render_serialized 对应的 Serializer 写法
- `ruby/01_service_objects.md` — 复杂业务逻辑移至 Service
