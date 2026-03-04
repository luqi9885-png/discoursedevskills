# Plugin 21 — 插件数据库操作完整规范

> 来源：`plugins/discourse-ai/app/models/ai_api_request_stat.rb`（事务+DB.exec 批量汇总）、`plugins/discourse-ai/app/models/ai_summary.rb`（upsert）、`plugins/discourse-ai/app/models/rag_document_fragment.rb`（DB.query 多表 JOIN）、`plugins/discourse-ai/app/models/ai_persona.rb`（DB.query_single 查空 ID）、`plugins/discourse-ai/lib/sentiment/post_classification.rb`（upsert_all）、`plugins/discourse-reactions/plugin.rb`（DB.query 报表 + insert_all 迁移关联数据）、`plugins/discourse-assign/app/jobs/scheduled/enqueue_reminders.rb`（DB.query_single + sanitize_sql）、`plugins/discourse-assign/db/migrate/20231011152903_ensure_notifications_consistency.rb`（迁移中 DB.exec 修复数据）

## 概述

Discourse 插件有两套数据库访问方式：

- **ActiveRecord ORM**：适合有验证/回调的单条 CRUD、简单条件查询
- **DB（MiniSQL）**：全局常量 `::DB = MiniSqlMultisiteConnection.instance`，写原生 SQL，适合跨表聚合、批量操作、报表、迁移中修复数据

两者经常在同一方法里混用：用 AR 做条件筛选（`pluck`/`where`），用 `DB.exec` 做批量写入。

---

## 来源验证

| 验证点 | 文件 | 关键方法 |
|--------|------|---------|
| `transaction { DB.exec × 2 }` | `ai_api_request_stat.rb:46-96` | `rollup!` — INSERT INTO…SELECT + DELETE |
| `DB.query` 多表 JOIN 转 Hash 索引 | `rag_document_fragment.rb:42-67` | `indexing_status` — 3 个 LEFT JOIN + `reduce({})` |
| `DB.query` 报表 + 日期遍历补零 | `discourse-reactions/plugin.rb:317` | `add_report("reactions")` |
| `DB.query_single` 查第一个空 ID | `ai_persona.rb:309-322` | `create_user!` — generate_series |
| `DB.query_single` 返回 user_id 列表 | `enqueue_reminders.rb:36-60` | `user_ids` — 多 JOIN + HAVING |
| `DB.exec` 判断精确影响一行 | `assigner.rb:111` | `DB.exec(sql, args) == 1` |
| `upsert` 单条插入或更新 | `ai_summary.rb:14-36` | `store!` — `unique_by:` + `update_only:` |
| `upsert_all` 批量 | `post_classification.rb:198` | `ClassificationResult.upsert_all` |
| `insert_all` + 事务 + 建 ID 映射 | `discourse-reactions/plugin.rb:502` | `on(:first_post_moved)` |
| `DB.exec` 在迁移中修复孤儿数据 | `20231011152903_...rb` | 3 条 `DB.exec(<<~SQL)` |
| `sanitize_sql` 处理动态 SQL 片段 | `enqueue_reminders.rb:32` | `ActiveRecord::Base.sanitize_sql` |

---

## 1. DB 方法速查

| 方法 | 返回值 | 适用场景 |
|------|-------|---------|
| `DB.query(sql, args)` | 结构体数组，`.字段名` 访问 | 多行 SELECT，需要按列名访问 |
| `DB.query_single(sql, args)` | 扁平数组（所有行所有列展开） | 单列多行 / 单行多列 / 单值 |
| `DB.query_hash(sql, args)` | `[{"col" => val}]` 的数组 | 需要 Hash 格式时 |
| `DB.exec(sql, args)` | 受影响行数（Integer） | INSERT / UPDATE / DELETE |

**参数绑定**：必须用命名参数 `:symbol` 或位置参数 `$1`，**不能字符串插值传参数值**：

