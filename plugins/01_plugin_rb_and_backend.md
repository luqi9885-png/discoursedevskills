# 插件开发规范 01 — plugin.rb 骨架与后端 API

## 概述
Discourse 插件通过 `plugin.rb` 声明元信息、注册资源，并在 `after_initialize` 块内挂载所有后端扩展。掌握 plugin.rb 的结构是插件开发的第一步。

## 来源验证

- `plugins/discourse-solved/plugin.rb` — 完整插件：模型、路由、事件、序列化扩展
- `plugins/discourse-user-notes/plugin.rb` — 中型插件：Guardian 扩展、自定义字段、Reports
- `plugins/discourse-reactions/plugin.rb` — 复杂插件：多模型、修饰器、add_to_serializer

---

## 核心规范

### 1. plugin.rb 标准骨架（顺序固定）

```ruby
# frozen_string_literal: true

# name: my-plugin                   ← 插件唯一 ID（连字符格式）
# about: 功能描述一句话
# meta_topic_id: 12345             ← meta.discourse.org 上的话题 ID
# version: 0.1
# authors: 作者名
# url: https://github.com/...

enabled_site_setting :my_plugin_enabled  # ← 与 config/settings.yml 中的 key 对应

register_asset "stylesheets/my-plugin.scss"
register_asset "stylesheets/mobile/my-plugin.scss", :mobile   # 仅移动端
register_asset "stylesheets/desktop/my-plugin.scss", :desktop  # 仅桌面端

register_svg_icon "circle-check"   # 注册插件使用的 SVG 图标

module ::MyPlugin
  PLUGIN_NAME = "my-plugin"        # ← 与 engine_name 一致
end

require_relative "lib/my_plugin/engine"

after_initialize do
  # 所有后端逻辑写在这里
end
```

**规则**：
- 注释头部的 `name:` 必须与目录名、`PLUGIN_NAME` 一致
- `enabled_site_setting` 在 `after_initialize` 外，确保设置先于业务代码加载
- `module ::MyPlugin` 定义命名空间，`::` 前缀确保在全局作用域

### 2. Engine 定义（必须有）

```ruby
# lib/my_plugin/engine.rb
module MyPlugin
  class Engine < ::Rails::Engine
    engine_name MyPlugin::PLUGIN_NAME
    isolate_namespace MyPlugin
    config.autoload_paths << File.join(config.root, "lib")
  end
end
```

Engine 负责：路由隔离、自动加载 `lib/` 下的代码。

### 3. after_initialize 内的结构（推荐顺序）

```ruby
after_initialize do
  # 1. SeedFu 数据文件路径
  SeedFu.fixture_paths << Rails.root.join("plugins", "my-plugin", "db", "fixtures").to_s

  # 2. require_relative 各文件（或 %w[] 批量加载）
  require_relative "lib/my_plugin/guardian_extensions"
  require_relative "app/models/my_plugin/foo"

  # 3. reloadable_patch：prepend/include 扩展核心类
  reloadable_patch do
    ::Guardian.prepend(MyPlugin::GuardianExtensions)
    ::Topic.prepend(MyPlugin::TopicExtension)
    ::PostSerializer.prepend(MyPlugin::PostSerializerExtension)
  end

  # 4. 路由挂载
  Discourse::Application.routes.append { mount MyPlugin::Engine, at: "/my_plugin" }

  # 5. add_to_serializer：向已有序列化器添加字段
  add_to_serializer(:post, :my_field) { object.my_method }
  add_to_serializer(:post, :my_field, include_condition: -> { scope.is_staff? }) { object.secret }

  # 6. 事件监听
  on(:post_created) { |post| MyPlugin.handle_post(post) }
  on(:post_destroyed) { |post| MyPlugin.cleanup(post) }

  # 7. 其他注册（报告、自定义字段、修饰器等）
  add_directory_column("my_stat", query: MY_SQL)
  register_modifier(:search_rank_sort_priorities) { |priorities, _| priorities }
end
```

### 4. 扩展核心类：prepend 模式

```ruby
# lib/my_plugin/guardian_extensions.rb
module MyPlugin
  module GuardianExtensions
    def can_do_thing?(target)
      return false if !authenticated?
      return true if is_staff?
      # 自定义逻辑
      target.user_id == current_user.id
    end
  end
end

# plugin.rb 中
reloadable_patch do
  ::Guardian.prepend(MyPlugin::GuardianExtensions)
end
```

**为什么用 `prepend` 而不是 `include`**：
- `prepend` 将模块方法插入 MRO 链的最前面，可以 `super` 调用原有方法
- `include` 只能在原有方法之后调用，无法覆盖

```ruby
# 扩展带 super 的示例
module MyPlugin::TopicExtension
  extend ActiveSupport::Concern

  prepended { has_one :my_record, dependent: :destroy }  # prepended 对应 included

  def my_method
    result = super   # 调用原有 Topic#my_method
    result.merge(extra: "plugin_data")
  end
end
```

### 5. add_to_serializer：最常用的扩展方式

