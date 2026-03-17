# Developer Guide — 开发者协议

> **本文件是开发者角色的完整执行协议。**
> AI 收到开发任务后，按本文件的顺序执行，无需等待人类逐步指示。
> 人类只需给出任务描述，AI 自主完成从任务解析到经验记录的全流程。

---

## 总体流程

```
任务描述
   │
   ▼
[Step 1] 任务解析        → 拆解功能模块，识别技术领域
   │
   ▼
[Step 2] 技能加载        → 查 SKILL_INDEX，按依赖顺序读取最小必要技能集
   │
   ▼
[Step 3] 实现规划        → 向人类确认实现方案（一次），不反复询问
   │
   ▼
[Step 4] 分阶段实现      → 每完成一个阶段，明确说明依据哪个技能文件的哪个章节
   │
   ▼
[Step 5] Git 操作        → 按 GIT_WORKFLOW.md 创建 issue、分支、提交、PR
   │
   ▼
[Step 6] 经验捕获        → 按 EXPERIENCE_GUIDE.md 记录偏差和决策
```

---

## Step 1：任务解析协议

收到任务描述后，AI 必须在开始任何实现之前，完成以下解析并**输出解析结果给人类确认**：

### 1.1 功能拆解

将任务描述拆解为以下维度：

```
实体与数据
  └─ 这个功能会创建/读取/修改哪些数据？需要新表还是自定义字段？

API 与路由
  └─ 是否需要新的后端端点？前端路由？

权限
  └─ 谁能访问这个功能？需要什么权限检查？

UI
  └─ 影响哪些界面？Admin 后台 / 用户前台 / 两者？

异步操作
  └─ 是否有定时任务、后台操作、事件监听？

通知与反馈
  └─ 是否需要发送通知？实时更新？

设置与配置
  └─ 是否有可配置项？需要国际化？
```

### 1.2 输出格式（AI 必须按此格式输出解析结果）

```
## 任务解析

**功能摘要**：[一句话]

**技术模块**：
- 数据层：[描述，或"无"]
- API 层：[描述，或"无"]
- 权限：[描述]
- 前端 Admin：[描述，或"无"]
- 前端用户：[描述，或"无"]
- 异步/事件：[描述，或"无"]
- 通知：[描述，或"无"]
- 设置/i18n：[描述，或"无"]

**待加载技能**：
- Phase 1（必读）：plugins/01_structure → plugins/01_rb_and_backend → plugins/02_api_frontend → patterns/01_architecture
- Phase 2（按需）：[列出需要的模块及文件]
- Phase 3（横切）：[如需测试则列出]

**实现顺序草案**：
1. [阶段名称]：[依赖的技能文件]
2. ...

**需要确认的不确定点**：[如有，列出；如无，写"无"]
```

> ⚠️ 输出解析结果后，**等待人类确认或修正**，再进入 Step 2。
> 如果人类说"直接开始"，则视为确认，立即进入 Step 2。


---

## Step 2：技能加载协议

### 2.1 加载规则

1. 打开 `standards/SKILL_INDEX.md`
2. 先加载 Phase 1（4 个基础文件），全部读完
3. 根据 Step 1 解析结果，从 Phase 2 表中选取对应模块，**按表中的阅读顺序**依次读取
4. 如需测试，加载 Phase 3

**渐进式披露原则**：
- 不要一次性加载所有 60 个文件
- 每个阶段只加载该阶段需要的文件
- 读完一个文件后，提取"快速参考"和"反模式"章节写入工作内存，正文可以不全记

### 2.2 技能读取时的注意点

每读完一个技能文件，AI 必须在内部确认：
- [ ] 了解该技能的边界（"边界声明"章节）
- [ ] 记住该技能的反模式（"反模式"章节）
- [ ] 注意与其他技能文件的交叉点（"关联规范"章节）

如果两个技能文件在某点上有矛盾，**以更新的修订历史日期的版本为准**，并将此矛盾记入 `experience/inbox/`。

---

## Step 3：实现规划

加载完技能后，向人类输出实现计划：

```
## 实现计划

**已读技能**：[列出读取的文件]

**实现阶段**：

### 阶段 1：[名称，如"插件骨架"]
- 创建文件：[列出]
- 依据：[技能文件 + 具体章节]
- 预期产出：[描述]

### 阶段 2：[名称，如"数据层"]
- 创建/修改文件：[列出]
- 依据：[技能文件 + 具体章节]
- 预期产出：[描述]

...（以此类推）

**需要的 Discourse 版本信息**：[如有特定版本依赖]
**潜在风险点**：[基于技能文件中的反模式章节列出]
```

