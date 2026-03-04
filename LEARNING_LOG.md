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

---

### 2026-03-02 — PostCreator + Reviewable + Search 系统

- **PostCreator 核心创建流程**：
  - `PostCreator.create(user, opts)` 是唯一正确的帖子创建入口
  - create 方法执行顺序：valid? → transaction（build_stats, create_topic, save_post, track_topic, update_stats） → publish → trigger_after_events → enqueue_jobs → auto_close
  - 新话题 vs 回复：回复用 `DistributedMutex.synchronize("topic_id_xxx")` 保证 post_number 单调递增无间隙
  - skip_jobs 模式：事务内调用 PostCreator 必须 `skip_jobs: true`，commit 后手动 `enqueue_jobs`
  - 三个 DiscourseEvent 触发点：`:before_create_post`（验证前）、`:topic_created`（新话题）、`:post_created`（每次）
  - 唯一帖子检查：Redis key `unique[-pm]-post-{user_id}:{raw_hash}` 防重复提交
  - auto_close：私信/公开话题都有 max_posts 自动关闭逻辑

- **Reviewable 审核系统**：
  - 4 种内置类型：ReviewableFlaggedPost / ReviewableQueuedPost / ReviewableUser / ReviewablePost
  - 状态机：pending → approved / rejected / ignored / deleted
  - 必须用 `needs_review!` 而非 `.create`：自动处理重复 target（upsert 回 pending）
  - Score 系统：trust_level_bonus + type_bonus + take_action_bonus，决定优先级和自动动作阈值
  - 子类必须实现 `build_actions` + `perform_{action_id}` 方法
  - Plugin 注册：`DiscoursePluginRegistry.register_reviewable_type(MyReviewable, plugin: self)`

- **Search 搜索系统**：
  - 底层：PostgreSQL tsvector/tsquery + gin 索引（post_search_data 表）
  - 多语言：中文用 CppjiebaRb 分词，日文用 TinyJapaneseSegmenter，其他用 PostgreSQL stemmer
  - 高级过滤器语法全部内联在搜索词中（status:solved、category:name、tags:tag1+tag2 等）
  - Plugin 扩展：`Search.advanced_filter(/\Astatus:solved\z/i) { |posts| }` 注册自定义过滤
  - Plugin 扩展排序：`Search.advanced_order(:name) { |posts| posts.reorder(...) }`
  - 搜索日志：每次搜索自动记录到 search_logs 表

- **验证来源**：
  - `lib/post_creator.rb`、`app/models/post.rb`
  - `app/models/reviewable.rb`、`app/models/reviewable_flagged_post.rb`
  - `lib/search.rb`（700+ 行）

- **输出文件**：
  - `ruby/11_post_topic_creator.md`（新建）
  - `ruby/12_reviewable.md`（新建）
  - `ruby/13_search.md`（新建）

---

## 🔜 下一步建议

优先顺序：
1. **Mailer 邮件系统**：`app/mailers/`、UserNotifications、邮件模板、ActionMailer 规范
2. **TopicQuery**：`lib/topic_query.rb`，话题列表构建、filter 机制
3. **DiscourseConnect / SSO**：`lib/discourse_connect_base.rb`
4. **Wizard 系统**：`lib/wizard/`，安装向导框架

---

### 2026-03-03 — registerValueTransformer+DAG 与 modifyClass 规范

- **方向纠正**：上一批（PostCreator/Reviewable/Search）是站点机制知识，非插件开发技术。本批聚焦插件开发实际所需的前端扩展技术。

- **registerValueTransformer + DAG**：
  - DAG 是有向无环图，基于 `dag-map` 库，用 `before`/`after` 声明排序约束
  - `post-menu-buttons` 是最常用的 transformer：插件通过 `dag.add/replace/delete/reposition` 操作按钮列表
  - context 提供 `buttonKeys`（核心按钮常量）、`firstButtonKey`、`lastHiddenButtonKey` 等定位锚点
  - `add` 时 before/after 可传数组作为优先级回退（引用不存在的 key 不报错）
  - `replace` 替换已有按钮（位置不变）；`delete` 移除；`reposition` 改位置不改 value
  - mutable transformer（post-menu-buttons）直接改 dag 不需要 return；immutable transformer（topic-list-item-class）需要 return 新值
  - 被注入组件通过 `static shouldRender(args)` 控制是否渲染

