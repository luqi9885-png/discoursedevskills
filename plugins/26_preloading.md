# plugins/26 — 预加载（Preloading）N+1 防治规范

> 来源：
> - `plugins/discourse-assign/plugin.rb`（TopicList/TopicView/BookmarkQuery/Search.on_preload + add_to_class preload_xxx）
> - `plugins/discourse-solved/plugin.rb`（register_topic_preloader_associations + Search.custom_topic_eager_load）
> 修订历史：
> - 2026-03-04 初版

## 边界声明

- **本文件负责**：插件中预防 N+1 查询的完整系统，包括 `on_preload` 回调、`register_topic_preloader_associations`、`add_to_class` 预加载存储方法、Serializer 中读取预加载值
- **不负责，见其他文件**：
  - `add_to_serializer` 基本用法 → `plugins/22_serializer_extensions.md`
  - ActiveRecord `includes/preload/eager_load` 核心用法 → `ruby/02_models_concerns.md`
- **与以下文件有交叉，已在 TOPIC_MAP 协调**：
  - `plugins/22_serializer_extensions.md`（本文写 on_preload 存数据；那边写 Serializer 如何读数据，两者配合）

---

## 概述

插件在 topic list、topic view、搜索结果中追加数据时，若每条记录单独查询则产生 N+1 问题。Discourse 提供了四个 `on_preload` 钩子，让插件在批量查询阶段一次性加载所有关联数据，再通过 `add_to_class` 定义的临时存储方法注入到每个对象，Serializer 最后读取这些预加载值。

---

## 来源验证

| 验证点 | 文件 | 关键行 | 差异 |
|--------|------|--------|------|
| TopicList.on_preload | `discourse-assign/plugin.rb:196` | `assignments_map = assignments.group_by(&:topic_id)` | 批量查询后 group_by 建索引，按 topic 逐一注入 |
| TopicView.on_preload | `discourse-assign/plugin.rb:190` | `topic_view.posts.includes(:assignment)` | 直接用 AR includes，不需要手动 map |
| BookmarkQuery.on_preload | `discourse-assign/plugin.rb:166` | `assignments.index_by(&:topic_id)` | 用 index_by 而非 group_by（每个 topic 只有一条） |
| Search.on_preload | `discourse-assign/plugin.rb:253` | `results.posts.map(&:topic)` | 从 posts 反查 topics，与 TopicList 入参不同 |
| add_to_class preload 存储 | `discourse-assign/plugin.rb:493` | `add_to_class(:topic, :preload_assigned_to)` | 在 Topic 上动态添加实例变量存储方法 |
| register_topic_preloader_associations | `discourse-solved/plugin.rb:196` | `register_topic_preloader_associations(:solved)` | 用于 AR association 预加载，更简洁 |
| Search.custom_topic_eager_load | `discourse-solved/plugin.rb:197` | `Search.custom_topic_eager_load { [:solved] }` | 声明式注册，让 Search 自动 eager load |

---

## 核心规范

### 1. 完整模式：add_to_class + on_preload + Serializer

三步缺一不可，顺序固定：

**第一步：在 Model 上定义临时存储方法（add_to_class）**

```ruby
# plugin.rb — after_initialize 内
# 来自 discourse-assign/plugin.rb:493 真实代码

# 写入方法（由 on_preload 调用）
add_to_class(:topic, :preload_assigned_to) { |val| @assigned_to = val }
add_to_class(:topic, :preload_assignment_status) { |val| @assignment_status = val }

# 读取方法（由 Serializer 调用）
add_to_class(:topic, :assigned_to) { @assigned_to }
add_to_class(:topic, :assignment_status) { @assignment_status }
```

**第二步：在 on_preload 中批量查询并注入**

