# javascript/06 — MessageBus 前端订阅规范

> 来源：
> - `frontend/discourse/app/instance-initializers/read-only.js`（最简单的标准模式）
> - `frontend/discourse/app/instance-initializers/banner.js`（带初始状态预加载）
> - `frontend/discourse/app/instance-initializers/subscribe-user-notifications.js`（多频道 + position 参数）
> - `plugins/discourse-assign/assets/javascripts/discourse/components/flagged-topic-listener.js`（旧式 Component）
> - `plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`（modifyClass Controller 中订阅）
> - `plugins/discourse-assign/lib/assigner.rb`（后端 MessageBus.publish 对应）

## 边界声明

- **本文件负责**：前端 MessageBus 订阅/取消订阅的完整使用规范
- **不负责，见其他文件**：
  - 后端 `MessageBus.publish` 权限控制 → `ruby/03_controllers.md`
  - Ember Service 通用模式 → `javascript/02_services.md`

---

## 概述

MessageBus 是 Discourse 的实时推送机制（长轮询），后端通过 `MessageBus.publish(channel, data, user_ids:)` 推送，前端通过 `this.messageBus.subscribe(channel, callback)` 接收。前端 `messageBus` 是 Ember Service，通过 `@service messageBus` 注入。

---

## 来源验证

| 验证点 | 文件 | 关键代码 | 与其他验证点差异 |
|--------|------|---------|----------------|
| instance-initializer 中标准订阅模式 | `instance-initializers/read-only.js` | `@service messageBus` + `subscribe/unsubscribe` + `@bind` | 最简模式，单频道 |
| 带预加载状态的订阅 | `instance-initializers/banner.js` | `PreloadStore.get` + `subscribe` | 页面初始值来自 preload，实时更新来自 MessageBus |
| 多频道 + position 参数 | `instance-initializers/subscribe-user-notifications.js:40-70` | `subscribe(channel, handler, position)` | 带 `last_id` 避免重放历史消息 |
| Controller modifyClass 中订阅 | `plugins/discourse-assign/initializers/extend-for-assigns.js:466` | `subscribe()` / `unsubscribe()` 方法覆盖 | 旧式 Controller 模式，需调用 `super` |
| 旧式 Component 中订阅 | `plugins/discourse-assign/components/flagged-topic-listener.js` | `didInsertElement` / `willDestroyElement` | 旧式 Component 生命周期钩子 |
| 后端对应的 publish | `plugins/discourse-assign/lib/assigner.rb:447,528` | `MessageBus.publish("/staff/topic-assignment", {...}, user_ids:)` | 权限控制：user_ids 限定接收者 |

---

## 核心规范

### 1. instance-initializer 中订阅（现代推荐模式）

插件推送全局状态时，在 `instance-initializer` 中使用独立的类处理生命周期（来自 `read-only.js` 真实代码）：

```javascript
// discourse/instance-initializers/my-plugin-listener.js
import { setOwner } from "@ember/owner";
import { service } from "@ember/service";
import { bind } from "discourse/lib/decorators";

class MyPluginListener {
  @service messageBus;
  @service site;           // 按需注入其他 Service
  @service currentUser;

  constructor(owner) {
    setOwner(this, owner);  // 必须：让 @service 注入生效

    // 只在登录用户时订阅私人频道
    if (this.currentUser) {
      this.messageBus.subscribe(
        `/my-plugin/user/${this.currentUser.id}`,
        this.onUserMessage,
        this.currentUser.my_plugin_channel_position  // 可选：避免重放历史消息
      );
    }

    // 公开频道（所有用户）
    this.messageBus.subscribe("/my-plugin/global", this.onGlobalMessage);
  }

  teardown() {
    // 必须与 subscribe 完全对应，传入同一个 handler 引用
    if (this.currentUser) {
      this.messageBus.unsubscribe(
        `/my-plugin/user/${this.currentUser.id}`,
        this.onUserMessage
      );
    }
    this.messageBus.unsubscribe("/my-plugin/global", this.onGlobalMessage);
  }

  @bind  // 必须：绑定 this，否则回调中 this 为 undefined
  onUserMessage(data) {
    // data 是后端 MessageBus.publish 的第二个参数
    this.site.set("myPluginUserStatus", data.status);
  }

  @bind
  onGlobalMessage(data) {
    // 处理全局消息
  }
}

export default {
  after: "message-bus",  // 确保在 MessageBus 初始化后执行

  initialize(owner) {
    this.instance = new MyPluginListener(owner);
  },

  teardown() {
    this.instance.teardown();
    this.instance = null;
  },
};
```