- **modifyClass**：
  - 工厂函数语法 `(Superclass) => class extends Superclass { ... }` 是必须的
  - 覆盖 `@tracked` getter 时必须同时覆盖 setter，否则运行时报错
  - 扩展 subscribe/unsubscribe 必须调用 `super.subscribe/unsubscribe(...arguments)`
  - 第三个参数 `{ ignoreMissing: true }` 用于可选组件（admin 专属组件等）
  - 类型名约定：`type:kebab-case`，路径中 `/` 改 `.`，去后缀
  - 验证了 5 种类型：model/controller/component/route/service

- **验证来源**（≥2 处每条规范）：
  - `plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`（725 行，modifyClass 5 处 + registerValueTransformer）
  - `plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`（modifyClass + registerValueTransformer）
  - `plugins/discourse-reactions/assets/javascripts/discourse/initializers/discourse-reactions.gjs`（dag.replace + ignoreMissing）
  - `frontend/discourse/app/lib/dag.js`（DAG 类全部方法）
  - `frontend/discourse/app/components/post/menu.gjs`（buttonKeys 定义 + context 字段）

- **输出文件**：
  - `plugins/04_value_transformer_dag.md`（新建）
  - `plugins/05_modify_class.md`（新建）


---

### 2026-03-03（续）— 插件开发全面扩展（UI、数据库、后端）

**本批目标**：覆盖插件开发者实际需要的全部重要功能模块，包括 Post UI 注入、Modal 弹窗、用户侧 UI、管理后台、数据库/Engine、后端 modifier。

- **06_tracked_post_and_render_outlet.md**：addTrackedPostProperties + renderAfterWrapperOutlet，static shouldRender，完整端到端流程（serializer→tracked→UI→MessageBus）
- **07_modal_and_ui_components.md**：DModal 骨架（具名 block body/footer）、@tagName="form" 提交模式、自定义 Service 封装 modal.show（task-actions 模式）、TrackedObject、DButton 参数速查
- **08_user_ui_extensions.md**：registerUserMenuTab（id/panelComponent/icon/count/linkWhenActive）、registerNotificationTypeRenderer、用户偏好 Connector 插槽速查、ComboBox 双向绑定、用户活动页导航 Connector
- **09_admin_ui.md**：setAdminPluginIcon、Admin 插件专属页面目录结构、Admin Connector 插槽速查、add_report（labels/data/DB.query）、动态列报表
- **10_plugin_model_and_db.md**：Rails Engine（isolate_namespace/autoload_paths/mount）、迁移规范（change vs up/down，禁止修改历史）、Model 命名空间隔离/polymorphic/scope/validate、Engine Controller（requires_plugin/render_json_error）、预加载防 N+1（TopicList/TopicView/BookmarkQuery.on_preload）
- **11_register_modifier_backend.md**：管道机制（block 返回值成为下一个输入）、三类参数（QueryBuilder/AR Relation/Array）、must return value、always in after_initialize

**验证来源**：discourse-assign 插件全文、discourse-reactions/plugin.rb、discourse-chat-integration/admin/、lib/discourse_plugin_registry.rb

**输出文件**：plugins/06 ~ plugins/11 共 6 个文件

---

### 2026-03-03（续二）— discourse-ai 深度挖掘，覆盖 4 个核心缺口

**数据来源**：`plugins/discourse-ai/`（全目录 + 重点文件），`lib/hijack.rb`

**完成内容：**

