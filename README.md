# Discourse Dev Skills — 开发规范知识库

> 基于 Discourse 真实项目代码多处比对学习，整理成可复用的编程开发规范。

---

## 📂 目录结构

```
discoursedevskills/
├── README.md                    ← 本文件：总览 + 学习规范
├── LEARNING_LOG.md              ← 学习进度记录（每次学习后更新）
│
├── ruby/                        ← Ruby / Rails 后端规范
│   ├── 01_service_objects.md
│   ├── 02_models_concerns.md
│   ├── 03_controllers.md
│   ├── 04_serializers.md
│   ├── 05_rspec_testing.md
│   └── 06_migrations_db.md
│
├── javascript/                  ← JavaScript / Ember 前端规范
│   ├── 01_component_gjs.md
│   ├── 02_services.md
│   ├── 03_models_adapters.md
│   ├── 04_routes_controllers.md
│   └── 05_qunit_testing.md
│
├── system_specs/                ← 系统测试 / Page Objects
│   ├── 01_page_objects.md
│   └── 02_system_spec_patterns.md
│
├── tooling/                     ← 工具链规范
│   ├── 01_commands.md
│   └── 02_linting_formatting.md
│
└── patterns/                    ← 通用架构模式
    ├── 01_service_base_pattern.md
    └── 02_api_design.md
```

---

## 📏 Skills 整理规范

每个 skill 文件必须遵守以下结构：

### 文件命名
- 使用 `序号_主题名称.md` 格式，如 `01_service_objects.md`
- 序号确保阅读顺序

### 文件内部结构（强制）

```markdown
# [主题名称]

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
```

### 质量标准
1. **多处验证**：每条规范必须在代码库中找到 ≥2 处实例
2. **代码优先**：用真实代码说话，不写空洞描述
3. **精简原则**：每个文件聚焦一个主题，不超 200 行
4. **可操作性**：读完能直接用，不是理论

---

## 🗂 学习方法

1. 选定主题 → 在代码库中搜索相关文件
2. 阅读 2-4 个典型实现，比对异同
3. 提炼规律，记录到对应 skill 文件
4. 更新 `LEARNING_LOG.md` 打卡
5. 下次继续前先看 LEARNING_LOG 回顾进度

---

## 📌 当前学习重点（按优先级）

- [ ] Ruby Service Objects（Service::Base 模式）
- [ ] Glimmer Component (.gjs) 规范
- [ ] Page Objects 系统测试模式
- [ ] Model Concerns 模块化
- [ ] RSpec + fab! 测试规范
- [ ] ActiveRecord 性能规范
