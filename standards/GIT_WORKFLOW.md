# Git 工作流规范

> 适用于 discoursedevskills 仓库的所有提交行为。
> 无论是主动学习、经验捕获还是文档迭代，统一遵守本规范。
> **开发者只需专注任务，提交由 AI 按本规范执行，无需手动构造 message。**

---

## Commit 类型速查

| 类型 | 触发场景 | 示例 |
|------|---------|------|
| `docs` | 新增或修改主知识文档（plugins/ruby/javascript/等） | `docs(plugins/30): add withPluginApi advanced patterns` |
| `exp` | 新增 experience 条目（mistakes/decisions/inbox） | `exp(mistakes): guardian scope nil in serializer context` |
| `refactor` | 将 experience 内容升级写入主文档 | `refactor(plugins/27): incorporate guardian scope lesson` |
| `feat` | 新增整块知识（新文件） | `feat(ruby/14): add ActionMailer plugin mailer patterns` |
| `fix` | 修正已有主文档的错误或遗漏 | `fix(javascript/08): correct @data POJO constraint description` |
| `chore` | 更新 README/TOPIC_MAP/LEARNING_LOG 等元数据 | `chore: update TOPIC_MAP + LEARNING_LOG after session` |
| `wip` | 草稿阶段，源码已读但未完成验证 | `wip: plugins/30 draft — pending second source verification` |
| `standards` | 修改 standards/ 目录下的规范文件本身 | `standards: add inbox upgrade criteria to EXPERIENCE_GUIDE` |

---

## Scope 命名规则

```
docs(plugins/30)          ← 文件路径（不含 .md）
exp(mistakes)             ← experience 子目录名
exp(decisions)            ← experience 子目录名
refactor(ruby/08)         ← 被升级的目标文件路径
feat(javascript/09)       ← 新文件路径
fix(patterns/02)          ← 被修正的文件路径
chore                     ← 无需 scope
standards                 ← 无需 scope
```

---

## Commit Message 格式

```
<type>(<scope>): <简明描述（英文，动宾结构）>
```

**规则：**
- 描述用英文，动宾结构，不超过 72 字符
- 不加句号
- scope 小写，使用 `/` 分隔路径层级

**正确示例：**
```bash
docs(plugins/27): add two-pattern search filter registration
exp(mistakes): guardian variable scope error in search context
exp(decisions): chose DB.exec over ActiveRecord for bulk update
refactor(plugins/27): incorporate guardian scope distinction from exp
fix(javascript/08): correct FormKit @data snapshot behavior note
chore: update README directory + TOPIC_MAP after session
wip: ruby/14 draft — ActionMailer base class pattern read, pending plugin example
```

**错误示例：**
```bash
update files                    ← 无类型无描述
feat: done                      ← 太模糊
docs(plugins/30): 新增文件。    ← 中文 + 有句号
```

---

## 会话级提交节奏

每次工作会话按以下节奏提交，**不允许积攒多个任务后一次性提交**：

```
1. 源码阅读完成，草稿未验证    → wip commit
2. 验证通过，文件完整          → docs/feat commit
3. 开发中发现问题，写入经验     → exp commit
4. 升级 experience 到主文档   → refactor commit（同时在原 exp 文件标注已升级）
5. 会话结束，更新元数据        → chore commit
```

**会话结束固定命令（AI 执行）：**
```bash
git add -A && git diff --cached --stat && git commit -m "chore: update README + TOPIC_MAP + LEARNING_LOG after session"
```

---

## 提交前检查清单（AI 必须执行）

```bash
# 1. 确认暂存内容与 message 一致
git diff --cached --stat

# 2. 确认没有遗漏文件
find . -name "*.md" | grep -v ".git" | sort

# 3. 提交
git add -A && git commit -m "<type>(<scope>): <描述>"
```

---

## 禁止行为

- ❌ 积攒多次修改后用一个 commit 提交（应按节奏拆分）
- ❌ 将 docs/exp/refactor 混在同一个 commit
- ❌ commit message 使用中文或过于模糊的描述
- ❌ 在未检查 `git diff --cached --stat` 前提交
- ❌ wip 状态的文件在未完成验证前用 `docs` 类型提交
