# Patterns 05 — Plugin 后端 DSL 完整参考

> 来源：`lib/plugin/instance.rb`、`plugins/discourse-solved/plugin.rb`

---

## 概述

Plugin 的 `plugin.rb` 是入口，所有方法都在 `Plugin::Instance` 实例上调用（顶层 DSL）。

### 关键原则

```ruby
# 多数方法内部通过 reloadable_patch 注入，且自动 enabled? guard
# 以下两种调用等价：
reloadable_patch do |plugin|
  SomeClass.include(MyModule) if plugin.enabled?
end
```

---

## 1. 生命周期钩子

```ruby
# plugin 激活后运行（大多数 DSL 放这里）
after_initialize do
  # 注册 Serializer 字段、DiscourseEvent 监听、业务逻辑等
end

# 监听插件启用/禁用
on_enabled_change do |old_value, new_value|
  if new_value
    MyPlugin.setup!
  else
    MyPlugin.teardown!
  end
end
```

**注意**：`register_xxx_custom_field_type`、`enabled_site_setting` 等放在 `after_initialize` 外面（全局 patch，不依赖 DB）。

---

## 2. 插件设置

```ruby
enabled_site_setting :my_plugin_enabled
# plugin.enabled? 会读取这个 SiteSetting
# plugin.rb 内 on() 回调会自动检查 enabled?
```

---

## 3. Serializer 扩展

```ruby
# 给序列化器添加字段（plugin 禁用时自动不输出）
add_to_serializer(:post, :can_accept_answer) do
  scope.can_accept_answer?(topic, object)   # scope = guardian, object = post
end

add_to_serializer(:user_card, :accepted_answers) do
  DiscourseSolved::Queries.solved_count(object.id)
end

# 带条件显示（include_condition）
add_to_serializer(:topic_view, :solved_data,
                  include_condition: -> { SiteSetting.solved_enabled }) do
  object.topic.solved
end

# 不受 plugin enabled? 限制（respect_plugin_enabled: false）
add_to_serializer(:basic_category, :custom_field,
                  respect_plugin_enabled: false) do
  object.custom_fields["my_field"]
end
```

**常用 serializer 名**：`:post`、`:topic_view`、`:basic_topic`、`:user`、`:user_card`、`:user_summary`、`:basic_category`、`:site`、`:basic_group`

---

## 4. 类方法扩展

```ruby
# 给实例添加方法（enabled? guard 自动）
add_to_class(:composer_messages_finder, :check_topic_is_solved) do
  # self = composer_messages_finder 实例
  return if !SiteSetting.solved_enabled
  topic.solved.present?
end

# 给类添加 class method（enabled? guard 自动）
add_class_method(:topic, :solved_topics) do
  joins(:solved).where.not(discourse_solved_solved_topics: { topic_id: nil })
end

# 添加 Model 回调（after_create, before_save 等）
add_model_callback(:topic, :after_create) do
  # self = topic 实例
  return unless SiteSetting.solved_enabled
  # 初始化操作...
end
```

---

## 5. register_modifier（改变核心逻辑）

```ruby
# 修改搜索排名优先级
register_modifier(:search_rank_sort_priorities) do |priorities, search|
  if SiteSetting.prioritize_solved_topics_in_search
    priorities.push(["EXISTS (SELECT 1 FROM solved_topics WHERE topic_id = topics.id)", 1.1])
  end
  priorities
end

# 修改 user_action_stream 查询
register_modifier(:user_action_stream_builder) do |builder|
  builder.where("t.deleted_at IS NULL")
end
```

**原理**：`apply_modifier(name, initial_value, *more_args)` → 链式调用所有 enabled plugin 的 block，每次把上一个输出作为下一个输入。Plugin 禁用时自动跳过。

---

## 6. DiscourseEvent 监听

