# Skill Index — 任务到技能的映射表

> **供 AI 开发者角色查阅。**
> 收到开发任务后，先用本表确定需要加载哪些技能文件，再按顺序阅读，不要全量加载。
>
> 基础层（Phase 1）每次必读。功能层（Phase 2）按任务特征按需加载。横切层（Phase 3）按需补充。

---

## Phase 1：基础层（所有插件任务必读，共 4 个文件）

> 无论任务是什么，先读这 4 个。它们建立插件的骨架认知。

| 顺序 | 文件 | 作用 |
|------|------|------|
| 1 | `plugins/01_plugin_structure.md` | 目录布局、文件命名规范 |
| 2 | `plugins/01_plugin_rb_and_backend.md` | plugin.rb DSL、后端扩展入口 |
| 3 | `plugins/02_plugin_api_frontend.md` | withPluginApi、apiInitializer 基础 |
| 4 | `patterns/01_plugin_architecture.md` | 整体架构模式、扩展点全览 |

---

## Phase 2：功能层（按任务特征选读）

### 数据与持久化

| 功能需求 | 必读文件 | 阅读顺序 | 核心关注点 |
|---------|---------|---------|-----------|
| 新增数据库表 | `ruby/06_migrations_db.md` → `plugins/10_plugin_model_and_db.md` | 06→10 | SafeMigrate、Plugin Engine、add_column 安全规范 |
| 自定义字段（无新表） | `patterns/04_custom_fields.md` → `plugins/21_database_operations.md` | 04→21 | CustomFields vs PluginStore 的选择 |
| 复杂 DB 操作 | `plugins/21_database_operations.md` | 21 | DB.exec/upsert_all/PluginStore |

### 后端逻辑

| 功能需求 | 必读文件 | 阅读顺序 | 核心关注点 |
|---------|---------|---------|-----------|
| Service 对象 / 业务逻辑 | `ruby/01_service_objects.md` | 01 | Service::Base DSL、step/result |
| 权限控制 | `ruby/08_guardian.md` → `plugins/23_routes_and_guardian.md` | 08→23 | can?/ensure! 的使用，插件 GuardianExtensions |
| REST API 端点 | `plugins/23_routes_and_guardian.md` → `ruby/03_controllers.md` → `ruby/04_serializers.md` | 23→03→04 | 路由注册、Controller 规范、响应序列化 |
| 序列化扩展 | `ruby/04_serializers.md` → `plugins/22_serializer_extensions.md` | 04→22 | add_to_serializer、preloaded_custom_fields |
| 后台任务（异步） | `ruby/07_jobs.md` | 07 | Jobs::Base/Scheduled、enqueue 时机 |
| 事件监听 | `plugins/18_on_event_system.md` | 18 | on() 注册、40+ 事件速查表 |
| 行为管道拦截 | `plugins/11_register_modifier_backend.md` | 11 | register_modifier 与 on() 的区别 |
| 限流保护 | `ruby/10_rate_limiter.md` | 10 | RateLimiter.new/performed! |
| N+1 预防 | `plugins/26_preloading.md` → `plugins/22_serializer_extensions.md` | 26→22 | on_preload 注册、Serializer 读取预加载值 |

### 前端 — 管理后台

| 功能需求 | 必读文件 | 阅读顺序 | 核心关注点 |
|---------|---------|---------|-----------|
| Admin 单页面 | `plugins/09_admin_ui.md` | 09 | 插件 Admin 页面入口、Connector |
| Admin 多子页面 | `plugins/09_admin_ui.md` → `plugins/15_admin_multi_page.md` | 09→15 | addAdminPluginConfigurationNav、路由命名规律 |
| Admin 数据层（RestModel） | `plugins/20_admin_rest_model_adapter.md` | 20 | RestModel/RestAdapter、createProperties/pathFor |
| Admin 报告图表 | `plugins/28_reports.md` | 28 | Report.add_report 折线图/表格 |

### 前端 — 用户界面

| 功能需求 | 必读文件 | 阅读顺序 | 核心关注点 |
|---------|---------|---------|-----------|
| 自定义前端路由/页面 | `plugins/16_plugin_route_map.md` → `javascript/03_routes.md` | 16→03 | route-map 注册、Route/Template 规范 |
| Glimmer 组件 | `javascript/01_component_gjs.md` | 01 | .gjs 格式、@tracked、生命周期 |
| 弹窗 / Modal | `plugins/07_modal_and_ui_components.md` | 07 | DModal、showModal、attrs 传参 |
| 表单 | `javascript/08_form_kit.md` | 08 | @data POJO 要求、Field 控件体系 |
| 下拉选择器 | `javascript/07_select_kit.md` | 07 | ComboBox/MultiSelect/modifySelectKit |
| 用户菜单 / 偏好页扩展 | `plugins/08_user_ui_extensions.md` | 08 | addUserMenuItems、addPreference |
| topic-list 自定义列 | `plugins/17_topic_list_columns.md` | 17 | addCustomColumn、registerValueTransformer |
| Composer 工具栏 / 批量操作 | `plugins/12_api_initializer_and_ui_injection.md` | 12 | addComposerToolbarPopupMenuOption、addBulkActionButton |
| 前端 Service | `plugins/14_plugin_service_frontend.md` | 14 | 四种注册模式、@tracked 状态 |
| 实时消息推送 | `javascript/06_message_bus.md` | 06 | subscribe/unsubscribe/@bind/teardown |

### 通知与搜索

| 功能需求 | 必读文件 | 阅读顺序 | 核心关注点 |
|---------|---------|---------|-----------|
| 发送通知 | `ruby/09_notification.md` | 09 | create/types/push |
| 通知合并策略 | `ruby/09_notification.md` → `plugins/19_notification_consolidation.md` | 09→19 | ConsolidateNotifications 规则 |
| 搜索过滤扩展 | `ruby/13_search.md` → `plugins/27_search_filter.md` | 13→27 | register_search_advanced_filter 两种模式 |

### 插件设置与国际化

| 功能需求 | 必读文件 | 阅读顺序 | 核心关注点 |
|---------|---------|---------|-----------|
| 插件设置项 | `plugins/03_plugin_settings.md` | 03 | default/type/client_editable |
| 国际化文本 | `plugins/24_i18n_and_locale.md` | 24 | server.en.yml / client.en.yml 两套 |

---

## Phase 3：横切层（按需补充，不强制）

| 场景 | 文件 | 说明 |
|------|------|------|
| 编写 RSpec 测试 | `ruby/05_rspec_testing.md` → `plugins/25_plugin_testing.md` | 先读核心，再读插件特有组织方式 |
| 编写系统测试 | `system_specs/01_page_objects.md` → `system_specs/02_system_spec_patterns.md` | Page Objects + 通用模式 |
| 代码风格检查 | `tooling/02_linting_formatting.md` | Rubocop/ESLint 配置规范 |
| 开发命令速查 | `tooling/01_commands.md` | 常用 rake/bin 命令 |

---

## 快速诊断：我的任务需要哪些 Phase 2 模块？

```
任务描述中包含...          → 加载的模块
─────────────────────────────────────────────────
"新表" / "存储数据"         → 数据与持久化
"API" / "端点" / "接口"     → 后端逻辑 / REST API 端点
"权限" / "只有xxx能"         → 权限控制
"定时" / "后台" / "异步"    → 后台任务
"Admin" / "管理后台"         → 前端-管理后台
"用户页面" / "前端界面"     → 前端-用户界面
"通知"                       → 通知与搜索
"搜索"                       → 通知与搜索
"国际化" / "多语言"          → 插件设置与国际化
"测试"                       → Phase 3 横切层
```
