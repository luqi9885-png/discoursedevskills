# Ruby 08 — Guardian 权限系统

> 来源：`lib/guardian.rb`、`lib/guardian/ensure_magic.rb`、`lib/guardian/topic_guardian.rb`、
> `lib/guardian/user_guardian.rb`、`lib/guardian/post_guardian.rb`
> 补写（原文件丢失）：2026-03-02

---

## 概述

Guardian 是 Discourse 的权限守卫层，所有"能不能做某件事"的判断都在这里。架构特点：

- **主类 + Mixin 组合**：`Guardian` 主类负责通用逻辑，每类资源独立一个 Module（`TopicGuardian`、`PostGuardian` 等），通过 `include` 组合进主类
- **AnonymousUser 内部类**：用户为 nil 时自动降级为 `AnonymousUser`，所有权限方法无需判断 nil
- **EnsureMagic**：`ensure_can_edit!` 自动对应 `can_edit?`，无权则 raise `Discourse::InvalidAccess`
- **分发机制**：`can_see?(obj)` / `can_edit?(obj)` 自动根据对象类型分发到具体方法

---

## 1. 主类结构

```ruby
class Guardian
  include BookmarkGuardian
  include CategoryGuardian
  include EnsureMagic        # ensure_xxx! 魔法
  include PostGuardian
  include TopicGuardian
  include UserGuardian
  # ... 其他 mixin

  def initialize(user = nil, request = nil)
    # nil 用户自动降级为 AnonymousUser，避免 nil.admin? 报错
    @user = user.presence || Guardian::AnonymousUser.new
    @request = request
  end

  def user
    @user.presence   # AnonymousUser 返回 nil（因为 blank? 为 true）
  end
  alias current_user user
end
```

---

## 2. AnonymousUser：安全默认值

```ruby
class AnonymousUser
  def blank?        = true     # 使 user.presence 返回 nil
  def admin?        = false
  def staff?        = false
  def moderator?    = false
  def anonymous?    = true
  def approved?     = false
  def silenced?     = false

  def secure_category_ids = []
  def groups              = []

  def has_trust_level?(level)      = false
  def in_any_groups?(group_ids)    = false
  def whisperer?                   = false
end
```

**意义**：Guardian 内部所有 `@user.admin?` 调用无需判断 nil，匿名用户的所有权限方法都安全返回 false / 空值。

---

## 3. 角色判断方法

```ruby
# Guardian 实例上的角色判断
guardian.anonymous?         # !authenticated?
guardian.authenticated?     # @user.present?（AnonymousUser 是 blank，返回 false）
guardian.is_admin?          # @user.admin?
guardian.is_staff?          # @user.staff?（admin 或 moderator）
guardian.is_moderator?      # @user.moderator?
guardian.is_developer?      # admin + Developer.user_ids 或 developer_emails
guardian.is_staged?         # @user.staged?
guardian.is_silenced?       # @user.silenced?

# 私有方法（在 mixin 内部使用）
is_my_own?(obj)     # obj.user_id == @user.id 或 obj.user == @user
is_me?(other)       # @user == other（User 对象比较）
is_not_me?(other)   # !is_me?(other)

# Group 权限
@user.in_any_groups?(SiteSetting.some_allowed_groups_map)
# SiteSetting 中 type: group_list 的字段自动生成 _map 方法（返回 group id 数组）
```

---

## 4. 分发机制：can_see? / can_edit? / can_delete?

```ruby
# 主类的通用分发方法
def can_see?(obj)
  see_method = method_name_for(:see, obj)  # → :can_see_topic?
  see_method && public_send(see_method, obj)
end

def can_edit?(obj)
  can_do?(:edit, obj)   # → can_edit_topic?
end

def can_delete?(obj)
  can_do?(:delete, obj) # → can_delete_topic?
end

private

def method_name_for(action, obj)
  method_name = :"can_#{action}_#{obj.class.name.underscore}?"
  # Topic → :can_see_topic?
  # Post  → :can_see_post?
  method_name if respond_to?(method_name)
end

def can_do?(action, obj)
  if obj && authenticated?
    action_method = method_name_for(action, obj)
    action_method ? public_send(action_method, obj) : true
  else
    false
  end
end
```

**规律**：只要 mixin 里定义了 `can_see_topic?`，调用 `guardian.can_see?(topic)` 就会自动路由过去。

---

## 5. EnsureMagic：自动生成 ensure! 方法

```ruby
module EnsureMagic
  def method_missing(method, *args, &block)
    if method.to_s =~ /\Aensure_(.*)\!\z/
      can_method = :"#{Regexp.last_match[1]}?"   # ensure_can_edit! → can_edit?

      if respond_to?(can_method)
        unless send(can_method, *args, &block)
          raise Discourse::InvalidAccess.new("#{can_method} failed")
        end
        return
      end
    end
    super
  end

  def ensure_can_see!(obj)
    raise Discourse::InvalidAccess.new unless can_see?(obj)
  end
end
```

```ruby
# Controller 中用法（无权则自动 raise → 返回 403）
guardian.ensure_can_accept_answer!(topic, post)  # 对应 can_accept_answer?(topic, post)
guardian.ensure_can_edit!(post)                   # 对应 can_edit?(post)
guardian.ensure_can_see!(topic)                   # 特殊实现，参数是对象
```

---

## 6. Mixin 权限方法编写规范

