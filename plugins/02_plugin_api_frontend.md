# 前端 Plugin API 规范

## 概述
Discourse 前端插件通过 `withPluginApi` 函数获取 Plugin API 对象，再调用各种 API 方法扩展前端行为。两种扩展方式：Initializer（主动调用 API）和 Connector（声明式插槽注入）。

## 来源验证

- `plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/components/solved-accept-answer-button.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/connectors/after-topic-status/solved-status.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/connectors/category-custom-settings/solved-settings.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/connectors/user-card-metadata/accepted-answers.gjs`
- `plugins/discourse-solved/assets/javascripts/discourse/pre-initializers/extend-category-for-solved.js`
- `frontend/discourse/app/lib/plugin-api.gjs` — Plugin API 源码

---

## 核心规范

### 1. 目录结构

```
assets/javascripts/discourse/
├── initializers/               ← 主动注册扩展（必须 export default { name, initialize }）
│   ├── my-feature.gjs
│   └── my-admin-nav.js
├── pre-initializers/           ← 在 initializers 之前运行
│   └── extend-model.js
├── connectors/                 ← 声明式插槽注入（文件名 = 组件名）
│   ├── after-topic-status/
│   │   └── my-status.gjs      ← 插槽名/组件文件
│   └── category-custom-settings/
│       └── my-settings.gjs
├── components/                 ← 复用组件
│   └── my-button.gjs
└── routes/                     ← 扩展路由
    └── ...
```

### 2. Initializer 标准骨架

```javascript
// initializers/my-feature.gjs
import { withPluginApi } from "discourse/lib/plugin-api";

export default {
  name: "my-feature",          // 唯一名称，用于排序和调试

  initialize() {
    withPluginApi((api) => {
      // 所有 api.xxx 调用放在这里
    });
  },
};
```

**每个 `withPluginApi` 回调是独立的**，可以多次调用，各自关注不同功能：

```javascript
initialize() {
  withPluginApi((api) => {
    customizePostMenu(api);     // 按功能拆分
  });

  withPluginApi((api) => {
    api.replaceIcon("...", "...");
  });
}
```

### 3. 常用 API 方法速查

#### 3.1 扩展 Model（modifyClass）

```javascript
api.modifyClass(
  "model:topic",
  (Superclass) =>
    class extends Superclass {
      @tracked accepted_answer;   // 添加响应式属性

      setAcceptedSolution(data) {
        this.accepted_answer = data;  // 添加方法
      }
    }
);
```

**注意**：`modifyClass` 使用类继承语法（`(Superclass) => class extends Superclass`），不是 reopen。

#### 3.2 修改 Post 菜单按钮（registerValueTransformer）

```javascript
api.registerValueTransformer(
  "post-menu-buttons",
  ({
    value: dag,
    context: { post, firstButtonKey, lastHiddenButtonKey },
  }) => {
    if (post.can_do_my_thing) {
      dag.add(
        "my-button",          // 按钮唯一 key
        MyButtonComponent,
        {
          before: firstButtonKey,   // 位置：在某个 key 之前
          after: "like",            // 或：在某个 key 之后
        }
      );
    }
  }
);
```

#### 3.3 在 Post 内容区域渲染组件

```javascript
api.renderAfterWrapperOutlet(
  "post-content-cooked-html",
  class extends Component {
    // 条件渲染：返回 false 不渲染
    static shouldRender(args) {
      return args.post?.post_number === 1 && args.post?.topic?.has_accepted_answer;
    }

    <template>
      <MyComponent @post={{@post}} />
    </template>
  }
);
```

#### 3.4 添加被追踪的 Post 属性

```javascript
// 声明需要从后端序列化数据中追踪的属性（@tracked）
api.addTrackedPostProperties(
  "can_accept_answer",
  "accepted_answer",
  "topic_accepted_answer"
);
```

后端 `add_to_serializer(:post, :can_accept_answer)` 的字段，前端必须用这个 API 声明才能响应式。

#### 3.5 注册 MessageBus 消息回调

```javascript
api.registerCustomPostMessageCallback("accepted_solution", async (controller, message) => {
  const topic = controller.model;
  if (topic) {
    topic.setAcceptedSolution(message.accepted_answer);
  }
});
```

对应后端：`MessageBus.publish("/topic/#{topic.id}", { type: :accepted_solution, ... })`

#### 3.6 替换图标

```javascript
api.replaceIcon(
  "notification.solved.accepted_notification",
  "square-check"
);
```

#### 3.7 添加搜索选项

```javascript
api.addAdvancedSearchOptions({
  statusOptions: [
    { name: i18n("search.advanced.statuses.solved"), value: "solved" },
  ],
});

api.addSearchSuggestion("status:solved");
```

#### 3.8 添加路由查询参数

