# 工具链命令规范

## 概述
Discourse 有一套标准工具链，所有开发操作都通过 `bin/` 目录下的脚本完成，
而不是直接调用 `bundle exec` 或 `npx`。

## 来源验证
- `discourse/CLAUDE.md`（官方指引）
- `discourse/bin/` 目录

---

## 核心命令

### Ruby 测试

```bash
# 运行整个文件的 RSpec
bin/rspec spec/path/file_spec.rb

# 运行单个 example（指定行号）
bin/rspec spec/path/file_spec.rb:123

# 运行系统测试
bin/rspec spec/system/some_feature_spec.rb
```

### JavaScript 测试

```bash
# 查看帮助
bin/qunit --help

# 运行单个测试文件
bin/qunit path/to/test-file.js

# 运行整个目录
bin/qunit path/to/tests/directory
```

### Lint

```bash
# Lint 指定文件（每次改完都要执行！）
bin/lint path/to/file path/to/another/file

# 自动修复
bin/lint --fix path/to/file

# 修复最近改动的文件
bin/lint --fix --recent
```

### 包管理

```bash
# JavaScript — 使用 pnpm（不用 npm/yarn）
pnpm install
pnpm add <package>

# Ruby — 使用 bundle
bundle install
bundle exec rails ...   # 如果没有 bin/ 包装时才用
```

### 数据库迁移

```bash
# 生成新迁移（必须用这个命令，不要手动建文件）
bin/rails generate migration MigrationName

# 运行迁移
bin/rails db:migrate
```

---

## 重要原则

1. **改完必须 lint**：`CLAUDE.md` 明确要求，每次改动后执行 `bin/lint`
2. **用 `bin/` 而非 `bundle exec`**：项目提供了封装脚本
3. **JS 用 pnpm**：不要用 npm 或 yarn（`.npmrc` 配置了 pnpm）

---

## 关联规范
- `ruby/05_rspec_testing.md` — 测试规范
