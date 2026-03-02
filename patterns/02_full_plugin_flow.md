# Patterns 02 — 完整 Plugin 开发流程

> 来源：`plugins/discourse-solved/`（完整分析）
> 验证：结构对比 `plugins/poll/`、`plugins/discourse-assign/`

---

## 概述

一个完整的 Discourse Plugin 由以下几层组成，每层职责明确：

```
plugin.rb              ← 入口：注册所有钩子、扩展、设置
config/
  settings.yml         ← SiteSettings 定义
  routes.rb            ← 后端路由
app/
  controllers/         ← 处理请求（继承 ApplicationController）
  models/              ← Plugin 专属数据模型
  serializers/         ← 序列化扩展
lib/
  engine.rb            ← Rails Engine（隔离命名空间）
  xxx_extension.rb     ← prepend 进核心类的模块
db/
  migrate/             ← 数据库迁移（表名加命名空间前缀）
assets/javascripts/
  initializers/        ← 前端入口，调用 withPluginApi
  pre-initializers/    ← 在 Ember 对象注入之前运行
  components/          ← Glimmer 组件 (.gjs)
  connectors/          ← 插入到核心的 outlet 插槽
  routes/              ← Plugin 专属路由
  xxx-route-map.js     ← 声明新路由，挂载到已有路由树
```

---

## 1. plugin.rb：后端入口

### 1.1 文件头声明

```ruby
# name: discourse-solved
# about: 简短描述
# meta_topic_id: 30155
# version: 0.1
# authors: Sam Saffron
# url: https://github.com/...

enabled_site_setting :solved_enabled   # 关联一个 SiteSetting 作为总开关
```

### 1.2 资源注册

```ruby
register_svg_icon "far-square-check"
register_asset "stylesheets/solutions.scss"
register_asset "stylesheets/mobile/solutions.scss", :mobile
```

### 1.3 命名空间与 Engine

```ruby
module ::DiscourseSolved
  PLUGIN_NAME = "discourse-solved"
  ENABLE_ACCEPTED_ANSWERS_CUSTOM_FIELD = "enable_accepted_answers"
end

require_relative "lib/discourse_solved/engine"
```

### 1.4 after_initialize 块

**所有核心逻辑放在 `after_initialize` 里**，确保 Rails 初始化完成后再执行：

```ruby
after_initialize do
  # 1. 核心业务方法（module 方法）
  module ::DiscourseSolved
    def self.accept_answer!(post, acting_user, topic: nil)
      DistributedMutex.synchronize("discourse_solved_toggle_answer_#{topic.id}") do
        ActiveRecord::Base.transaction do
          solved = DiscourseSolved::SolvedTopic.new(topic:, answer_post: post, accepter: acting_user)
          solved.save!
        end
        MessageBus.publish("/topic/#{topic.id}", { type: :accepted_solution, accepted_answer: })
        DiscourseEvent.trigger(:accepted_solution, post)
      end
    end
  end

  # 2. prepend 扩展核心类（reloadable_patch 使开发模式下可热重载）
  reloadable_patch do
    ::Guardian.prepend(DiscourseSolved::GuardianExtensions)
    ::Topic.prepend(DiscourseSolved::TopicExtension)
    ::PostSerializer.prepend(DiscourseSolved::PostSerializerExtension)
    ::TopicViewSerializer.prepend(DiscourseSolved::TopicViewSerializerExtension)
    # 多个 Serializer 批量 include 同一个 mixin
    [::TopicListItemSerializer, ::SuggestedTopicSerializer].each do |klass|
      klass.include(DiscourseSolved::TopicAnswerMixin)
    end
  end

  # 3. 向已有 Serializer 添加字段（DSL，不用 prepend）
  add_to_serializer(:post, :can_accept_answer) { scope.can_accept_answer?(topic, object) }
  add_to_serializer(:post, :accepted_answer) { topic&.solved&.answer_post_id == object.id }
  add_to_serializer(:user_card, :accepted_answers) { DiscourseSolved::Queries.solved_count(object.id) }

  # 4. 监听核心事件
  on(:post_destroyed) { |post| DiscourseSolved.unaccept_answer!(post) }

  # 5. 注册 modifier（修改核心行为）
  register_modifier(:search_rank_sort_priorities) do |priorities, _search|
    priorities.push([condition, 1.1]) if SiteSetting.prioritize_solved_topics_in_search
  end

  # 6. 其他注册
  add_api_key_scope(:solved, { answer: { actions: %w[discourse_solved/answer#accept] } })
  register_html_builder("server:before-head-close") { |c| DiscourseSolved::BeforeHeadClose.new(c).html }
end
```

