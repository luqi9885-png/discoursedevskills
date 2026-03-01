# 插件开发架构

## 概述
Discourse 插件通过 `plugin.rb` 入口文件，利用一套声明式钩子 API 向核心系统注入功能，无需修改核心代码。

## 来源验证

- `plugins/discourse-solved/plugin.rb` — 完整插件典范（374 行，覆盖大多数 API）
- `plugins/discourse-reactions/plugin.rb` — 带 Service 层的插件
- `plugins/discourse-solved/lib/discourse_solved/engine.rb` — Rails Engine 声明
- `plugins/discourse-solved/config/routes.rb` — 插件路由
- `plugins/discourse-solved/app/controllers/discourse_solved/answer_controller.rb`
- `plugins/discourse-solved/app/models/discourse_solved/solved_topic.rb`
- `lib/plugin/instance.rb` — 全部钩子 API 定义

---

## 核心规范

### 1. 插件目录结构（标准）

```
plugins/my-plugin/
├── plugin.rb                    ← 入口文件（必须）
├── about.json                   ← 元数据（测试依赖声明）
├── config/
│   ├── routes.rb                ← 路由（有自定义 API 时必须）
│   ├── settings.yml             ← SiteSettings
│   └── locales/
│       ├── client.en.yml        ← 前端 i18n
│       └── server.en.yml        ← 后端 i18n
├── app/
│   ├── controllers/my_plugin/
│   ├── models/my_plugin/
│   ├── serializers/
│   ├── services/my_plugin/
│   └── jobs/regular/my_plugin/
├── lib/
│   └── my_plugin/
│       ├── engine.rb
│       └── guardian_extensions.rb
├── db/migrate/
├── assets/
│   ├── javascripts/discourse/
│   └── stylesheets/
└── spec/
```

### 2. plugin.rb 文件结构（固定顺序）

```ruby
# frozen_string_literal: true

# name: my-plugin
# about: 一句话描述
# meta_topic_id: 12345
# version: 0.1
# authors: Your Name
# url: https://github.com/...

# 1. 插件开关 SiteSetting（必须在顶层声明）
enabled_site_setting :my_plugin_enabled

# 2. 静态资源注册（顶层）
register_svg_icon "icon-name"
register_asset "stylesheets/my-plugin.scss"

# 3. 命名空间模块（顶层定义）
module ::MyPlugin
  PLUGIN_NAME = "my-plugin"
end

# 4. 加载 Engine（有路由时必须，顶层）
require_relative "lib/my_plugin/engine"

# 5. 初始化钩子（所有运行时代码放这里）
after_initialize do
  # require 具体实现文件
  require_relative "lib/my_plugin/guardian_extensions"

  # 扩展核心类
  reloadable_patch do
    ::Guardian.prepend(MyPlugin::GuardianExtensions)
    ::Post.prepend(MyPlugin::PostExtension)
  end

  # 监听事件
  on(:post_created) { |post, opts| MyPlugin::DoSomething.call(post: post) }

  # 向序列化器添加字段
  add_to_serializer(:post, :my_field) { object.my_field }

  # 向类添加方法
  add_to_class(:topic, :my_method) { custom_fields["my_key"] }
end
```

**关键**：所有访问 DB、Rails 类、SiteSettings 的代码必须在 `after_initialize` 块内，否则 Rails 未完全启动会报错。

### 3. Rails Engine（有路由时必须）

```ruby
# lib/my_plugin/engine.rb
module MyPlugin
  class Engine < ::Rails::Engine
    engine_name PLUGIN_NAME
    isolate_namespace MyPlugin
    config.autoload_paths << File.join(config.root, "lib")
  end
end
```

```ruby
# config/routes.rb
MyPlugin::Engine.routes.draw do
  post "/accept" => "answer#accept"
  get  "/list"   => "items#index"
end

Discourse::Application.routes.draw do
  mount MyPlugin::Engine, at: "my-plugin"
end
```

访问路径：`POST /my-plugin/accept`，`GET /my-plugin/list`

### 4. 插件 Controller

