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
| 完整 Plugin 开发流程 | ✅ 完成 | patterns/02_full_plugin_flow.md | 2026-03-02 |
| 系统测试通用模式 | ✅ 完成 | system_specs/02_system_spec_patterns.md | 2026-03-02 |
| Lint + 格式化规范 | ✅ 完成 | tooling/02_linting_formatting.md | 2026-03-02 |
| Plugin 完整流程 | 🔲 待开始 | patterns/02_full_plugin_flow.md | - |
| Guardian 权限系统（重建） | ✅ 完成 | ruby/08_guardian.md | 2026-03-02 |
| MessageBus 实时通信 | ✅ 完成 | javascript/06_message_bus.md | 2026-03-02 |
| RateLimiter 限流模式 | ✅ 完成 | ruby/10_rate_limiter.md | 2026-03-02 |

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

### 2026-03-02 — Guardian 权限系统（重建）+ MessageBus + RateLimiter

- **Guardian（重建）**：
  - 主类 + Mixin 组合（TopicGuardian / PostGuardian 等独立 include）
  - AnonymousUser 内部类：所有方法返回安全默认值，避免 nil check
  - 角色判断：is_admin? / is_staff? / is_moderator? / is_my_own? / is_me?
  - 分发机制：can_see?(obj) → method_name_for(:see, obj) → can_see_topic?
  - EnsureMagic：method_missing 实现，ensure_can_edit! 对应 can_edit?，无权 raise Discourse::InvalidAccess
  - 权限方法模式：false 守卫 → true 短路（高权限） → 细化条件
  - Group 权限：@user.in_any_groups?(SiteSetting.xxx_groups_map)
  - alias 复用：can_archive_topic? 等 5 个方法 alias 同一实现
  - Controller 三种用法：布尔判断 / ensure! / 传 scope 给 Serializer
  - Plugin 扩展：module GuardianExtensions + prepend + reloadable_patch

- **MessageBus**：
  - Service 层：直接 return message-bus-client 库实例
  - subscribe(channel, callback, lastId?) / unsubscribe(同一函数引用)
  - Instance-initializer 模式：constructor 订阅 + teardown 取消 + @bind 固定 this
  - 常见频道：/notification/{id} / /topic/{id} / /categories / /client_settings
  - Plugin：registerCustomPostMessageCallback(type, (controller, message) => {})
  - 后端：MessageBus.publish(channel, data, user_ids:, group_ids:)
  - 安全受众：secure_audience_publish_messages 限制私信/受限分类的推送范围
  - 测试环境 isTesting() 时不启动

- **RateLimiter**：
  - Redis List + Lua 脚本实现滑动窗口（原子操作，线程安全）
  - RateLimiter.new(user, type, max, secs, global:, aggressive:, apply_limit_to_staff:)
  - performed! / can_perform? / remaining / seconds_to_wait / rollback! / clear!
  - Staff 默认豁免（apply_limit_to_staff: false），测试/profile 模式自动禁用
  - Controller 模式：return if staff? + 双窗口（分钟+小时）+ rescue LimitExceeded
  - IP 限流：key 包含 request.remote_ip（覆盖匿名用户）
  - Model 层：include OnCreateRecord + rate_limit DSL + default_rate_limiter 读 SiteSetting
  - Key 规范：{操作}-{时间粒度}-{user_id 或 ip}

- **验证来源**：
  - `lib/guardian.rb`（完整主类）
  - `lib/guardian/ensure_magic.rb`、`topic_guardian.rb`
  - `lib/rate_limiter.rb`、`lib/rate_limiter/on_create_record.rb`
  - `app/controllers/session_controller.rb`（IP 限流 + rescue 模式）
  - `plugins/discourse-solved/app/controllers/.../answer_controller.rb`
  - `frontend/.../instance-initializers/message-bus.js`
  - `frontend/.../instance-initializers/subscribe-user-notifications.js`
  - `plugins/discourse-solved/.../extend-for-solved-button.gjs`（registerCustomPostMessageCallback）

