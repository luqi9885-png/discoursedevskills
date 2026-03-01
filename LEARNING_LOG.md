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
| Background Jobs | ✅ 完成 | ruby/07_jobs.md | 2026-03-01 |
| ActiveRecord 迁移与DB规范 | ✅ 完成 | ruby/06_migrations_db.md | 2026-03-29 |
| Ember Service 单例模式 | ✅ 完成 | javascript/02_services.md | 2026-03-29 |
| QUnit 前端测试 | ✅ 完成 | javascript/05_qunit_testing.md | 2026-03-29 |
| Guardian 权限系统 | ✅ 完成 | ruby/08_guardian.md | 2026-03-29 |
| Ember Routes | ✅ 完成 | javascript/03_routes.md | 2026-03-29 |
| Ember Store 与 Model | ✅ 完成 | javascript/04_models.md | 2026-03-29 |
| Plugin 完整流程 | 🔲 待开始 | patterns/02_full_plugin_flow.md | - |

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

- **输出文件**：`ruby/04_serializers.md`

---

### 2026-03-01 — RSpec + fab! 规范

- **输出文件**：`ruby/05_rspec_testing.md`

---

### 2026-03-01 — Background Jobs 规范

- **学习内容**：
  - 三类 Job：Regular / Scheduled / Onceoff
  - `Jobs.enqueue / enqueue_in / enqueue_at` API
  - 幂等性设计（find_by + return 守卫）

- **输出文件**：`ruby/07_jobs.md`

---

### 2026-03-29 — ActiveRecord 迁移与数据库规范

- **学习内容**：
  - 两类迁移目录：`db/migrate/`（部署前）vs `db/post_migrate/`（部署后）
  - SafeMigrate 保护机制：拦截 DROP TABLE / DROP COLUMN / RENAME（必须用 post_migrate）
  - 大表加索引必须用 `algorithm: :concurrently` + `disable_ddl_transaction!`
  - 先 `remove_index if_exists: true` 再 `add_index concurrently`（SafeMigrate 强制要求）
  - 删列两步流程：① Model 中 `ignored_columns`，② post_migrate 中 `ColumnDropper.execute_drop`
  - 函数索引、部分索引用原生 SQL（AR DSL 不支持）
  - 数据迁移用 SQL，不用 AR 模型，配合 `ON CONFLICT` 实现幂等

- **验证来源**：
  - `db/migrate/20251023042602_add_extra_index_topic_tags.rb`
  - `db/migrate/20260109041508_add_index_category_on_topics.rb`
  - `db/migrate/20251107123438_add_index_to_tag_groups.rb`
  - `db/migrate/20251024015907_populate_image_quality_setting.rb`
  - `db/migrate/20251111073356_create_tag_localizations.rb`
  - `db/post_migrate/20250821155127_drop_dark_hex_from_color_scheme_color.rb`
  - `lib/migration/safe_migrate.rb`
  - `lib/migration/column_dropper.rb`

- **输出文件**：`ruby/06_migrations_db.md`

---

### 2026-03-29 — Ember Service 单例模式

- **学习内容**：
  - `@disableImplicitInjections` 装饰器：Discourse 规范，必须显式声明依赖
  - `@service xxx;` 注入其他 Service
  - `@tracked` 响应式状态，`TrackedMap / TrackedArray / TrackedObject` 响应式集合
  - ES # 私有字段（`#title`），内部方法 `_method()` 约定
  - Evented mixin 模式（app-events 事件总线）
  - 自定义工厂模式：`static isServiceFactory = true; static create() { return instance; }`
  - `deprecated()` 包装废弃 API（指定 since / dropFrom）
  - `KeyValueStore` 持久化到 localStorage，带命名空间

- **验证来源**：
  - `frontend/discourse/app/services/app-events.js`
  - `frontend/discourse/app/services/document-title.js`
  - `frontend/discourse/app/services/header.js`
  - `frontend/discourse/app/services/emoji-store.js`
  - `frontend/discourse/app/services/capabilities.js`

- **输出文件**：`javascript/02_services.md`

---

### 2026-03-29 — QUnit 前端测试规范

- **学习内容**：
  - 三类测试：Unit（`setupTest`）/ Integration（`setupRenderingTest`）/ Acceptance
  - Unit 用 `.js`，Integration 用 `.gjs`（需要 JSX 模板语法）
  - `getOwner(this).lookup("service:name")` 获取 Service 实例
  - `this.set("key", value)` 传数据到模板
  - `assert.dom()` DOM 断言：`.exists()` / `.hasText()` / `.hasAttribute()`
  - `assert.step()` + `assert.verifySteps()` 验证回调调用顺序
  - `hooks.beforeEach` 初始化，`hooks.afterEach` 清理
  - 所有 DOM helper（`click / fillIn / select`）必须 `await`

- **验证来源**：
  - `frontend/discourse/tests/unit/services/emoji-store-test.js`
  - `frontend/discourse/tests/integration/components/admin-filter-controls-test.gjs`

