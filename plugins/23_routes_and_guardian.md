# Plugin 23 — 插件路由注册与 Guardian 权限扩展

> 来源：discourse-assign、discourse-solved、discourse-ai、discourse-calendar 等主流插件的典型模式

## 概述

- **路由注册**：后端 Rails routes + 前端 Ember routes，两端都要注册
- **Guardian 权限扩展**：给核心权限系统追加插件专属 `can_xxx?` 方法

---

## Part 1 — 路由注册

### 1. 后端 Rails 路由

```ruby
# 方式一：plugin.rb 中直接注册（少量路由）
Discourse::Application.routes.draw do
  scope "/my-plugin", defaults: { format: :json } do
    get    "topics/:topic_id/items"     => "my_plugin/items#index"
    post   "topics/:topic_id/items"     => "my_plugin/items#create"
    delete "topics/:topic_id/items/:id" => "my_plugin/items#destroy"
    get    "u/:username/items"          => "my_plugin/user_items#index"
  end
end

# 方式二：Engine routes（大型插件，结构清晰）
# config/routes.rb
MyPlugin::Engine.routes.draw do
  resources :items, only: %i[index show create update destroy] do
    collection { get :search }
    member     { put :activate }
  end
  get "status" => "status#show"
end

# plugin.rb 挂载 Engine
Discourse::Application.routes.draw do
  mount MyPlugin::Engine, at: "/my-plugin"
end
```

**URL 对应关系（Engine 方式）：**

```
GET    /my-plugin/items           => MyPlugin::ItemsController#index
POST   /my-plugin/items           => MyPlugin::ItemsController#create
GET    /my-plugin/items/:id       => MyPlugin::ItemsController#show
PUT    /my-plugin/items/:id       => MyPlugin::ItemsController#update
DELETE /my-plugin/items/:id       => MyPlugin::ItemsController#destroy
GET    /my-plugin/items/search    => MyPlugin::ItemsController#search
PUT    /my-plugin/items/:id/activate => MyPlugin::ItemsController#activate
```

### 2. 控制器文件位置

```
后端 Controller：
  app/controllers/my_plugin/items_controller.rb
  class MyPlugin::ItemsController < ::ApplicationController

前端 Route：
  assets/javascripts/discourse/routes/my-plugin-items.js（列表，复数）
  assets/javascripts/discourse/routes/my-plugin-item.js（详情，单数）

前端 Template/Component：
  assets/javascripts/discourse/templates/my-plugin-items.hbs
  assets/javascripts/discourse/components/my-plugin-item.gjs
```

### 3. 前端 Ember 路由

```javascript
// assets/javascripts/discourse/routes/my-plugin-items.js
import DiscourseRoute from "discourse/routes/discourse";

export default class MyPluginItemsRoute extends DiscourseRoute {
  model() {
    return this.store.findAll("my-plugin-item");
  }
}

// assets/javascripts/discourse/routes/my-plugin-item.js
import DiscourseRoute from "discourse/routes/discourse";

export default class MyPluginItemRoute extends DiscourseRoute {
  model(params) {
    return this.store.find("my-plugin-item", params.id);
  }
}
```

---

## Part 2 — Guardian 权限扩展

### 4. GuardianExtensions module

```ruby
# lib/my_plugin/guardian_extensions.rb
module MyPlugin
  module GuardianExtensions
    # 标准三段式：SiteSetting → authenticated → 业务逻辑
    def can_create_my_plugin_item?(topic)
      return false unless SiteSetting.my_plugin_enabled?  # 1. 插件开关
      return false unless authenticated?                   # 2. 登录检查
      return false if topic.archived? || topic.closed?    # 3. 业务逻辑（轻量，不查 DB）
      can_see?(topic)                                      # 4. 复用核心权限
    end

    def can_delete_my_plugin_item?(item)
      return false unless SiteSetting.my_plugin_enabled?
      return false unless authenticated?
      # 自己的内容，或管理员/版主
      item.user_id == current_user&.id || is_staff?
    end

    def can_see_my_plugin_items?(topic)
      SiteSetting.my_plugin_enabled? && can_see?(topic)
    end

    def can_manage_my_plugin?
      is_admin?
    end
  end
end

# plugin.rb
after_initialize do
  require_relative "lib/my_plugin/guardian_extensions"
  Guardian.include(MyPlugin::GuardianExtensions)
end
```

