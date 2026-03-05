# plugins/27 — 搜索扩展（register_search_advanced_filter / add_filter_custom_filter）

> 来源：
> - `plugins/discourse-assign/plugin.rb:898~998`（三种 filter 模式 + add_filter_custom_filter）
> - `plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js:304`（前端 addAdvancedSearchOptions）
> - `lib/plugin/instance.rb:283~315`（DSL 定义：add_filter_custom_filter / register_search_advanced_filter）
> 修订历史：
> - 2026-03-05 初版

## 边界声明

- **本文件负责**：插件扩展搜索过滤的两种后端 DSL（`register_search_advanced_filter` / `add_filter_custom_filter`）及对应前端 `addAdvancedSearchOptions`
- **不负责，见其他文件**：
  - Search 核心架构（`Search.advanced_filter` 基础）→ `ruby/13_search.md`
  - `register_modifier(:search_rank_sort_priorities)` 调整搜索排序权重 → `plugins/11_register_modifier_backend.md`

---

## 概述

插件扩展搜索有两条路：

| DSL | 触发方式 | 典型用途 |
|-----|---------|---------|
| `register_search_advanced_filter` | 正则匹配搜索词（`/in:assigned/`） | 全文搜索框里输入关键字触发 |
| `add_filter_custom_filter` | 话题列表 URL 参数（`?assigned=username`） | 话题列表过滤，不是搜索框 |

两者都在 `after_initialize` 块内调用，都接收 `posts` scope 并返回新 scope。

---

## 来源验证

| 验证点 | 文件 | 行号 | 差异 |
|--------|------|------|------|
| 无捕获组正则（`/in:assigned/`） | `discourse-assign/plugin.rb` | 970 | 块参数只有 `\|posts\|`，无 match |
| 带捕获组正则（`/assigned:(.+)$/`） | `discourse-assign/plugin.rb` | 982 | 块参数 `\|posts, match\|`，match 是捕获的字符串 |
| 权限 guard（`@guardian`） | `discourse-assign/plugin.rb` | 971 | 块内 `@guardian` 是当前用户的 Guardian 实例 |
| `next` 提前返回 | `discourse-assign/plugin.rb` | 971 | 条件不满足时 `next`（不是 return），scope 不变 |
| `add_filter_custom_filter` | `discourse-assign/plugin.rb` | 898 | 话题列表参数，块参数 `\|scope, filter_values, guardian\|` |
| 前端 addAdvancedSearchOptions | `extend-for-assigns.js` | 304 | 在搜索 UI 的「高级搜索」面板注册选项 |

---

## 核心规范

### 1. register_search_advanced_filter — 搜索框关键字过滤

作用于全文搜索（`/search?q=...`），正则匹配用户输入的搜索词。

**模式 A：无捕获组（开关式）**

```ruby
# 来自 discourse-assign/plugin.rb:970 真实代码
# 触发：搜索词中含 "in:assigned"
register_search_advanced_filter(/in:assigned/) do |posts|
  next if !@guardian.can_assign?   # @guardian 是当前用户，无权限则不过滤（跳过）

  posts.where("topics.id IN (SELECT a.topic_id FROM assignments a WHERE a.active)")
end

register_search_advanced_filter(/in:unassigned/) do |posts|
  next if !@guardian.can_assign?

  posts.where("topics.id NOT IN (SELECT a.topic_id FROM assignments a WHERE a.active)")
end
```

**模式 B：带捕获组（带参数值）**

```ruby
# 来自 discourse-assign/plugin.rb:982 真实代码
# 触发：搜索词中含 "assigned:username" 或 "assigned:groupname"
register_search_advanced_filter(/assigned:(.+)$/) do |posts, match|
  # match 是正则第一个捕获组的内容（字符串）
  next if !@guardian.can_assign? || match.blank?

  if user_id = User.find_by_username(match)&.id
    posts.where(<<~SQL, user_id)
      topics.id IN (
        SELECT a.topic_id FROM assignments a
        WHERE a.assigned_to_id = ? AND a.assigned_to_type = 'User' AND a.active
      )
    SQL
  elsif group_id = Group.find_by(name: match)&.id
    posts.where(<<~SQL, group_id)
      topics.id IN (
        SELECT a.topic_id FROM assignments a
        WHERE a.assigned_to_id = ? AND a.assigned_to_type = 'Group' AND a.active
      )
    SQL
  end
  # 两个 elsif 都不匹配时隐式返回 nil，等同于不过滤
end
```

