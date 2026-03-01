# 插件开发规范 02 — 前端：initializer / connector / component

## 概述
Discourse 插件前端代码通过三个机制扩展 UI：Initializer（启动时运行）、Outlet Connector（向现有 UI 插入组件）、Glimmer Component（独立 UI 单元）。核心工具是 `withPluginApi` 和 Plugin API。

## 来源验证

- `plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/initializers/add-topic-list-class.js`
- `plugins/discourse-solved/assets/javascripts/discourse/pre-initializers/extend-category-for-solved.js`
- `plugins/discourse-solved/assets/javascripts/discourse/connectors/after-topic-status/solved-status.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/connectors/user-card-metadata/accepted-answers.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/connectors/category-custom-settings/solved-settings.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/components/solved-accept-answer-button.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/components/solved-accepted-answer.gjs`
- `frontend/discourse/app/lib/plugin-api.gjs` — Plugin API 全部方法定义

---

## 核心规范

### 1. 目录结构规范

```
assets/javascripts/discourse/
├── initializers/              ← 应用启动时运行（大多数扩展在这里）
│   └── my-plugin.gjs
├── pre-initializers/          ← 比 initializer 更早运行（扩展 Model）
│   └── extend-model.js
├── connectors/                ← 向已有 UI 位置插入组件
│   └── [outlet-name]/        ← 目录名 = outlet 名称
│       └── my-component.gjs  ← 文件名随意（一个 outlet 可多个文件）
├── components/               ← 纯粹的 Glimmer 组件
│   └── my-button.gjs
├── services/                 ← Ember Service 单例
│   └── my-service.js
└── routes/                   ← 路由定义（少用）
```

### 2. Initializer 骨架

```javascript
// assets/javascripts/discourse/initializers/my-plugin.js
import { withPluginApi } from "discourse/lib/plugin-api";

export default {
  name: "my-plugin",          // 唯一名称，用于启动顺序控制

  initialize() {
    withPluginApi((api) => {
      // 在这里调用 Plugin API 方法
    });
  },
};
```

**withPluginApi** 的作用：确保在 Discourse 框架初始化完成后才执行插件代码，避免时序问题。

### 3. Pre-Initializer（扩展 Model）

比普通 initializer 更早执行，用于 `reopen` Model（如给 `Category` 添加 computed property）：

```javascript
// assets/javascripts/discourse/pre-initializers/extend-category.js
import { computed, get } from "@ember/object";
import Category from "discourse/models/category";

export default {
  name: "extend-category-for-my-plugin",
  before: "inject-discourse-objects",    // 指定执行顺序

  initialize() {
    Category.reopen({
      enable_my_feature: computed(
        "custom_fields.enable_my_feature",
        {
          get(fieldName) {
            return get(this.custom_fields, fieldName) === "true";
          },
        }
      ),
    });
  },
};
```

**规则**：`before: "inject-discourse-objects"` 是标准写法，确保在 Model 注入之前扩展。

### 4. Outlet Connector（最轻量的 UI 扩展）

向现有 UI 的特定位置插入内容，目录名即 outlet 名称：

```javascript
// connectors/after-topic-status/solved-status.gjs
import icon from "discourse/helpers/d-icon";
import { i18n } from "discourse-i18n";

const SolvedStatus = <template>
  {{#if @outletArgs.topic.has_accepted_answer}}
    <span class="topic-status --solved">
      {{icon "far-square-check"}}
    </span>
  {{/if}}
</template>;

export default SolvedStatus;
```

**关键**：
- `@outletArgs` — 由 outlet 位置注入的参数（每个 outlet 不同）
- `@context` — outlet 渲染的上下文（有时用于区分场景）
- 文件是 functional component（不需要 class）

常见 outlet 名称：
- `after-topic-status` — topic 状态图标后
- `topic-navigation` — topic 页面导航区
- `user-card-metadata` — 用户卡片元数据区
- `user-summary-stat` — 用户摘要统计区
- `category-custom-settings` — 分类自定义设置页
- `bread-crumbs-right` — 面包屑右侧
- `user-activity-bottom` — 用户活动页底部
- `post-bottom` — 帖子底部

### 5. Glimmer Component（.gjs 格式）

插件中的 UI 组件使用 Glimmer（.gjs 格式）：

```javascript
// components/my-button.gjs
import Component from "@glimmer/component";
import { tracked } from "@glimmer/tracking";
import { action } from "@ember/object";
import { service } from "@ember/service";
import DButton from "discourse/components/d-button";
import { ajax } from "discourse/lib/ajax";
import { popupAjaxError } from "discourse/lib/ajax-error";

export default class MyButton extends Component {
  // 静态方法：控制该按钮是否渲染（用于 post-menu-buttons 等 DAG 系统）
  static hidden(args) {
    return !args.post.can_do_thing;
  }

  @service appEvents;
  @service currentUser;
  @service siteSettings;

  @tracked loading = false;

  @action
  async handleClick() {
    this.loading = true;
    try {
      await ajax("/my_endpoint", { type: "POST", data: { id: this.args.post.id } });
      this.appEvents.trigger("my-plugin:thing-done", this.args.post);
    } catch (e) {
      popupAjaxError(e);
    } finally {
      this.loading = false;
    }
  }

  <template>
    <DButton
      @action={{this.handleClick}}
      @disabled={{this.loading}}
      @icon="circle-check"
      @label="my_plugin.button_label"
      @title="my_plugin.button_title"
    />
  </template>
}
```

