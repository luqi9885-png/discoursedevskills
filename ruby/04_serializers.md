# Serializers 规范

## 概述
Discourse 使用 `ActiveModel::Serializer` 将 Ruby 对象序列化为 JSON。通过继承层级、Mixin 复用和条件属性，实现灵活的 API 响应控制。

## 来源验证

- `app/serializers/application_serializer.rb` — 基类，提供缓存能力
- `app/serializers/basic_user_serializer.rb` — 轻量用户序列化
- `app/serializers/user_serializer.rb` — 完整用户序列化（继承层级典范）
- `app/serializers/basic_category_serializer.rb` — 含 has_one 关联的典型
- `app/serializers/basic_topic_serializer.rb` + `listable_topic_serializer.rb` — 继承扩展模式
- `app/serializers/flag_serializer.rb` — 条件 / 覆盖属性方法

---

## 核心规范

### 1. 继承层级：Basic → 标准 → Detailed

Discourse 序列化器采用三层继承：

```
ApplicationSerializer          ← 基类（缓存、scope）
  └── BasicTopicSerializer     ← 最少字段，只够生成链接
        └── ListableTopicSerializer  ← 列表页所需字段
              └── TopicListItemSerializer  ← 完整列表项

  └── BasicUserSerializer      ← id + username + avatar
        └── UserCardSerializer ← 卡片所需字段
              └── UserSerializer  ← 完整个人页
```

**原则**：先写 Basic，只包含生成链接需要的最小字段集。

### 2. 标准骨架

```ruby
# frozen_string_literal: true

class FlagSerializer < ApplicationSerializer
  # 1. 声明属性（字段名 = 方法名 或 model 属性）
  attributes :id, :name, :description, :enabled, :system

  # 2. 关联（has_one / has_many）
  has_one :uploaded_logo, embed: :object, serializer: CategoryUploadSerializer

  # 3. 覆盖属性方法（当需要转换值时）
  def name
    I18n.t("#{i18n_prefix}.title", default: object.name)
  end

  # 4. 条件属性方法 include_xxx?（返回 bool）
  def include_name?
    SiteSetting.enable_names?
  end

  # 私有辅助方法
  private

  def i18n_prefix
    "#{@options[:target]}.#{object.name_key}"
  end
end
```

### 3. 条件属性：include_xxx?

当某字段需要条件显示时，定义同名 `include_xxx?` 方法：

```ruby
attributes :can_edit, :parent_category_id, :custom_fields

def include_can_edit?
  scope && scope.can_edit?(object)        # 有权限才输出
end

def include_parent_category_id?
  parent_category_id.present?             # 非空才输出
end

def include_custom_fields?
  custom_fields.present?
end
```

来源：`basic_category_serializer.rb`

### 4. 关联序列化：has_one / has_many

```ruby
# embed: :object → 内嵌完整对象（不只是 id）
has_one :invited_by, embed: :object, serializer: BasicUserSerializer
has_many :groups, embed: :object, serializer: BasicGroupSerializer

# 默认 embed: :ids → 只输出 id，前端再关联
has_many :tags, embed: :ids
```

来源：`user_serializer.rb`, `basic_category_serializer.rb`

### 5. Mixin 共享属性集

当多个序列化器需要相同字段集，提取 Mixin：

```ruby
# app/serializers/concerns/user_status_mixin.rb
module UserStatusMixin
  extend ActiveSupport::Concern

  included do
    attributes :status
  end

  def status
    object.user_status
  end

  def include_status?
    SiteSetting.enable_user_status
  end
end

# 使用
class BasicUserSerializer < ApplicationSerializer
  include UserStatusMixin
  # ...
end
```

Mixin 放在 `app/serializers/concerns/` 下，命名以 `Mixin` 结尾。

### 6. staff_attributes / private_attributes

`UserSerializer` 定义了 Discourse 扩展的属性分组 DSL：

```ruby
# 只对 staff 输出
staff_attributes :post_count, :can_be_deleted

# 只对用户本人输出
private_attributes :locale, :muted_category_ids, :watched_tags

# 不可信内容（富文本，需特别处理）
untrusted_attributes :bio_raw, :bio_cooked
```

这些是 Discourse 自定义 DSL，**不是** ActiveModel::Serializer 原生功能。来源：`user_serializer.rb`

### 7. scope 访问当前用户

`scope` 对象是当前请求的 `guardian`（权限对象），用于权限判断：

```ruby
def can_edit
  scope.can_edit?(object)
end

def include_email?
  scope.can_see_email?(object)
end
```

等同于 Controller 里的 `guardian.can_xxx?`。

### 8. 缓存片段（cache_fragment）

基类 `ApplicationSerializer` 提供 `cache_fragment` 和 `cache_anon_fragment`：

```ruby
def expensive_field
  cache_fragment("key_#{object.id}") do
    # 耗时计算
  end
end

def anon_only_field
  cache_anon_fragment("anon_#{object.id}") do
    # 只对匿名用户缓存
  end
end
```

来源：`application_serializer.rb`

---

## 文件位置规范

```
app/serializers/
├── application_serializer.rb      ← 基类，禁止业务逻辑
├── basic_xxx_serializer.rb        ← 最小字段集
├── xxx_serializer.rb              ← 完整序列化
├── detailed_xxx_serializer.rb     ← 超详细（如管理页）
└── concerns/
    └── xxx_mixin.rb               ← 共享属性集，命名用 Mixin 后缀
```

---

## 反模式（避免这样做）

```ruby
# ❌ 在序列化器里写查询逻辑
def comments_count
  Comment.where(post_id: object.id).count   # N+1 问题！
end

# ✅ 在 Controller 里预加载，序列化器只读取
# controller: @post = Post.includes(:comments).find(id)
def comments_count
  object.comments.size   # 使用已加载的关联
end

# ❌ 在序列化器里写权限判断的业务规则
def include_secret_field?
  object.user_id == scope.user.id && object.created_at > 1.day.ago  # 逻辑太重

# ✅ 将判断委托给 guardian
def include_secret_field?
  scope.can_see_secret?(object)
end
```

---

## 关联规范

- `ruby/03_controllers.md` — Controller 如何 render serializer
- `ruby/01_service_objects.md` — Service 的输出对象被 Serializer 序列化
- `ruby/02_models_concerns.md` — Model Concern 与 Serializer Mixin 的对比
