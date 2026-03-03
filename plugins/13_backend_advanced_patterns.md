# Plugin 13 — 后端高级模式：EntryPoint、hijack、Controller Concern 与 plugin.rb 高级 DSL

> 来源：`plugins/discourse-ai/plugin.rb`、`plugins/discourse-ai/lib/ai_helper/entry_point.rb`、`plugins/discourse-ai/lib/summarization/entry_point.rb`、`plugins/discourse-ai/app/controllers/discourse_ai/ai_helper/assistant_controller.rb`、`plugins/discourse-ai/app/controllers/concerns/ai_credit_limit_handler.rb`、`plugins/discourse-ai/app/controllers/discourse_ai/admin/ai_personas_controller.rb`

## 概述

大型插件（如 discourse-ai）将 `after_initialize` 的注入逻辑拆分到各功能模块的 `EntryPoint` 类中，由 plugin.rb 统一遍历调用，保持 plugin.rb 简洁。`hijack` 用于将长时阻塞请求（如 AI 生成）从 Unicorn worker 解放出来异步执行。Controller Concern 封装跨控制器的公共行为（如限流、错误响应）。

---

## 来源验证

- `plugins/discourse-ai/plugin.rb`（EntryPoint 调用模式、add_admin_route、register_reviewable_type、Rails.autoloaders）
- `plugins/discourse-ai/lib/ai_helper/entry_point.rb`（inject_into 模式）
- `plugins/discourse-ai/lib/summarization/entry_point.rb`（inject_into + register_modifier + on 事件）
- `plugins/discourse-ai/app/controllers/discourse_ai/ai_helper/assistant_controller.rb`（hijack + RATE_LIMITS + Controller Concern include）
- `plugins/discourse-ai/app/controllers/concerns/ai_credit_limit_handler.rb`（Concern rescue_from 模式）
- `plugins/discourse-ai/app/controllers/discourse_ai/admin/ai_personas_controller.rb`（Admin Controller 完整示例）

---

## 核心规范

### 1. EntryPoint 模式（大型插件功能模块化）

将 `after_initialize` 的注入逻辑按功能拆分到独立类，plugin.rb 只做调度：

```ruby
# lib/my_feature/entry_point.rb
module MyPlugin
  module MyFeature
    class EntryPoint
      def inject_into(plugin)
        # 所有 plugin.xxx DSL 通过 plugin 参数调用
        plugin.add_to_serializer(:current_user, :can_use_my_feature) do
          scope.user.in_any_groups?(SiteSetting.my_feature_allowed_groups_map)
        end

        plugin.add_to_serializer(:topic_view, :my_topic_field) do
          object.topic.my_custom_data
        end

        plugin.register_modifier(:topic_query_create_list_topics) do |topics, options|
          next topics unless SiteSetting.my_feature_enabled
          topics.includes(:my_related_model)
        end

        plugin.on(:post_created) do |post|
          next unless SiteSetting.my_feature_enabled && post.topic
          Jobs.enqueue(:my_background_job, topic_id: post.topic_id)
        end
      end
    end
  end
end
```

plugin.rb 统一调用：

```ruby
# plugin.rb
after_initialize do
  # 遍历所有模块 EntryPoint，统一注入
  [
    MyPlugin::MyFeature::EntryPoint.new,
    MyPlugin::AnotherFeature::EntryPoint.new,
    MyPlugin::ThirdFeature::EntryPoint.new,
  ].each { |mod| mod.inject_into(self) }
end
```

**与小插件的区别**：小插件（如 discourse-solved）直接在 `after_initialize` 内写所有逻辑；大型插件（如 discourse-ai）用 EntryPoint 保持 plugin.rb 短小干净。

---

### 2. Rails.autoloaders 自动加载 lib/ 目录

discourse-ai 用 Rails autoloader 而不是 `require_relative` 批量加载：