```ruby
# app/controllers/my_plugin/items_controller.rb
class MyPlugin::ItemsController < ::ApplicationController
  requires_plugin MyPlugin::PLUGIN_NAME   # 插件禁用时自动 404，必须！

  def index
    guardian.ensure_can_see_items!
    items = MyPlugin::Item.where(user: current_user)
    render_json_dump(items, serializer: MyPlugin::ItemSerializer)
  end

  def create
    MyPlugin::CreateItem.call(
      guardian: guardian,
      params: params.permit(:name, :value)
    )
    render json: success_json
  end
end
```

### 5. 插件 Model（独立表）

```ruby
# app/models/my_plugin/item.rb
module MyPlugin
  class Item < ActiveRecord::Base
    self.table_name = "my_plugin_items"   # 表名必须加插件前缀

    belongs_to :topic
    belongs_to :user

    validates :topic_id, presence: true
    validates :user_id, presence: true
  end
end

# db/migrate/20240101000000_create_my_plugin_items.rb
class CreateMyPluginItems < ActiveRecord::Migration[7.2]
  def change
    create_table :my_plugin_items do |t|
      t.integer :topic_id, null: false
      t.integer :user_id, null: false
      t.string  :name, null: false
      t.timestamps
    end
    add_index :my_plugin_items, :topic_id
    add_index :my_plugin_items, :user_id
  end
end
```

### 6. 核心钩子 API 速查

#### reloadable_patch + prepend — 扩展核心类

```ruby
reloadable_patch do
  ::Guardian.prepend(MyPlugin::GuardianExtensions)   # 权限扩展
  ::Post.prepend(MyPlugin::PostExtension)             # 模型方法扩展
  ::TopicViewSerializer.prepend(MyPlugin::TopicViewSerializerExtension)
end
```

用 `prepend` 而非 `include`：prepend 让方法插入继承链最前面，可以调用 `super` 访问原方法。

#### add_to_serializer — 向序列化器添加字段

```ruby
# 最简：无条件添加
add_to_serializer(:post, :my_flag) { object.my_flag }

# 有条件：include_condition 为 false 时字段不出现在 JSON
add_to_serializer(:post, :can_accept, include_condition: -> { scope.can_accept?(object) }) do
  true
end

# 添加到 user_card 序列化器
add_to_serializer(:user_card, :solved_count) { MyPlugin.solved_count(object.id) }

# 不受插件开关控制（全局常量类型字段）
add_to_serializer(:site, :my_setting, respect_plugin_enabled: false) do
  SiteSetting.my_setting
end
```

#### on — 监听 DiscourseEvent

```ruby
on(:post_created)   { |post, opts, user| }
on(:post_edited)    { |post, topic_changed| }
on(:post_destroyed) { |post| }
on(:topic_created)  { |topic, opts, user| }
on(:user_created)   { |user| }
on(:user_logged_in) { |user| }
on(:before_create_post) { |post, opts| }
```

插件禁用时 `on` 自动不执行（`Plugin::Instance#on` 内部有 `if enabled?` 守卫）。

#### add_to_class — 向现有类添加实例方法

```ruby
add_to_class(:topic, :my_solved?) do
  custom_fields["solved"] == true
end

add_to_class(:user, :my_plugin_stats) do
  MyPlugin::UserStat.for_user(id)
end
```

#### register_modifier — 拦截核心行为（返回修改后的值）

```ruby
# 影响搜索排名
register_modifier(:search_rank_sort_priorities) do |priorities, search|
  if SiteSetting.my_feature_enabled
    priorities.push(["EXISTS (SELECT 1 FROM my_plugin_items WHERE topic_id = topics.id)", 1.2])
  end
  priorities
end

# 过滤查询 builder
register_modifier(:user_action_stream_builder) do |builder|
  builder.where("my_table.active = true")
end
```

#### add_model_callback — 向模型添加回调

```ruby
add_model_callback(:post, :after_save) do
  MyPlugin.index_post(self) if saved_change_to_raw?
end
```

### 7. Guardian 扩展

