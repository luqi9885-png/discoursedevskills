# Git 工作流规范（高级工程师标准）

> 适用于 discoursedevskills 仓库。
> **AI 完整自主执行本规范，人类只需给出任务目标。**
> 规范版本：v2.0 | 更新：2026-03-18

---

## 前置条件：gh 认证

所有 gh 操作需要先完成认证（一次性，人工执行）：

```bash
gh auth login
# 选择 GitHub.com → HTTPS → 浏览器认证
gh auth status   # 验证认证状态
```

---

## 一、分支策略

### 分支类型与用途

| 分支前缀 | 用途 | 生命周期 | 示例 |
|---------|------|---------|------|
| `main` | 稳定主干，只接受合并，不直接提交 | 永久 | `main` |
| `learn/` | 主动学习会话（学习者角色） | 会话结束后合并删除 | `learn/plugin-serializer-extensions` |
| `dev/` | 插件开发任务（开发者角色） | 任务结束后合并删除 | `dev/my-plugin-name` |
| `fix/` | 修正主文档错误，关联 issue | 修复完成后合并删除 | `fix/42-guardian-scope-description` |
| `exp/` | 实验性探索，不保证合并 | 探索结束后评估 | `exp/webhook-pattern-research` |
| `hotfix/` | 紧急修复主干文档严重错误 | 合并后立即删除 | `hotfix/wrong-api-method-name` |

### 核心规则

- **`main` 受保护**：不允许直接 `git push origin main`，所有变更通过 PR 合并
- **一个任务一个分支**：不在同一分支上混合不同主题
- **分支从 `main` 切出**：`git checkout main && git pull && git checkout -b <branch>`
- **合并后必须删除分支**：本地和远程都删除


---

## 二、Commit 规范

### 类型速查

| 类型 | 触发场景 |
|------|---------|
| `feat` | 新增完整知识文件（首次创建） |
| `docs` | 修改或补充已有主文档内容 |
| `exp` | 新增 experience/ 条目 |
| `refactor` | experience 升级写入主文档 |
| `fix` | 修正主文档错误（关联 issue） |
| `chore` | 更新 README/TOPIC_MAP/LEARNING_LOG 等元数据 |
| `wip` | 草稿阶段，未完成验证 |
| `standards` | 修改 standards/ 规范文件本身 |

### Message 格式

```
<type>(<scope>): <动宾描述，英文，≤72字符>

[可选 body：补充说明，关联 issue]

closes #<issue-number>
```

**Scope 规则：**
```
feat(ruby/14)             ← 文件路径，不含 .md
docs(plugins/27)          ← 同上
exp(mistakes)             ← experience 子目录名
refactor(ruby/08)         ← 被升级的目标文件
fix(javascript/08)        ← 被修正的文件
chore                     ← 无 scope
standards                 ← 无 scope
```

**正确示例：**
```
feat(plugins/30): add withPluginApi advanced hook patterns

Covers: apiInitializer v2, registerBehavior, destroyable.
Verified against discourse-assign and discourse-solved.

closes #15
```

```
exp(mistakes): guardian scope nil when used outside request context

fix(javascript/08): correct @data POJO constraint — Model instances fail silently
```

**禁止：**
```
update files          ← 无类型
feat: done            ← 无意义描述
docs: 新增文件。       ← 中文 + 句号
```


---

## 三、完整会话工作流

### 学习者会话（learn/ 分支）

#### 启动（AI 执行）

```bash
# 1. 同步主干
git checkout main
git pull origin main

# 2. 创建 issue（追踪本次学习目标）
gh issue create \
  --title "Learn: <主题名称>" \
  --body "目标：整理 <主题> 规范到 <文件路径>
验证来源：<源码目录>
预期输出：<文件路径>" \
  --label "learning"

# 记录返回的 issue number，例如 #16

# 3. 创建分支（分支名用 issue 号前缀）
git checkout -b learn/16-plugin-serializer-extensions

# 4. 验证起点
git log --oneline -3
```

#### 开发中（按节奏提交，不积攒）

```bash
# 草稿阶段（源码读完，未完成验证）
git add <文件>
git commit -m "wip(plugins/30): draft serializer extensions — pending second verification"

# 完成阶段（验证通过）
git add <文件>
git commit -m "feat(plugins/30): add serializer extension patterns

Covers add_to_serializer, preloaded_custom_fields, scope conditional.
Verified against discourse-assign (2 locations) and discourse-solved.

closes #16"
```

#### 收尾（AI 执行）

```bash
# 1. 更新元数据
git add README.md TOPIC_MAP.md LEARNING_LOG.md
git commit -m "chore: update README + TOPIC_MAP + LEARNING_LOG for plugins/30"

# 2. 推送分支
git push -u origin learn/16-plugin-serializer-extensions

# 3. 创建 PR
gh pr create \
  --title "Learn: add plugins/30 serializer extension patterns" \
  --body "closes #16

## 本次完成
- 新增 plugins/30_xxx.md
- 验证来源：2处独立源码

## TOPIC_MAP 更新
- 已登记新主题边界" \
  --base main

# 4. 合并 PR（squash 保持主干整洁）
gh pr merge --squash --delete-branch

# 5. 同步本地 main
git checkout main
git pull origin main
```

---

### 开发者会话（dev/ 分支）

#### 启动（AI 执行）