- **12_api_initializer_and_ui_injection.md**：
  - `apiInitializer` vs `withPluginApi` 选择规则（`api.container.lookup` 等价）
  - `api.onToolbarCreate / toolbar.addButton`（Composer 工具栏，toolbarEvent 方法速查）
  - `api.registerTopicFooterButton`（话题脚注按钮，dependentKeys 响应式依赖）
  - `api.headerIcons.add`（顶部导航栏图标）
  - `api.addCommunitySectionLink`（侧边栏社区链接，baseSectionLink 继承模式）
  - `api.renderInOutlet` vs `renderAfterWrapperOutlet`（区别对比表）
  - `api.addBulkActionButton`（批量操作按钮，actionType: performAndRefresh）
  - `api.addTopicAdminMenuButton`（话题管理菜单项）
  - `api.registerReportModeComponent` + `optionalRequire`（自定义报表渲染 + 可选依赖）

- **13_backend_advanced_patterns.md**：
  - EntryPoint 模式：`inject_into(plugin)` 方法按功能模块拆分 after_initialize 逻辑
  - `Rails.autoloaders.main.push_dir(File.join(__dir__, "lib"), namespace: DiscourseAi)` — 自动加载（必须在 after_initialize 之前）
  - `hijack do...end` — 长时 IO 请求解放 Worker，rescue 必须写在 block 内部
  - Controller Concern（`extend ActiveSupport::Concern` + `included do rescue_from`）
  - RATE_LIMITS 常量 + `action_name` 按 action 分级限流
  - plugin.rb 高级 DSL：`add_admin_route`、`register_seedfu_fixtures`、`register_reviewable_type`、`register_problem_check`、`add_api_key_scope`、`add_model_callback`
  - `add_to_serializer include_condition` 动态条件（lambda，可引用 scope/object/其他字段）
  - Admin Controller 继承 `::Admin::AdminController` 规范

- **14_plugin_service_frontend.md**：
  - 轻量状态服务：`@tracked` + 操作方法，跨组件共享
  - localStorage 偏好服务：`#loadPreference()` 私有方法 + 迁移 key 逻辑 + `router.currentRoute.attributes` 动态读取路由数据
  - 请求去重服务：ES # 私有字段 `#pendingRequests = new Map()` + Promise 缓存 + then/catch 自动清理
  - constructor/willDestroy 生命周期：appEvents on/off + messageBus subscribe/unsubscribe（必须成对）
  - `discourseDebounce` + 无限滚动分页（isFetching 防重 + hasMore 终止）
  - `addSidebarSection` 动态添加侧边栏区块（需要先 `api.addSidebarPanel` 注册面板）
  - `TrackedArray` + `scheduleOnce("afterRender")` 防多次触发

- **15_admin_multi_page.md**：
  - `addAdminPluginConfigurationNav(PLUGIN_ID, [{label, route, description}])` 子导航配置
  - 路由名规律：`adminPlugins.show.{plugin-id}-{子页面}`
  - `admin-xxx-plugin-route-map.js`：`resource: "admin.adminPlugins.show"` + 嵌套路由（new/edit）
  - 文件目录：`admin/assets/javascripts/discourse/routes/admin-plugins/show/` + `templates/admin-plugins/show/`
  - Route 加载数据：`store.findAll` / `store.find` / `.content` 取数组
  - 模板 import 路径计算（按目录深度数 `../` 级别）
  - Admin Store Model + Adapter（`basePath` 覆盖 URL 前缀）
  - `add_admin_route("i18n_key", "plugin-id", { use_new_show_route: true })`（必须加 `use_new_show_route`）

**验证来源**：
- `plugins/discourse-ai/assets/javascripts/discourse/initializers/` 全部文件
- `plugins/discourse-ai/services/` 全部 4 个 Service 文件
- `plugins/discourse-ai/admin/assets/` 路由 + 模板目录
- `plugins/discourse-ai/app/controllers/` AI helper + Admin personas controller
- `plugins/discourse-ai/lib/{ai_helper,summarization}/entry_point.rb`
- `lib/hijack.rb`（hijack 完整实现）

**输出文件**：plugins/12 ~ plugins/15 共 4 个文件

## 2026-03-04 — javascript/06_message_bus.md

**状态**：✅ 完成

