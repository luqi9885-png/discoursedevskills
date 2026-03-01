# 学习进度记录 LEARNING_LOG

> 每次学习完一个主题后在此记录，防止遗忘进度。

---

## 格式说明

```
### [日期] [主题]
- 学习内容：
- 验证来源：
- 输出文件：
- 遗留问题：
- 下次继续：
```

---

## 进度总览

| 主题 | 状态 | 输出文件 | 日期 |
|------|------|----------|------|
| 项目结构总览 & Skills体系 | ✅ 完成 | README.md | 2026-03-01 |
| Ruby Service Objects | ✅ 完成 | ruby/01_service_objects.md | 2026-03-01 |
| Glimmer Component (.gjs) | ✅ 完成 | javascript/01_component_gjs.md | 2026-03-01 |
| Page Objects | ✅ 完成 | system_specs/01_page_objects.md | 2026-03-01 |
| 工具链命令 | ✅ 完成 | tooling/01_commands.md | 2026-03-01 |
| Model Concerns | ✅ 完成 | ruby/02_models_concerns.md | 2026-03-01 |
| Controllers 规范 | ✅ 完成 | ruby/03_controllers.md | 2026-03-01 |
| Serializers 规范 | ✅ 完成 | ruby/04_serializers.md | 2026-03-01 |
| RSpec + fab! 规范 | ✅ 完成 | ruby/05_rspec_testing.md | 2026-03-01 |
| ActiveRecord 性能 | 🔲 待开始 | ruby/06_migrations_db.md | - |
| Ember Services (.js) | 🔲 待开始 | javascript/02_services.md | - |
| QUnit 测试 | 🔲 待开始 | javascript/05_qunit_testing.md | - |
| Background Jobs | ✅ 完成 | ruby/07_jobs.md | 2026-03-01 |

---

## 详细记录

### 2026-03-01 — 项目结构总览 & Skills 体系建立

- **学习内容**：
  - 阅读 CLAUDE.md 获取官方开发规范
  - 了解项目整体目录：Ruby 后端（app/controllers, models, services, serializers）+ Ember 前端（frontend/discourse/app）
  - 关键发现：前端使用 .gjs（Glimmer）格式，后端 Service 使用 Service::Base 模式
  - 工具链：pnpm（JS）/ bundle（Ruby）/ bin/rspec / bin/lint

- **验证来源**：
  - `discourse/CLAUDE.md`
  - `discourse/app/services/flags/create_flag.rb`
  - `discourse/frontend/discourse/app/components/d-button.gjs`
  - `discourse/spec/system/page_objects/pages/`

- **输出文件**：`README.md`（框架搭建）

---

### 2026-03-01 — Ruby Service Objects

- **学习内容**：
  - Service::Base DSL：policy / params / model / step / transaction
  - 依赖注入机制（关键字参数自动注入）
  - policy vs step 的语义区别
  - context 共享状态、optional model、外部 Policy 类

- **验证来源**：
  - `app/services/flags/create_flag.rb`
  - `app/services/flags/update_flag.rb`
  - `app/services/flags/destroy_flag.rb`
  - `app/services/user/silence.rb`

- **输出文件**：`ruby/01_service_objects.md`

---

### 2026-03-01 — Model Concerns

- **学习内容**：
  - `extend ActiveSupport::Concern` 标准骨架
  - `included do` 块内写 AR 宏（关联/scope/callback）
  - `class_methods do` 块写类方法
  - 命名规范：Has + 名词 / 形容词able
  - 复杂 Concern（HasCustomFields 含 Redis 操作）

- **验证来源**：
  - `app/models/concerns/trashable.rb`
  - `app/models/concerns/has_custom_fields.rb`
  - `app/models/concerns/cached_counting.rb`
  - `app/models/concerns/searchable.rb`

- **输出文件**：`ruby/02_models_concerns.md`

---

### 2026-03-01 — Controllers 规范

- **学习内容**：
  - ApplicationController 基类职责
  - Guardian 授权调用方式
  - 统一响应格式与 render_json

- **验证来源**：
  - `app/controllers/application_controller.rb`
  - `app/controllers/topics_controller.rb`
  - `app/controllers/categories_controller.rb`