### 6. Plugin API 常用方法

```javascript
withPluginApi((api) => {

  // --- 模型扩展 ---
  // 向 Topic model 添加 tracked 属性（用于响应式更新）
  api.modifyClass("model:topic", (Superclass) =>
    class extends Superclass {
      @tracked my_field;

      myMethod() { ... }
    }
  );

  // --- Post 扩展 ---
  // 声明需要从序列化器同步的 post 属性
  api.addTrackedPostProperties("can_accept_answer", "accepted_answer");

  // --- Post Menu 按钮 ---
  // 向帖子菜单添加按钮（DAG 系统）
  api.registerValueTransformer(
    "post-menu-buttons",
    ({ value: dag, context: { post, firstButtonKey } }) => {
      if (post.can_do_thing) {
        dag.add("my-button", MyButtonComponent, { before: firstButtonKey });
      }
    }
  );

  // --- 渲染位置 ---
  // 在 outlet 后渲染组件
  api.renderAfterWrapperOutlet(
    "post-content-cooked-html",
    class extends Component {
      static shouldRender(args) { return args.post.post_number === 1; }
      <template><MyComponent @post={{@post}} /></template>
    }
  );

  // --- Topic List 类名 ---
  api.registerValueTransformer(
    "topic-list-item-class",
    ({ value, context }) => {
      if (context.topic.has_accepted_answer) {
        value.push("status-solved");
      }
      return value;
    }
  );

  // --- MessageBus 消息处理 ---
  api.registerCustomPostMessageCallback("my_event", async (controller, message) => {
    const topic = controller.model;
    if (topic && message.my_data) {
      topic.set("my_field", message.my_data);
    }
  });

  // --- 搜索扩展 ---
  api.addDiscoveryQueryParam("solved", { replace: true, refreshModel: true });
  api.addAdvancedSearchOptions({ statusOptions: [{ name: "Solved", value: "solved" }] });
  api.addSearchSuggestion("status:solved");

  // --- 图标替换 ---
  api.replaceIcon("notification.my_type", "circle-check");

});
```

### 7. ajax：前端 API 调用

```javascript
import { ajax } from "discourse/lib/ajax";
import { popupAjaxError } from "discourse/lib/ajax-error";

// GET 请求
const data = await ajax("/my_endpoint", { type: "GET" });

// POST 请求
await ajax("/solution/accept", {
  type: "POST",
  data: { id: post.id },
});

// 错误处理（自动显示 flash 提示）
try {
  await ajax(...);
} catch (e) {
  popupAjaxError(e);
}
```

### 8. appEvents：组件间通信

```javascript
import { service } from "@ember/service";

// 触发事件
@service appEvents;

this.appEvents.trigger("my-plugin:something-happened", payload);

// 监听事件（在 constructor 或 willDestroy 中注册/注销）
constructor(owner, args) {
  super(owner, args);
  this.appEvents.on("my-plugin:something-happened", this, "handleEvent");
}

willDestroy() {
  this.appEvents.off("my-plugin:something-happened", this, "handleEvent");
  super.willDestroy(...arguments);
}
```

### 9. Connector 中访问 siteSettings 和 currentUser

Connector 是简单 template 时可直接使用全局注入：

```javascript
// connectors 如果需要 class：
import Component from "@ember/component";
import { classNames, tagName } from "@ember-decorators/component";

@tagName("div")
@classNames("my-outlet", "my-class")
export default class MyConnector extends Component {
  // this.siteSettings — 自动注入
  // this.currentUser — 自动注入
  // this.model / this.user — 由 outlet 注入，按 outlet 不同

  <template>
    {{#if this.siteSettings.my_setting}}
      <span>{{this.user.accepted_answers}}</span>
    {{/if}}
  </template>
}
```

---

## 反模式（避免这样做）

```javascript
// ❌ 直接操作 DOM（Discourse 使用 Glimmer 响应式渲染）
document.querySelector(".post").innerHTML += "<div>...</div>";

// ❌ 在 initializer 外直接执行代码（框架可能未准备好）
import Topic from "discourse/models/topic";
Topic.reopen({ myField: null });   // 应放在 pre-initializer 里

// ❌ 不用 withPluginApi 直接调用 API
api.modifyClass(...);   // api 未定义！

// ✅ 必须包在 withPluginApi 里
withPluginApi((api) => {
  api.modifyClass(...);
});

// ❌ 在 try-catch 里静默吞掉错误
try {
  await ajax(...);
} catch (e) {
  // 啥都不做
}
// ✅ 至少用 popupAjaxError 提示用户
catch (e) { popupAjaxError(e); }
```

---

## 关联规范

- `plugins/01_plugin_rb_and_backend.md` — 后端如何通过 add_to_serializer 传数据到前端
- `javascript/01_component_gjs.md` — .gjs 组件格式完整规范
- `plugins/03_plugin_settings.md` — client: true 设置才能在前端用 siteSettings.xxx