> 等待人类确认后，进入 Step 4 开始实现。

---

## Step 4：分阶段实现

### 4.1 执行规则

- **逐阶段推进**：完成一个阶段 → 明确说明完成 → 进入下一阶段
- **引用透明**：每写一个关键实现，说明依据哪个技能文件的哪个章节
- **偏差即记录**：如果实际实现与技能文件描述不符（无论原因），立即记入 `experience/inbox/`，不要等到会话结束

### 4.2 阶段完成确认格式

每完成一个实现阶段，输出：

```
## ✅ 阶段 N 完成：[阶段名称]

**完成内容**：
- [文件名]：[实现了什么]

**依据技能**：
- [技能文件路径] → [章节名]：[应用了什么规范]

**偏差记录**：
- [如有偏差，说明原因] 或 "无偏差"

**下一阶段**：[阶段名称]，即将开始...
```

### 4.3 遇到不确定时的处理

- 技能文件中有明确规范 → 直接按规范执行，无需询问
- 技能文件中没有覆盖 → 说明"SKILL_INDEX 未覆盖此点，按通用 Rails/Ember 最佳实践处理"，并记入 `experience/inbox/`
- 存在多种可选方案 → 说明各方案的权衡，**一次性**给出建议，等待人类选择


---

## Step 5：Git 操作

完整流程见 `standards/GIT_WORKFLOW.md` 三节（开发者会话）。

**快速索引**：

```bash
# 启动
git checkout main && git pull origin main
gh issue create --title "Dev: <插件名> — <功能描述>" --label "development"
git checkout -b dev/<issue-number>-<plugin-slug>

# 开发中（每完成一个阶段提交）
git add <相关文件>
git commit -m "feat(dev/<plugin>): <阶段描述>"

# 记录 experience
git add experience/
git commit -m "exp(mistakes|decisions): <描述>"

# 收尾
git push -u origin dev/<branch>
gh pr create --title "Dev: <完整描述>" --body "closes #<issue>" --base main
gh pr merge --squash --delete-branch
git checkout main && git pull origin main
```

---

## Step 6：经验捕获

会话结束前，AI 必须回顾：

**触发 mistakes/ 记录的情形：**
- 遇到 bug，找到根因
- 技能文件描述与实际 Discourse 行为不符
- 花了超过两轮对话才解决的问题

**触发 decisions/ 记录的情形：**
- 在两种技术方案之间做了选择（有理由）
- 选择了技能文件未覆盖的方案

**触发 inbox/ 记录的情形：**
- 观察到某个规律但尚未确认
- 对技能文件某个描述有疑问但未验证

完整格式见 `standards/EXPERIENCE_GUIDE.md`。

---

## 附录：开发者会话的时间分配建议

| 阶段 | 预期占比 | 说明 |
|------|---------|------|
| Step 1 任务解析 | ~5% | 快速，不要过度分析 |
| Step 2 技能加载 | ~15% | 读技能文件是投资，节省后续试错 |
| Step 3 规划确认 | ~5% | 一次确认，不反复 |
| Step 4 实现 | ~65% | 核心工作 |
| Step 5 Git 操作 | ~5% | 自动化，不占思考时间 |
| Step 6 经验捕获 | ~5% | 必须执行，不可省略 |

---

## 附录：常见任务的技能加载快速参考

### "给现有 topic 增加一个自定义字段，并在前端显示"
```
Phase 1（必读）→
patterns/04_custom_fields →
plugins/21_database_operations →
plugins/22_serializer_extensions →
plugins/26_preloading →
javascript/01_component_gjs（如需前端组件）
```

### "创建一个带 Admin 管理界面和 REST API 的全栈插件"
```
Phase 1（必读）→
ruby/06_migrations_db → plugins/10_plugin_model_and_db →
ruby/08_guardian → plugins/23_routes_and_guardian →
ruby/04_serializers → plugins/22_serializer_extensions →
plugins/09_admin_ui → plugins/15_admin_multi_page →
plugins/20_admin_rest_model_adapter →
plugins/03_plugin_settings → plugins/24_i18n_and_locale
```

### "监听用户行为事件，发送通知，记录日志"
```
Phase 1（必读）→
plugins/18_on_event_system →
ruby/09_notification →
ruby/07_jobs（如需异步）→
plugins/21_database_operations（如需持久化日志）
```