```ruby
# 简单字段（总是包含）
add_to_serializer(:user_card, :accepted_answers) do
  MyPlugin::Queries.count(object.id)
end

# 条件字段（include_condition proc）
add_to_serializer(:post, :can_accept_answer) do
  scope.can_accept_answer?(topic, object)
end

# 带 include_condition 的写法
add_to_serializer(
  :topic_list_item,
  :my_extra_data,
  include_condition: -> { scope.is_staff? }
) { object.admin_only_data }
```

**注意**：`object` 是被序列化的 AR 对象，`scope` 是 `guardian`。

### 6. 事件系统：on / DiscourseEvent.trigger

```ruby
# 监听内置事件
on(:post_created) { |post| MyPlugin.handle_new_post(post) }
on(:user_created) { |user| MyPlugin.setup_user(user) }
on(:post_destroyed) { |post| MyPlugin.cleanup_post(post) }

# 触发自定义事件（通常在插件主模块中）
DiscourseEvent.trigger(:my_plugin_thing_happened, object)

# 其他插件监听自定义事件
on(:my_plugin_thing_happened) { |object| OtherPlugin.react(object) }
```

常见内置事件：`post_created`, `post_edited`, `post_destroyed`, `topic_created`, `user_created`, `user_approved`, `before_post_publish_changes`

### 7. register_modifier：修改内置行为

```ruby
register_modifier(:search_rank_sort_priorities) do |priorities, _search|
  if SiteSetting.my_plugin_boost_solved
    priorities.push(["EXISTS (SELECT 1 FROM ...)", 1.1])
  else
    priorities
  end
end
```

修饰器让插件在不破坏核心逻辑的前提下修改特定行为节点的输出。

### 8. add_to_class：向现有类添加方法

```ruby
# 向 Guardian 添加方法（不用 prepend，适合纯新增方法）
add_to_class(Guardian, :can_delete_user_notes?) do
  (SiteSetting.user_notes_moderators_delete? && user.staff?) || user.admin?
end

# 向任意类添加方法
add_to_class(:composer_messages_finder, :check_topic_is_solved) do
  return if !SiteSetting.solved_enabled
  { id: "solved_topic", ... }
end
```

### 9. add_model_callback：向模型添加回调

```ruby
add_model_callback(UserWarning, :after_commit, on: :create) do
  # self 是 UserWarning 实例
  user = User.find_by_id(self.user_id)
  MyPlugin.add_note(user, "Warning created")
end

add_model_callback(User, :before_destroy) do
  MyPlugin::ReactionUser.where(user_id: self.id).delete_all
end
```

### 10. SiteSetting 守卫

在每个功能入口处检查插件开关：

```ruby
on(:post_created) do |post|
  return unless SiteSetting.my_plugin_enabled   # 插件总开关
  return if post.topic.private_message?         # 业务条件
  MyPlugin.process(post)
end
```

---

## 文件目录结构（标准布局）

```
plugins/my-plugin/
├── plugin.rb                 ← 入口文件（元信息 + 注册）
├── about.json               ← 测试依赖声明（可选）
├── config/
│   ├── routes.rb            ← Engine 路由
│   ├── settings.yml         ← SiteSettings 声明
│   └── locales/
│       ├── server.en.yml    ← 后端 I18n（邮件、错误等）
│       └── client.en.yml    ← 前端 I18n（JS 使用）
├── db/
│   ├── migrate/             ← ActiveRecord 迁移文件
│   └── fixtures/            ← SeedFu 初始数据
├── app/
│   ├── controllers/my_plugin/
│   ├── models/my_plugin/
│   ├── serializers/my_plugin/
│   └── jobs/
│       ├── regular/my_plugin/
│       └── scheduled/my_plugin/
├── lib/
│   └── my_plugin/
│       ├── engine.rb         ← 必须有
│       ├── guardian_extensions.rb
│       └── topic_extension.rb
├── assets/
│   ├── stylesheets/
│   └── javascripts/
│       └── discourse/        ← 前端代码（见 JS 规范）
└── spec/                    ← 测试
```

---

## 反模式（避免这样做）

```ruby
# ❌ 在 after_initialize 外写业务逻辑
def my_helper; ...; end      # 不在 after_initialize 内，可能加载顺序错误

# ❌ 直接 monkey-patch 核心类而不用 reloadable_patch
Guardian.class_eval { def can_x?; true; end }  # 无法 reload，调试麻烦

# ✅ 使用 reloadable_patch
reloadable_patch do
  Guardian.prepend(MyPlugin::GuardianExtensions)
end

# ❌ 在 include_condition 里做数据库查询
add_to_serializer(:post, :my_field, include_condition: -> { MyModel.exists? }) { ... }
# ✅ include_condition 只做内存判断
add_to_serializer(:post, :my_field, include_condition: -> { scope.is_staff? }) { ... }
```

---

## 关联规范

- `plugins/02_plugin_frontend.md` — 前端 initializer / connector / component 规范
- `plugins/03_plugin_settings.md` — settings.yml 完整语法
- `ruby/01_service_objects.md` — 插件业务逻辑同样应用 Service::Base
- `ruby/07_jobs.md` — 插件 Job 写法与核心相同
