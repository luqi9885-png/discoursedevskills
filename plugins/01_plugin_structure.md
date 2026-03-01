# 插件结构与 plugin.rb 规范

## 概述
Discourse 插件是独立的 Rails Engine，挂载在主应用下。`plugin.rb` 是唯一入口，通过钩子函数扩展核心功能，不修改核心代码。

## 来源验证

- `plugins/discourse-solved/plugin.rb` — 典型中型插件全貌
- `plugins/discourse-user-notes/plugin.rb` — 简洁小型插件
- `plugins/discourse-reactions/plugin.rb` — 复杂大型插件
- `plugins/discourse-solved/lib/discourse_solved/engine.rb` — Engine 定义
- `plugins/discourse-solved/config/routes.rb` — 插件路由
- `plugins/discourse-solved/config/settings.yml` — 插件设置

---

## 核心规范

### 1. 插件目录结构

```
plugins/my-plugin/
├── plugin.rb                    ← 唯一入口（必须）
├── about.json                   ← 测试配置（可选）
├── package.json                 ← 前端包声明（有 JS 时必须）
├── tsconfig.json                ← TS 配置（有 TS 时）
│
├── config/
│   ├── routes.rb                ← 插件路由
│   ├── settings.yml             ← 插件 SiteSettings
│   └── locales/
│       ├── server.en.yml        ← 后端 i18n
│       └── client.en.yml        ← 前端 i18n
│
├── app/
│   ├── controllers/my_plugin/   ← 命名空间化
│   ├── models/my_plugin/
│   ├── serializers/my_plugin/
│   ├── jobs/regular/            ← Regular Jobs
│   └── jobs/scheduled/          ← Scheduled Jobs
│
├── lib/my_plugin/
│   ├── engine.rb                ← Rails Engine 定义
│   ├── guardian_extensions.rb   ← Guardian 扩展
│   ├── topic_extension.rb       ← Model 扩展（prepend 模式）
│   └── ...
│
├── assets/
│   ├── javascripts/discourse/   ← 前端代码
│   └── stylesheets/             ← CSS/SCSS
│
├── db/
│   └── migrate/                 ← 数据库迁移
│
└── spec/                        ← 测试
```

### 2. plugin.rb 文件头注释（必须）

```ruby
# frozen_string_literal: true

# name: my-plugin-name          ← 与目录名一致
# about: 一句话描述插件功能
# meta_topic_id: 12345          ← Discourse Meta 讨论帖 ID
# version: 0.1
# authors: Your Name
# url: https://github.com/...
```

### 3. plugin.rb 顶层结构

```ruby
# 1. 声明依赖的 SiteSetting 开关（关闭时插件不加载）
enabled_site_setting :my_plugin_enabled

# 2. 注册静态资源
register_asset "stylesheets/my_plugin.scss"
register_asset "stylesheets/mobile/my_plugin.scss", :mobile
register_svg_icon "circle-check"

# 3. 声明命名空间模块 + 常量
module ::MyPlugin
  PLUGIN_NAME = "my-plugin"
end

# 4. 加载 Engine
require_relative "lib/my_plugin/engine"

# 5. 所有运行时逻辑放在 after_initialize 块内
after_initialize do
  # 所有 require_relative、reloadable_patch、钩子注册都在这里
end
```

**关键**：`after_initialize` 之外只能做常量定义和资源注册，所有业务逻辑必须在 `after_initialize` 内。

### 4. Engine 定义（lib/my_plugin/engine.rb）

```ruby
# frozen_string_literal: true

module MyPlugin
  class Engine < ::Rails::Engine
    engine_name MyPlugin::PLUGIN_NAME
    isolate_namespace MyPlugin
    config.autoload_paths << File.join(config.root, "lib")
  end
end
```

### 5. 路由注册

```ruby
# config/routes.rb
MyPlugin::Engine.routes.draw do
  post "/accept" => "answer#accept"
  get  "/by_user" => "items#by_user"
end

# 挂载到主应用（在 plugin.rb after_initialize 中）
Discourse::Application.routes.append do
  mount MyPlugin::Engine, at: "/my_plugin"
end
```

### 6. 扩展核心类：prepend 模式

Discourse 插件**不能直接 monkey-patch** 核心类，必须用 `prepend`，并包裹在 `reloadable_patch` 中：

```ruby
# lib/my_plugin/topic_extension.rb
module MyPlugin::TopicExtension
  extend ActiveSupport::Concern

  prepended { has_one :my_record, class_name: "MyPlugin::MyRecord", dependent: :destroy }

  def my_custom_method
    # 覆盖或新增方法
  end
end

# plugin.rb after_initialize 中
reloadable_patch do
  ::Topic.prepend(MyPlugin::TopicExtension)
  ::Guardian.prepend(MyPlugin::GuardianExtension)
  ::PostSerializer.prepend(MyPlugin::PostSerializerExtension)
end
```

