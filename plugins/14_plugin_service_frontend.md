# Plugin 14 — 插件自定义 Ember Service 完整规范

> 来源：`plugins/discourse-ai/assets/javascripts/discourse/services/gists.js`、`plugins/discourse-ai/assets/javascripts/discourse/services/ai-credits.js`、`plugins/discourse-ai/assets/javascripts/discourse/services/ai-conversations-sidebar-manager.js`、`plugins/discourse-ai/assets/javascripts/discourse/services/image-caption-popup.js`

## 概述

插件 Service 放在 `assets/javascripts/discourse/services/` 目录下，Ember 通过文件路径自动注册（无需手动声明）。Service 适合管理跨组件共享状态、封装 AJAX 请求、处理持久化偏好、管理 MessageBus 订阅等。本文覆盖四种典型的插件 Service 模式，均来自 discourse-ai 真实代码。

---

## 来源验证

- `plugins/discourse-ai/assets/javascripts/discourse/services/gists.js`（localStorage 偏好 + router 属性）
- `plugins/discourse-ai/assets/javascripts/discourse/services/ai-credits.js`（请求去重 + ES# 私有字段）
- `plugins/discourse-ai/assets/javascripts/discourse/services/ai-conversations-sidebar-manager.js`（appEvents + MessageBus + constructor/willDestroy 生命周期 + addSidebarSection）
- `plugins/discourse-ai/assets/javascripts/discourse/services/image-caption-popup.js`（轻量状态服务）

---

## 核心规范

### 1. 轻量状态服务（跨组件共享 @tracked 状态）

最简单的服务形态：通过 `@tracked` 共享状态，供多个组件读取/修改：

```javascript
// services/my-feature-popup.js
import { tracked } from "@glimmer/tracking";
import Service, { service } from "@ember/service";

export default class MyFeaturePopup extends Service {
  @service composer;       // 注入其他 Service
  @service appEvents;

  // 共享的 @tracked 状态（任何组件 @service myFeaturePopup 后即可读写）
  @tracked showPopup = false;
  @tracked targetId = null;
  @tracked isLoading = false;
  @tracked currentData = null;

  // 操作方法（组件调用）
  openPopup(id) {
    this.targetId = id;
    this.showPopup = true;
  }

  closePopup() {
    this.showPopup = false;
    this.targetId = null;
    this.currentData = null;
  }

  toggleLoading(isLoading) {
    if (isLoading) {
      this.isLoading = true;
    } else {
      this.isLoading = false;
    }
  }

  // 操作 Composer 内容（通过 appEvents 通信）
  replaceComposerText(match, replacement) {
    this.appEvents.trigger("composer:replace-text", match, replacement);
  }
}
```

组件中使用：

```javascript
import Component from "@glimmer/component";
import { action } from "@ember/object";
import { service } from "@ember/service";

export default class MyTrigger extends Component {
  @service myFeaturePopup;  // 注入 Service（名称为 services/ 文件名的 camelCase）

  @action
  handleClick() {
    this.myFeaturePopup.openPopup(this.args.targetId);
  }

  <template>
    {{#if this.myFeaturePopup.showPopup}}
      <MyPopupComponent />
    {{/if}}
    <DButton @action={{this.handleClick}} @icon="star" />
  </template>
}
```

---

### 2. localStorage 偏好管理服务（带 router 集成）

管理用户布局偏好，保存到 localStorage，并根据 router 状态动态计算属性：

