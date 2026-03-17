# Discourse Dev Skills — 开发规范知识库

> 基于 Discourse 真实项目代码多处比对学习，整理成可复用的编程开发规范。
> 源码目录：`/Users/qilu/Desktop/develop/discourse/`

---

## 👤 角色选择

本仓库服务两种 AI 角色，进入任务前先确认角色：

| 角色 | 任务 | 入口 |
|------|------|------|
| **学习者** | 主动学习 Discourse 源码，整理结构化知识到主文档 | 👇 启动流程（学习者） |
| **开发者** | 执行外部插件开发任务，读取 skills 参考，记录经验 | 👇 启动流程（开发者）|

> Git 规范：`standards/GIT_WORKFLOW.md`
> 经验编写规范：`standards/EXPERIENCE_GUIDE.md`

---

## 🚀 每次开始前（启动流程 — 学习者）

> **记忆可能丢失，每次必须先执行以下步骤再开始任何工作。**

1. **读 LEARNING_LOG.md**：确认上次进度、已完成项、待开始项
2. **用 git log 验证**：`git log --oneline -5` 确认上次 commit 内容与 LOG 一致
3. **内容核验**：对上次 WIP 未完成、或本次要修改的文件，执行：
   ```bash
   head -5 plugins/xx_xxx.md   # 读首行标题
   ```
   与 TOPIC_MAP 中该文件的「内容摘要」比对，确认文件实际内容与登记一致。
   发现不符 → **优先修复，再开始新任务**
4. **读 TOPIC_MAP.md**：确认即将写的主题是否与已有文件边界冲突
5. **完成新任务**：阅读源码 → 整理到对应 md 文件

---

## 🏁 每次结束时（收尾流程 — 学习者）

> **每次任务结束必须按顺序执行，不得跳过。**

1. **核验提交内容与 message 一致**：
   ```bash
   git diff --cached --stat
   ```
   确认实际变更文件与即将写的 commit message 完全对应。
   不一致 → 拆分提交或修正 message，**不允许将就提交**

2. **核验磁盘文件与 README 目录一致**：
   ```bash
   find . -name "*.md" | grep -v README | grep -v LEARNING_LOG | grep -v TOPIC_MAP | grep -v ".git" | sort
   ```
   与 README 目录结构逐一比对（覆盖全部子文件夹）。
   磁盘有但 README 没有 → 补登记；README 有但磁盘没有 → 标记或删除条目

3. **更新 TOPIC_MAP.md**：新文件完成后，为本次新文件登记主题边界 + 内容摘要（首行标题 + 主要章节关键词）

4. **更新 LEARNING_LOG.md**：按格式填写完整（状态、验证来源、关联更新三项不得省略），填写本次学习内容、验证来源、输出文件

5. **git commit**：按阶段提交，不要积攒（见提交规范），必须检验提交内容和提交信息是否正确

```bash
# WIP 阶段（源码读完，文件草稿未验证完）
git add -A
git commit -m "wip: 草稿 plugins/16_topic_list — 源码阅读完，待第二处验证"

# 完成阶段（两处验证通过，文件完整）
git add -A
git commit -m "feat: 完成 plugins/16_topic_list.md — topic-list-item-class"

# 修改已有文件
git add -A
git commit -m "fix: 补充 ruby/08_guardian.md — 新增与 register_modifier 组合的反模式"

# 仅更新进度记录
git add LEARNING_LOG.md README.md TOPIC_MAP.md
git commit -m "chore: 更新进度记录，标记 16 完成，下一步 17_notification"
```

> 完整 Git 类型速查见 `standards/GIT_WORKFLOW.md`

---

## 🛠 每次开始前（启动流程 — 开发者）

> 开发者角色：只读取 skills 参考，专注执行外部开发任务，不主动学习或修改主文档。