`reloadable_patch` 保证开发环境代码重载时扩展正确重新应用。

### 7. 向 Serializer 添加字段（最常用 API）

```ruby
# 向已有 Serializer 添加字段，不需要 prepend
add_to_serializer(:post, :can_accept_answer) do
  scope.can_accept_answer?(topic, object)
end

add_to_serializer(:user_card, :accepted_answers) do
  MyPlugin::Queries.solved_count(object.id)
end

# 带条件
add_to_serializer(:post, :secret_data) { object.secret }
add_to_serializer(:post, :include_secret_data?) { scope.is_staff? }
```

**命名**：`:post` 对应 `PostSerializer`，`:user_card` 对应 `UserCardSerializer`，去掉 "Serializer" 后缀。

### 8. 监听事件（on 钩子）

```ruby
on(:post_destroyed) { |post| MyPlugin.cleanup!(post) }
on(:user_created)   { |user| MyPlugin.setup_defaults!(user) }
on(:topic_created)  { |topic, params, user| }
```

常用事件：`post_created`, `post_edited`, `post_destroyed`, `topic_created`, `user_created`, `user_logged_in`。

### 9. 向类动态添加方法（add_to_class）

```ruby
# 向已有类添加方法，不需要 prepend
add_to_class(:composer_messages_finder, :check_topic_is_solved) do
  return if !SiteSetting.my_plugin_enabled
  { id: "my_message", templateName: "education" }
end

add_to_class(Guardian, :can_do_my_thing?) do
  user.staff? || user.in_group?(:my_group)
end
```

### 10. 注册修饰器（register_modifier）

修饰器用于修改核心方法的返回值，比直接 prepend 更安全：

```ruby
register_modifier(:search_rank_sort_priorities) do |priorities, _search|
  if SiteSetting.my_feature_enabled
    priorities.push(["topics.hot_score > 100", 1.1])
  end
  priorities
end
```

### 11. 添加 Report

```ruby
add_report("my_report") do |report|
  report.data = []
  report.modes = [:table]
  report.labels = [
    { type: :text, property: :name, title: "Name" }
  ]
  MyModel.where(created_at: report.start_date..report.end_date).each do |record|
    report.data << { x: record.created_at, y: record.count }
  end
  report.total = MyModel.count
end
```

### 12. SiteSettings（config/settings.yml）

```yaml
my_plugin:
  my_plugin_enabled:          ← 主开关，与 enabled_site_setting 对应
    default: true
    client: true              ← true 表示前端可访问 SiteSetting.xxx

  my_feature_trust_level:
    default: 1
    enum: "TrustLevelSetting"

  my_allowed_groups:
    default: "1|2"            ← | 分隔多个 group id
    type: group_list
    allow_any: false

  my_text_setting:
    type: string
    default: ""

  my_tag_setting:
    type: tag_list
    default: ""
```

### 13. Controller 规范

插件 Controller 继承 `::ApplicationController`，并声明 `requires_plugin`：

```ruby
class MyPlugin::ItemsController < ::ApplicationController
  requires_plugin MyPlugin::PLUGIN_NAME    # 插件禁用时自动返回 404

  def create
    guardian.ensure_can_do_my_thing!
    # ...
    render_json_dump(result_serializer)
  end
end
```

### 14. Model 规范

插件 Model 声明 `self.table_name` 并加命名空间前缀：

```ruby
module MyPlugin
  class MyRecord < ActiveRecord::Base
    self.table_name = "my_plugin_my_records"   # 表名加插件前缀，避免冲突

    belongs_to :topic
    belongs_to :user

    validates :topic_id, presence: true
  end
end
```

迁移文件中表名也要加前缀：
```ruby
create_table :my_plugin_my_records do |t|
  t.integer :topic_id, null: false
  t.timestamps
end
```

---

## 反模式（避免这样做）

```ruby
# ❌ 直接修改核心类（不用 prepend）
class Topic
  def my_method; end   # 会导致热重载失败
end

# ❌ 在 after_initialize 外写业务逻辑
Discourse::Application.routes.append { ... }  # 应在 after_initialize 内

# ❌ 表名不加插件前缀
create_table :my_records do |t| ... end  # 可能与其他插件冲突

# ❌ 直接修改 ::ApplicationController
ApplicationController.prepend(MyModule)  # 用 add_to_class 代替

# ✅ 正确做法
reloadable_patch { ::Topic.prepend(MyPlugin::TopicExtension) }
```

---

## 关联规范

- `plugins/02_plugin_api_frontend.md` — 前端 Plugin API 详解
- `ruby/01_service_objects.md` — 插件内的 Service 与主应用相同模式
- `ruby/07_jobs.md` — 插件内的 Job 与主应用相同模式