### 2. 带初始状态预加载（banner.js 模式）

当页面初始状态由后端预加载、后续更新通过 MessageBus 推送时：

```javascript
// 来自 banner.js 真实代码
import EmberObject from "@ember/object";
import { setOwner } from "@ember/owner";
import { service } from "@ember/service";
import { bind } from "discourse/lib/decorators";
import PreloadStore from "discourse/lib/preload-store";

class MyPluginInit {
  @service site;
  @service messageBus;

  constructor(owner) {
    setOwner(this, owner);

    // 1. 先用预加载数据设置初始值（页面 SSR 时已有数据，无闪烁）
    const initialData = PreloadStore.get("my_plugin_data") || {};
    this.site.set("myPluginData", EmberObject.create(initialData));

    // 2. 再订阅实时更新
    this.messageBus.subscribe("/my-plugin/data", this.onDataUpdate);
  }

  teardown() {
    this.messageBus.unsubscribe("/my-plugin/data", this.onDataUpdate);
  }

  @bind
  onDataUpdate(data) {
    if (data) {
      this.site.set("myPluginData", EmberObject.create(data));
    } else {
      this.site.set("myPluginData", null);
    }
  }
}

export default {
  after: "message-bus",
  initialize(owner) { this.instance = new MyPluginInit(owner); },
  teardown() { this.instance.teardown(); this.instance = null; },
};
```

### 3. Controller modifyClass 中订阅（discourse-assign 真实模式）

当需要在 Topic Controller 中响应实时消息时，通过 `modifyClass` 覆盖 `subscribe`/`unsubscribe`：

```javascript
// 来自 extend-for-assigns.js 真实代码
api.modifyClass(
  "controller:topic",
  (Superclass) =>
    class extends Superclass {
      subscribe() {
        super.subscribe(...arguments);  // 必须调用 super，保留核心订阅

        this.messageBus.subscribe("/staff/topic-assignment", (data) => {
          const topic = this.model;
          if (data.topic_id !== topic.id) {
            return;
          }

          // 根据消息类型更新本地数据
          if (data.type === "assigned") {
            topic.set("assigned_to_user", data.assigned_to);
          } else if (data.type === "unassigned") {
            topic.set("assigned_to_user", null);
          }

          // 触发 appEvents 通知其他组件更新
          this.appEvents.trigger("header:update-topic", topic);
        });
      }

      unsubscribe() {
        super.unsubscribe(...arguments);  // 必须调用 super

        if (!this.model?.id) {
          return;
        }
        this.messageBus.unsubscribe("/staff/topic-assignment");
      }
    }
);
```

### 4. Glimmer Component 中订阅（现代写法）

```javascript
// assets/javascripts/discourse/components/my-plugin-live-status.gjs
import Component from "@glimmer/component";
import { service } from "@ember/service";
import { bind } from "discourse/lib/decorators";

export default class MyPluginLiveStatus extends Component {
  @service messageBus;

  constructor(owner, args) {
    super(owner, args);
    this.messageBus.subscribe(
      `/my-plugin/topic/${this.args.topicId}`,
      this.onStatusUpdate
    );
  }

  willDestroy() {
    super.willDestroy();
    this.messageBus.unsubscribe(
      `/my-plugin/topic/${this.args.topicId}`,
      this.onStatusUpdate
    );
  }

  @bind
  onStatusUpdate(data) {
    // 更新组件状态
  }

  <template>
    {{! 模板内容 }}
  </template>
}
```

### 5. 后端 publish（对应前端订阅）