```ruby
# 来自 discourse-assign/plugin.rb:196 真实代码
TopicList.on_preload do |topics, topic_list|
  next unless SiteSetting.my_plugin_enabled?

  # 权限检查（先过滤，减少不必要的查询）
  next if !topic_list.current_user&.can_do_something? || topics.empty?

  # 1. 一次性批量查询（绝对不在循环内查询）
  items = MyPlugin::Item
    .strict_loading                          # 防止懒加载触发新查询
    .active
    .where(topic: topics)
    .includes(:assigned_to, :target)         # 预加载关联，防止 Serializer 触发 N+1
    .group_by(&:topic_id)                    # 建索引：topic_id => [items]

  # 2. 逐个注入（从索引取，O(1)，不查 DB）
  topics.each do |topic|
    item = items[topic.id]
    # NOTE: preloading to `nil` is necessary to avoid N+1 queries
    # 即使没有数据也必须调用，否则 Serializer 会触发 DB 查询
    topic.preload_my_item(item)
  end
end
```

**第三步：Serializer 读取预加载值（不触发新查询）**

```ruby
add_to_serializer(:topic_list_item, :my_item_data) do
  object.my_item   # 读取 @my_item 实例变量，无 DB 查询
end

add_to_serializer(:topic_list_item, :include_my_item_data?) do
  object.instance_variable_defined?(:@my_item)  # 安全检查是否已预加载
end
```

### 2. TopicList.on_preload（话题列表）

```ruby
# 来自 discourse-assign/plugin.rb 真实代码，简化版
TopicList.on_preload do |topics, topic_list|
  next unless SiteSetting.my_plugin_enabled?
  next if topics.empty?

  # group_by：一个 topic 可能有多条记录时用
  items_map = MyPlugin::Item.where(topic: topics).group_by(&:topic_id)

  # index_by：一个 topic 只有一条记录时用（更简洁）
  # items_map = MyPlugin::Item.where(topic: topics).index_by(&:topic_id)

  topics.each do |topic|
    topic.preload_my_item(items_map[topic.id])  # nil 也要注入
  end
end
```

### 3. TopicView.on_preload（话题详情页）

```ruby
# 来自 discourse-assign/plugin.rb:190 真实代码
TopicView.on_preload do |topic_view|
  next unless SiteSetting.my_plugin_enabled?

  # topic_view.posts 是当前页的帖子集合
  # 直接用 AR includes，Discourse 会在同一个查询中加载关联
  topic_view.instance_variable_set(
    :@posts,
    topic_view.posts.includes(:my_plugin_item)
  )
end
```

### 4. BookmarkQuery.on_preload（书签列表）

```ruby
# 来自 discourse-assign/plugin.rb:166 真实代码
BookmarkQuery.on_preload do |bookmarks, _bookmark_query|
  next unless SiteSetting.my_plugin_enabled?

  # 从 bookmarks 提取所有 topic
  topics =
    Bookmark.select_type(bookmarks, "Topic")
      .map(&:bookmarkable)
      .concat(
        Bookmark.select_type(bookmarks, "Post").map { |bm| bm.bookmarkable.topic }
      )
      .uniq
      .compact

  return if topics.empty?

  items = MyPlugin::Item.where(topic: topics).index_by(&:topic_id)

  topics.each do |topic|
    topic.preload_my_item(items[topic.id])
  end
end
```

### 5. Search.on_preload（搜索结果）

```ruby
# 来自 discourse-assign/plugin.rb:253 真实代码
Search.on_preload do |results, search|
  next unless SiteSetting.my_plugin_enabled?
  next if results.posts.empty?

  # 注意：search results 入参是 posts，需要从 posts 提取 topics
  topics = results.posts.map(&:topic).uniq.compact

  items = MyPlugin::Item.where(topic: topics).index_by(&:topic_id)

  results.posts.each do |post|
    post.topic.preload_my_item(items[post.topic_id])
  end
end

# 声明式注册（更简洁，让 Search 自动 eager load AR association）
# 来自 discourse-solved/plugin.rb:197 真实代码
Search.custom_topic_eager_load { [:my_plugin_item] }
```

### 6. register_topic_preloader_associations（最简洁的 AR association 预加载）

当插件数据是 AR association（`has_one :solved` 等）时，用此方法代替 `on_preload`：

```ruby
# 来自 discourse-solved/plugin.rb:196 真实代码
# 声明后，TopicList 和 TopicView 加载时自动 includes(:my_plugin_item)
register_topic_preloader_associations(:my_plugin_item)

# 同时注册 category list 预加载
register_category_list_topics_preloader_associations(:my_plugin_item)
```

