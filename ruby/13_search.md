# Ruby/Patterns 13 — Search 搜索系统

> 来源：`lib/search.rb`（1658 行）、`app/models/post_search_data.rb`、
> `app/models/topic_search_data.rb`、`lib/search/grouped_search_results.rb`

---

## 概述

Discourse 使用 PostgreSQL 全文搜索（tsvector/tsquery）作为后端，通过 `Search` 类封装查询、高级过滤、多语言分词、排序和 Plugin 扩展点。

---

## 来源验证

- `lib/search.rb`（主类）
- `app/models/post_search_data.rb` / `topic_search_data.rb`（索引数据表）
- `lib/search/grouped_search_results.rb`（结果容器）
- `discourse/frontend/discourse/app/services/search.js`（前端调用）

---

## 核心规范

### 1. 基本调用

```ruby
# 简单查询（class method）
results = Search.execute("discourse ruby", guardian: Guardian.new(user))

# 完整控制
searcher = Search.new("discourse ruby", {
  guardian: guardian,
  type_filter: "topic",      # 限定类型：topic/user/category/tag/private_messages
  page: 1,
  search_type: :full_page,   # :full_page（完整搜索页）或 :header（搜索框）
  ip_address: request.remote_ip,
  user_id: current_user&.id,
})
results = searcher.execute
```

### 2. 结果结构（GroupedSearchResults）

```ruby
results.posts      # 帖子/话题搜索结果（Post 对象数组，含 topic 关联）
results.users      # 用户搜索结果
results.categories # 分类搜索结果
results.tags       # 标签搜索结果

results.term       # 搜索词
results.search_log_id   # 搜索日志 ID（用于点击追踪）

results.grouped_query_limit  # 是否达到分组上限
```

### 3. 高级搜索过滤器语法

所有高级过滤都在查询词中内联，`process_advanced_search!` 方法解析后从 term 中移除：

| 语法 | 功能 |
|------|------|
| `status:open` | 未关闭、未归档的话题 |
| `status:closed` | 已关闭 |
| `status:archived` | 已归档 |
| `status:noreplies` | 只有 1 楼的话题 |
| `category:name` | 指定分类（含子分类） |
| `=category:name` | 精确分类（不含子分类） |
| `#slug` | 按分类 slug 或标签过滤 |
| `tags:tag1,tag2` | 包含任一标签 |
| `tags:tag1+tag2` | 同时包含所有标签 |
| `user:username` | 指定用户的帖子 |
| `in:title` 或 `t` | 仅搜索标题 |
| `in:first` 或 `f` | 仅首楼 |
| `in:replies` | 仅回复（非首楼） |
| `in:pinned` | 置顶话题 |
| `in:wiki` | Wiki 帖子 |
| `in:likes` | 我点赞的帖子 |
| `in:bookmarks` | 我收藏的帖子 |
| `in:posted` | 我发过的帖子 |
| `in:seen` / `in:unseen` | 已读/未读 |
| `before:2024-01-01` | 早于指定日期 |
| `after:2024-01-01` | 晚于指定日期 |
| `order:latest` | 按最新排序 |
| `order:likes` | 按点赞数排序 |
| `order:views` | 按浏览量排序 |
| `min_posts:5` | 最少楼数 |
| `max_posts:10` | 最多楼数 |
| `with:images` | 含图片帖子 |
| `in:personal` | 我的私信 |
| `badge:name` | 拥有该徽章的用户发的帖 |

### 4. PostgreSQL 全文搜索实现

```ruby
# search_data 字段存储 tsvector
# 建索引（post_search_data 表）
# CREATE INDEX post_search_data_search_data_idx ON post_search_data USING gin(search_data)

# Search 中的 ts_query 构建
def self.ts_query(term:, ts_config: nil, joiner: nil)
  # 把搜索词转为 tsquery
  ts_config ||= "'#{Search.ts_config}'"
  all = term.scan(/\S+/).map { |word| "#{Search.escape_string(word)}:*" }.join(" & ")
  "to_tsquery(#{ts_config}, '#{all}')"
end

# 实际查询（posts 基础查询）
posts = posts.where(
  "post_search_data.search_data @@ #{Search.ts_query(term: @original_term)}"
)
```

### 5. 多语言分词