---

## 2. 后端：config/

### 2.1 config/settings.yml

```yaml
discourse_solved:
  solved_enabled:
    default: true
    client: true        # client: true → 前端 siteSettings 可读

  accept_all_solutions_allowed_groups:
    default: "1|2|14"
    mandatory_values: "1|2"   # 不可删除的最小集合
    type: group_list
    client: false
    allow_any: false
    refresh: true
    validator: "AtLeastOneGroupValidator"

  solved_topics_auto_close_hours:
    default: 0
    min: 0
    max: 175200          # 20 years

  solved_add_schema_markup:
    type: enum
    default: "always"
    choices: ["never", "always", "answered only"]
```

### 2.2 config/routes.rb

```ruby
DiscourseSolved::Engine.routes.draw do
  post "/accept"    => "answer#accept"
  post "/unaccept"  => "answer#unaccept"
  get  "/by_user"   => "solved_topics#by_user"
end

# 挂载 Engine：所有 plugin 路由统一加前缀
Discourse::Application.routes.draw { mount DiscourseSolved::Engine, at: "solution" }
# → POST /solution/accept, POST /solution/unaccept
```

---

## 3. 后端：lib/

### 3.1 lib/engine.rb

```ruby
module DiscourseSolved
  class Engine < ::Rails::Engine
    engine_name PLUGIN_NAME
    isolate_namespace DiscourseSolved     # 类名、路由都隔离在模块内
    config.autoload_paths << File.join(config.root, "lib")
  end
end
```

### 3.2 Guardian 扩展

```ruby
module DiscourseSolved
  module GuardianExtensions
    def can_accept_answer?(topic, post)
      return false if !authenticated?
      return false if !allow_accepted_answers?(topic.category_id, topic.tags.map(&:name))
      return true if is_staff?
      return true if current_user.in_any_groups?(SiteSetting.accept_all_solutions_allowed_groups_map)
      topic.user_id == current_user.id && !topic.closed
    end
  end
end
# plugin.rb: ::Guardian.prepend(DiscourseSolved::GuardianExtensions)
# → ensure_can_accept_answer! 自动生成（EnsureMagic）
```

### 3.3 Serializer 扩展（prepend）

```ruby
module DiscourseSolved::TopicViewSerializerExtension
  extend ActiveSupport::Concern

  prepended { attributes :accepted_answer }      # 声明新字段

  def include_accepted_answer?                    # 条件渲染
    SiteSetting.solved_enabled? && object.topic.solved.present?
  end

  def accepted_answer
    object.topic.accepted_answer_post_info
  end
end
```

---

## 4. 后端：app/

### 4.1 Plugin 专属 Model

```ruby
module DiscourseSolved
  class SolvedTopic < ActiveRecord::Base
    self.table_name = "discourse_solved_solved_topics"   # 明确指定表名（带命名空间前缀）

    belongs_to :topic
    belongs_to :answer_post, class_name: "Post", foreign_key: "answer_post_id"
    belongs_to :accepter, class_name: "User", foreign_key: "accepter_user_id"
    belongs_to :topic_timer, dependent: :destroy

    validates :topic_id, presence: true
    validates :answer_post_id, presence: true
    validates :accepter_user_id, presence: true
  end
end
```

### 4.2 Controller

```ruby
class DiscourseSolved::AnswerController < ::ApplicationController
  requires_plugin DiscourseSolved::PLUGIN_NAME    # 未启用则 404

  def accept
    limit_accepts                                   # 限流
    post = Post.find(params[:id].to_i)
    topic = post.topic

    guardian.ensure_can_accept_answer!(topic, post) # 权限守卫（EnsureMagic）

    accepted_answer = DiscourseSolved.accept_answer!(post, current_user, topic:)
    render_json_dump(accepted_answer)
  end

  private

  def limit_accepts
    return if current_user.staff?
    RateLimiter.new(nil, "accept-hr-#{current_user.id}", 20, 1.hour).performed!
    RateLimiter.new(nil, "accept-min-#{current_user.id}", 4, 30.seconds).performed!
  end
end
```

### 4.3 db/migrate/

