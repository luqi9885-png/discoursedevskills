# TOPIC_MAP.md — 主题边界注册表

> 每个关键词只有一个「权威文件」。其他文件涉及相关内容时，只写本文件特有的扩展点，
> 通用逻辑一律引用权威文件，不重复。

| 主题关键词 | 权威文件（唯一） | 可引用但不重复写的文件 | 边界说明 |
|-----------|----------------|---------------------|---------|
| Guardian 权限（核心） | ruby/08_guardian.md | plugins/23_routes_and_guardian.md | 后者只写插件中 GuardianExtensions module + Guardian.include 注册，`can?` 内部逻辑不重复 |
| Ember Service 通用 | javascript/02_services.md | plugins/14_plugin_service_frontend.md | 后者只写 plugin 特有的注册方式（withPluginApi），Service 生命周期不重复 |
| DB Migration | ruby/06_migrations_db.md | plugins/10_plugin_model_and_db.md | 后者只写 plugin 引入新表的特殊注意点，SafeMigrate 规范不重复 |
| Admin UI 基础 | plugins/09_admin_ui.md | plugins/15_admin_multi_page.md | 后者只写多子页面导航扩展，单页 Admin 不重复 |
| apiInitializer 基础 | plugins/02_plugin_api_frontend.md | plugins/12_api_initializer_and_ui_injection.md | 后者只写高级注入场景，基础 withPluginApi 结构不重复 |
| ActiveRecord CRUD | ruby/02_models_concerns.md | plugins/21_database_operations.md | 后者只写插件中 AR 与 DB（MiniSQL）混合使用模式，AR 核心 API 不重复 |
| 序列化器（核心） | ruby/04_serializers.md | plugins/22_serializer_extensions.md | 后者只写 add_to_serializer / 插件追加字段，核心 Serializer 类结构不重复 |
| Rails 路由（核心） | ruby/03_controllers.md | plugins/23_routes_and_guardian.md | 后者只写插件注册路由的方式，路由 DSL 细节不重复 |
| Guardian 权限（插件） | plugins/23_routes_and_guardian.md | plugins/22_serializer_extensions.md | 后者只写 scope.can_xxx? 在序列化器中的用法，不重复 GuardianExtensions 定义 |
| i18n / 国际化 | plugins/24_i18n_and_locale.md | — | 独立主题，无交叉 |
| on() 事件系统 | plugins/18_on_event_system.md | — | 独立主题；register_modifier 见 plugins/11 |
| 通知合并策略 | plugins/19_notification_consolidation.md | ruby/09_notification.md | 后者只写通知创建/发送，合并策略不重复 |
| Admin RestModel + Adapter | plugins/20_admin_rest_model_adapter.md | plugins/15_admin_multi_page.md | 后者路由加载数据时引用 store API，Model/Adapter 定义不重复 |
| DB（MiniSQL）操作 | plugins/21_database_operations.md | — | DB.query/exec/query_single、upsert_all、PluginStore、CustomFields 均在此 |
| Plugin 基础结构 | plugins/01_plugin_structure.md | plugins/01_plugin_rb_and_backend.md | 后者聚焦 plugin.rb DSL，目录布局不重复 |
| topic-list 自定义列 | plugins/17_topic_list_columns.md | — | 独立主题 |
| topic-list 路由/模板 | plugins/16_plugin_route_map.md | — | 独立主题；与 23 路由有交叉：16 写前端 route-map，23 写后端 Rails routes |
| register_modifier（后端） | plugins/11_register_modifier_backend.md | — | 独立主题；与 on() 事件系统区别见 plugins/18 |
| EntryPoint / hijack | plugins/13_backend_advanced_patterns.md | — | 独立主题 |
| Modal / DModal | plugins/07_modal_and_ui_components.md | — | 独立主题 |
| 用户菜单 / 偏好扩展 | plugins/08_user_ui_extensions.md | — | 独立主题 |
| Composer 工具栏 / 批量操作 | plugins/12_api_initializer_and_ui_injection.md | — | 独立主题 |
| Admin 多子页面导航 | plugins/15_admin_multi_page.md | — | 独立主题 |
| MessageBus 前端 | javascript/06_message_bus.md | — | 前端 subscribe/unsubscribe；后端 publish 权限控制见 ruby/03_controllers.md |
| Jobs（核心） | ruby/07_jobs.md | — | Regular + Scheduled Jobs 完整规范 |
| Reviewable 审核 | ruby/12_reviewable.md | — | 独立主题 |
| Search 扩展 | ruby/13_search.md | — | 独立主题 |
| RateLimiter | ruby/10_rate_limiter.md | — | 独立主题 |
| PostCreator | ruby/11_post_topic_creator.md | — | 独立主题 |
| Notification（核心） | ruby/09_notification.md | plugins/19_notification_consolidation.md | 后者只写合并策略，通知创建/类型不重复 |
| System Spec | system_specs/01_page_objects.md | system_specs/02_system_spec_patterns.md | 后者引用前者的 Page Objects，不重复定义 |
