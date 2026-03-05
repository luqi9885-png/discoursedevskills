# TOPIC_MAP.md — 主题边界注册表

> 每个关键词只有一个「权威文件」。其他文件涉及相关内容时，只写本文件特有的扩展点，
> 通用逻辑一律引用权威文件，不重复。
>
> **内容摘要格式**：`首行标题；章节：关键词1/关键词2/关键词3`
> 启动时用 `head -5 文件名` 读首行标题与此比对，发现不符立即修复。

| 主题关键词 | 权威文件 | 内容摘要（用于启动核验） | 可引用文件 | 边界说明 |
|-----------|---------|----------------------|-----------|---------|
| Guardian 权限（核心） | ruby/08_guardian.md | Guardian 权限系统；章节：can?/ensure!/scope/register_policy | plugins/23_routes_and_guardian.md | 后者只写插件 GuardianExtensions module + Guardian.include 注册 |
| Ember Service 通用 | javascript/02_services.md | Ember Service 单例模式；章节：@tracked/@service/生命周期 | plugins/14_plugin_service_frontend.md | 后者只写 plugin 特有注册方式（withPluginApi），生命周期不重复 |
| DB Migration | ruby/06_migrations_db.md | 迁移安全规范 + SafeMigrate；章节：add_column/change_column/index | plugins/10_plugin_model_and_db.md | 后者只写 plugin 引入新表的特殊注意点，SafeMigrate 规范不重复 |
| Admin UI 基础 | plugins/09_admin_ui.md | 管理后台 UI；章节：插件页/Report/Connector | plugins/15_admin_multi_page.md | 后者只写多子页面导航扩展，单页 Admin 不重复 |
| apiInitializer 基础 | plugins/02_plugin_api_frontend.md | Plugin API 前端；章节：withPluginApi/apiInitializer | plugins/12_api_initializer_and_ui_injection.md | 后者只写高级注入场景，基础 withPluginApi 结构不重复 |
| ActiveRecord CRUD | ruby/02_models_concerns.md | ActiveSupport::Concern 模式；章节：belongs_to/scope/validate | plugins/21_database_operations.md | 后者只写插件中 AR 与 DB（MiniSQL）混合使用模式 |
| 序列化器（核心） | ruby/04_serializers.md | 三层继承 + 条件属性；章节：attributes/include_xxx?/scope | plugins/22_serializer_extensions.md | 后者只写 add_to_serializer / 插件追加字段，核心 Serializer 类结构不重复 |
| Rails 路由（核心） | ruby/03_controllers.md | ApplicationController 规范；章节：before_action/rescue_from | plugins/23_routes_and_guardian.md | 后者只写插件注册路由方式，路由 DSL 细节不重复 |
| Guardian 权限（插件） | plugins/23_routes_and_guardian.md | 插件路由注册 + GuardianExtensions；章节：can_xxx?/ensure_can_xxx!/Rails routes/Ember routes | plugins/22_serializer_extensions.md | 后者只写 scope.can_xxx? 在序列化器中的用法 |
| i18n / 国际化 | plugins/24_i18n_and_locale.md | 插件国际化；章节：server.en.yml/client.en.yml/I18n.t/i18n()/with_locale | — | 独立主题，无交叉 |
| on() 事件系统 | plugins/18_on_event_system.md | on() 事件系统；章节：first_post_moved/post_destroyed/topic_status_updated/site_setting_changed | — | 独立主题；register_modifier 见 plugins/11 |
| 通知合并策略 | plugins/19_notification_consolidation.md | 通知合并规范；章节：ConsolidateNotifications/DeletePreviousNotifications/register_notification_consolidation_plan | ruby/09_notification.md | 后者只写通知创建/发送，合并策略不重复 |
| Admin RestModel + Adapter | plugins/20_admin_rest_model_adapter.md | Admin RestModel + Adapter；章节：createProperties/updateProperties/pathFor/apiNameFor/jsonMode | plugins/15_admin_multi_page.md | 后者路由加载数据时引用 store API，Model/Adapter 定义不重复 |
| DB（MiniSQL）操作 | plugins/21_database_operations.md | DB/AR 混合操作；章节：DB.query/upsert_all/PluginStore/CustomFields/soft_delete | — | DB.query/exec/query_single、upsert_all、PluginStore、CustomFields 均在此 |
| Serializer 扩展（插件） | plugins/22_serializer_extensions.md | 插件 Serializer 扩展；章节：add_to_serializer/include_xxx?/preloaded_custom_fields/topic_view vs topic_list_item | ruby/04_serializers.md | 后者写核心 Serializer 结构，本文只写插件追加字段 |
| Plugin 基础结构 | plugins/01_plugin_structure.md | Plugin 目录结构；章节：文件布局/命名规范/plugin.rb | plugins/01_plugin_rb_and_backend.md | 后者聚焦 plugin.rb DSL，目录布局不重复 |
| topic-list 自定义列 | plugins/17_topic_list_columns.md | topic-list 自定义列与行样式；章节：addCustomColumn/registerValueTransformer | — | 独立主题 |
| 插件前端路由 | plugins/16_plugin_route_map.md | 插件前端路由；章节：route-map/Route/Template | plugins/23_routes_and_guardian.md | 16 写前端 route-map，23 写后端 Rails routes，不重复 |
| register_modifier（后端） | plugins/11_register_modifier_backend.md | register_modifier 后端管道；章节：modifier注册/返回值/与on()区别 | — | 独立主题；与 on() 区别见 plugins/18 |
| EntryPoint / hijack | plugins/13_backend_advanced_patterns.md | EntryPoint / hijack / Controller Concern；章节：inject_into/hijack/prepend | — | 独立主题 |
| Modal / DModal | plugins/07_modal_and_ui_components.md | DModal + 自定义 Service；章节：showModal/DModal/attrs | — | 独立主题 |
| 用户菜单 / 偏好扩展 | plugins/08_user_ui_extensions.md | 用户菜单/偏好/活动页扩展；章节：addUserMenuItems/addPreference | — | 独立主题 |
| Composer 工具栏 | plugins/12_api_initializer_and_ui_injection.md | apiInitializer 高级/Composer 工具栏；章节：addComposerToolbarPopupMenuOption/addButton | — | 独立主题 |
| Admin 多子页面 | plugins/15_admin_multi_page.md | Admin 多子页面导航；章节：addAdminPluginConfigurationNav/route命名规律 | — | 独立主题 |
| Jobs（核心） | ruby/07_jobs.md | Regular / Scheduled Jobs；章节：Jobs::Base/Jobs::Scheduled/execute/enqueue | — | 独立主题 |
| Reviewable 审核 | ruby/12_reviewable.md | Reviewable 审核系统；章节：build/perform/guardian | — | 独立主题 |
| Plugin API 杂项 | plugins/29_plugin_api_misc.md | Plugin API 杂项；章节：registerTopicFooterButton/addNavigationBarItem/addBulkActionButton/addKeyboardShortcut/addPostSmallActionIcon/replaceIcon | plugins/12_api_initializer_and_ui_injection.md | 后者写 Composer/高级注入，本文写其余散落 api 方法 |
| 搜索过滤扩展（插件） | plugins/27_search_filter.md | 搜索扩展规范；章节：register_search_advanced_filter/add_filter_custom_filter/addAdvancedSearchOptions | ruby/13_search.md | 后者写 Search 核心架构，本文只写插件扩展 DSL |
| Admin 报告 + 用户目录列 | plugins/28_reports.md | Admin 报告；章节：Report.add_report/折线图/表格/add_directory_column | plugins/09_admin_ui.md | 后者写 Admin 页面布局，本文只写报告数据 |
| Search 扩展 | ruby/13_search.md | Search 扩展；章节：register_modifier/custom_filter | — | 独立主题 |
| RateLimiter | ruby/10_rate_limiter.rb | RateLimiter；章节：performed!/RateLimiter.new | — | 独立主题 |
| PostCreator | ruby/11_post_topic_creator.md | PostCreator；章节：create/TopicCreator | — | 独立主题 |
| Notification（核心） | ruby/09_notification.md | 后端通知系统；章节：create/types/push | plugins/19_notification_consolidation.md | 后者只写合并策略，通知创建/类型不重复 |
| System Spec | system_specs/01_page_objects.md | Page Objects 系统测试；章节：PageObject/has_css?/fill_in | system_specs/02_system_spec_patterns.md | 后者引用前者的 Page Objects，不重复定义 |
| FormKit 表单框架 | javascript/08_form_kit.md | FormKit 规范；章节：@data/@validation/Field 控件体系/form.Object/InputGroup/onRegisterApi | javascript/07_select_kit.md | SelectKit 作为 field.Custom 嵌入时需绑定 field.value/field.set |
| SelectKit 组件 | javascript/07_select_kit.md | SelectKit 规范；章节：ComboBox/MultiSelect/EmailGroupUserChooser/modifySelectKit | — | 独立主题；withPluginApi 基础见 plugins/02_plugin_api_frontend.md |
| 预加载 / N+1 防治 | plugins/26_preloading.md | 预加载 N+1 防治；章节：TopicList.on_preload/TopicView.on_preload/BookmarkQuery.on_preload/Search.on_preload/register_topic_preloader_associations | plugins/22_serializer_extensions.md | 后者写 Serializer 读取预加载值，本文写 on_preload 存数据 |
| 插件 RSpec 测试 | plugins/25_plugin_testing.md | 插件 RSpec 测试规范；章节：Fabricator定义/request spec/model spec/job spec/shared_context | ruby/05_rspec_testing.md | 后者写核心 RSpec 模式，本文只写插件特有组织方式和 Fabricator 定义 |
| MessageBus 前端 | javascript/06_message_bus.md | MessageBus 前端订阅规范；章节：subscribe/unsubscribe/@bind/instance-initializer/teardown | — | 前端 subscribe/unsubscribe；后端 publish 权限控制见 ruby/03_controllers.md |
