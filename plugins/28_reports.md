# plugins/28 — Report.add_report + add_directory_column

> 来源：
> - `plugins/discourse-solved/plugin.rb:213~250`（Report.add_report 标准模式 + add_category_filter）
> - `plugins/discourse-solved/plugin.rb:307~339`（add_directory_column + SQL query）
> - `plugins/discourse-reactions/plugin.rb:289~368`（add_report 多列表格模式 + DB.query）
> - `plugins/discourse-solved/config/locales/server.en.yml`（reports i18n key 结构）
> - `app/models/report.rb:96~200`（Report attr_accessor 完整列表）
> 修订历史：
> - 2026-03-05 初版

## 边界声明

- **本文件负责**：插件向 Admin Dashboard 贡献统计图表（`Report.add_report`）和向用户目录贡献自定义列（`add_directory_column`）
- **不负责，见其他文件**：
  - DB.query / DB.exec 用法 → `plugins/21_database_operations.md`
  - Admin UI 页面布局 → `plugins/09_admin_ui.md`

---

## 概述

`Report.add_report` 在 Admin Dashboard（`/admin/reports/xxx`）添加图表，支持折线图（默认）和表格两种模式。`add_directory_column` 在用户目录（`/users`）添加新的统计列，数据由自定义 SQL 维护。

两者都在 `after_initialize` 内调用，都需要 i18n key 配合。

---

## 来源验证

| 验证点 | 文件 | 行号 | 差异 |
|--------|------|------|------|
| 折线图模式（`{x:date, y:count}`） | `discourse-solved/plugin.rb` | 213 | 默认 labels，data 用 `{x, y}` |
| 表格模式（`modes: [:table]` + 自定义 labels） | `discourse-reactions/plugin.rb` | 289 | 需显式设 `report.modes = [:table]` 和 `report.labels` |
| `add_category_filter` | `discourse-solved/plugin.rb` | 222 | 返回 `[category_id, include_subcategories]`，需解构 |
| `report.icon` | `discourse-reactions/plugin.rb` | 292 | 字符串 icon name（SVG icon ID） |
| `add_directory_column` + SQL | `discourse-solved/plugin.rb` | 307 | SQL 必须包含 `period_type` 参数和 UPDATE directory_items 结构 |
| i18n key 结构 | `discourse-solved/config/locales/server.en.yml` | 27 | `reports.报告名.title/xaxis/yaxis` |

---

## 核心规范

### 1. Report.add_report — 折线图模式（最常用）

```ruby
# 来自 discourse-solved/plugin.rb:213 真实代码
Report.add_report("accepted_solutions") do |report|
  report.data = []

  # 1. 构建基础查询
  accepted_solutions =
    DiscourseSolved::SolvedTopic
      .joins(:topic)
      .where.not(topics: { archetype: Archetype.private_message })

  # 2. 可选：加入分类过滤（Admin Dashboard 上的过滤 UI 自动生成）
  category_id, include_subcategories = report.add_category_filter
  if category_id
    if include_subcategories
      accepted_solutions = accepted_solutions.where(
        "topics.category_id IN (?)", Category.subcategory_ids(category_id)
      )
    else
      accepted_solutions = accepted_solutions.where("topics.category_id = ?", category_id)
    end
  end

  # 3. 按日期范围过滤 + 按天分组 → 填充 data
  accepted_solutions
    .where("...created_at >= ?", report.start_date)
    .where("...created_at <= ?", report.end_date)
    .group("DATE(...created_at)")
    .order("DATE(...created_at)")
    .count
    .each { |date, count| report.data << { x: date, y: count } }

  # 4. 总数和前30天对比（可选，Dashboard 上显示趋势）
  report.total = accepted_solutions.count
  report.prev30Days =
    accepted_solutions
      .where("...created_at >= ?", report.start_date - 30.days)
      .where("...created_at <= ?", report.start_date)
      .count
end
```

**折线图 data 格式：**