**选择指南：**

| 场景 | 推荐方式 |
|------|---------|
| 插件数据是 AR has_one/has_many | `register_topic_preloader_associations` |
| 数据来自独立表（非 AR association） | `TopicList.on_preload` + `add_to_class` |
| 需要复杂权限检查或数据聚合 | `TopicList.on_preload` + `add_to_class` |
| 搜索结果预加载 | `Search.custom_topic_eager_load` 或 `Search.on_preload` |

---

## 各场景入参速查

| 钩子 | 块参数 | 关键字段 |
|------|--------|---------|
| `TopicList.on_preload` | `\|topics, topic_list\|` | `topics`（Array\<Topic\>），`topic_list.current_user` |
| `TopicView.on_preload` | `\|topic_view\|` | `topic_view.topic`，`topic_view.posts` |
| `BookmarkQuery.on_preload` | `\|bookmarks, bookmark_query\|` | `Bookmark.select_type(bookmarks, "Topic/Post")` |
| `Search.on_preload` | `\|results, search\|` | `results.posts`，`search.guardian` |

---

## 反模式

```ruby
# 错误1：在 Serializer 中直接查询（最常见的 N+1 根源）
add_to_serializer(:topic_list_item, :my_item) do
  MyPlugin::Item.find_by(topic_id: object.id)  # 每个 topic 一次查询！100 条 = 100 次查询
end
# 正确：先在 on_preload 中批量加载，Serializer 只读实例变量

# 错误2：on_preload 中没有为无数据的 topic 注入 nil
TopicList.on_preload do |topics, _|
  items = MyPlugin::Item.where(topic: topics).index_by(&:topic_id)
  topics.each do |topic|
    topic.preload_my_item(items[topic.id]) if items[topic.id]  # 跳过了 nil！
  end
end
# 结果：没有数据的 topic 在 Serializer 中会触发 DB 查询（因为实例变量未定义）
# 正确：无条件注入，nil 也要注入
topics.each { |topic| topic.preload_my_item(items[topic.id]) }

# 错误3：on_preload 中没有用 strict_loading，关联被懒加载
items = MyPlugin::Item.where(topic: topics)  # 没有 includes(:assigned_to)
items.each { |item| item.assigned_to.name }  # 每条触发一次查询！
# 正确：includes 所有后续会访问的关联
MyPlugin::Item.where(topic: topics).includes(:assigned_to)

# 错误4：on_preload 内循环查询 DB
TopicList.on_preload do |topics, _|
  topics.each do |topic|
    topic.preload_my_item(MyPlugin::Item.find_by(topic: topic))  # N 次查询！
  end
end
# 正确：先批量查询建 map，再循环注入

# 错误5：Serializer 用 include_condition 但 on_preload 没有对应的 nil 注入
add_to_serializer(:topic_list_item, :include_my_item?) do
  object.instance_variable_defined?(:@my_item)  # 如果 nil 没有注入，永远返回 false
end
```

---

## 快速参考

```ruby
# ── plugin.rb 标准三件套 ──────────────────────────────────

# 1. 存储方法
add_to_class(:topic, :preload_my_item) { |v| @my_item = v }
add_to_class(:topic, :my_item) { @my_item }

# 2. 批量加载
TopicList.on_preload do |topics, topic_list|
  next unless SiteSetting.my_plugin_enabled?
  next if topics.empty?
  map = MyPlugin::Item.where(topic: topics).includes(:user).index_by(&:topic_id)
  topics.each { |t| t.preload_my_item(map[t.id]) }  # nil 也注入
end

# 3. Serializer 读取
add_to_serializer(:topic_list_item, :my_item_data) { object.my_item&.as_json }
add_to_serializer(:topic_list_item, :include_my_item_data?) do
  object.instance_variable_defined?(:@my_item)
end

# ── 简洁版（AR association 时）────────────────────────────
register_topic_preloader_associations(:my_plugin_item)
Search.custom_topic_eager_load { [:my_plugin_item] }
```