```ruby
# 表名加命名空间前缀，避免与核心或其他 plugin 冲突
class CreateDiscourseSolvedSolvedTopics < ActiveRecord::Migration[7.2]
  def change
    create_table :discourse_solved_solved_topics do |t|
      t.integer :topic_id, null: false
      t.integer :answer_post_id, null: false
      t.integer :accepter_user_id, null: false
      t.integer :topic_timer_id
      t.timestamps
    end
  end
end
```


---

## 5. 前端结构

### 5.1 initializer：前端入口

```js
// assets/javascripts/discourse/initializers/extend-for-solved-button.gjs
export default {
  name: "extend-for-solved-button",
  initialize() {
    withPluginApi(initializeWithApi);       // 可多次调用 withPluginApi 拆分不同功能
    withPluginApi((api) => {
      api.replaceIcon("notification.solved.accepted_notification", "square-check");
    });
  },
};

function initializeWithApi(api) {
  // 扩展核心 model
  api.modifyClass("model:topic", (Superclass) =>
    class extends Superclass {
      @tracked accepted_answer;

      setAcceptedSolution(acceptedAnswer) {
        this.postStream?.posts?.forEach((post) => {
          post.setProperties(
            acceptedAnswer?.post_number === post.post_number
              ? { accepted_answer: true, topic_accepted_answer: true }
              : { accepted_answer: false, topic_accepted_answer: !!acceptedAnswer }
          );
        });
        this.accepted_answer = acceptedAnswer;
      }
    }
  );

  // 注册 post-menu 按钮
  api.registerValueTransformer("post-menu-buttons", ({ value: dag, context: { post } }) => {
    const solvedButton = post.accepted_answer
      ? SolvedUnacceptAnswerButton
      : post.can_accept_answer
        ? SolvedAcceptAnswerButton
        : null;
    solvedButton && dag.add("solved", solvedButton, { before: firstButtonKey });
  });

  // 注册实时消息回调
  api.registerCustomPostMessageCallback("accepted_solution", async (controller, message) => {
    controller.model?.setAcceptedSolution(message.accepted_answer);
  });
}
```

### 5.2 pre-initializer：在 Ember 注入前运行

```js
// assets/javascripts/discourse/pre-initializers/extend-category-for-solved.js
export default {
  name: "extend-category-for-solved",
  before: "inject-discourse-objects",   // 在核心注入之前运行

  initialize() {
    Category.reopen({
      enable_accepted_answers: computed("custom_fields.enable_accepted_answers", {
        get(fieldName) { return get(this.custom_fields, fieldName) === "true"; },
      }),
    });
  },
};
```

**使用场景**：需要在 Ember 初始化早期修改 Model 类时用 pre-initializer；其余用 initializer + withPluginApi。

### 5.3 connector：注入到核心 outlet

```gjs
// assets/javascripts/discourse/connectors/after-topic-status/solved-status.gjs
// 路径规则：connectors/<outlet-name>/<任意名称>.gjs

const SolvedStatus = <template>
  {{#if (or @outletArgs.topic.has_accepted_answer @outletArgs.topic.accepted_answer)}}
    <span class="topic-status --solved">{{icon "far-square-check"}}</span>
  {{else if (and @outletArgs.topic.can_have_answer (eq @outletArgs.context "topic-list"))}}
    <span class="topic-status --unsolved">{{icon "far-square"}}</span>
  {{/if}}
</template>;

export default SolvedStatus;
```

`@outletArgs` 接收 outlet 传入的上下文数据。

### 5.4 plugin 专属路由

```js
// assets/javascripts/discourse/solved-route-map.js
export default {
  resource: "user.userActivity",    // 挂载到已有路由下

  map() {
    this.route("solved");           // → user.userActivity.solved
  },
};
```

### 5.5 Component：调用后端 API

```gjs
// components/solved-accept-answer-button.gjs
export default class SolvedAcceptAnswerButton extends Component {
  static hidden(args) { return args.post.topic_accepted_answer; }  // 控制是否渲染

  @service appEvents;
  @tracked saving = false;

  @action
  async acceptAnswer() {
    this.saving = true;
    try {
      const acceptedAnswer = await ajax("/solution/accept", {
        type: "POST",
        data: { id: this.args.post.id },
      });
      this.args.post.topic.setAcceptedSolution(acceptedAnswer);  // 更新前端状态
      this.appEvents.trigger("discourse-solved:solution-toggled", this.args.post);
    } catch (e) {
      popupAjaxError(e);
    } finally {
      this.saving = false;
    }
  }

  <template>
    <DButton @action={{this.acceptAnswer}} @disabled={{this.saving}} @icon="far-square-check" />
  </template>
}
```

