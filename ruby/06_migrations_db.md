# Ruby 06 — ActiveRecord 迁移与数据库规范

> 来源：`db/migrate/`、`db/post_migrate/`、`lib/migration/`、`CLAUDE.md`

---

## 1. 迁移文件基础规范

### 1.1 文件头

```ruby
# frozen_string_literal: true

class AddSlugToTags < ActiveRecord::Migration[8.0]
  def change
    # ...
  end
end
```

- 文件顶部必须有 `# frozen_string_literal: true`
- 类名使用 PascalCase，与文件名时间戳后半段对应
- 继承 `ActiveRecord::Migration[8.0]`（使用当前 Rails 版本号）

### 1.2 生成命令

```bash
bin/rails generate migration AddColumnToTable
bin/rails g post_migration DropOldColumn   # 生成后部署迁移
```

---

## 2. 两类迁移目录

| 目录 | 用途 | 执行时机 |
|------|------|----------|
| `db/migrate/` | 常规迁移（加列、建表、加索引） | 部署前（主服务器启动前） |
| `db/post_migrate/` | 后部署迁移（删列、删表、数据清理） | 部署后（服务已运行时） |

### 为什么需要 post_migrate？

Discourse 的 `SafeMigrate` 模块在常规迁移中会拦截以下危险操作，并抛出异常：

- `DROP TABLE` / `ALTER TABLE ... RENAME TO`
- `ALTER TABLE ... DROP COLUMN` / `ALTER TABLE ... RENAME COLUMN`

正确做法：先废弃（ignored_columns），后在 post_migrate 中删除。

---

## 3. 索引规范：大表必须用 CONCURRENTLY

### 3.1 标准写法（AR 方式）

```ruby
class AddExtraIndexTopicTags < ActiveRecord::Migration[8.0]
  disable_ddl_transaction!  # ← 必须！CONCURRENTLY 不能在事务中执行

  def change
    remove_index :topic_tags, %i[tag_id topic_id], if_exists: true
    add_index :topic_tags, %i[tag_id topic_id], unique: true, algorithm: :concurrently
  end
end
```

**规则：**
- 使用 `algorithm: :concurrently` 避免锁表
- 类顶部必须声明 `disable_ddl_transaction!`
- 并发添加前先 `remove_index ... if_exists: true`（SafeMigrate 要求先 drop 再 concurrent create）

### 3.2 原始 SQL 写法（复杂索引场景）

```ruby
class AddIndexCategoryOnTopics < ActiveRecord::Migration[8.0]
  disable_ddl_transaction!

  def up
    execute "DROP INDEX IF EXISTS index_topics_on_category_id;"
    execute "CREATE INDEX CONCURRENTLY index_topics_on_category_id ON topics (category_id) WHERE deleted_at IS NULL AND (archetype <> 'private_message');"
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

用于：部分索引（WHERE 子句）、函数索引（如 `lower(name)`）等 AR DSL 不支持的场景。

### 3.3 函数索引示例

```ruby
def up
  # 先处理重复数据再建唯一索引
  execute <<~SQL
    UPDATE tag_groups
    SET name = name || id
    WHERE EXISTS(SELECT * FROM tag_groups t WHERE lower(t.name) = lower(tag_groups.name) AND t.id < tag_groups.id)
  SQL

  add_index :tag_groups, "lower(name)", unique: true
end
```

---

## 4. 建表规范

```ruby
class CreateTagLocalizations < ActiveRecord::Migration[8.0]
  def change
    create_table :tag_localizations do |t|
      t.references :tag, null: false          # 外键引用（bigint id）
      t.string :locale, limit: 20, null: false
      t.string :name, null: false
      t.string :description, null: true, limit: 1000

      t.timestamps null: false
    end

    add_index :tag_localizations, %i[tag_id locale], unique: true
  end
end
```

**要点：**
- `null: false` 约束在数据库层面强制
- string 字段加 `limit:` 防止超长数据
- `t.timestamps` 几乎必须加
- 索引与建表分开写（便于阅读），或直接在 `create_table` 块外用 `add_index`

---

## 5. 加列规范

```ruby
class AddNotifyOnLinkedPostsToUserOptions < ActiveRecord::Migration[7.1]
  def change
    add_column :user_options, :notify_on_linked_posts, :boolean, default: true, null: false
  end