```ruby
report.data << { x: Date.today, y: 42 }   # x: 日期，y: 数值
```

### 2. Report.add_report — 表格模式（多列）

```ruby
# 来自 discourse-reactions/plugin.rb:289 真实代码（简化）
add_report("reactions") do |report|
  # 1. 设置图标和模式
  report.icon = "discourse-emojis"   # Admin Dashboard 中的图标
  report.modes = [:table]             # 显示为表格而非折线图

  report.data = []

  # 2. 定义表格列（labels）
  report.labels = [
    { type: :date,   property: :day,        title: I18n.t("reports.reactions.labels.day") },
    { type: :number, property: :like_count, html_title: "<emoji>" },
  ]
  # 动态追加更多列
  reactions.each do |reaction|
    report.labels << {
      type: :number,
      property: "#{reaction}_count",
      html_title: PrettyText.unescape_emoji(CGI.escapeHTML(":#{reaction}:")),
    }
  end

  # 3. 用 DB.query 查询（SQL 性能敏感时直接用原始 SQL）
  results = DB.query(<<~SQL, start_date: report.start_date.to_date, end_date: report.end_date.to_date)
    SELECT reaction_value, count(id) as count,
           date_trunc('day', created_at)::date as day
    FROM my_reactions
    WHERE created_at::DATE >= :start_date::DATE
      AND created_at::DATE <= :end_date::DATE
    GROUP BY reaction_value, day
  SQL

  # 4. 按日期遍历，组装每行数据
  (report.start_date.to_date..report.end_date.to_date).each do |date|
    data = { day: date }
    results.select { |r| r.day == date }.each do |result|
      data["#{result.reaction_value}_count"] = result.count
    end
    report.data << data
  end
end
```

**表格 label type 可选值：**

| type | 说明 |
|------|------|
| `:date` | 日期列，自动格式化 |
| `:number` | 数字列 |
| `:text` | 文本列 |
| `:link` | 链接列，需要 `{ property: :url, title_property: :label }` |

### 3. Report 对象常用属性速查

```ruby
report.data          # Array，每个元素是 Hash（折线图用 {x:, y:}，表格用自定义 key）
report.total         # Integer，总数（显示在图表右上角）
report.prev30Days    # Integer，前30天数，用于计算趋势百分比
report.start_date    # Time，查询起始时间（已设好，直接用）
report.end_date      # Time，查询结束时间（已设好，直接用）
report.labels        # Array，表格列定义
report.modes         # Array，[:chart]（默认）或 [:table] 或 [:chart, :table]
report.icon          # String，SVG icon name
report.higher_is_better  # Boolean，true=数值越高越好（默认），影响趋势颜色
report.dates_filtering   # Boolean，是否显示日期过滤器（默认 true）
```

### 4. i18n key 结构

```yaml
# config/locales/server.en.yml
en:
  reports:
    my_report_name:          # 与 add_report("my_report_name") 对应
      title: "My Report"
      xaxis: "Day"           # 折线图 X 轴标签
      yaxis: "Count"         # 折线图 Y 轴标签
      description: "..."     # 可选，显示在报告顶部
```

### 5. add_directory_column — 用户目录自定义列

用户目录（`/users`）按周期统计每个用户的指标，`add_directory_column` 注册新列并提供更新 SQL：

```ruby
# 来自 discourse-solved/plugin.rb:307 真实代码
query = <<~SQL
  -- 第一步：重置当前周期的值为 0（必须）
  UPDATE directory_items di
     SET solutions = 0
   WHERE di.period_type = :period_type AND di.solutions IS NOT NULL;

  -- 第二步：计算并更新每个用户的统计值
  WITH x AS (
    SELECT p.user_id, COUNT(DISTINCT st.id) AS solutions
    FROM discourse_solved_solved_topics AS st
    JOIN posts AS p ON p.id = st.answer_post_id
      AND COALESCE(st.created_at, :since) > :since  -- :since 是周期起始时间
      AND p.deleted_at IS NULL
    JOIN topics AS t ON t.id = st.topic_id
      AND t.archetype <> 'private_message'
      AND t.deleted_at IS NULL
    JOIN users AS u ON u.id = p.user_id
    WHERE u.id > 0 AND u.active
      AND u.silenced_till IS NULL AND u.suspended_till IS NULL
    GROUP BY p.user_id
  )
  UPDATE directory_items di
     SET solutions = x.solutions
    FROM x
   WHERE x.user_id = di.user_id
     AND di.period_type = :period_type;
SQL

add_directory_column("solutions", query:)
```