```ruby
# lib/my_plugin/guardian_extensions.rb
module MyPlugin::GuardianExtensions
  # 新增权限方法
  def can_do_my_thing?(resource)
    return false unless authenticated?
    return true if is_staff?
    resource.user_id == current_user.id
  end

  # 强制断言版本（失败 raise InvalidAccess）
  def ensure_can_do_my_thing!(resource)
    raise Discourse::InvalidAccess unless can_do_my_thing?(resource)
  end

  # 覆盖核心权限（用 super 访问原逻辑）
  def can_see_post?(post)
    super && MyPlugin.post_visible_for?(post, current_user)
  end
end
```

### 8. SiteSettings 声明（settings.yml）

```yaml
my_plugin:
  my_plugin_enabled:        # 与 enabled_site_setting 对应
    default: true
    client: true            # true = 前端 Ember 也能访问

  my_feature_groups:
    type: group_list
    default: "1|2"          # 1=admins, 2=moderators
    allow_any: false

  my_count:
    default: 10
    min: 1
    max: 1000
    client: false

  my_enum:
    type: enum
    default: "option_a"
    choices:
      - "option_a"
      - "option_b"
```

后端访问：`SiteSetting.my_plugin_enabled`、`SiteSetting.my_count`

### 9. 插件间事件通信

```ruby
# 触发自定义事件（让其他插件可以监听）
DiscourseEvent.trigger(:my_plugin_item_created, item, user)

# 监听（在自己或其他插件中）
on(:my_plugin_item_created) do |item, user|
  SomeOtherPlugin.handle_new_item(item)
end
```

---

## 命名空间规范

```
# 模块名 = 驼峰化插件名
插件目录: discourse-solved     → 模块: DiscourseSolved
插件目录: my-plugin            → 模块: MyPlugin

# 文件路径对应类名
lib/my_plugin/guardian_extensions.rb
→ MyPlugin::GuardianExtensions

app/controllers/my_plugin/items_controller.rb
→ MyPlugin::ItemsController

app/models/my_plugin/item.rb
→ MyPlugin::Item

# DB 表名 = 插件名_资源名（snake_case）
MyPlugin::Item → table: my_plugin_items
DiscourseSolved::SolvedTopic → table: discourse_solved_solved_topics
```

---

## 反模式（避免这样做）

```ruby
# ❌ 不用 reloadable_patch，直接 monkey-patch（热重载后失效）
Guardian.class_eval { def can_do? = true }

# ✅ 使用 reloadable_patch + prepend

# ❌ after_initialize 外访问 Rails 类（启动时序问题）
Topic.where(...)       # 顶层执行，可能报错

# ✅ 所有运行时逻辑放 after_initialize 内

# ❌ 忘记 requires_plugin（插件关闭时接口仍暴露）
class MyPlugin::ItemsController < ::ApplicationController
  def index; end
end

# ✅ 必须加 requires_plugin
class MyPlugin::ItemsController < ::ApplicationController
  requires_plugin MyPlugin::PLUGIN_NAME
end

# ❌ DB 表不加前缀（与核心或其他插件冲突风险）
create_table :items do ...

# ✅ 加插件名前缀
create_table :my_plugin_items do ...

# ❌ 在 prepend 模块中直接 include（方法顺序错误）
module GuardianExtensions
  def can_see?(target)
    super || my_check(target)   # 这里 super 调用顺序正确
  end
end
::Guardian.include(GuardianExtensions)   # include 会放到继承链末尾

# ✅ 使用 prepend（放到继承链最前）
::Guardian.prepend(GuardianExtensions)
```

---

## 关联规范

- `ruby/01_service_objects.md` — 插件 Service 完全遵循 Service::Base 模式
- `ruby/03_controllers.md` — 插件 Controller 相同约定（guardian、render_json_dump）
- `ruby/06_migrations_db.md` — 插件迁移规范同核心（表名加前缀是唯一区别）
- `ruby/07_jobs.md` — 插件 Job 放 app/jobs/regular|scheduled/my_plugin/ 下
- `javascript/02_plugin_api.md` — 前端 withPluginApi 钩子（待整理）