```ruby
# 正确：命名参数
DB.query("SELECT id FROM posts WHERE user_id = :uid AND created_at > :since",
         uid: user.id, since: 7.days.ago)

# 正确：IN 列表
DB.query("SELECT id FROM topics WHERE id IN (:ids)", ids: [1, 2, 3])

# 错误：字符串插值传参数值 → SQL 注入漏洞！
DB.query("SELECT id FROM posts WHERE user_id = #{params[:user_id]}")
```

动态表名/列名（非用户输入）可以用字符串插值：`"SELECT * FROM #{table_name} WHERE ..."`

---

## 2. DB.query — 多表 JOIN，结果转 Hash 索引

来自 `rag_document_fragment.rb:indexing_status`，真实代码：

```ruby
def self.indexing_status(persona, uploads)
  embeddings_table = DiscourseAi::Embeddings::Schema.for(self).table  # 动态表名，安全插值

  results =
    DB.query(
      <<~SQL,
        SELECT
          uploads.id,
          SUM(CASE WHEN (rdf.upload_id IS NOT NULL) THEN 1 ELSE 0 END)               AS total,
          SUM(CASE WHEN (eft.rag_document_fragment_id IS NOT NULL) THEN 1 ELSE 0 END) AS indexed,
          SUM(CASE WHEN (rdf.upload_id IS NOT NULL AND eft.rag_document_fragment_id IS NULL) THEN 1 ELSE 0 END) AS left
        FROM uploads
        LEFT OUTER JOIN rag_document_fragments rdf
          ON uploads.id = rdf.upload_id
          AND rdf.target_id = :target_id
          AND rdf.target_type = :target_type
        LEFT OUTER JOIN #{embeddings_table} eft
          ON rdf.id = eft.rag_document_fragment_id
        WHERE uploads.id IN (:upload_ids)
        GROUP BY uploads.id
      SQL
      target_id:  persona.id,
      target_type: persona.class.to_s,
      upload_ids: uploads.map(&:id),
    )

  # 转成 Hash 索引，O(1) 查找
  results.reduce({}) do |acc, r|
    acc[r.id] = { total: r.total, indexed: r.indexed, left: r.left }
    acc
  end
end
```

`DB.query` 返回的结构体直接用 `.字段名`（即 SQL AS 别名）访问，`reduce({})` 模式将数组转成按 id 索引的 Hash 供后续 O(1) 查找。

---

## 3. DB.query 在 add_report 内（报表场景）

来自 `discourse-reactions/plugin.rb:317`，真实代码：

```ruby
add_report("reactions") do |report|
  report.icon  = "discourse-emojis"
  report.modes = [:table]
  report.data  = []
  report.labels = [
    { type: :date,   property: :day,        title: I18n.t("reports.reactions.labels.day") },
    { type: :number, property: :like_count, html_title: "👍" },
  ]

  reactions_results =
    DB.query(<<~SQL, start_date: report.start_date.to_date, end_date: report.end_date.to_date)
      SELECT
        reactions.reaction_value,
        count(reaction_users.id)                                   AS reactions_count,
        date_trunc('day', reaction_users.created_at)::date         AS day
      FROM discourse_reactions_reactions AS reactions
      LEFT OUTER JOIN discourse_reactions_reaction_users AS reaction_users
        ON reactions.id = reaction_users.reaction_id
      WHERE reactions.reaction_users_count IS NOT NULL
        AND reaction_users.created_at::DATE >= :start_date::DATE
        AND reaction_users.created_at::DATE <= :end_date::DATE
      GROUP BY reactions.reaction_value, day
    SQL

  # 遍历日期范围，缺失的天补 0，确保报表连续
  (report.start_date.to_date..report.end_date.to_date).each do |date|
    data = { day: date }

    reactions_results.select { |r| r.day == date }.each do |result|
      data["#{result.reaction_value}_count"] ||= 0
      data["#{result.reaction_value}_count"] += result.reactions_count
    end

    report.data << data
  end
end
```

**关键点**：`report.start_date.to_date` / `report.end_date.to_date` 传给 SQL 参数；日期范围遍历补零确保每天都有数据行；结构体字段用 `.day`/`.reactions_count` 访问。