```javascript
// services/my-preferences.js
import { tracked } from "@glimmer/tracking";
import Service, { service } from "@ember/service";

const LAYOUT_KEY = "my-plugin-layout";

export const LIST_LAYOUT = "list";
export const CARD_LAYOUT = "card";

export default class MyPreferences extends Service {
  @service router;

  // @tracked preference：localStorage → 初始值，set 时同步写入
  @tracked preference = this.#loadPreference();

  // ES 私有方法，从 localStorage 加载（含迁移逻辑）
  #loadPreference() {
    const OLD_KEY = "my-plugin-old-layout";  // 处理旧版 key 迁移
    const oldValue = localStorage.getItem(OLD_KEY);
    const currentValue = localStorage.getItem(LAYOUT_KEY);

    if (oldValue && !currentValue) {
      localStorage.setItem(LAYOUT_KEY, oldValue);
      localStorage.removeItem(OLD_KEY);
      return oldValue;
    } else if (oldValue) {
      localStorage.removeItem(OLD_KEY);
    }

    return currentValue;
  }

  // 从 router 动态读取当前路由的数据（计算属性）
  get routeAttributes() {
    return this.router.currentRoute.attributes;
  }

  // 从多个来源尝试获取当前 topic 列表
  get topics() {
    const listTopics = this.routeAttributes?.list?.topics;
    if (listTopics) { return listTopics; }

    const directTopics = this.routeAttributes?.topics;
    if (directTopics) { return directTopics; }

    return null;
  }

  get hasContentToShow() {
    return this.topics?.some((topic) => topic.my_plugin_field);
  }

  get currentPreference() {
    return this.preference;
  }

  // 设置偏好并同步到 localStorage
  setPreference(value) {
    this.preference = value;
    localStorage.setItem(LAYOUT_KEY, value);
  }
}
```

---

### 3. AJAX 请求去重服务（ES # 私有字段 + Promise 缓存）

封装 API 请求，对并发重复请求进行去重（单一缓存 in-flight request）：

```javascript
// services/my-plugin-api.js
import Service, { service } from "@ember/service";
import { ajax } from "discourse/lib/ajax";

export default class MyPluginApi extends Service {
  @service currentUser;

  // ES # 私有字段（外部无法访问）
  #pendingRequests = new Map();

  // 公开方法：检查特定 ID 的状态
  async checkStatus(entityIds) {
    const result = await this.#fetchStatus({ entity_ids: entityIds });
    return result.entities || {};
  }

  // 公开方法：检查某个实体是否可用
  async isEntityAvailable(entityId) {
    const result = await this.checkStatus([entityId]);
    const status = result[entityId];
    return status?.available ?? true;  // 默认 true（宽松降级）
  }

  // 私有：带请求去重的 AJAX 封装
  async #fetchStatus(params) {
    // 用参数的 JSON 字符串作为 key，对相同请求去重
    const cacheKey = JSON.stringify(params);

    // 如果有相同参数的请求正在进行中，复用同一个 Promise
    if (this.#pendingRequests.has(cacheKey)) {
      return this.#pendingRequests.get(cacheKey);
    }

    const request = ajax("/my_plugin/status", {
      type: "GET",
      data: params,
    })
      .then((result) => {
        this.#pendingRequests.delete(cacheKey);  // 完成后清理
        return result;
      })
      .catch((error) => {
        this.#pendingRequests.delete(cacheKey);  // 出错也要清理
        throw error;
      });

    this.#pendingRequests.set(cacheKey, request);
    return request;
  }
}
```

---

### 4. constructor/willDestroy 生命周期 + MessageBus + appEvents

需要在 Service 初始化时订阅事件、销毁时取消的复杂 Service：