**SQL 中可用的参数：**

| 参数 | 说明 |
|------|------|
| `:period_type` | 统计周期类型（day/week/month/quarter/year/all），由系统传入 |
| `:since` | 周期起始时间，用于过滤数据范围 |

**DB Migration（必须先建列）：**

```ruby
# db/migrate/20240101000000_add_solutions_to_directory_items.rb
class AddSolutionsToDirectoryItems < ActiveRecord::Migration[7.0]
  def change
    add_column :directory_items, :solutions, :integer
  end
end
```

---

## 反模式

```ruby
# 错误1：忘记初始化 report.data = []，导致 nil.push 报错
Report.add_report("my_report") do |report|
  # 没有 report.data = []
  SomeModel.count_by_day.each { |date, count| report.data << { x: date, y: count } }
end
# 正确：第一行设 report.data = []

# 错误2：折线图 data 用 {date:, count:} 而非 {x:, y:}
report.data << { date: date, count: count }  # 前端无法识别
# 正确：折线图固定用 {x:, y:}
report.data << { x: date, y: count }

# 错误3：add_directory_column SQL 没有先 UPDATE ... SET col = 0
# 导致旧数据残留，切换统计周期时数值不刷新
# 正确：SQL 第一步必须 UPDATE ... SET my_col = 0 WHERE period_type = :period_type

# 错误4：add_directory_column 没有对应的 Migration
# directory_items 不存在该列，SQL UPDATE 会报错
# 正确：先写 Migration add_column :directory_items, :my_col, :integer

# 错误5：report.add_category_filter 返回值没有解构
category_id = report.add_category_filter  # 实际返回 [category_id, include_subcategories] 或 nil
# 正确：
category_id, include_subcategories = report.add_category_filter
```

---

## 快速参考

```ruby
# ── 折线图报告 ─────────────────────────────────────────
Report.add_report("my_report") do |report|
  report.data = []
  category_id, include_subcategories = report.add_category_filter

  base = MyModel.where(created_at: report.start_date..report.end_date)
  base = base.where(category_id:) if category_id

  base.group("DATE(created_at)").order("DATE(created_at)").count
      .each { |date, count| report.data << { x: date, y: count } }

  report.total = MyModel.count
end

# ── 表格报告 ───────────────────────────────────────────
add_report("my_table_report") do |report|
  report.modes = [:table]
  report.data = []
  report.labels = [
    { type: :date,   property: :day,   title: I18n.t("reports.my_table_report.labels.day") },
    { type: :number, property: :count, title: I18n.t("reports.my_table_report.labels.count") },
  ]
  results = DB.query("SELECT ...", start_date: report.start_date.to_date, end_date: report.end_date.to_date)
  (report.start_date.to_date..report.end_date.to_date).each do |date|
    report.data << { day: date, count: results.find { |r| r.day == date }&.count || 0 }
  end
end

# ── 用户目录列 ─────────────────────────────────────────
add_directory_column("my_stat", query: <<~SQL)
  UPDATE directory_items SET my_stat = 0
   WHERE period_type = :period_type AND my_stat IS NOT NULL;
  WITH x AS (SELECT user_id, COUNT(*) AS my_stat FROM ... WHERE created_at > :since GROUP BY user_id)
  UPDATE directory_items di SET my_stat = x.my_stat FROM x
   WHERE x.user_id = di.user_id AND di.period_type = :period_type;
SQL
```