- **输出文件**：`javascript/05_qunit_testing.md`

### 2026-03-29 — Ember Store 与 Model 规范

- **学习内容**：
  - Store 是 Service（`@service store`），不是 Ember Data；自定义实现 identity map 单例缓存
  - `find / findAll / createRecord / update / destroyRecord / appendResults` 核心方法
  - `createRecord` 有 id → hydrate 进 identity map；无 id → `isNew=true` 的新记录
  - Identity map：同 type+id 全局只有一个实例，hydrate 时只调 `setProperties` 不 new 新对象
  - `RestModel` 基类：`isSaving / isNew / isCreated` 状态；`save()` 自动判断新建/更新
  - `createProperties()` 必须重写；`static munge(json)` 预处理服务端 JSON
  - `@discourseComputed`：依赖追踪计算属性，比 Ember `@computed` 更简洁
  - Ember macros：`@alias / @equal / @notEmpty / @or / @and / @fmt`
  - `@tracked` + `@trackedArray`：Glimmer 响应式属性
  - `@cached + @dependentKeyCompat`：惰性计算 + 兼容旧依赖系统
  - 静态方法：批量操作或不需要实例时直接 ajax，不走 store
  - `@singleton`：`User.current()` 全局唯一实例
  - `ResultSet`：`findAll` 的返回值，含 `totalRows / loadMoreUrl / canLoadMore`

- **验证来源**：
  - `services/store.js`（Store 实现）
  - `models/rest.js`（RestModel 基类）
  - `models/topic.js`（复杂 model：munge、@discourseComputed、静态方法）
  - `models/user.js`（@singleton、@userOption、findDetails）

- **输出文件**：`javascript/04_models.md`

---

### 2026-03-29 — Ember Routes 规范

- **学习内容**：
  - 所有 Route 继承 `DiscourseRoute`（不直接用 Ember 的 Route）
  - 路由在 `app-route-map.js` 集中声明；`resetNamespace: true` 去掉父路由名称前缀
  - 生命周期顺序：`beforeModel → model → afterModel → setupController → activate → deactivate`
  - `beforeModel`：权限守卫，throw RouteException 或 replaceWith 阻断
  - `model(params, transition)`：返回值成为 controller.model；子路由用 `this.modelFor()` 复用
  - `afterModel`：model 加载后的异步操作（加载详情、重定向 404）
  - `setupController`：向 controller 写入 model 和初始状态，设置全局 Service 状态
  - `activate/deactivate`：订阅/取消订阅 MessageBus，deactivate 必须清理防内存泄漏
  - `queryParams = { key: { replace: true } }`：不产生浏览器历史记录
  - `@action` 方法可被子路由和 Controller `send()` 冒泡调用
  - `titleToken()` 返回字符串或数组控制页面标题
  - 专用基类模式：`RestrictedUserRoute`、工厂函数 `build-topic-route.js` 等

- **验证来源**：
  - `routes/discourse.js`（基类）
  - `routes/about.js`（简单 model hook）
  - `routes/topic.js`（复杂 model、setupController、@action）
  - `routes/user.js`（beforeModel 守卫、afterModel、activate/deactivate）
  - `routes/preferences.js`（复用父 model）
  - `routes/restricted-user.js`（专用基类）
  - `routes/app-route-map.js`（路由定义）

- **输出文件**：`javascript/03_routes.md`

---

### 2026-03-29 — Guardian 权限系统

- **学习内容**：
  - Guardian 主类 + Mixin 模块结构（每类资源独立 module，include 组合）
  - AnonymousUser 内部类：所有 `xxx?` 返回安全默认值，权限方法无需判断 nil
  - 角色判断：`is_admin? / is_staff? / is_moderator? / is_my_own? / is_me?`
  - 通用分发机制：`can_see?(obj)` → `method_name_for(:see, obj)` → `can_see_topic?`
  - EnsureMagic：`ensure_can_edit!` 自动对应 `can_edit?`，无权则 raise InvalidAccess
  - 权限方法编写规范：先 false 守卫 → 高权限短路 → 细化条件
  - Group 权限模式：`@user.in_any_groups?(SiteSetting.xxx_groups_map)`
  - Controller 中三种用法：布尔判断 / ensure! / 传 scope 给 Serializer

- **验证来源**：
  - `lib/guardian.rb`
  - `lib/guardian/ensure_magic.rb`
  - `lib/guardian/topic_guardian.rb`
  - `lib/guardian/user_guardian.rb`
  - `lib/guardian/post_guardian.rb`

- **输出文件**：`ruby/08_guardian.md`

---

## 🔜 下一步建议

优先顺序：
1. **`patterns/02_full_plugin_flow.md`** — 完整 Plugin 开发流程（前后端联调）
2. **`system_specs/02_system_spec_patterns.md`** — 系统测试通用模式