- **输出文件**：`ruby/03_controllers.md`

---

### 2026-03-01 — Serializers 规范

- **学习内容**：
  - 三层继承层级（Basic → 标准 → Detailed）
  - 条件属性 include_xxx? 方法
  - has_one / has_many embed 方式
  - Serializer Mixin（concerns/ 目录）
  - staff_attributes / private_attributes Discourse 扩展 DSL
  - scope = guardian，用于权限判断
  - cache_fragment / cache_anon_fragment 缓存机制

- **验证来源**：
  - `app/serializers/application_serializer.rb`
  - `app/serializers/basic_user_serializer.rb`
  - `app/serializers/user_serializer.rb`
  - `app/serializers/basic_category_serializer.rb`
  - `app/serializers/basic_topic_serializer.rb`
  - `app/serializers/listable_topic_serializer.rb`
  - `app/serializers/flag_serializer.rb`
  - `app/serializers/concerns/user_status_mixin.rb`

- **输出文件**：`ruby/04_serializers.md`

- **下次继续**：`ruby/06_migrations_db.md`（ActiveRecord 迁移与性能）或 `ruby/07_jobs.md`（Background Jobs）

---

### 2026-03-01 — RSpec + fab! 规范

- **学习内容**：
  - `fab!` 宏：before(:context) 级别创建，比 let 快
  - Fabrication gem 使用模式
  - Service 测试的结果检查方式

- **验证来源**：`spec/` 目录各测试文件

- **输出文件**：`ruby/05_rspec_testing.md`

---

## 🔜 下一步建议

优先顺序：
1. **`ruby/06_migrations_db.md`** — ActiveRecord 迁移规范（add_column 安全写法、索引规范）
2. **`javascript/02_services.md`** — Ember Service 单例模式
3. **`patterns/01_service_base_pattern.md`** — 深度分析 Service::Base 源码
4. **`javascript/05_qunit_testing.md`** — QUnit 前端测试规范

---

### 2026-03-01 — Serializers 规范

- **学习内容**：
  - 三层继承层级（Basic → 标准 → Detailed）
  - 条件属性 `include_xxx?` 方法
  - `has_one / has_many` 的 `embed: :object` vs `embed: :ids`
  - Serializer Mixin（`concerns/` 目录，命名用 `Mixin` 后缀）
  - `staff_attributes / private_attributes / untrusted_attributes` Discourse 扩展 DSL
  - `scope` = guardian，用于权限判断
  - `cache_fragment / cache_anon_fragment` 缓存机制

- **验证来源**：
  - `app/serializers/application_serializer.rb`
  - `app/serializers/basic_user_serializer.rb`
  - `app/serializers/user_serializer.rb`
  - `app/serializers/basic_category_serializer.rb`
  - `app/serializers/basic_topic_serializer.rb`
  - `app/serializers/listable_topic_serializer.rb`
  - `app/serializers/flag_serializer.rb`
  - `app/serializers/concerns/user_status_mixin.rb`

- **输出文件**：`ruby/04_serializers.md`

---

### 2026-03-01 — Background Jobs 规范

- **学习内容**：
  - 三类 Job：Regular（手动触发）/ Scheduled（定时）/ Onceoff（单次）
  - 标准骨架：`module Jobs; class Xxx < ::Jobs::Base; def execute(args)`
  - `Jobs.enqueue / enqueue_in / enqueue_at` API
  - Scheduled Job 的 `every N.unit` 频率声明
  - 扇出模式：Scheduled 只分发，Regular 处理单条
  - 幂等性设计（`find_by` + `return` 守卫）
  - 参数只传 JSON 可序列化类型（id 而非对象）

- **验证来源**：
  - `app/jobs/base.rb`
  - `app/jobs/regular/post_alert.rb`
  - `app/jobs/regular/close_topic.rb`
  - `app/jobs/regular/push_notification.rb`
  - `app/jobs/scheduled/heartbeat.rb`
  - `app/jobs/scheduled/badge_grant.rb`
  - `app/jobs/scheduled/enqueue_digest_emails.rb`

- **输出文件**：`ruby/07_jobs.md`

- **下次继续**：`ruby/06_migrations_db.md` — ActiveRecord 迁移与数据库性能规范