```ruby
# plugin.rb 顶部（after_initialize 之外）
module ::DiscourseAi
  PLUGIN_NAME = "discourse-ai"
end

# 将 lib/ 目录注册到 Rails autoloader，命名空间为 DiscourseAi
# 这样 lib/my_feature/entry_point.rb 就能自动对应 DiscourseAi::MyFeature::EntryPoint
Rails.autoloaders.main.push_dir(File.join(__dir__, "lib"), namespace: DiscourseAi)

require_relative "lib/engine"  # Engine 文件仍需手动 require

after_initialize do
  # 可以直接使用 DiscourseAi::MyFeature::EntryPoint 无需手动 require
  DiscourseAi::MyFeature::EntryPoint.new.inject_into(self)
end
```

**与小插件的对比：**

```ruby
# 小插件（require_relative 模式，适合 < 20 个文件）
after_initialize do
  %w[
    app/models/my_plugin/my_model.rb
    app/controllers/my_plugin/my_controller.rb
  ].each { |path| require_relative path }
end

# 大插件（autoloader 模式，文件多时更优雅）
Rails.autoloaders.main.push_dir(File.join(__dir__, "lib"), namespace: MyPlugin)
# 直接使用 MyPlugin::FeatureName::ClassName，Rails 按路径自动加载
```

---

### 3. hijack — 长时请求解放 Worker

用于需要等待外部 IO 或 AI 生成的 Controller action，将处理移到后台线程，立即释放 Unicorn Worker：

```ruby
# app/controllers/discourse_ai/summarization/summary_controller.rb

def summarize
  raise Discourse::InvalidAccess unless guardian.can_request_summary?

  topic = Topic.find(params[:topic_id])

  # hijack：立即释放 worker，在后台线程执行 block
  # block 内的 render 正常工作，会在连接保持打开期间写回响应
  hijack do
    result = DiscourseAi::Summarization.summarize(topic, current_user)
    render json: { summary: result }, status: :ok
  end
end

# rescue 写在 hijack 外部，hijack 内部的异常需在 block 内处理
def suggest
  hijack do
    begin
      result = SomeSlowOperation.call(params)
      render json: result, status: :ok
    rescue MyPlugin::SpecificError => e
      render_json_error e.message, status: :unprocessable_entity
    rescue SomeExternalApi::Error
      render_json_error I18n.t("my_plugin.errors.api_failed"), status: 502
    end
  end
end
```

**hijack 注意事项：**
- 适用场景：外部 API 调用、AI 推理、文件处理等 IO 密集型操作（通常 > 500ms）
- block 内的 `render` 与普通 action 完全一样
- `rescue` 必须写在 `hijack do...end` **内部**，外部的 `rescue_from` 对 hijack 后的异常不生效
- 测试中 hijack 降级为同步执行（`rack.hijack` 不存在时直接调用 block）

---

### 4. Controller Concern — 跨控制器公共行为

将多个 Controller 共享的行为（rescue_from、before_action、私有方法）抽成 Concern：

```ruby
# app/controllers/concerns/ai_credit_limit_handler.rb
module AiCreditLimitHandler
  extend ActiveSupport::Concern

  included do
    # included 块：在引入该 Concern 的 Controller 类上下文中执行
    # 这里的 rescue_from 会注册到引入的 Controller
    rescue_from LlmCreditAllocation::CreditLimitExceeded do |e|
      render_credit_limit_error(e)
    end
  end

  private

  def render_credit_limit_error(exception)
    allocation = exception.allocation

    message = I18n.t(
      "my_plugin.limit_exceeded_#{current_user&.admin? ? "admin" : "user"}",
      reset_time: allocation&.formatted_reset_time.presence || "",
    )

    render json: {
      error: "credit_limit_exceeded",
      message: message,
      details: {
        reset_time_relative: allocation&.relative_reset_time,
        reset_time_absolute: allocation&.formatted_reset_time,
      },
    }, status: :too_many_requests
  end
end
```

在 Controller 中引入：

```ruby
# app/controllers/discourse_ai/ai_helper/assistant_controller.rb
module DiscourseAi
  module AiHelper
    class AssistantController < ::ApplicationController
      include AiCreditLimitHandler  # 引入后 rescue_from 自动注册

      requires_plugin PLUGIN_NAME
      requires_login
    end
  end
end
```

---

### 5. RATE_LIMITS 常量 + 按 action 分级限流

