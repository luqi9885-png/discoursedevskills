# Plugin 10 — 插件独立 Model、Engine 与数据库迁移

> 来源：`plugins/discourse-assign/app/models/assignment.rb`、`plugins/discourse-assign/lib/discourse_assign/engine.rb`、`plugins/discourse-assign/db/migrate/`、`plugins/discourse-reactions/lib/discourse_reactions/engine.rb`

## 概述

需要持久化独立数据的插件，通过创建专属数据库表（migrations）、ActiveRecord Model、以及 Rails Engine（命名空间隔离）来实现。Engine 将插件的 controllers/models/routes 封装在独立命名空间下，防止与 Discourse 核心或其他插件冲突。

---

## 来源验证

- `plugins/discourse-assign/app/models/assignment.rb`（完整 ActiveRecord Model，含关联、scope、验证）
- `plugins/discourse-assign/db/migrate/20210627100932_create_assignments.rb`（建表迁移）
- `plugins/discourse-assign/db/migrate/20211006223156_add_target_to_assignments.rb`（演进式迁移，含数据迁移）
- `plugins/discourse-assign/lib/discourse_assign/engine.rb`（Engine with autoload_paths）
- `plugins/discourse-reactions/lib/discourse_reactions/engine.rb`（最简 Engine）
- `plugins/discourse-assign/plugin.rb`（require_relative engine + after_initialize 中 require 各文件）

---

## 核心规范

### 1. Rails Engine 定义

每个有独立 Model/Controller 的插件都需要一个 Engine：

```ruby
# lib/my_plugin/engine.rb
module MyPlugin
  class Engine < ::Rails::Engine
    engine_name PLUGIN_NAME     # PLUGIN_NAME 在 plugin.rb 顶部定义
    isolate_namespace MyPlugin  # 隔离路由和 helpers

    # 如果 lib/ 下有需要自动加载的文件
    config.autoload_paths << File.join(config.root, "lib")
  end
end
```

**最简 Engine（无需 autoload）：**

```ruby
# lib/discourse_reactions/engine.rb
module DiscourseReactions
  class Engine < ::Rails::Engine
    engine_name PLUGIN_NAME
    isolate_namespace DiscourseReactions
  end
end
```

**plugin.rb 中挂载 Engine：**

```ruby
# plugin.rb 顶部（after_initialize 之外）
module ::MyPlugin
  PLUGIN_NAME = "my-plugin"
end

require_relative "lib/my_plugin/engine"

after_initialize do
  # 在 Rails 路由中挂载 Engine
  Discourse::Application.routes.append do
    mount MyPlugin::Engine, at: "/"
  end

  # require 各个 model/controller/service 文件
  %w[
    app/models/my_plugin/my_model.rb
    app/controllers/my_plugin/my_controller.rb
    app/serializers/my_serializer.rb
  ].each { |path| require_relative path }
end
```

---

### 2. 插件专属数据库表（迁移）

初始建表迁移：

```ruby
# db/migrate/20240101000000_create_my_plugin_items.rb
class CreateMyPluginItems < ActiveRecord::Migration[7.0]
  def change
    create_table :my_plugin_items do |t|
      t.integer  :topic_id,        null: false
      t.integer  :user_id,         null: false
      t.string   :status,          null: false, default: "pending"
      t.text     :note
      t.boolean  :active,          null: false, default: true
      t.timestamps
    end

    add_index :my_plugin_items, :topic_id
    add_index :my_plugin_items, :user_id
    add_index :my_plugin_items, [:topic_id, :user_id], unique: true
  end
end
```

**演进式迁移（新增列 + 数据回填）：**

```ruby
# db/migrate/20240201000000_add_priority_to_my_plugin_items.rb
class AddPriorityToMyPluginItems < ActiveRecord::Migration[7.0]
  def up
    add_column :my_plugin_items, :priority, :integer

    # 数据回填（存量数据）
    execute <<~SQL
      UPDATE my_plugin_items
      SET priority = 0
      WHERE priority IS NULL
    SQL

    change_column :my_plugin_items, :priority, :integer, null: false, default: 0
  end

  def down
    remove_column :my_plugin_items, :priority
  end
end
```

**关键原则：**
- 迁移文件名以时间戳开头（`YYYYMMDDHHMMSS_`），确保顺序执行
- `change` 方法只用于可自动逆转的操作（add_column、create_table）
- 有数据迁移或复杂逻辑时用 `up`/`down` 分开写
- 后续变更绝不修改历史迁移，只添加新迁移

---

### 3. 插件 ActiveRecord Model

```ruby
# app/models/my_plugin/my_item.rb
module MyPlugin
  class MyItem < ActiveRecord::Base
    # 关联（与 Discourse 核心表关联时不加 module 前缀）
    belongs_to :topic
    belongs_to :assigned_to, polymorphic: true  # 多态关联
    belongs_to :created_by_user, class_name: "User"

    # Scope
    scope :active,   -> { where(active: true) }
    scope :inactive, -> { where(active: false) }
    scope :for_user, ->(user) { active.where(assigned_to: user) }

    # 类方法
    class << self
      def valid_statuses
        SiteSetting.my_plugin_statuses.split("|")
      end

      def default_status
        valid_statuses.first
      end
    end

    # 验证
    before_validation :set_default_status
    validate :validate_status

    # 实例方法
    def assigned_to_user?
      assigned_to.is_a?(User)
    end

    def deactivate!
      update!(active: false)
      Jobs.enqueue(:my_plugin_deactivate, item_id: id)
    end

    private

    def set_default_status
      self.status ||= self.class.default_status
    end

    def validate_status
      unless self.class.valid_statuses.include?(status)
        errors.add(:status, :invalid)
      end
    end
  end
end
```