1. **确认角色**：本次任务是外部插件开发，非主动学习
2. **读取相关 skills**：在 `plugins/` `ruby/` `javascript/` 等目录按需查阅
3. **读取 experience/**：快速浏览 `experience/mistakes/` 和 `experience/decisions/`，避免重复踩已知的坑
4. **确认 Git 规范**：`standards/GIT_WORKFLOW.md`（仅需读一次，后续 AI 自动执行）
5. **开始任务**：专注实现功能，遇到问题随时记录到 `experience/inbox/`

## 🏁 每次结束时（收尾流程 — 开发者）

1. **回顾任务**：是否遇到 bug、卡点、非显而易见的决策？
2. **执行经验捕获**：读取 `standards/EXPERIENCE_GUIDE.md`，按规范写入 `experience/`
3. **评估升级**：是否有内容应升级至主文档？（EXPERIENCE_GUIDE 第五节）
4. **exp commit**：`exp(mistakes|decisions): <描述>`
5. **refactor commit**（如有升级）：`refactor(plugins/xx): <描述>`
6. **chore commit**：更新 `experience/README.md` 条目数表

> 开发者**不更新** TOPIC_MAP.md / LEARNING_LOG.md（这是学习者的职责）

---

## 📝 README 维护原则

README 是整个知识库的导航核心，修改需谨慎。

**允许自动更新（无需人工审核）：**
- 目录结构中的 ✅ / 🔲 状态标记
- 待完成任务表的内容（新增、完成、调整优先级）
- 工具链速查中的路径或命令

**必须人工审核批准才能修改：**
- 每次启动的标准流程（步骤逻辑）
- Skills 整理规范（文件结构模板、质量标准）
- 学习方法（方法论）
- README 维护原则本身

> 每次任务结束必须检查实际操作过程中readme核心规范造成的问题，然后整理分析。
如有规范性、原则性的修改建议（结合实际，做出因果推断），AI 应提出建议并等待人工确认，不得自行写入。
这可以使得学习机制不断增强

---

## 📂 目录结构

> ✅ 已完成 | 🔲 待开始。以此为准，不要依赖记忆。

```
discoursedevskills/
├── README.md                        ← 本文件：总览 + 规范 + 工作流
├── LEARNING_LOG.md                  ← 学习进度记录（学习者每次必读必写）
├── TOPIC_MAP.md                     ← 主题边界注册表（学习者每次新文件后更新）
│
├── standards/                       ← 工作规范（AI 执行，人类可参考）
│   ├── GIT_WORKFLOW.md              ← Git commit 类型、格式、节奏规范
│   └── EXPERIENCE_GUIDE.md         ← AI 经验编写指南（命名/格式/去重/升级）
│
├── experience/                      ← 开发经验捕获层（开发者角色写入）
│   ├── README.md                    ← 经验层说明 + 当前条目数
│   ├── mistakes/                    ← 踩坑记录
│   ├── decisions/                   ← 设计决策记录
│   └── inbox/                       ← 原始观察暂存（7 天内处理）
│
├── ruby/                            ← Ruby / Rails 后端规范
│   ├── 01_service_objects.md        ✅ Service::Base DSL
│   ├── 02_models_concerns.md        ✅ ActiveSupport::Concern 模式
│   ├── 03_controllers.md            ✅ ApplicationController 规范
│   ├── 04_serializers.md            ✅ 三层继承 + 条件属性
│   ├── 05_rspec_testing.md          ✅ fab! + Fabrication
│   ├── 06_migrations_db.md          ✅ 迁移安全规范 + SafeMigrate
│   ├── 07_jobs.md                   ✅ Regular / Scheduled Jobs
│   ├── 08_guardian.md               ✅ Guardian 权限系统
│   ├── 09_notification.md           ✅ Notification 系统
│   ├── 10_rate_limiter.md           ✅ RateLimiter 限流模式
│   ├── 11_post_topic_creator.md     ✅ PostCreator & TopicCreator 核心创建流程
│   ├── 12_reviewable.md             ✅ Reviewable 审核系统
│   └── 13_search.md                 ✅ Search 搜索系统
│
├── javascript/                      ← JavaScript / Ember 前端规范
│   ├── 01_component_gjs.md          ✅ Glimmer Component (.gjs)
│   ├── 02_services.md               ✅ Ember Service 单例模式
│   ├── 03_routes.md                 ✅ Ember Routes
│   ├── 04_models.md                 ✅ Ember Store 与 Model
│   ├── 05_qunit_testing.md          ✅ Unit + Integration 测试
│   ├── 06_message_bus.md            ✅ MessageBus 前端订阅规范（subscribe/unsubscribe/@bind/instance-initializer）
│   ├── 07_select_kit.md             ✅ SelectKit 组件规范（ComboBox/MultiSelect/EmailGroupUserChooser/modifySelectKit）
│   └── 08_form_kit.md               ✅ FormKit 表单框架（@data/@validation/Field 控件/form.Object/onRegisterApi）
│
├── system_specs/                    ← 系统测试 / Page Objects
│   ├── 01_page_objects.md           ✅ Page Objects 系统测试
│   └── 02_system_spec_patterns.md   ✅ 系统测试通用模式
│
├── plugins/                         ← Plugin 开发规范
│   ├── 01_plugin_structure.md       ✅ Plugin 目录结构（文件布局，命名规范）
│   ├── 01_plugin_rb_and_backend.md  ✅ plugin.rb + 后端扩展入口
│   ├── 02_plugin_frontend.md        ✅ 前端扩展
│   ├── 02_plugin_api_frontend.md    ✅ Plugin API
│   ├── 03_plugin_settings.md        ✅ Plugin 设置
│   ├── 04_value_transformer_dag.md  ✅ registerValueTransformer + DAG
│   ├── 05_modify_class.md           ✅ modifyClass 规范
│   ├── 06_tracked_post_and_render_outlet.md ✅ addTrackedPostProperties + renderAfterWrapperOutlet
│   ├── 07_modal_and_ui_components.md ✅ DModal + 自定义 Service（弹窗 UI）
│   ├── 08_user_ui_extensions.md     ✅ 用户菜单 / 偏好 / 活动页扩展
│   ├── 09_admin_ui.md               ✅ 管理后台 UI（插件页 / Report / Connector）
│   ├── 10_plugin_model_and_db.md    ✅ 插件 Model + Engine + 数据库迁移
│   ├── 11_register_modifier_backend.md ✅ register_modifier 后端管道规范
│   ├── 12_api_initializer_and_ui_injection.md ✅ apiInitializer 高级 / Composer 工具栏 / 批量操作
│   ├── 13_backend_advanced_patterns.md ✅ EntryPoint / hijack / Controller Concern / 高级 DSL
│   ├── 14_plugin_service_frontend.md ✅ 插件 Ember Service 四种模式
│   ├── 15_admin_multi_page.md       ✅ Admin 多子页面（addAdminPluginConfigurationNav）
│   ├── 16_plugin_route_map.md       ✅ 插件前端路由（route-map / Route / Template）
│   ├── 17_topic_list_columns.md     ✅ topic-list 自定义列与行样式
│   ├── 18_on_event_system.md        ✅ on() 事件系统完整规范（40+ 事件速查表）
│   ├── 19_notification_consolidation.md ✅ 通知合并（ConsolidateNotifications / DeletePreviousNotifications）
│   ├── 20_admin_rest_model_adapter.md ✅ Admin RestModel + RestAdapter 完整数据层规范
│   ├── 21_database_operations.md    ✅ DB/AR 混合操作、PluginStore、CustomFields、upsert_all
│   ├── 22_serializer_extensions.md  ✅ add_to_serializer、预加载、scope、topic_view vs topic_list_item
│   ├── 23_routes_and_guardian.md    ✅ 插件路由注册 + GuardianExtensions + ensure_can_xxx! 自动派生
│   ├── 24_i18n_and_locale.md        ✅ server/client 两套 locale 文件、复数形式、用户语言切换
│   ├── 25_plugin_testing.md         ✅ 插件 RSpec 测试（Fabricator/request spec/model spec/job spec/shared_context）
│   ├── 26_preloading.md             ✅ 预加载 N+1 防治（TopicList/TopicView/BookmarkQuery/Search.on_preload）
│   ├── 27_search_filter.md          ✅ 搜索扩展（register_search_advanced_filter/add_filter_custom_filter/addAdvancedSearchOptions）
│   ├── 28_reports.md                ✅ Admin 报告（Report.add_report 折线图/表格 + add_directory_column）
│   └── 29_plugin_api_misc.md        ✅ Plugin API 杂项（registerTopicFooterButton/addNavigationBarItem/addBulkActionButton/addKeyboardShortcut 等）
│
├── patterns/                        ← 通用架构模式
│   ├── 01_plugin_architecture.md    ✅ Plugin 架构模式
│   ├── 02_full_plugin_flow.md       ✅ 完整 Plugin 开发流程
│   ├── 03_discourse_event.md        ✅ DiscourseEvent 事件系统
│   ├── 04_custom_fields.md          ✅ Custom Fields
│   └── 05_plugin_backend_dsl.md     ✅ Plugin 后端 DSL 完整参考
│
└── tooling/                         ← 工具链规范
    ├── 01_commands.md               ✅ 常用开发命令
    └── 02_linting_formatting.md     ✅ Lint + 格式化规范
```

> ⚠️ **目录即真相**：README 的目录结构必须与磁盘文件一一对应。发现不一致时，以磁盘为准，立即修正 README。

---

## 📋 待完成任务（按优先级）

| 优先级 | 输出文件 | 主题 | 源码参考目录 |
|--------|----------|------|-------------|
| — | — | 已系统覆盖主要插件开发技能，待实战反馈补充 | — |

---

## 📏 Skills 整理规范

每个 skill 文件必须遵守以下结构：

### 文件命名
- 使用 `序号_主题名称.md` 格式，如 `01_service_objects.md`
- 序号在各自文件夹内唯一，不得重复
- 新建文件前先检查 TOPIC_MAP.md，确认主题尚未被覆盖

### 文件内部结构（强制）

```markdown
# [主题名称]

> 来源：`源码路径`
> 修订历史：
> - YYYY-MM-DD 初版
> - YYYY-MM-DD 补充 xxx（来源：xxx）

## 边界声明
- **本文件负责**：xxx（一句话）
- **不负责，见其他文件**：
  - xxx → 文件路径
  - xxx → 文件路径
- **与以下文件有交叉，已在 TOPIC_MAP 协调**：
  - 文件路径（本文只写 xxx，那边只写 yyy）

## 概述
一句话说明这个规范解决什么问题。

## 来源验证
列出在真实代码中验证过的文件路径（至少 2 处，必须来自相对独立的文件或插件）。

| 验证点 | 文件路径 | 关键行 / 方法名 | 与其他验证点的差异 |
|--------|---------|---------------|-----------------|
| 验证 1 | path/to/file.rb | `def can_xxx` | 基础用法 |
| 验证 2 | path/to/other.rb | `def can_xxx` | 差异点：使用了 scope 参数 |

## 核心规范

### [子规范 1]
说明 + 代码示例（从真实代码摘取或简化）

### [子规范 2]
...

## 反模式（避免这样做）
对比说明错误做法。

## 关联规范
指向其他 skill 文件。

## 快速参考
速查表或最常用代码片段，读完能直接用。
```

### 质量标准
1. **多处验证**：每条规范必须在代码库中找到 ≥2 处实例，来自相对独立的文件（不同插件或不同模块）
2. **验证差异记录**：若两处实现有差异，必须说明以哪个为准及原因
3. **代码优先**：用真实代码说话，不写空洞描述
4. **精简原则**：每个文件聚焦一个主题，不超 500 行
5. **可操作性**：读完能直接用，不是理论
5. **学习库验证**：随着知识增长，除了源码多处验证，还需要比对已经学习整理到学习库的相关内容


---

## 📓 LEARNING_LOG 格式规范

每次学习必须按以下格式记录，不得省略字段：

```markdown
## YYYY-MM-DD — [文件路径]

**状态**：🔄 进行中 / ✅ 完成 / ❌ 中止

**本次完成**：
- 阅读了 xxx 文件
- 整理了 xxx 规范，写入 xxx 章节

**下次从这里开始**（进行中时必填）：
- 待完成：还需验证第二处来源（目标文件：xxx）
- 待完成：反模式章节未写

**验证来源**：
- `路径 A`：第 xx 行，印证了 xxx，关键方法 `def xxx`
- `路径 B`：第 xx 行，与 A 的差异在于 xxx，以 A 为准因为 xxx

**中止原因**（中止时必填）：
- 原因说明，判断下次是否值得继续还是重来

**关联更新**：
- TOPIC_MAP.md 已更新 / 待更新
- README 目录 ✅ 已更新
```

---

## 🗺 TOPIC_MAP 格式规范

`TOPIC_MAP.md` 是防止内容重叠的核心文件，每完成一个新文件后必须更新。

```markdown
# TOPIC_MAP.md — 主题边界注册表

> 每个关键词只有一个「权威文件」。其他文件涉及相关内容时，只写本文件特有的扩展点，
> 通用逻辑一律引用权威文件，不重复。

| 主题关键词 | 权威文件（唯一） | 内容摘要（用于启动核验） | 可引用但不重复写的文件 | 边界说明 |
|-----------|----------------|----------------------|---------------------|---------|
| Guardian 权限 | ruby/08_guardian.md | Guardian 权限系统；章节：can?/ensure!/scope/register_policy | plugins/02_plugin_rb_and_backend.md | 后者只写"plugin.rb 里如何注册 guardian 扩展点"，`can?` 判断逻辑不重复 |
| Ember Service 通用 | javascript/02_services.md | Ember Service 单例模式；章节：@tracked/@service/生命周期 | plugins/14_plugin_service_frontend.md | 后者只写 plugin 特有的注册方式（`withPluginApi`），Service 本身的生命周期不重复 |
```

> 新建文件时必须先在此表登记，确认没有与已有权威文件重叠。若发现已有权威文件不够用，
> 先修改权威文件，再决定是否新建。
>
> **内容摘要格式**：`首行标题；章节：关键词1/关键词2/关键词3`
> 启动时用 `head -5 文件名` 读首行标题与此比对，发现不符立即修复。

---

## 🗂 学习方法

1. 选定主题 → 查 TOPIC_MAP.md 确认边界，避免与已有文件重叠
2. 在代码库中搜索相关文件，找 2-4 个典型实现
3. 比对异同 → 记录差异点和取舍依据
4. 整理到对应 skill 文件，填写"边界声明"和"验证来源"表格
5. 更新 TOPIC_MAP.md，登记新文件的主题边界
6. 更新 LEARNING_LOG.md 打卡
7. git commit（见提交规范）

---

## 🔧 工具链速查

```bash
# 查看进度
cd /Users/qilu/Desktop/develop/discoursedevskills
git log --oneline -10
cat LEARNING_LOG.md
cat TOPIC_MAP.md

# 搜索源码（Ruby）
grep -r "pattern" /Users/qilu/Desktop/develop/discourse/app/ --include="*.rb"
ls /Users/qilu/Desktop/develop/discourse/lib/guardian/

# 搜索源码（JavaScript）
ls /Users/qilu/Desktop/develop/discourse/frontend/discourse/app/services/
ls /Users/qilu/Desktop/develop/discourse/frontend/discourse/app/routes/

# 检查目录与 README 是否一致
find . -name "*.md" | grep -v README | grep -v LEARNING_LOG | grep -v TOPIC_MAP | sort
```