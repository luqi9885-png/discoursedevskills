# Plugin 11 — register_modifier 后端规范

> 来源：`plugins/discourse-assign/plugin.rb`、`plugins/discourse-reactions/plugin.rb`、`lib/discourse_plugin_registry.rb`、`lib/plugin/instance.rb`

## 概述

`register_modifier` 是后端插件修改 Discourse 核心行为的管道机制。核心代码在关键点调用 `DiscoursePluginRegistry.apply_modifier(:modifier_name, value, *extra_args)`，插件注册的 block 按顺序接收并可修改该值，最终返回修改后的结果。与 `prepend` 不同，modifier 无需继承即可修改多个插件的协同行为。

---

## 来源验证

- `plugins/discourse-assign/plugin.rb`（`register_modifier(:topics_filter_options)`）
- `plugins/discourse-reactions/plugin.rb`（`register_modifier(:user_action_stream_builder)`、`register_modifier(:post_action_users_list)`）
- `lib/discourse_plugin_registry.rb`（`register_modifier`、`apply_modifier` 实现，289-310 行）
- `lib/plugin/instance.rb`（`register_modifier` 委托，229-230 行）

---

## 核心规范

### 1. 基本语法

```ruby
# plugin.rb 内 after_initialize 中
register_modifier(:modifier_name) do |value, *extra_args|
  # 修改并返回 value
  # extra_args 是 apply_modifier 调用时额外传入的参数
  value
end
```

`apply_modifier` 的执行逻辑：

```ruby
# 核心代码（lib/discourse_plugin_registry.rb）
def self.apply_modifier(name, arg, *more_args)
  registered_modifiers = @modifiers[name]
  return arg if !registered_modifiers

  registered_modifiers.each do |plugin_instance, block|
    arg = block.call(arg, *more_args) if plugin_instance.enabled?
  end
  arg
end
```

关键：**block 的返回值成为下一个 modifier 的输入**（管道模式）。若插件被禁用，其 block 自动跳过。

---

### 2. 修改 Query Builder（builder 类型）

`register_modifier(:user_action_stream_builder)` 接收一个 query builder 对象，通过链式调用 `left_join`、`where` 等方法修改查询：

```ruby
# reactions 插件：过滤掉"既 Like 又 React"的重复记录
register_modifier(:user_action_stream_builder) do |builder|
  builder.left_join(<<~SQL)
    discourse_reactions_reaction_users
    ON discourse_reactions_reaction_users.post_id = a.target_post_id
    AND discourse_reactions_reaction_users.user_id = a.acting_user_id
  SQL

  builder.where("discourse_reactions_reaction_users.id IS NULL")

  # builder 类型：直接修改对象，不需要显式返回（返回 builder 或 nil 均可）
  builder
end
```

---

### 3. 修改 ActiveRecord Relation（query 类型）

`register_modifier(:post_action_users_list)` 接收 AR relation，通过 `.where` 添加过滤条件：

```ruby
# reactions 插件：在用户头像列表中过滤掉既 Like 又 React 的用户
register_modifier(:post_action_users_list) do |query, post|
  # 第一个参数：AR relation（可继续链式调用）
  # 第二个参数：apply_modifier 传入的 extra_arg（此处为 post 对象）

  where_clause = <<~SQL
    post_actions.id NOT IN (
      SELECT post_actions.id
      FROM post_actions
      INNER JOIN discourse_reactions_reaction_users
        ON discourse_reactions_reaction_users.post_id = post_actions.post_id
        AND discourse_reactions_reaction_users.user_id = post_actions.user_id
      WHERE post_actions.post_id = #{post.id}
    )
  SQL

  query.where(where_clause)
  # 必须返回修改后的 query
end
```

---

### 4. 修改数组（array 类型）

`register_modifier(:topics_filter_options)` 接收结果数组，在其中追加或修改选项：