AI 插件对不同 action 配置不同限流规则：

```ruby
RATE_LIMITS = {
  "default" => { amount: 6, interval: 3.minutes },
  "caption_image" => { amount: 20, interval: 1.minute },
  "suggest" => { amount: 10, interval: 5.minutes },
}.freeze

before_action :rate_limiter_performed!

private

def rate_limiter_performed!
  # 根据当前 action 选择限流配置，没有匹配就用 default
  action_rate_limit = RATE_LIMITS[action_name] || RATE_LIMITS["default"]

  RateLimiter.new(
    current_user,
    "my_plugin_#{action_name}",  # key 带 action name 防止跨 action 干扰
    action_rate_limit[:amount],
    action_rate_limit[:interval],
  ).performed!
end
```

---

### 6. plugin.rb 高级 DSL

plugin.rb 中的高级注册方法（超出基础 DSL 的部分）：

```ruby
after_initialize do
  # 1. 注册 Admin 侧边栏导航（会在 Admin → 侧边栏自动出现）
  add_admin_route("my_plugin.admin.title", "my-plugin", { use_new_show_route: true })
  # use_new_show_route: true → 使用 adminPlugins.show.my-plugin 路由（新式）
  # 不加 use_new_show_route → 使用旧式 adminPlugins.my-plugin

  # 2. 注册初始化数据（seedfu fixtures）
  register_seedfu_fixtures(
    Rails.root.join("plugins", "my-plugin", "db", "fixtures", "my_category")
  )

  # 3. 注册可审核类型
  register_reviewable_type MyPlugin::ReviewableMyItem

  # 4. 注册站点健康检查
  register_problem_check ProblemCheck::MyPluginHealth
  # ProblemCheck 放在 app/models/problem_check/ 目录

  # 5. 扩展 API Key scope（限制 API Key 只能调特定 action）
  add_api_key_scope(
    :my_plugin,
    {
      update_items: {
        actions: %w[my_plugin/admin/items#update],
      },
    },
  )

  # 6. 注册 Site Setting 区域（Admin UI 中的设置分组）
  register_site_setting_area("my-feature/general")
  register_site_setting_area("my-feature/advanced")

  # 7. 注册 model 回调（外部 model 上挂事件）
  add_model_callback(SomeExternalModel, :after_save) do
    MyPlugin::SomeCache.flush!
  end
end
```

---

### 7. add_to_serializer include_condition

在 EntryPoint 中用 `include_condition` 控制字段是否被序列化（减少不必要计算）：

```ruby
# lib/my_feature/entry_point.rb
plugin.add_to_serializer(
  :topic_view,
  :my_cached_record,
  include_condition: -> { false },  # 永远不序列化（仅供内部计算用）
) do
  # 这个值作为缓存/中间计算，供其他字段引用
  @my_cached_record ||= object.topic.my_records.find_by(type: "main")
end

plugin.add_to_serializer(
  :topic_view,
  :my_full_data,
  include_condition: -> do
    # 动态条件：只在满足时才序列化
    SiteSetting.my_feature_enabled &&
      scope.can_see_my_feature?(object.topic) &&
      my_cached_record.present?
  end,
) do
  {
    id: my_cached_record.id,
    content: my_cached_record.content,
    updated_at: my_cached_record.updated_at,
  }
end
```

---

### 8. Admin Controller 继承 Admin::AdminController

插件的 Admin 相关 Controller 应继承 `::Admin::AdminController` 而不是 `::ApplicationController`：