```ruby
# 根据 SiteSetting.default_locale 选 PostgreSQL 文本配置
Search.ts_config("zh_TW")  # => "simple"（中文用 simple stemmer）
Search.ts_config("en")     # => "english"

# 中文分词（Cppjieba）
if segment_chinese?
  # 使用 CppjiebaRb 对中文文本做词语切分
  segments = CppjiebaRb.segment(text, mode: :mix)
  data = segments.join(" ")
end

# 日文分词（TinyJapaneseSegmenter）
if segment_japanese?
  data = TinyJapaneseSegmenter.segment(data).join(" ")
end
```

### 6. 排序

```ruby
# 相关性排序（默认）
# 使用 ts_rank_cd + 近期帖子 boost
posts = sort_by_relevance(posts, ...)

# 最新排序
posts.reorder("topics.bumped_at DESC")

# 自定义排序（Plugin 注册）
Search.advanced_order(:my_order, enabled: -> { SiteSetting.my_plugin_enabled }) do |posts|
  posts.reorder("my_score DESC")
end

# 查询时使用：order:my_order
```

### 7. Plugin 扩展 — 自定义过滤器

```ruby
# plugin.rb
Search.advanced_filter(/\Astatus:solved\z/i) do |posts|
  posts.where("topics.id IN (SELECT topic_id FROM solved_topics)")
end

Search.advanced_filter(/\Astatus:unsolved\z/i) do |posts|
  posts.where(
    "topics.id NOT IN (SELECT topic_id FROM solved_topics) AND topics.posts_count > 1"
  )
end

# Plugin 也可注册带命名的 filter（用于 UI 展示）
Search.advanced_filter(
  /\Astatus:solved\z/i,
  name: "solved"
) do |posts|
  posts.joins(:solved_topics)
end
```

### 8. Plugin 扩展 — 预加载与回调

```ruby
# 搜索结果预加载 custom fields
Search.preloaded_topic_custom_fields << "my_field"

# 搜索结果 eager load 额外 join
Search.custom_topic_eager_load(
  ["my_table"],
  enabled: -> { SiteSetting.my_plugin_enabled }
) do |posts|
  posts.joins("LEFT JOIN my_table ON my_table.topic_id = topics.id")
end

# 搜索结果后处理
Search.on_preload do |results, search|
  if results.posts.any?
    topics = results.posts.map(&:topic)
    Topic.preload_custom_fields(topics, ["my_field"])
  end
end
```

### 9. 搜索日志

```ruby
# 每次搜索自动记录到 search_logs 表
SearchLog.log(
  term: @clean_term,
  search_type: @opts[:search_type],   # :full_page / :header / :topic
  ip_address: @opts[:ip_address],
  user_agent: @opts[:user_agent],
  user_id: @opts[:user_id],
)

# 点击搜索结果时记录 click
SearchLog.click_through(search_log_id: id, topic_id: topic_id)
```

### 10. search_data 索引更新

```ruby
# 帖子创建/修改后自动触发（Post 模型）
after_commit :index_search

def index_search
  jobs_args = { post_id: self.id }
  jobs_args[:reindex_topic] = true if saved_change_to_attribute?(:title)
  Jobs.enqueue(:index_post, jobs_args)
end

# app/jobs/regular/index_post.rb
# 调用 PostSearchData.update_index(post_id)
# 生成 tsvector 存入 post_search_data.search_data
```

---

## 反模式（避免这样做）

```ruby
# ❌ 直接用 SQL LIKE — 不支持相关性、分词、多语言
Post.where("raw LIKE ?", "%discourse%")

# ❌ 忽略 guardian — 搜索结果可能包含用户无权查看的内容
Search.execute("term")  # 不传 guardian，使用匿名权限

# ✅ 总是传 guardian
Search.execute("term", guardian: Guardian.new(user))
```

---

## 关联规范

- `ruby/08_guardian.md` — 搜索结果权限过滤
- `patterns/05_plugin_backend_dsl.md` — `register_custom_filter_by_status` DSL

---

## 快速参考

```ruby
# 基本搜索
results = Search.execute("关键词 category:general", guardian: guardian)
results.posts          # 结果帖子
results.posts.first.topic  # 关联话题

# 构建 ts_query（在自定义 SQL 中）
Search.ts_query(term: "discourse")
# => "to_tsquery('english', 'discourse:*')"

# Plugin 注册过滤器
Search.advanced_filter(/\Astatus:solved\z/i) { |posts| posts.joins(:solved_topics) }

# Plugin 注册排序
Search.advanced_order(:solved_first) do |posts|
  posts.order("CASE WHEN solved_topics.topic_id IS NULL THEN 1 ELSE 0 END")
end

# 预加载 custom fields 到搜索结果
Search.preloaded_topic_custom_fields << "my_field"
```