```ruby
# assign 插件：在话题过滤器中添加 "assigned:" 选项
register_modifier(:topics_filter_options) do |results, guardian|
  # 第一个参数：当前已有的过滤选项数组
  # 第二个参数：guardian（权限对象）

  if guardian.can_assign?
    results << {
      name: "assigned:",
      description: I18n.t("discourse_assign.filter.description.assigned"),
      type: "username_group_list",
      extra_entries: [
        { name: "nobody", description: I18n.t("...") },
        { name: "*",      description: I18n.t("...") },
      ],
      priority: 1,
    }
  end

  results  # 必须返回结果数组
end
```

---

### 5. 修改布尔值或简单值

有些 modifier 用于开关判断，block 返回修改后的 boolean/string：

```ruby
# 示例：控制某功能是否对特定用户开放
register_modifier(:include_discourse_reactions_data_on_topic_list) do |value, user|
  # value 是当前布尔值（默认 false）
  # user 是当前用户对象
  next value unless user  # 未登录时保持原值

  # 根据用户条件决定是否开放
  value || user.staff?
end
```

---

### 6. 常用 modifier 名称（来自 discourse-assign 和 discourse-reactions）

| modifier 名 | 参数类型 | 用途 |
|------------|---------|------|
| `topics_filter_options` | Array, guardian | 话题列表过滤选项 |
| `user_action_stream_builder` | QueryBuilder | 用户动态流查询 |
| `post_action_users_list` | AR Relation, post | 帖子点赞用户列表 |
| `include_discourse_reactions_data_on_topic_list` | Boolean, user | 是否在列表显示 reactions |
| `topic_query_create_list_topics` | AR Relation, options | TopicQuery 列表结果 |
| `similar_topic_candidate_ids` | Array, args | 相似话题候选 ID |
| `chat_allowed_bot_user_ids` | Array, guardian | Chat 允许的 bot 用户 |

---

### 7. 测试中取消注册（unregister_modifier）

```ruby
# spec 中（仅限测试环境）
before do
  DiscoursePluginRegistry.register_modifier(plugin, :my_modifier, &my_block)
end

after do
  DiscoursePluginRegistry.unregister_modifier(plugin, :my_modifier, &my_block)
end
```

`unregister_modifier` 只允许在测试环境（`Rails.env.test?`）中调用，生产环境禁止。

---

## 反模式（避免这样做）

```ruby
# block 里忘记返回 value（管道断裂，后续 modifier 收到 nil）
register_modifier(:topics_filter_options) do |results, guardian|
  results << { name: "my_option:" }
  # 忘记 return results！ → 下一个 modifier 收到 nil
end

# 正确：必须返回值
register_modifier(:topics_filter_options) do |results, guardian|
  results << { name: "my_option:" }
  results  # 显式返回
end

# 在 after_initialize 之外调用 register_modifier
register_modifier(:some_modifier) { |v| v }  # 在 after_initialize 外执行，可能出错

# 应始终在 after_initialize 内
after_initialize do
  register_modifier(:some_modifier) { |v| v }
end

# 修改核心数据结构的 modifier 忽略 extra_args（丢失上下文）
register_modifier(:post_action_users_list) do |query|
  # 丢掉了 post 参数！无法按 post 过滤
  query.where("...")
end

# 正确：接收所有参数
register_modifier(:post_action_users_list) do |query, post|
  query.where("post_actions.post_id = ?", post.id)
end
```

---

## 关联规范

- `plugins/01_plugin_rb_and_backend.md` — plugin.rb 整体结构与 after_initialize
- `ruby/02_models_concerns.md` — prepend 模式（修改核心类的另一种方式）
- `ruby/08_guardian.md` — guardian 权限对象

---

## 快速参考

```ruby
# 修改数组：追加选项
register_modifier(:modifier_name) do |results, guardian|
  results << { name: "my_option" } if guardian.can_do_thing?
  results
end

# 修改 AR Relation：添加 WHERE 条件
register_modifier(:modifier_name) do |query, extra|
  query.where("table.column = ?", extra.id)
end

# 修改 Query Builder：添加 JOIN/WHERE
register_modifier(:modifier_name) do |builder|
  builder.left_join("other_table ON other_table.post_id = a.target_post_id")
  builder.where("other_table.id IS NULL")
  builder
end

# 修改布尔值：按条件翻转
register_modifier(:some_flag) do |value, user|
  next value unless user
  value || user.staff?
end
```