---

### 4. Engine 中的 Controller

```ruby
# app/controllers/my_plugin/items_controller.rb
module MyPlugin
  class ItemsController < ::ApplicationController
    requires_plugin PLUGIN_NAME  # 插件未启用时自动返回 404

    before_action :ensure_logged_in
    before_action :ensure_can_manage, only: [:create, :update, :destroy]

    def index
      items = MyPlugin::MyItem.active.where(topic_id: params[:topic_id])
      render json: ActiveModel::ArraySerializer.new(
        items,
        each_serializer: MyItemSerializer,
        scope: guardian
      )
    end

    def create
      item = MyPlugin::MyItem.new(
        topic_id: params[:topic_id],
        assigned_to: current_user,
        created_by_user: current_user,
        note: params[:note]
      )

      if item.save
        render json: MyItemSerializer.new(item, scope: guardian, root: false)
      else
        render_json_error(item)
      end
    end

    def destroy
      item = MyPlugin::MyItem.find(params[:id])
      item.deactivate!
      render json: success_json
    end

    private

    def ensure_can_manage
      raise Discourse::InvalidAccess unless guardian.can_manage_my_plugin?
    end
  end
end
```

**Engine 路由（config/routes.rb）：**

```ruby
# config/routes.rb（插件自己的路由文件）
MyPlugin::Engine.routes.draw do
  get    "/my_plugin/items"     => "items#index"
  post   "/my_plugin/items"     => "items#create"
  delete "/my_plugin/items/:id" => "items#destroy"
end
```

---

### 5. 预加载防止 N+1

插件数据常需要在列表中展示，必须使用 Discourse 的预加载钩子：

```ruby
# plugin.rb 内 after_initialize 中

# TopicList 预加载
TopicList.on_preload do |topics, topic_list|
  next unless SiteSetting.my_plugin_enabled?

  items = MyPlugin::MyItem.active.where(topic: topics).index_by(&:topic_id)
  topics.each do |topic|
    topic.preload_my_plugin_item(items[topic.id])
  end
end

# TopicView 预加载
TopicView.on_preload do |topic_view|
  next unless SiteSetting.my_plugin_enabled?
  topic_view.instance_variable_set(
    :@posts,
    topic_view.posts.includes(:my_plugin_item)
  )
end

# Bookmark 预加载
BookmarkQuery.on_preload do |bookmarks, _|
  next unless SiteSetting.my_plugin_enabled?
  topic_ids = bookmarks.map { |b| b.bookmarkable.try(:topic_id) }.compact.uniq
  items = MyPlugin::MyItem.active.where(topic_id: topic_ids).index_by(&:topic_id)
  # 预加载到 topic...
end
```

---

## 反模式（避免这样做）

```ruby
# 直接在 plugin.rb 顶部 require 文件（不在 after_initialize 中）
require_relative "app/models/my_plugin/my_item"  # 错误！在 after_initialize 之前执行，数据库可能未就绪

# 应在 after_initialize 中 require
after_initialize do
  require_relative "app/models/my_plugin/my_item"
end

# 修改历史迁移文件（绝对禁止）
# 20240101000000_create_my_plugin_items.rb 已运行后不能修改
# 应新建 20240201000000_add_priority_to_my_plugin_items.rb

# Model 不在 Module 命名空间内（与核心或其他插件冲突）
class MyItem < ActiveRecord::Base  # 错误！可能与其他插件冲突
# 应用：
module MyPlugin
  class MyItem < ActiveRecord::Base  # 隔离在命名空间内
  end
end
```

---

## 关联规范

- `ruby/02_models_concerns.md` — ActiveRecord Model 模式
- `ruby/06_migrations_db.md` — 迁移安全规范
- `plugins/01_plugin_rb_and_backend.md` — plugin.rb 整体结构

---

## 快速参考

```ruby
# Engine 最简结构
module MyPlugin
  class Engine < ::Rails::Engine
    engine_name PLUGIN_NAME
    isolate_namespace MyPlugin
    config.autoload_paths << File.join(config.root, "lib")  # 若有 lib/ 文件
  end
end

# plugin.rb 引导
module ::MyPlugin; PLUGIN_NAME = "my-plugin"; end
require_relative "lib/my_plugin/engine"

after_initialize do
  Discourse::Application.routes.append { mount MyPlugin::Engine, at: "/" }
  %w[app/models/my_plugin/my_item.rb].each { |p| require_relative p }
end

# 迁移：新建表
create_table :my_plugin_items do |t|
  t.integer :topic_id, null: false
  t.integer :user_id,  null: false
  t.boolean :active,   null: false, default: true
  t.timestamps
end
add_index :my_plugin_items, :topic_id

# 迁移：演进（up/down 分开）
def up
  add_column :my_plugin_items, :note, :text
  execute "UPDATE my_plugin_items SET note = '' WHERE note IS NULL"
end
def down
  remove_column :my_plugin_items, :note
end
```