- **输出文件**：
  - `ruby/08_guardian.md`（重建）
  - `javascript/06_message_bus.md`（新建）
  - `ruby/10_rate_limiter.md`（新建）

---

### 2026-03-02 — Lint + 格式化规范

- **学习内容**：
  - 工具全景：RuboCop + stree（Ruby）、ESLint + Prettier（JS）、ember-template-lint（GJS）、Stylelint（SCSS）、yaml-lint、i18n-lint
  - `.rubocop.yml` 继承 `rubocop-discourse: stree-compat.yml`，项目文件只做覆盖
  - 6 条 Plugin 专用 Cops：CallRequiresPlugin / UsePluginInstanceOn / NamespaceMethods / NamespaceConstants / UseRequireRelative / NoMonkeyPatching
  - `Discourse/NoResetColumnInformationInMigrations`：迁移中禁止调用 reset_column_information
  - `RSpec/InstanceVariable`：models spec 中禁用实例变量，用 let
  - rubocop:disable 注释用法，区块 vs 单行
  - RuboCop（质量）与 Syntax Tree（格式）分工：两者分别运行
  - `eslint.config.mjs` Flat Config：继承 @discourse/lint-configs/eslint + 项目额外规则
  - 关键 ESLint 规则：qunit/no-assert-equal、ember/no-classic-components、discourse/moved-packages-import-paths
  - `.prettierrc.cjs`：完全继承官方配置，ESLint 与 Prettier 通过 eslint-config-prettier 分工不冲突
  - Lefthook：pre-commit（只扫 staged_files，并行快）/ fix-staged / fix-all 三个 hook 组
  - `package.json` scripts：pnpm lint:js / lint:css / lint:hbs / lint:prettier / lint:types
  - i18n_lint：禁 HTML 实体，插值用 %{var} 非 {{var}}
  - 日常流程：`bin/lint --fix --recent` → staged files 全量修复

- **验证来源**：
  - `.rubocop.yml`（完整配置）
  - `eslint.config.mjs`（完整配置）
  - `.prettierrc.cjs`
  - `lefthook.yml`（完整配置）
  - `package.json` lint scripts
  - `plugins/` 中实际 rubocop:disable 使用示例

- **输出文件**：`tooling/02_linting_formatting.md`

---

### 2026-03-02 — 系统测试通用模式

- **学习内容**：
  - System Spec vs Request Spec：端到端 vs HTTP 层；浏览器交互 vs JSON 验证
  - `fab!` 在 describe 开始前创建（共享），`Fabricate` 在 before/it 内按需创建（隔离）
  - `sign_in(user)` 通过 `/session/xxx/become.json` 路由登录；不调用即匿名
  - `SiteSetting.xxx = value` 在测试内赋值，结束后自动还原；Mocha stubs 同理
  - 直接 Capybara DSL：`visit`、`find`、`expect(page).to have_css`、`send_keys`
  - Page Object 模式：继承 `Base`（含 `Capybara::DSL`）；`has_xxx?`/`has_no_xxx?`；操作方法返回 self
  - `within(selector)` 缩小查找范围，避免 CSS 冲突
  - 私有辅助方法：`it` 读起来像用户故事，步骤细节下沉到 private
  - 常量集中定义选择器（`ACCEPTED_BUTTON_SELECTOR = "..."`），修改只改一处
  - `SystemHelpers`：`try_until_success`、`wait_for_animation`、`wait_for_attribute`、`resize_window`、`using_browser_timezone`、`fake_scroll_down_long`、`pause_test`
  - Playwright 直接操作：`page.driver.with_playwright_page { |pw| pw.mouse.click(x, y, clickCount: 3) }`
  - Plugin system spec：放 `plugins/xxx/spec/system/`，直接复用核心 Page Object

