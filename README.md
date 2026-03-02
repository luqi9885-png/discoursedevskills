# Discourse Dev Skills — 开发规范知识库

> 基于 Discourse 真实项目代码多处比对学习，整理成可复用的编程开发规范。
> 源码目录：`/Users/qilu/Desktop/develop/discourse/`

---

## 🚀 每次启动的标准流程

> **记忆可能丢失，每次必须先执行以下步骤再开始任何工作。**

1. **读 LEARNING_LOG.md**：确认上次进度、已完成项、待开始项
2. **用 git log 验证**：`git log --oneline -5` 确认上次 commit 内容与 LOG 一致
3. **完成新任务**：阅读源码 → 整理到对应 md 文件
4. **更新 LEARNING_LOG.md**：填写本次学习内容、验证来源、输出文件
5. **git commit**：每完成一个主题就立即 commit，不要积攒

```bash
# 每完成一个主题立即 commit
git add -A
git commit -m "feat: 完成 ruby/08_guardian.md — Guardian权限系统"

# 若只更新进度记录
git add LEARNING_LOG.md README.md
git commit -m "chore: 更新进度记录，标记guardian完成，下一步routes"
```

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

> 如有规范性、原则性的修改建议，AI 应提出建议并等待人工确认，不得自行写入。

---

## 📂 目录结构

> ✅ 已完成 | 🔲 待开始。以此为准，不要依赖记忆。

```
discoursedevskills/
├── README.md                        ← 本文件：总览 + 学习规范 + 工作流
├── LEARNING_LOG.md                  ← 学习进度记录（每次必读必写）
│
├── ruby/                            ← Ruby / Rails 后端规范
│   ├── 01_service_objects.md        ✅ Service::Base DSL
│   ├── 02_models_concerns.md        ✅ ActiveSupport::Concern 模式
│   ├── 03_controllers.md            ✅ ApplicationController 规范
│   ├── 04_serializers.md            ✅ 三层继承 + 条件属性
│   ├── 05_rspec_testing.md          ✅ fab! + Fabrication
│   ├── 06_migrations_db.md          ✅ 迁移安全规范 + SafeMigrate
│   ├── 07_jobs.md                   ✅ Regular / Scheduled Jobs
│   └── 08_guardian.md               ✅ Guardian 权限系统
│
├── javascript/                      ← JavaScript / Ember 前端规范
│   ├── 01_component_gjs.md          ✅ Glimmer Component (.gjs)
│   ├── 02_services.md               ✅ Ember Service 单例模式
│   ├── 03_routes.md                 ✅ Ember Routes
│   └── 04_models.md                 ✅ Ember Store 与 Model
│   └── 05_qunit_testing.md          ✅ Unit + Integration 测试
│
├── system_specs/                    ← 系统测试 / Page Objects
│   ├── 01_page_objects.md           ✅ Page Objects 系统测试
│   └── 02_system_spec_patterns.md   ✅ 系统测试通用模式
│
├── plugins/                         ← Plugin 开发规范
│   ├── 01_plugin_structure.md       ✅ Plugin 目录结构
│   ├── 01_plugin_rb_and_backend.md  ✅ plugin.rb + 后端扩展
│   ├── 02_plugin_frontend.md        ✅ 前端扩展
│   ├── 02_plugin_api_frontend.md    ✅ Plugin API
│   └── 03_plugin_settings.md        ✅ Plugin 设置
│
├── patterns/                        ← 通用架构模式
│   ├── 01_plugin_architecture.md    ✅ Plugin 架构模式
│   └── 02_full_plugin_flow.md       ✅ 完整 Plugin 开发流程
│
└── tooling/                         ← 工具链规范
    ├── 01_commands.md               ✅ 常用开发命令
    └── 02_linting_formatting.md     ✅ Lint + 格式化规范
```

---

## 📋 待完成任务（按优先级）

| 优先级 | 输出文件 | 主题 | 源码参考目录 |
|--------|----------|------|-------------|
（已完成全部规划目标，待制定下一批）

---

## 📏 Skills 整理规范

每个 skill 文件必须遵守以下结构：

### 文件命名
- 使用 `序号_主题名称.md` 格式，如 `01_service_objects.md`
- 序号确保阅读顺序

### 文件内部结构（强制）

```markdown
# [主题名称]

> 来源：`源码路径`

## 概述
一句话说明这个规范解决什么问题。

## 来源验证
列出在真实代码中验证过的文件路径（至少 2 处）。

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
1. **多处验证**：每条规范必须在代码库中找到 ≥2 处实例
2. **代码优先**：用真实代码说话，不写空洞描述
3. **精简原则**：每个文件聚焦一个主题，不超 500 行
4. **可操作性**：读完能直接用，不是理论

---

## 🗂 学习方法

1. 选定主题 → 在代码库中搜索相关文件
2. 阅读 2-4 个典型实现，比对异同
3. 提炼规律，记录到对应 skill 文件
4. 更新 `LEARNING_LOG.md` 打卡
5. 下次继续前先看 LEARNING_LOG 回顾进度

---

## 🔧 工具链速查

```bash
# 查看进度
cd /Users/qilu/Desktop/develop/discoursedevskills
git log --oneline -10
cat LEARNING_LOG.md

# 搜索源码（Ruby）
grep -r "pattern" /Users/qilu/Desktop/develop/discourse/app/ --include="*.rb"
ls /Users/qilu/Desktop/develop/discourse/lib/guardian/

# 搜索源码（JavaScript）
ls /Users/qilu/Desktop/develop/discourse/frontend/discourse/app/services/
ls /Users/qilu/Desktop/develop/discourse/frontend/discourse/app/routes/
```