```ruby
module MyPlugin
  module Admin
    class ItemsController < ::Admin::AdminController
      # Admin::AdminController 已经包含：
      # - before_action :ensure_logged_in
      # - before_action :ensure_staff（非 staff 自动 403）
      # - before_action :ensure_admin（某些操作要求 admin）

      requires_plugin PLUGIN_NAME
      # requires_plugin 会在插件禁用时返回 404

      before_action :find_item, only: %i[edit update destroy]

      def index
        items = MyPlugin::Item.ordered.includes(:user)
        render json: items.map { |i| MyItemSerializer.new(i, root: false) }
      end

      def create
        item = MyPlugin::Item.new(item_params)
        if item.save
          render json: MyItemSerializer.new(item, root: false), status: :created
        else
          render_json_error item
        end
      end

      def update
        if @item.update(item_params)
          render json: MyItemSerializer.new(@item, root: false)
        else
          render_json_error @item
        end
      end

      def destroy
        if @item.destroy
          head :no_content
        else
          render_json_error @item
        end
      end

      private

      def find_item
        @item = MyPlugin::Item.find(params[:id])
      end

      def item_params
        params.require(:item).permit(:name, :description, :enabled, group_ids: [])
      end
    end
  end
end
```

---

## 反模式（避免这样做）

```ruby
# hijack 内部的异常没有被捕获
hijack do
  result = ExternalApi.call(params)
  render json: result  # 如果 ExternalApi.call 抛异常，响应会 hang 住
end

# 正确：hijack 内必须 rescue 所有可能的异常
hijack do
  begin
    result = ExternalApi.call(params)
    render json: result
  rescue ExternalApi::Error => e
    render_json_error e.message, status: 502
  rescue StandardError => e
    Discourse.warn_exception(e, message: "My plugin external call failed")
    render_json_error "Internal error", status: 500
  end
end
```

```ruby
# Concern 的 rescue_from 写在 included 之外（不会注册到 Controller）
module MyConcern
  extend ActiveSupport::Concern

  rescue_from MyError do |e|  # 错误！这不在 included 块内
    render_json_error e.message
  end
end

# 正确：rescue_from 必须在 included do...end 块内
module MyConcern
  extend ActiveSupport::Concern

  included do
    rescue_from MyError do |e|  # 正确！会注册到引入的 Controller
      render_json_error e.message
    end
  end
end
```

```ruby
# Rails.autoloaders 在 after_initialize 内注册（太晚）
after_initialize do
  Rails.autoloaders.main.push_dir(...)  # 错误！应在 plugin.rb 顶层，after_initialize 之外
end

# 正确：在 plugin.rb 顶层（after_initialize 之前）
Rails.autoloaders.main.push_dir(File.join(__dir__, "lib"), namespace: MyPlugin)

after_initialize do
  # 此时 autoloader 已注册，MyPlugin::Feature 类可以正常使用
  MyPlugin::Feature::EntryPoint.new.inject_into(self)
end
```

---

## 关联规范

- `plugins/01_plugin_rb_and_backend.md` — plugin.rb 基础 DSL
- `plugins/10_plugin_model_and_db.md` — Rails Engine + Model 基础
- `ruby/10_rate_limiter.md` — RateLimiter 完整规范
- `ruby/03_controllers.md` — ApplicationController 规范

---

## 快速参考

```ruby
# EntryPoint 模式
class MyFeature::EntryPoint
  def inject_into(plugin)
    plugin.add_to_serializer(:current_user, :can_use) { scope.user.in_any_groups?(...) }
    plugin.register_modifier(:some_modifier) { |val| val }
    plugin.on(:post_created) { |post| Jobs.enqueue(:my_job, post_id: post.id) }
  end
end

# plugin.rb 中调用
after_initialize do
  [MyFeature::EntryPoint.new, OtherFeature::EntryPoint.new].each { |e| e.inject_into(self) }
end

# hijack（长时 IO 请求）
def slow_action
  hijack do
    begin
      result = ExternalService.call(params)
      render json: result
    rescue ExternalService::Error => e
      render_json_error e.message, status: 502
    end
  end
end

# Controller Concern
module MyConcern
  extend ActiveSupport::Concern
  included do
    rescue_from MyError { |e| render json: { error: e.message }, status: 429 }
  end
  private
  def shared_helper_method; end
end

# Admin Controller
class Admin::ItemsController < ::Admin::AdminController
  requires_plugin PLUGIN_NAME
  def index; render json: MyItem.all.map { |i| MySerializer.new(i, root: false) }; end
end

# Rails autoloader（plugin.rb 顶层）
Rails.autoloaders.main.push_dir(File.join(__dir__, "lib"), namespace: MyPlugin)
```