```ruby
# plugin 内用 on()，自动 enabled? guard
on(:post_destroyed) { |post| DiscourseSolved.unaccept_answer!(post) }

on(:filter_auto_bump_topics) do |_category, filters|
  filters.push(->(r) { r.where("...") })
end

on(:before_post_publish_changes) do |post_changes, topic_changes, options|
  category_id_changes = topic_changes.diff["category_id"].to_a
  # 处理分类变化...
end
```

---

## 7. 权限扩展

```ruby
# 通过 reloadable_patch + prepend 扩展 Guardian
reloadable_patch do
  ::Guardian.prepend(DiscourseSolved::GuardianExtensions)
end

# module 内：
module DiscourseSolved::GuardianExtensions
  def can_accept_answer?(topic, post)
    return false if !authenticated?
    return true if is_staff?
    topic.user_id == current_user.id
  end
end
# ensure_can_accept_answer!(topic, post) 自动生成
```

---

## 8. 路由扩展

```ruby
# 注册插件路由（plugin.rb 顶层）
Discourse::Application.routes.draw do
  scope "/discourse-solved" do
    put "answer" => "discourse_solved/answer#accept"
    delete "answer" => "discourse_solved/answer#unaccept"
  end
end
```

---

## 9. API Key Scope

```ruby
add_api_key_scope(
  :solved,
  {
    answer: {
      actions: %w[discourse_solved/answer#accept discourse_solved/answer#unaccept],
    },
  },
)
```

---

## 10. 其他常用方法

```ruby
# 预加载 Custom Fields（避免 N+1）
register_preloaded_category_custom_fields("enable_accepted_answers")
add_preloaded_topic_list_custom_field("my_field")

# 注册 SVG icon（让 replaceIcon 等可用）
register_svg_icon("square-check")

# 让 Post 接受额外的创建参数
add_permitted_post_create_param(:my_field, :string)

# 自定义 HTML 注入
register_html_builder("server:before-head-close") do |controller|
  "<meta name='my-plugin' content='true' />"
end

# 目录列（UserDirectory 页面）
add_directory_column("solutions", query: <<~SQL, icon: "check")
  SELECT count(*) FROM solved_topics WHERE solved_topics.answer_post_id = posts.id ...
SQL

# 注册自定义搜索过滤器
register_custom_filter_by_status("solved") do |scope|
  scope.joins(:solved).where.not(discourse_solved_solved_topics: { id: nil })
end
```

---

## 11. reloadable_patch（底层）

```ruby
# 直接操作 reloadable_patch（DSL 方法不够用时）
reloadable_patch do |plugin|
  ::Topic.include(MyTopicExtension) if plugin.enabled?

  # prepend（覆盖现有方法）
  ::Post.prepend(MyPostExtension)
end
```

**何时直接用**：需要 `include`/`prepend`/`extend` 模块，且没有对应的高级 DSL 方法。

---

## 快速参考表

| 需求 | 方法 |
|------|------|
| Serializer 输出字段 | `add_to_serializer(:name, :field) { }` |
| 给实例加方法 | `add_to_class(:model_name, :method) { }` |
| 给类加 class method | `add_class_method(:name, :method) { }` |
| 添加 Model 回调 | `add_model_callback(:name, :after_create) { }` |
| 修改核心逻辑 | `register_modifier(:name) { |val| val }` |
| 事件监听 | `on(:event) { |args| }` |
| Guardian 扩展 | `reloadable_patch { ::Guardian.prepend(Ext) }` |
| Custom Field 类型注册 | `register_topic_custom_field_type("k", :type)` |
| 预加载 Custom Fields | `register_preloaded_category_custom_fields("k")` |
| 搜索过滤 | `register_custom_filter_by_status("tag") { |scope| }` |
| 注册 icon | `register_svg_icon("name")` |
| 注入 HTML | `register_html_builder("hook") { |ctrl| html }` |
| API 权限 | `add_api_key_scope(:res, { action: ... })` |