### 5. ensure_can_xxx! 自动派生

`can_xxx?` → `ensure_can_xxx!` 由 Guardian 的 `method_missing` **自动派生**，失败时抛 `Discourse::InvalidAccess`，**不需要手动定义**：

```ruby
# Controller 中使用
def create
  guardian.ensure_can_create_my_plugin_item!(@topic)  # 无权限自动 raise
  # 以下只在有权限时执行
  MyPlugin::Item.create!(user: current_user, topic: @topic)
end

def destroy
  guardian.ensure_can_delete_my_plugin_item!(@item)
  @item.destroy!
  render json: success_json
end
```

### 6. 完整 Controller 示例

```ruby
# app/controllers/my_plugin/items_controller.rb
module MyPlugin
  class ItemsController < ::ApplicationController
    requires_plugin "my-plugin"

    before_action :ensure_logged_in, only: %i[create destroy]
    before_action :find_topic,       only: %i[index create]
    before_action :find_item,        only: %i[destroy]

    def index
      guardian.ensure_can_see_my_plugin_items!(@topic)
      items = MyPlugin::Item.where(topic_id: @topic.id).order(created_at: :desc)
      render_serialized(items, MyPlugin::ItemSerializer, root: "items")
    end

    def create
      guardian.ensure_can_create_my_plugin_item!(@topic)
      item = MyPlugin::Item.create!(user: current_user, topic: @topic, data: params[:data])
      render_serialized(item, MyPlugin::ItemSerializer)
    end

    def destroy
      guardian.ensure_can_delete_my_plugin_item!(@item)
      @item.destroy!
      render json: success_json
    end

    private

    def find_topic
      @topic = Topic.find(params[:topic_id])
      raise Discourse::NotFound unless @topic
    end

    def find_item
      @item = MyPlugin::Item.find(params[:id])
      raise Discourse::NotFound unless @item
    end
  end
end
```

### 7. Serializer 中暴露权限给前端

```ruby
after_initialize do
  # 把权限状态序列化给前端，前端据此显示/隐藏按钮
  add_to_serializer(:topic_view, :my_plugin_actions) do
    {
      can_create: scope.can_create_my_plugin_item?(object.topic),
      can_manage: scope.can_manage_my_plugin?,
    }
  end

  add_to_serializer(:topic_view, :include_my_plugin_actions?) do
    SiteSetting.my_plugin_enabled?
  end
end
```

---

## 反模式

```ruby
# 错误：Guardian 方法中查数据库（权限检查应轻量）
def can_create_my_plugin_item?(topic)
  MyPlugin::UserQuota.where(user_id: current_user.id).sum(:used) < 100  # 每次权限检查都查 DB！
end
# 正确：只做状态检查，配额检查放在 Controller 业务逻辑中

# 错误：忘记 SiteSetting 守卫
def can_create_my_plugin_item?(topic)
  authenticated?  # 插件禁用时仍返回 true！
end

# 错误：用 if 替代 ensure_!（逻辑错误时不会抛异常）
def create
  if guardian.can_create_my_plugin_item?(@topic)
    # ...
  end
  # 没权限时不报错，静默返回 nil
end
# 正确：用 ensure_! 让无权限时自动 raise InvalidAccess
def create
  guardian.ensure_can_create_my_plugin_item!(@topic)
end

# 错误：在 after_initialize 外 include Guardian
Guardian.include(MyPlugin::GuardianExtensions)  # 可能在 Guardian 加载前执行
# 正确
after_initialize { Guardian.include(MyPlugin::GuardianExtensions) }
```

---

## 快速参考

```ruby
# Guardian 扩展三段式
def can_do_thing?(obj)
  return false unless SiteSetting.my_plugin_enabled?
  return false unless authenticated?
  !obj.archived?  # 业务逻辑（轻量）
end

after_initialize { Guardian.include(MyPlugin::GuardianExtensions) }

# Controller 使用
guardian.ensure_can_do_thing!(@obj)  # 自动 raise InvalidAccess
guardian.can_do_thing?(@obj)         # 返回布尔值

# Serializer 暴露权限
add_to_serializer(:topic_view, :can_do_thing) { scope.can_do_thing?(object.topic) }
add_to_serializer(:topic_view, :include_can_do_thing?) { SiteSetting.my_plugin_enabled? }
```