**关键规则：**

- 块内 `@guardian` 是当前用户的 Guardian 实例（由 Search 注入）
- 无权限时用 `next`（不是 `return`），让搜索继续但不应用此过滤
- 返回 `nil` 或 `next` = 不修改 scope；返回新的 `posts` scope = 应用过滤
- 正则触发后搜索词中的匹配部分**会被自动移除**，不影响全文匹配

### 2. add_filter_custom_filter — 话题列表 URL 参数过滤

作用于话题列表（`/latest?assigned=username`），不是全文搜索。块参数与 `register_search_advanced_filter` 不同：

```ruby
# 来自 discourse-assign/plugin.rb:898 真实代码
# 触发：话题列表 URL 含 ?assigned=xxx 参数
add_filter_custom_filter("assigned") do |scope, filter_values, guardian|
  # filter_values：数组，来自 URL 参数（?assigned=a,b 则为 ["a,b"]）
  # guardian：当前用户 Guardian（直接传入，非 @guardian）
  # scope：Topic AR scope

  next if !guardian.can_assign? || filter_values.blank?

  # 处理逗号分隔的多个值
  names = filter_values.compact
    .flat_map { |value| value.to_s.split(",") }
    .map(&:strip)
    .reject(&:blank?)

  next if names.blank?

  # 特殊值处理
  if names.include?("nobody")
    next scope.where("topics.id NOT IN (SELECT a.topic_id FROM assignments a WHERE a.active)")
  end

  if names.include?("*")
    next scope.where("topics.id IN (SELECT a.topic_id FROM assignments a WHERE a.active)")
  end

  # 区分用户和群组（共享命名空间）
  found_names, user_ids = User.where(username_lower: names.map(&:downcase))
    .pluck(:username, :id).transpose
  found_names ||= []
  user_ids ||= []

  remaining_names = names - found_names
  group_ids = Group.visible_groups(guardian.user)
    .where(name: remaining_names).pluck(:id)

  next scope.none if user_ids.empty? && group_ids.empty?

  assignment_query = Assignment.none
  assignment_query = assignment_query.or(
    Assignment.active.where(assigned_to_type: "User", assigned_to_id: user_ids)
  ) if user_ids.present?
  assignment_query = assignment_query.or(
    Assignment.active.where(assigned_to_type: "Group", assigned_to_id: group_ids)
  ) if group_ids.present?

  scope.where(id: assignment_query.select(:topic_id))
end
```

**与 register_search_advanced_filter 的关键差异：**

| | `register_search_advanced_filter` | `add_filter_custom_filter` |
|---|---|---|
| 触发场景 | 全文搜索框 | 话题列表 URL 参数 |
| 权限对象 | `@guardian`（实例变量） | `guardian`（块参数） |
| scope 类型 | posts（Join topics） | topics |
| 参数来自 | 正则捕获组 | `filter_values` 数组 |

### 3. 前端：addAdvancedSearchOptions — 在搜索 UI 注册选项

让用户在搜索面板的「高级选项」里点选（而非手动输入）：

```javascript
// 来自 extend-for-assigns.js:304 真实代码
withPluginApi("1.0", (api) => {
  api.addAdvancedSearchOptions(
    // 只有有权限的用户才注册，否则传空对象
    api.getCurrentUser()?.can_assign
      ? {
          inOptionsForUsers: [
            {
              name: i18n("search.advanced.in.assigned"),
              value: "assigned",     // 点选后搜索词追加 "in:assigned"
            },
            {
              name: i18n("search.advanced.in.unassigned"),
              value: "unassigned",   // 追加 "in:unassigned"
            },
          ],
        }
      : {}
  );
});
```