```ruby
# 后端推送（来自 discourse-assign/lib/assigner.rb 真实代码）

# 推送给特定用户列表
MessageBus.publish(
  "/staff/topic-assignment",
  {
    type: "assigned",
    topic_id: topic.id,
    assigned_to: BasicUserSerializer.new(user, scope: Guardian.new, root: false).as_json,
  },
  user_ids: allowed_user_ids,   # 限定接收者（安全）
)

# 推送给所有用户
MessageBus.publish("/my-plugin/global", { status: "updated" })

# 推送给特定用户（私信频道）
MessageBus.publish(
  "/my-plugin/user/#{user.id}",
  { count: items.count },
  user_ids: [user.id],          # 与频道中的 user.id 对应，双重保护
)

# 推送给 staff
MessageBus.publish(
  "/my-plugin/staff-only",
  { data: "secret" },
  group_ids: [Group::AUTO_GROUPS[:staff]],
)
```

---

## 频道命名约定

| 频道模式 | 用途 | 后端 user_ids 控制 |
|---------|------|-------------------|
| `/my-plugin/global` | 所有用户可见的状态 | 不传（全部用户） |
| `/my-plugin/user/:id` | 用户私人频道 | `user_ids: [user.id]` |
| `/staff/my-plugin` | 仅 staff 可见 | `group_ids: [staff_group_id]` |
| `/my-plugin/topic/:id` | 话题级别实时 | 话题可见用户 |

---

## 反模式

```javascript
// 错误1：忘记 @bind，导致回调中 this 为 undefined
@service messageBus;

constructor(owner) {
  setOwner(this, owner);
  this.messageBus.subscribe("/channel", this.onMessage);  // onMessage 中 this 是 undefined！
}

onMessage(data) {
  this.site.set("x", data);  // TypeError: Cannot read properties of undefined
}

// 正确：加 @bind 装饰器
@bind
onMessage(data) {
  this.site.set("x", data);  // this 正确绑定
}

// 错误2：unsubscribe 传不同的函数引用（无法取消订阅，内存泄漏）
subscribe() {
  this.messageBus.subscribe("/channel", (data) => this.handle(data));  // 匿名函数！
}
unsubscribe() {
  this.messageBus.unsubscribe("/channel", (data) => this.handle(data));  // 新函数，无法匹配！
}
// 正确：用 @bind 方法作为具名引用
@bind onMessage(data) { this.handle(data); }
subscribe() { this.messageBus.subscribe("/channel", this.onMessage); }
unsubscribe() { this.messageBus.unsubscribe("/channel", this.onMessage); }

// 错误3：teardown 中忘记 unsubscribe（内存泄漏，路由切换后仍在监听）
class MyListener {
  constructor(owner) {
    setOwner(this, owner);
    this.messageBus.subscribe("/channel", this.onMessage);
  }
  // 没有 teardown 或 willDestroy！
}

// 错误4：modifyClass 中 subscribe/unsubscribe 忘记调用 super
subscribe() {
  // 没有 super.subscribe()！核心订阅被跳过
  this.messageBus.subscribe("/my-channel", this.handler);
}

// 错误5：不传 user_ids 的后端 publish（所有用户都能收到私密数据）
MessageBus.publish("/user-private-data", { secret: "xxx" })  // 危险！
// 正确
MessageBus.publish("/user-private-data", { secret: "xxx" }, user_ids: [user.id])
```

---

## 快速参考

```javascript
// instance-initializer 模板（推荐）
import { setOwner } from "@ember/owner";
import { service } from "@ember/service";
import { bind } from "discourse/lib/decorators";

class MyListener {
  @service messageBus;
  @service currentUser;

  constructor(owner) {
    setOwner(this, owner);
    this.messageBus.subscribe("/my-channel", this.onMessage);
  }

  teardown() {
    this.messageBus.unsubscribe("/my-channel", this.onMessage);
  }

  @bind
  onMessage(data) { /* 处理消息 */ }
}

export default {
  after: "message-bus",
  initialize(owner) { this.instance = new MyListener(owner); },
  teardown() { this.instance.teardown(); this.instance = null; },
};
```

```ruby
# 后端 publish
MessageBus.publish("/my-channel", { key: value }, user_ids: [user.id])
MessageBus.publish("/my-channel", { key: value }, group_ids: [group_id])
MessageBus.publish("/my-channel", { key: value })  # 全体用户（公开）
```