- **验证来源**：
  - `spec/system/about_page_spec.rb`（fab!、SiteSetting、Page Object 完整用法）
  - `spec/system/flagging_post_spec.rb`（sign_in、多角色、Modal）
  - `spec/system/topic_page_spec.rb`（send_keys、Playwright、with_playwright_page）
  - `spec/system/admin_filter_controls_spec.rb`（Component Page Object）
  - `spec/support/system_helpers.rb`（sign_in 实现、所有 helper 方法）
  - `spec/system/page_objects/pages/topic.rb`（Page Object 实现规范）
  - `spec/system/page_objects/pages/base.rb`（Base 类）
  - `plugins/discourse-solved/spec/system/solved_spec.rb`（Plugin spec 模式、私有方法）

- **输出文件**：`system_specs/02_system_spec_patterns.md`

---

### 2026-03-02 — 完整 Plugin 开发流程

- **学习内容**：
  - 完整 Plugin 目录结构及各层职责
  - `plugin.rb`：头声明、`enabled_site_setting`、`register_asset/svg_icon`、`after_initialize` 块
  - `after_initialize` 内的 DSL：`reloadable_patch`、`add_to_serializer`、`on(:event)`、`register_modifier`
  - Rails Engine：`isolate_namespace` 隔离命名空间，表名加前缀防冲突
  - `config/routes.rb`：Engine 内路由 + `mount Engine, at: "prefix"` 挂载到主 App
  - `config/settings.yml`：`client: true/false`、`type: group_list/enum`、`mandatory_values`
  - Controller：`requires_plugin`、`RateLimiter` 限流、`guardian.ensure_can_xxx!`
  - Guardian 扩展：`module GuardianExtensions` → `prepend` 进 `::Guardian`
  - Serializer 扩展：`prepended { attributes :field }` + `include_xxx?` 条件
  - Plugin Model：`self.table_name` 明确指定带前缀的表名
  - 前端 initializer：`withPluginApi`、`api.modifyClass`、`api.registerValueTransformer`
  - pre-initializer：`before: "inject-discourse-objects"` 在 Ember 注入前运行
  - connector：路径名即 outlet 名，`@outletArgs` 接收上下文
  - `solved-route-map.js`：`resource` 指定挂载点，`this.route()` 新增路由
  - Component 调用后端：`ajax` → `topic.setAcceptedSolution()` 更新前端状态
  - MessageBus 实时同步：后端 `publish`，前端 `registerCustomPostMessageCallback`
  - 完整请求链路：按钮点击 → ajax → Controller → 业务逻辑 → MessageBus → 全端同步

- **验证来源**：
  - `plugins/discourse-solved/plugin.rb`（完整）
  - `plugins/discourse-solved/lib/discourse_solved/engine.rb`
  - `plugins/discourse-solved/lib/discourse_solved/guardian_extensions.rb`
  - `plugins/discourse-solved/lib/discourse_solved/topic_view_serializer_extension.rb`
  - `plugins/discourse-solved/app/controllers/discourse_solved/answer_controller.rb`
  - `plugins/discourse-solved/app/models/discourse_solved/solved_topic.rb`
  - `plugins/discourse-solved/db/migrate/20250318024824_create_discourse_solved_solved_topics.rb`
  - `plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`
  - `plugins/discourse-solved/assets/javascripts/discourse/pre-initializers/extend-category-for-solved.js`
  - `plugins/discourse-solved/assets/javascripts/discourse/connectors/after-topic-status/solved-status.gjs`
  - `plugins/discourse-solved/assets/javascripts/discourse/components/solved-accept-answer-button.gjs`
  - `plugins/discourse-solved/assets/javascripts/discourse/solved-route-map.js`
  - `plugins/discourse-solved/config/settings.yml`
  - `plugins/discourse-solved/config/routes.rb`

- **输出文件**：`patterns/02_full_plugin_flow.md`

---

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
1. **规划下一批**：当前 roadmap 已完成，需要制定新的学习目标
