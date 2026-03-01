# JavaScript 02 — Ember Service 单例模式

> 来源：`frontend/discourse/app/services/`

---

## 1. Service 是什么

Ember Service 是应用级单例对象，在整个应用生命周期内只存在一个实例。适合用于：

- 全局共享状态（currentUser、siteSettings、session）
- 跨组件通信（appEvents 事件总线）
- 封装浏览器 API（capabilities、document-title）
- 持久化存储代理（emoji-store）

---

## 2. 基础骨架

```js
import Service, { service } from "@ember/service";
import { tracked } from "@glimmer/tracking";
import { disableImplicitInjections } from "discourse/lib/implicit-injections";

@disableImplicitInjections              // Discourse 规范：必须显式声明注入
export default class MyService extends Service {
  // 注入其他 Service
  @service appEvents;
  @service currentUser;
  @service siteSettings;

  // 响应式状态
  @tracked count = 0;

  // 普通方法
  increment() {
    this.count++;
  }
}
```

**文件命名**：`kebab-case.js`，如 `document-title.js`、`emoji-store.js`

---

## 3. @disableImplicitInjections 装饰器

Discourse 自定义规范，要求所有 Service **必须显式声明** 依赖，禁止隐式注入：

```js
// ✅ 正确：显式声明
@disableImplicitInjections
export default class Header extends Service {
  @service site;
  @service scrollDirection;
}

// ❌ 错误：依赖隐式注入（被 Discourse 禁止）
export default class Header extends Service {
  // 这种写法在 Discourse 中不允许
}
```

---

## 4. 常见模式

### 4.1 事件总线（mixin 模式）

```js
// services/app-events.js
import Evented from "@ember/object/evented";
import Service from "@ember/service";

export default class AppEvents extends Service.extend(Evented) {}
```

使用方：
```js
// 触发事件
this.appEvents.trigger("discourse:focus-changed", hasFocus);

// 监听事件（在组件中）
this.appEvents.on("discourse:focus-changed", this, "handleFocusChanged");
```

### 4.2 @tracked 响应式状态

```js
import { tracked } from "@glimmer/tracking";
import Service from "@ember/service";

export default class Header extends Service {
  @tracked headerOffset = 0;
  @tracked topicInfo = null;
  @tracked hamburgerVisible = false;
}
```

绑定到模板后，状态变化会自动触发重渲染。

### 4.3 私有字段（ES2022 # 语法）

```js
export default class DocumentTitle extends Service {
  #title = null;            // 私有状态，不暴露给外部
  #backgroundNotify = null;

  getTitle() {
    return this.#title;
  }

  setTitle(title) {
    this.#title = title;
    this._renderTitle();    // 内部方法用 _ 前缀
  }
}
```

### 4.4 TrackedMap / TrackedArray（响应式集合）

```js
import { TrackedMap } from "@ember-compat/tracked-built-ins";
import { TrackedArray, TrackedObject } from "@ember-compat/tracked-built-ins";

export default class Header extends Service {
  #hiders = new TrackedMap();    // 替代普通 Map，支持响应式
  contexts = new TrackedObject(); // 替代普通对象
}
```

### 4.5 持久化到 LocalStorage

```js
import KeyValueStore from "discourse/lib/key-value-store";

export default class EmojiStore extends Service {
  store = new KeyValueStore("discourse_emoji_reaction_");  // 命名空间

  get diversity() {
    return this._diversity ?? this.store.getObject("emojiSelectedDiversity") ?? 1;
  }

  set diversity(value) {
    this._diversity = value;
    this.store.setObject({ key: "emojiSelectedDiversity", value });
  }
}
```

### 4.6 自定义工厂（非标准 Service 实例化）

```js
// services/capabilities.js
class Capabilities {
  // 非 Service 类，直接实例化
  isAndroid = navigator.userAgent.includes("Android");
  // ...
}

export const capabilities = new Capabilities(); // 模块级单例

// 通过 Ember Service 工厂包装，使其可被注入
export default class CapabilitiesServiceShim {
  static isServiceFactory = true;

  static create() {
    return capabilities;  // 返回已有实例，不创建新实例
  }
}
```

---

## 5. deprecated 废弃 API 处理

```js
import deprecated from "discourse/lib/deprecated";

get topic() {
  deprecated(
    "`.topic` is deprecated. Use `.topicInfo` instead.",
    {
      id: "discourse.header-service-topic",
      since: "3.3.0.beta4-dev",
      dropFrom: "3.4.0",
    }
  );
  return this.topicInfoVisible ? this.topicInfo : null;
}
```

废弃 API 用 `deprecated()` 包装，保留兼容性的同时输出警告。

---

## 6. @dependentKeyCompat（遗留兼容）

```js
import { dependentKeyCompat } from "@ember/object/compat";

@dependentKeyCompat  // 让旧的 Ember observers 能追踪此 getter
get topicInfoVisible() {
  // ...
}
```

用于保持与遗留 `@observes` / computed 宏的兼容。新代码不需要。

---

## 7. 在组件中注入 Service

```js
// components/my-component.gjs
import { service } from "@ember/service";
import Component from "@glimmer/component";

export default class MyComponent extends Component {
  @service documentTitle;
  @service currentUser;

  updateTitle() {
    this.documentTitle.setTitle("New Title");
  }
}
```

---

## 8. Service 目录结构

```
frontend/discourse/app/services/
├── app-events.js          # 事件总线（Evented mixin）
├── capabilities.js        # 浏览器能力检测（自定义工厂）
├── composer.js            # 发帖 UI 状态（复杂业务逻辑）
├── document-title.js      # 文档标题管理
├── emoji-store.js         # Emoji 偏好持久化
├── header.js              # 顶栏响应式状态
├── session.js             # 会话信息
└── ...
```

---

## 9. 规范总结

| 规范 | 说明 |
|------|------|
| 文件名 | `kebab-case.js` |
| 类名 | PascalCase，与业务对应（不必以 Service 结尾） |
| `@disableImplicitInjections` | 必须加，强制显式声明依赖 |
| 依赖注入 | `@service xxx;` 显式声明 |
| 响应式状态 | 用 `@tracked` 而非普通属性 |
| 响应式集合 | `TrackedMap` / `TrackedArray` / `TrackedObject` |
| 私有状态 | 用 ES # 私有字段 |
| 内部辅助方法 | `_methodName()` 约定（_ 前缀） |
| LocalStorage | 通过 `KeyValueStore` 封装，带命名空间 |
| 废弃 API | `deprecated()` 包装，指定 since / dropFrom |