end
```

**PostgreSQL 注意事项：**
- 加带 `default` 的列：PG 12+ 支持 instant add（不锁表），安全
- 加不带 default 的列到大表：也安全（值为 NULL）
- **不要** 同时加列并设置 `NOT NULL` 且无默认值（需分步）

---

## 6. 删列的两步流程（安全废弃）

### 第一步：在 Model 中废弃

```ruby
# app/models/user.rb
class User < ApplicationRecord
  self.ignored_columns += ["old_column_name"]
end
```

### 第二步：在 post_migrate 中删除

```ruby
# db/post_migrate/20250821155127_drop_dark_hex_from_color_scheme_color.rb
class DropDarkHexFromColorSchemeColor < ActiveRecord::Migration[8.0]
  DROPPED_COLUMNS = { color_scheme_colors: %i[dark_hex] }

  def up
    DROPPED_COLUMNS.each { |table, columns| Migration::ColumnDropper.execute_drop(table, columns) }
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

`Migration::ColumnDropper.execute_drop` 会：
1. 移除触发器（readonly 保护）
2. 执行 `ALTER TABLE DROP COLUMN IF EXISTS`

---

## 7. 删表规范

```ruby
# db/post_migrate/20260108044513_drop_imap_sync_logs.rb
class DropImapSyncLogs < ActiveRecord::Migration[8.0]
  def change
    drop_table :imap_sync_logs, if_exists: true
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

- 必须放在 `post_migrate/`
- `if_exists: true` 提高幂等性
- 无法回滚时 `down` 方法 `raise ActiveRecord::IrreversibleMigration`

---

## 8. 数据迁移：使用 SQL 而非 AR 模型

```ruby
class PopulateImageQualitySetting < ActiveRecord::Migration[8.0]
  def up
    execute(<<~SQL)
      INSERT INTO site_settings (name, data_type, value, created_at, updated_at)
      SELECT 'image_quality', 7,
        CASE
          WHEN recompress_quality BETWEEN 0 AND 60 THEN '50'
          ELSE '90'
        END,
        NOW(), NOW()
      FROM (SELECT COALESCE(CAST(...) AS INTEGER), 90) AS recompress_quality) src
      ON CONFLICT (name) DO UPDATE SET value = EXCLUDED.value;
    SQL
  end

  def down
    execute("DELETE FROM site_settings WHERE name = 'image_quality';")
  end
end
```

**原则：**
- 数据迁移用原生 SQL，不用 AR 模型（模型可能已变化）
- `ON CONFLICT ... DO UPDATE` 实现幂等性
- `up` 和 `down` 分别实现，`down` 要能回滚数据

---

## 9. SafeMigrate 保护机制

`lib/migration/safe_migrate.rb` 在非 production 环境的常规迁移中拦截：

| 危险操作 | 拦截方式 | 正确替代 |
|----------|----------|----------|
| `DROP TABLE` | 抛出异常 | 放到 `post_migrate/` |
| `RENAME TABLE` | 抛出异常 | 放到 `post_migrate/` |
| `DROP COLUMN` | 抛出异常 | 先 `ignored_columns`，再 `post_migrate/` |
| `RENAME COLUMN` | 抛出异常 | 同上 |
| 未先 DROP 就 `CREATE INDEX CONCURRENTLY` | 抛出异常 | 先 `remove_index if_exists: true` |

---

## 10. 查询性能规范（来自 CLAUDE.md）

- 用 `explain` 分析慢查询
- 查询时指定具体列，避免 `SELECT *`
- 使用战略性索引（联合索引、部分索引）
- 计数用 `counter_cache`，避免 `COUNT(*)` 全表扫描

---

## 快速参考

```ruby
# 大表加索引（必须 CONCURRENTLY）
disable_ddl_transaction!
remove_index :table, :col, if_exists: true
add_index :table, :col, algorithm: :concurrently

# 删列两步：
# 1. Model: self.ignored_columns += ["col"]
# 2. post_migrate: Migration::ColumnDropper.execute_drop(:table, [:col])

# 删表
# post_migrate: drop_table :table, if_exists: true

# 不可回滚时
def down
  raise ActiveRecord::IrreversibleMigration
end
```