---

## 6. 完整请求链路（"采纳答案"为例）

```
用户点击"采纳"按钮
  ↓
SolvedAcceptAnswerButton#acceptAnswer()
  → ajax POST /solution/accept { id: post.id }
  ↓
DiscourseSolved::AnswerController#accept
  → requires_plugin（未启用则 404）
  → limit_accepts（限流）
  → guardian.ensure_can_accept_answer!(topic, post)   ← 权限
  → DiscourseSolved.accept_answer!(post, current_user)
    → DistributedMutex（防并发）
    → ActiveRecord::Base.transaction
      → SolvedTopic.new(...).save!
      → Notification.create!(...)
      → topic_timer 设置自动关闭
    → MessageBus.publish("/topic/#{id}", accepted_solution)  ← 实时广播
    → DiscourseEvent.trigger(:accepted_solution, post)       ← 事件钩子
  → render_json_dump(accepted_answer)
  ↓
前端收到 HTTP 响应
  → topic.setAcceptedSolution(acceptedAnswer)     ← 更新当前用户视图
  ↓
其他用户通过 MessageBus 收到实时消息
  → registerCustomPostMessageCallback 回调
  → topic.setAcceptedSolution(message.accepted_answer)
```

---

## 7. Plugin 开发 Checklist

| 步骤 | 文件 | 关键点 |
|------|------|--------|
| 1 | `plugin.rb` 头 | 头注释、`enabled_site_setting`、`require_relative engine` |
| 2 | `lib/.../engine.rb` | `isolate_namespace`、`config.autoload_paths` |
| 3 | `config/settings.yml` | `client: true` 前端可读；`type: group_list` 等特殊类型 |
| 4 | `config/routes.rb` | Engine 内路由 + `mount Engine, at: "prefix"` |
| 5 | `db/migrate/` | 表名加命名空间前缀（`plugin_name_xxx`） |
| 6 | `app/models/` | `self.table_name = "..."` 明确指定 |
| 7 | `app/controllers/` | 继承 `ApplicationController`，加 `requires_plugin` |
| 8 | `lib/.../guardian_extensions.rb` | `can_xxx?` 方法，`plugin.rb` 中 `prepend` 进 Guardian |
| 9 | `lib/.../xxx_serializer_extension.rb` | `prepended { attributes :field }`，`include_xxx?` 条件 |
| 10 | `plugin.rb` `after_initialize` | `reloadable_patch` + `add_to_serializer` + `on(:event)` |
| 11 | `assets/.../initializers/` | `export default { name, initialize() { withPluginApi(...) } }` |
| 12 | `assets/.../components/` | Glimmer 组件（`.gjs`），`ajax` 调用后端 |
| 13 | `assets/.../connectors/` | 目录名 = outlet 名；`@outletArgs` 获取上下文 |

---

## 快速参考

```ruby
# plugin.rb 常用 DSL
enabled_site_setting :plugin_enabled
register_asset "stylesheets/plugin.scss"
register_svg_icon "icon-name"

after_initialize do
  reloadable_patch do
    ::Guardian.prepend(MyPlugin::GuardianExtensions)
    ::PostSerializer.prepend(MyPlugin::PostSerializerExtension)
  end
  add_to_serializer(:post, :my_field) { object.my_value }
  on(:post_created) { |post| MyPlugin.handle(post) }
  register_modifier(:some_modifier) { |val| val }
end

# 路由
MyPlugin::Engine.routes.draw { post "/action" => "controller#action" }
Discourse::Application.routes.draw { mount MyPlugin::Engine, at: "my-plugin" }
```

```js
// 前端 initializer 骨架
export default {
  name: "my-plugin",
  initialize() {
    withPluginApi((api) => {
      api.modifyClass("model:topic", (Superclass) => class extends Superclass { ... });
      api.registerValueTransformer("post-menu-buttons", ({ value: dag }) => { ... });
      api.registerCustomPostMessageCallback("my_event", (controller, msg) => { ... });
      api.addDiscoveryQueryParam("my_param", { replace: true, refreshModel: true });
    });
  },
};
```