```bash
# 1. 同步主干
git checkout main && git pull origin main

# 2. 创建 issue
gh issue create \
  --title "Dev: <插件名> — <任务简述>" \
  --body "插件：<名称>
任务：<具体功能>
参考 skills：<相关文件路径>" \
  --label "development"

# 3. 创建分支
git checkout -b dev/<issue-number>-<plugin-name>
# 示例：git checkout -b dev/17-discourse-badge-extension
```

#### 开发中

```bash
# 功能完成节点提交（不等到全部完成）
git add <相关文件>
git commit -m "feat(dev/my-plugin): implement custom badge award endpoint"

# 发现问题，立即记录 experience
git add experience/mistakes/<date>_<slug>.md
git commit -m "exp(mistakes): serializer scope nil when topic_view missing preload"
```

#### 收尾（AI 执行）

```bash
# 1. 经验升级评估（见 EXPERIENCE_GUIDE.md 第五节）
# 如有升级：
git add plugins/xx.md
git commit -m "refactor(plugins/22): add preload anti-pattern from exp/mistakes"
git add experience/mistakes/<file>.md
git commit -m "exp(mistakes): mark preload-nil as upgraded to plugins/22"

# 2. 推送 + PR
git push -u origin dev/17-discourse-badge-extension
gh pr create \
  --title "Dev: discourse-badge-extension task complete" \
  --body "closes #17

## 完成内容
- 实现了 xxx
- Experience 条目：2 个 mistakes，1 个 decision

## 升级到主文档
- plugins/22 补充了预加载反模式" \
  --base main

# 3. 合并
gh pr merge --squash --delete-branch
git checkout main && git pull origin main
```


---

## 四、gh 常用操作速查

```bash
# Issue 管理
gh issue create --title "..." --body "..." --label "learning|development|bug"
gh issue list                         # 查看所有 open issues
gh issue list --label "development"   # 按标签过滤
gh issue close <number>               # 手动关闭（commit 中 closes #N 自动关闭）
gh issue view <number>                # 查看 issue 详情

# PR 管理
gh pr create --title "..." --body "..." --base main
gh pr list                            # 查看所有 open PRs
gh pr view <number>                   # 查看 PR 详情
gh pr merge <number> --squash --delete-branch   # squash 合并，推荐
gh pr merge <number> --merge --delete-branch    # 普通合并（保留所有 commit）
gh pr close <number>                  # 关闭 PR（不合并）

# Label 管理（首次使用需创建）
gh label create "learning" --color "0075ca" --description "主动学习会话"
gh label create "development" --color "e4e669" --description "插件开发任务"
gh label create "experience" --color "d93f0b" --description "经验升级"
gh label create "bug" --color "ee0701" --description "文档错误"

# 仓库状态
gh repo view                          # 查看仓库概览
gh run list                           # 查看 Actions 运行记录（如有配置）
```

---

## 五、异常处理

### 修正最近一次 commit（未 push）

```bash
# 修改文件后，追加到上次 commit
git add <文件>
git commit --amend --no-edit          # 保留原 message
git commit --amend -m "新 message"   # 修改 message
```

### 撤销最近一次 commit（保留文件变更）

```bash
git reset HEAD~1                      # 撤销 commit，文件改动保留在工作区
```

### 分支落后 main（需要同步）

```bash
git fetch origin
git rebase origin/main                # 推荐：保持线性历史
# 如有冲突 → 解决冲突 → git add . → git rebase --continue
```

### 临时中断：保存现场

```bash
git stash push -m "wip: 暂存 plugins/30 草稿"
# ... 处理其他事情 ...
git stash list                        # 查看暂存列表
git stash pop                         # 恢复最近一次暂存
```

### 误提交到 main（未 push）

```bash
# 创建正确分支，保留所有 commit
git checkout -b learn/correct-branch-name
git checkout main
git reset --hard origin/main          # main 回滚到远程状态
```

### PR 合并后清理本地残留分支

```bash
git fetch --prune                     # 清理已删除的远程分支引用
git branch -d learn/16-xxx            # 删除本地分支
```

---

## 六、push 时机规范

| 场景 | 操作 |
|------|------|
| 会话进行中，仅本地提交 | ✅ 正常，不强制 push |
| 准备创建 PR | `git push -u origin <branch>` **必须在 gh pr create 之前** |
| 会话中断，需要保存远程备份 | `git push origin <branch>` |
| 修正已 push 的分支（amend/rebase）| `git push --force-with-lease origin <branch>` ⚠️ **仅限个人特性分支，永远不对 main 使用** |
| 主干同步 | 通过 PR merge 后 `git pull origin main`，**不手动 push main** |

---

## 七、每日状态检查（AI 启动时必执行）

```bash
# 完整状态检查（一条命令）
git fetch origin && \
git log --oneline --graph --all -10 && \
echo "---" && \
git status && \
echo "---" && \
gh issue list --limit 10
```

输出解读：
- `* main` 后有 `origin/main`：本地主干与远程同步
- 有 `ahead`：本地有未推送 commit
- 有 `behind`：远程有更新，需要 pull
- Issue 列表：确认当前进行中的任务

---

## 八、禁止行为

- ❌ `git push origin main`（直接推送主干）
- ❌ `git push --force origin main`（强制推送主干）
- ❌ 在 `main` 分支上直接写文件和提交
- ❌ 积攒多次修改一次提交（应按节奏提交）
- ❌ 不创建 issue 直接开始工作（无法追踪，无法自动关闭）
- ❌ PR 合并后不删除分支（分支积累导致混乱）
- ❌ 将 docs/exp/refactor 混在同一 commit
- ❌ commit message 使用中文或过于模糊的描述