```javascript
// services/my-realtime-manager.js
import { tracked } from "@glimmer/tracking";
import { scheduleOnce } from "@ember/runloop";
import Service, { service } from "@ember/service";
import { TrackedArray } from "@ember-compat/tracked-built-ins";
import { ajax } from "discourse/lib/ajax";
import discourseDebounce from "discourse/lib/debounce";

const CHANNEL = "/my-plugin/updates";
const DEBOUNCE_MS = 100;

export default class MyRealtimeManager extends Service {
  @service appEvents;
  @service messageBus;
  @service router;

  @tracked items = [];
  @tracked isLoading = true;

  // 非响应式私有字段（不需要 @tracked）
  isFetching = false;
  page = 0;
  hasMore = true;

  // 构造函数：Service 初始化时自动调用
  constructor() {
    super(...arguments);

    // 订阅自定义 appEvent
    this.appEvents.on(
      "my-plugin:item-created",
      this,
      this._handleNewItem         // 必须传 this 作为绑定上下文
    );

    // 订阅 MessageBus 实时频道
    this.messageBus.subscribe(CHANNEL, (payload) => {
      this._applyUpdate(payload.id, payload.data);
    });
  }

  // 销毁时必须清理所有订阅（防止内存泄漏）
  willDestroy() {
    super.willDestroy(...arguments);  // 必须调用 super

    this.appEvents.off(
      "my-plugin:item-created",
      this,
      this._handleNewItem  // 必须传相同的函数引用才能取消订阅
    );

    this.messageBus.unsubscribe(CHANNEL);
  }

  // 懒加载数据（首次调用时触发）
  async fetchItems() {
    if (this.isFetching || !this.hasMore) { return; }

    const isFirstPage = this.page === 0;
    this.isFetching = true;

    try {
      const { items, meta } = await ajax("/my_plugin/items.json", {
        data: { page: this.page, per_page: 40 },
      });

      if (isFirstPage) {
        this.items = items;
      } else {
        this.items = [...this.items, ...items];  // 追加（不替换）
      }

      this.page += 1;
      this.hasMore = meta.has_more;
    } finally {
      this.isFetching = false;
      this.isLoading = false;
    }
  }

  // appEvent 处理器（实时插入新项目）
  _handleNewItem(item) {
    this.items = [item, ...this.items];  // 新项目插入头部
  }

  // MessageBus 更新处理器（更新已有项目）
  _applyUpdate(itemId, newData) {
    this.items = this.items.map((item) =>
      item.id === itemId ? { ...item, ...newData } : item
    );
  }

  // 防抖的滚动处理器
  _debouncedScroll = () => {
    discourseDebounce(this, this._checkScroll, DEBOUNCE_MS);
  };

  _checkScroll() {
    const element = this._scrollElement;
    if (!element) { return; }

    const { scrollTop, scrollHeight, clientHeight } = element;
    // 距离底部 100px 时触发加载
    if (scrollHeight - scrollTop - clientHeight < 100 && !this.isFetching && this.hasMore) {
      this.fetchItems();
    }
  }

  // 用 scheduleOnce 避免在同一 runloop 内多次操作 DOM
  _scheduleRerender() {
    scheduleOnce("afterRender", this, this._doRerender);
  }

  _doRerender() {
    // DOM 操作或触发事件
    this.appEvents.trigger("my-plugin:sidebar-updated");
  }
}
```

---

### 5. addSidebarSection（插件自定义侧边栏区块）

通过 Service 动态向 Discourse 侧边栏注册自定义区块（Section）：

```javascript
// 在 initializer 中将 api 传给 Service，让 Service 控制侧边栏
// initializers/my-plugin-sidebar.js
import { withPluginApi } from "discourse/lib/plugin-api";
import { MY_PANEL_KEY } from "../services/my-sidebar-manager";

export default {
  name: "my-plugin-sidebar",

  initialize(container) {
    const currentUser = container.lookup("service:current-user");
    if (!currentUser?.can_use_feature) { return; }

    withPluginApi((api) => {
      const manager = container.lookup("service:my-sidebar-manager");
      manager.api = api;  // 将 api 注入 Service

      // 注册自定义侧边栏面板
      api.addSidebarPanel(
        (BasePanel) =>
          class extends BasePanel {
            key = MY_PANEL_KEY;
            switchButtonLabel = "My Plugin";
            switchButtonIcon = "star";
            switchButtonDefaultUrl = "/my-plugin";
          }
      );
    });
  },
};
```

Service 中动态添加 Section：

```javascript
// services/my-sidebar-manager.js
import Service from "@ember/service";

export const MY_PANEL_KEY = "my-sidebar-panel";

export default class MySidebarManager extends Service {
  api = null;
  _registered = new Set();

  addSection(sectionName, sectionTitle, links) {
    if (this._registered.has(sectionName)) { return; }
    this._registered.add(sectionName);

    this.api.addSidebarSection(
      (BaseCustomSidebarSection) => {
        return class extends BaseCustomSidebarSection {
          get name() { return sectionName; }
          get title() { return sectionTitle; }
          get links() { return links; }
        };
      },
      MY_PANEL_KEY  // 指定归属的面板
    );
  }
}
```