**addAdvancedSearchOptions 可配置的字段：**

| 字段 | 说明 |
|------|------|
| `inOptionsForUsers` | 追加到「in:」选项列表（用户可见） |
| `inOptionsForAll` | 追加到「in:」选项列表（所有人可见） |
| `statusOptions` | 追加到「status:」选项列表 |

---

## 完整后端搜索扩展三件套

一个完整的插件搜索扩展通常同时用三种机制：

```ruby
# plugin.rb — after_initialize 内

# 1. 搜索框关键字过滤（用户手动输入 "in:xxx" 或 "xxx:value"）
register_search_advanced_filter(/in:my_filter/) do |posts|
  next if !@guardian.can_use_my_filter?
  posts.where("topics.id IN (SELECT topic_id FROM my_plugin_items WHERE active)")
end

# 2. 话题列表 URL 参数过滤（如 /latest?my_filter=value）
add_filter_custom_filter("my_filter") do |scope, filter_values, guardian|
  next if !guardian.can_use_my_filter? || filter_values.blank?
  value = filter_values.first
  scope.where("topics.id IN (SELECT topic_id FROM my_plugin_items WHERE value = ?)", value)
end

# 3. 告知搜索系统预加载关联（避免 N+1）
Search.custom_topic_eager_load { [:my_plugin_item] }
```

```javascript
// 前端：让用户在搜索 UI 里点选而不用手动输入
withPluginApi("1.0", (api) => {
  api.addAdvancedSearchOptions(
    api.getCurrentUser()?.can_use_my_filter
      ? { inOptionsForUsers: [{ name: i18n("my_plugin.search.label"), value: "my_filter" }] }
      : {}
  );
});
```

---

## 反模式

```ruby
# 错误1：用 return 代替 next（会抛 LocalJumpError）
register_search_advanced_filter(/in:assigned/) do |posts|
  return posts if !@guardian.can_assign?  # LocalJumpError！
end
# 正确：用 next
register_search_advanced_filter(/in:assigned/) do |posts|
  next if !@guardian.can_assign?
  posts.where(...)
end

# 错误2：add_filter_custom_filter 中用 @guardian
add_filter_custom_filter("assigned") do |scope, filter_values, guardian|
  next if !@guardian.can_assign?  # @guardian 在此 context 为 nil！
end
# 正确：用块参数 guardian
add_filter_custom_filter("assigned") do |scope, filter_values, guardian|
  next if !guardian.can_assign?
end

# 错误3：register_search_advanced_filter 中忘记处理 match 为空的情况
register_search_advanced_filter(/assigned:(.+)$/) do |posts, match|
  # match 理论上不会空（有捕获组），但防御性写法更安全
  posts.where("...", match)  # 若 match 为空字符串，SQL 可能返回错误结果
end
# 正确：加 blank? 守卫
register_search_advanced_filter(/assigned:(.+)$/) do |posts, match|
  next if match.blank?
  posts.where("...", match)
end

# 错误4：在 scope 上直接 .count 或 .to_a（破坏链式调用）
register_search_advanced_filter(/in:assigned/) do |posts|
  posts.where(...).count  # 返回整数，不是 scope！后续 Search 无法继续链式
end
# 正确：始终返回 scope
```

---

## 快速参考

```ruby
# 搜索框关键字（无参数）
register_search_advanced_filter(/in:my_keyword/) do |posts|
  next if !@guardian.can_xxx?
  posts.where("topics.id IN (SELECT topic_id FROM ...)")
end

# 搜索框关键字（带参数值）
register_search_advanced_filter(/my_filter:(.+)$/) do |posts, match|
  next if match.blank?
  posts.where("... = ?", match)
end

# 话题列表 URL 参数
add_filter_custom_filter("my_filter") do |scope, filter_values, guardian|
  next if !guardian.can_xxx? || filter_values.blank?
  value = filter_values.flat_map { |v| v.to_s.split(",") }.first
  scope.where("... = ?", value)
end
```