以 `TopicGuardian#can_edit_topic?` 为例，标准结构：

```ruby
def can_edit_topic?(topic)
  # 1. 快速 false 守卫（最常见的拒绝条件）
  return false if Discourse.static_doc_topic_ids.include?(topic.id) && !is_admin?
  return false unless can_see?(topic)

  # 2. 高权限短路（staff 等直接放行）
  return true if is_admin?
  return true if is_moderator? && can_create_post?(topic)
  return true if is_category_group_moderator?(topic.category)

  # 3. 细化条件（普通用户的复杂判断）
  return false if !can_create_topic_on_category?(topic.category)
  return false if topic.archived

  # 4. 最终条件
  is_my_own?(topic) && !topic.edit_time_limit_expired?(user) && !first_post&.locked?
end
```

**模式**：`return false if ...`（守卫） → `return true if ...`（短路） → 最终复杂条件。

---

## 7. 常见权限模式

### 7.1 Group 权限（SiteSetting group_list）

```ruby
# settings.yml 中 type: group_list 的字段自动生成 _map 方法
def can_send_private_messages?
  @user.in_any_groups?(SiteSetting.personal_message_enabled_groups_map)
end

def can_ignore_users?
  return false if anonymous?
  @user.staff? || @user.in_any_groups?(SiteSetting.ignore_allowed_groups_map)
end
```

### 7.2 Trust Level 权限

```ruby
def can_reply_as_new_topic?(topic)
  authenticated? && topic && @user.has_trust_level?(TrustLevel[1])
end

def can_mute_users?
  return false if anonymous?
  @user.staff? || @user.trust_level >= TrustLevel.levels[:basic]
end
```

### 7.3 alias 复用同一逻辑

```ruby
def can_perform_action_available_to_group_moderators?(topic)
  return false if anonymous? || topic.nil?
  return true if is_staff?
  return true if @user.has_trust_level?(TrustLevel[4])
  is_category_group_moderator?(topic.category)
end

alias can_archive_topic?    can_perform_action_available_to_group_moderators?
alias can_close_topic?      can_perform_action_available_to_group_moderators?
alias can_open_topic?       can_perform_action_available_to_group_moderators?
alias can_split_merge_topic? can_perform_action_available_to_group_moderators?
alias can_pin_unpin_topic?  can_perform_action_available_to_group_moderators?
```

### 7.4 可见性权限（read_restricted 分类）

```ruby
def can_see_topic?(topic, hide_deleted = true)
  return false unless topic
  return true if is_admin? && !SiteSetting.suppress_secured_categories_from_admin
  return false if hide_deleted && topic.deleted_at && !can_see_deleted_topics?(topic.category)

  if topic.private_message?
    return authenticated? && topic.all_allowed_users.where(id: @user.id).exists?
  end

  category = topic.category
  can_see_category?(category) &&
    (!category.read_restricted || !is_staged? || secure_category_ids.include?(category.id))
end
```

---

## 8. Controller 中的三种用法

```ruby
class PostsController < ApplicationController
  def update
    post = Post.find(params[:id])

    # 1. 布尔判断（决定显示哪些 UI）
    if guardian.can_edit?(post)
      # ...
    end

    # 2. ensure!（无权则 raise → 自动返回 403）
    guardian.ensure_can_edit!(post)

    # 3. 传 scope 给 Serializer（Serializer 用 scope.can_xxx? 判断是否包含字段）
    render_serialized(post, PostSerializer, scope: guardian)
  end
end

# Serializer 中
class PostSerializer < ApplicationSerializer
  attributes :can_edit

  def can_edit
    scope.can_edit?(object)   # scope 就是 guardian
  end
end
```

---

## 9. Plugin 中扩展 Guardian

```ruby
# lib/discourse_solved/guardian_extensions.rb
module DiscourseSolved
  module GuardianExtensions
    def can_accept_answer?(topic, post)
      return false if !authenticated?
      return false if !allow_accepted_answers?(topic.category_id, topic.tags.map(&:name))
      return true if is_staff?
      return true if current_user.in_any_groups?(SiteSetting.accept_all_solutions_allowed_groups_map)
      topic.user_id == current_user.id && !topic.closed
    end
  end
end

# plugin.rb 中：
reloadable_patch do
  ::Guardian.prepend(DiscourseSolved::GuardianExtensions)
end
# → ensure_can_accept_answer!(topic, post) 自动生成
```

---

## 快速参考

```ruby
# 常用判断
guardian.authenticated?
guardian.is_admin? / is_staff? / is_moderator?
guardian.is_my_own?(obj)     # 私有，在 mixin 内部调用
guardian.can_see?(obj)       # 自动路由到 can_see_xxx?
guardian.can_edit?(obj)      # 自动路由到 can_edit_xxx?
guardian.can_delete?(obj)    # 自动路由到 can_delete_xxx?

# Controller
guardian.ensure_can_edit!(post)              # 无权 → raise → 403
render_serialized(obj, Serializer, scope: guardian)

# Group 权限
@user.in_any_groups?(SiteSetting.xxx_groups_map)
@user.has_trust_level?(TrustLevel[2])

# 编写新权限方法（在 mixin 内）
def can_do_something?(resource)
  return false if anonymous?              # 快速守卫
  return true if is_staff?               # 高权限短路
  return true if is_my_own?(resource)    # 所有者
  resource.public? && authenticated?     # 细化条件
end
```