---

## 反模式（避免这样做）

```javascript
// constructor 订阅后 willDestroy 未取消订阅（内存泄漏）
constructor() {
  super(...arguments);
  this.appEvents.on("some-event", this, this._handler);
  this.messageBus.subscribe("/channel", this._handler);  // 错误：忘记取消
}

// 未实现 willDestroy
// 正确：必须在 willDestroy 中 off + unsubscribe
willDestroy() {
  super.willDestroy(...arguments);
  this.appEvents.off("some-event", this, this._handler);
  this.messageBus.unsubscribe("/channel");
}

// localStorage 直接存对象（会变成 "[object Object]"）
localStorage.setItem("key", { layout: "table" });  // 错误！
// 正确：JSON 序列化
localStorage.setItem("key", JSON.stringify({ layout: "table" }));
const val = JSON.parse(localStorage.getItem("key"));

// ES # 私有字段 + 类继承（私有字段不可继承）
class ChildService extends ParentService {
  useParentPrivate() { return this.#parentField; }  // 错误！无法访问父类私有字段
}
// 若需要在子类中访问，改用 _ 约定的私有字段（non-enforced privacy）

// appEvents.on 传匿名函数（无法取消订阅）
constructor() {
  this.appEvents.on("event", this, () => this.handle());  // 错误！每次都是新函数引用
}
willDestroy() {
  this.appEvents.off("event", this, () => this.handle());  // 无法匹配！
}
// 正确：使用命名方法
constructor() { this.appEvents.on("event", this, this.handle); }
willDestroy() { this.appEvents.off("event", this, this.handle); }
```

---

## 关联规范

- `javascript/02_services.md` — Ember Service 核心规范（@tracked、KeyValueStore）
- `javascript/06_message_bus.md` — MessageBus 订阅/取消规范
- `plugins/07_modal_and_ui_components.md` — Service 触发 Modal 的 task-actions 模式

---

## 快速参考

```javascript
// 轻量状态服务
export default class MyPopup extends Service {
  @tracked show = false;
  @tracked data = null;
  open(data) { this.data = data; this.show = true; }
  close() { this.show = false; this.data = null; }
}

// localStorage 偏好服务
export default class MyPrefs extends Service {
  @service router;
  @tracked preference = localStorage.getItem("my-key");
  setPreference(v) { this.preference = v; localStorage.setItem("my-key", v); }
  get currentRouteData() { return this.router.currentRoute.attributes?.list?.topics; }
}

// 请求去重服务（ES # 私有字段）
export default class MyApi extends Service {
  #pending = new Map();
  async fetchData(params) {
    const key = JSON.stringify(params);
    if (this.#pending.has(key)) { return this.#pending.get(key); }
    const req = ajax("/my-endpoint", { data: params })
      .then(r => { this.#pending.delete(key); return r; })
      .catch(e => { this.#pending.delete(key); throw e; });
    this.#pending.set(key, req);
    return req;
  }
}

// 生命周期 + 事件订阅
export default class MyManager extends Service {
  @service appEvents; @service messageBus;
  @tracked items = [];
  constructor() {
    super(...arguments);
    this.appEvents.on("my-plugin:created", this, this._onCreated);
    this.messageBus.subscribe("/my-channel", (data) => this._onUpdate(data));
  }
  willDestroy() {
    super.willDestroy(...arguments);
    this.appEvents.off("my-plugin:created", this, this._onCreated);
    this.messageBus.unsubscribe("/my-channel");
  }
  _onCreated(item) { this.items = [item, ...this.items]; }
  _onUpdate(data) { this.items = this.items.map(i => i.id === data.id ? {...i, ...data} : i); }
}
```