```javascript
api.addDiscoveryQueryParam("solved", { replace: true, refreshModel: true });
```

### 4. Connector（插槽注入）

Connector 是最简单的扩展方式，**无需注册**，只需在正确目录下放组件文件即可自动注入。

**目录命名** = 插槽名（对应 `<PluginOutlet @name="after-topic-status" />`）

```javascript
// connectors/after-topic-status/my-status.gjs
import icon from "discourse/helpers/d-icon";

const MyStatus = <template>
  {{#if @outletArgs.topic.has_my_feature}}
    <span class="topic-status --my-feature">
      {{icon "circle-check"}}
    </span>
  {{/if}}
</template>;

export default MyStatus;
```

**访问插槽参数**：`@outletArgs.xxx`（对应 `<PluginOutlet @outletArgs={{...}} />`）

**常用插槽名**：
- `after-topic-status` — 话题状态图标区域后
- `category-custom-settings` — 分类设置页
- `user-card-metadata` — 用户卡片元数据区
- `user-summary-stat` — 用户统计页
- `topic-navigation` — 话题导航区
- `bread-crumbs-right` — 面包屑右侧
- `post-menu-before-extra-controls` — Post 菜单

### 5. Pre-Initializer

在所有 initializer 之前运行，用于扩展 Model 类（`Category`、`Topic` 等）：

```javascript
// pre-initializers/extend-category.js
import { computed, get } from "@ember/object";
import Category from "discourse/models/category";

export default {
  name: "extend-category-for-my-plugin",
  before: "inject-discourse-objects",   // 必须在这个之前运行

  initialize() {
    Category.reopen({
      my_feature_enabled: computed(
        "custom_fields.my_feature_enabled",
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

### 6. AJAX 调用

```javascript
import { ajax } from "discourse/lib/ajax";
import { popupAjaxError } from "discourse/lib/ajax-error";

// 在 action 中
@action
async doSomething() {
  this.saving = true;
  try {
    const result = await ajax("/my_plugin/action", {
      type: "POST",
      data: { id: this.args.post.id },
    });
    // 处理结果
  } catch (e) {
    popupAjaxError(e);    // 自动弹出错误提示
  } finally {
    this.saving = false;
  }
}
```

### 7. 访问全局服务（@service）

```javascript
import { service } from "@ember/service";

export default class MyComponent extends Component {
  @service appEvents;
  @service currentUser;
  @service siteSettings;
  @service store;
  @service router;

  @action
  doThing() {
    this.appEvents.trigger("my-plugin:thing-done", this.args.post);
    this.currentUser.id;          // 当前登录用户 id
    this.siteSettings.my_setting; // SiteSettings
  }
}
```

### 8. i18n 国际化

```javascript
import { i18n } from "discourse-i18n";

// 在 .gjs template 中
<template>
  <span>{{i18n "my_plugin.my_key"}}</span>
  <span>{{i18n "my_plugin.with_count" count=this.count}}</span>
</template>

// 在 JS 中
const label = i18n("my_plugin.button_label");
```

对应 `config/locales/client.en.yml`：
```yaml
en:
  js:
    my_plugin:
      my_key: "My translated text"
      with_count:
        one: "1 item"
        other: "%{count} items"
      button_label: "Do Action"
```

### 9. Category Custom Fields 读写模式

后端注册：
```ruby
# plugin.rb
Site.preloaded_category_custom_fields << "my_feature_enabled"
```

前端读取（pre-initializer 扩展 Category model）：
```javascript
Category.reopen({
  my_feature_enabled: computed("custom_fields.my_feature_enabled", {
    get(fieldName) {
      return get(this.custom_fields, fieldName) === "true";
    },
  }),
});
```

Connector 中读取：
```javascript
// @outletArgs.category.my_feature_enabled
```

---

## 反模式

```javascript
// ❌ 直接访问 DOM 而不用 decorateCookedElement
document.querySelectorAll('.cooked').forEach(el => { ... });

// ✅ 用 API
api.decorateCookedElement((element) => {
  element.querySelectorAll('img').forEach(...);
}, { onlyStream: true });

// ❌ 在 initializer 外调用 withPluginApi
withPluginApi((api) => { ... });   // 顶层调用，不在 initialize() 内

// ✅ 必须在 initialize() 内
export default {
  name: "my-plugin",
  initialize() {
    withPluginApi((api) => { ... });
  },
};

// ❌ 在 Connector 中使用 this.model（connector 没有 model）
// ✅ 用 @outletArgs.xxx 访问插槽传入的参数
```

---

## 关联规范

- `plugins/01_plugin_structure.md` — 后端 plugin.rb 对应的前端结构
- `javascript/01_component_gjs.md` — Glimmer 组件语法规范