**本次完成**：
- 阅读 `frontend/discourse/app/instance-initializers/read-only.js`（标准单频道模式）
- 阅读 `frontend/discourse/app/instance-initializers/banner.js`（PreloadStore + 订阅组合）
- 阅读 `frontend/discourse/app/instance-initializers/subscribe-user-notifications.js`（多频道 + position 参数）
- 阅读 `plugins/discourse-assign/components/flagged-topic-listener.js`（旧式 Component 模式）
- 阅读 `plugins/discourse-assign/initializers/extend-for-assigns.js:455-530`（modifyClass Controller 覆盖 subscribe/unsubscribe）
- 阅读 `plugins/discourse-assign/lib/assigner.rb:447,528`（后端 MessageBus.publish 对应）
- 整理 `javascript/06_message_bus.md`（379 行）

**验证来源**：
- `instance-initializers/read-only.js`：`@service messageBus` + `@bind` + `setOwner` 标准三件套，最简模式
- `instance-initializers/banner.js`：PreloadStore 初始值 + subscribe 实时更新，与 read-only.js 差异：多了 EmberObject.create 包装
- `instance-initializers/subscribe-user-notifications.js:40-70`：带第三参数 `position`（`last_id`）避免重放历史消息
- `extend-for-assigns.js:462-520`：modifyClass Controller 必须调 `super.subscribe()`，否则核心订阅丢失
- `assigner.rb:447,528`：后端 `user_ids:` 参数限定接收者，与前端频道路径对应

**关联更新**：
- TOPIC_MAP.md 已更新（新增 MessageBus 前端条目）
- README 目录 ✅ 已更新（javascript/06 加入目录，待完成任务已更新）
- 同次提交：创建 TOPIC_MAP.md（之前不存在）、README 补充 21~24 目录条目

## 2026-03-04 — plugins/25_plugin_testing.md

**状态**：✅ 完成

**本次完成**：
- 阅读 `discourse-assign/spec/fabricators/assignment_fabricator.rb`（transient + 动态关联）
- 阅读 `discourse-assign/spec/fabricators/notification_fabricator.rb`（from: 继承）
- 阅读 `discourse-assign/spec/requests/assign_controller_spec.rb`（request spec 完整模式）
- 阅读 `discourse-assign/spec/models/assignment_spec.rb`（model spec + described_class）
- 阅读 `discourse-assign/spec/jobs/regular/assign_notification_spec.rb`（job spec + Mocha）
- 阅读 `discourse-assign/spec/support/assign_allowed_group.rb`（shared_context）
- 阅读 `discourse-reactions/spec/fabricators/reaction_fabricator.rb`（class_name 命名空间）
- 阅读 `discourse-reactions/spec/requests/custom_reactions_controller_spec.rb`（多 fab! 场景）
- 整理 `plugins/25_plugin_testing.md`（469 行）
- 收尾：发现 README 漏登记 8 个文件（patterns/03~05、ruby/09~13），已补入

**验证来源**：
- `assign_controller_spec.rb`：`fab!` + `sign_in` + `require_relative support` + `response.parsed_body`，标准 request spec 模式
- `custom_reactions_controller_spec.rb`：与 assign 差异在大量 fab! 预建关联数据 + before 中设多个 SiteSetting
- `assignment_fabricator.rb`：`transient :post` + `target { |attrs| }` 动态关联，assign 中独有
- `reaction_fabricator.rb`：`class_name: "DiscourseReactions::Reaction"` 字符串形式，有命名空间时必须
- `assign_notification_spec.rb`：Mocha `stubs/expects` 风格，Discourse 统一用 Mocha 不用 RSpec double
- `assign_allowed_group.rb`：`shared_context` + helper 方法 + `include_context` 复用模式

**关联更新**：
- TOPIC_MAP.md ✅ 已更新（新增 plugins/25 条目）
- README 目录 ✅ 已更新（补登记 patterns/03~05、ruby/09~13、plugins/25）
- 待完成任务已更新（移除 plugins/25）
