# JavaScript 06 — MessageBus 实时通信

> 来源：`message-bus.js`（service + instance-initializer）、
> `subscribe-user-notifications.js`、`extend-for-solved-button.gjs`、
> `app/models/topic.rb`（后端 publish 模式）

---

## 概述

MessageBus 是 Discourse 的实时推送层，基于长轮询实现服务器→客户端的单向推送。

```
后端                                      前端
────                                      ────
MessageBus.publish(channel, data)   →   messageBus.subscribe(channel, callback)
                                                ↓
                                          callback(data) 更新 UI / state
```

---

## 1. 前端 Service

```js
// services/message-bus.js
import MessageBus from "message-bus-client";
export default class MessageBusService {
  static isServiceFactory = true;
  static create() { return MessageBus; }  // 返回第三方库实例作为 Service
}
```

注入：任意 Service / Component 中 `@service messageBus`。

---

## 2. 前端订阅 API

```js
messageBus.subscribe(channel, callback, lastId?)
messageBus.unsubscribe(channel, callback)   // 必须传同一个函数引用！
```

---

## 3. Instance-initializer 模式（推荐）

```js
class SubscribeUserNotificationsInit {
  @service messageBus;
  @service currentUser;
  @service appEvents;

  constructor(owner) {
    setOwner(this, owner);
    if (!this.currentUser) return;

    this.messageBus.subscribe(
      `/notification/${this.currentUser.id}`,
      this.onNotification,
      this.currentUser.notification_channel_position  // lastId：断线续传
    );
    this.messageBus.subscribe("/categories", this.onCategories);
  }

  teardown() {
    // 销毁时必须取消所有订阅，否则内存泄漏
    this.messageBus.unsubscribe(
      `/notification/${this.currentUser.id}`, this.onNotification
    );
    this.messageBus.unsubscribe("/categories", this.onCategories);
  }

  @bind   // 保证回调中 this 指向类实例
  onNotification(data) {
    this.currentUser.setProperties({
      unread_notifications: data.unread_notifications,
      unread_high_priority_notifications: data.unread_high_priority_notifications,
    });
    this.appEvents.trigger("notifications:changed");
  }

  @bind
  onCategories(data) {
    (data.categories || []).forEach(c => this.site.updateCategory(c));
    (data.deleted_categories || []).forEach(id => this.site.removeCategory(id));
  }
}

export default {
  after: "message-bus",   // MessageBus 启动后才订阅
  initialize(owner) { this.instance = new SubscribeUserNotificationsInit(owner); },
  teardown() { this.instance.teardown(); this.instance = null; },
};
```

**要点**：`@bind` 固定 this；`teardown()` 取消所有订阅；`after: "message-bus"` 保证顺序。

---

## 4. 常见频道规律

| 频道 | 用途 |
|------|------|
| `/notification/{user_id}` | 通知计数变更 |
| `/user-drafts/{user_id}` | 草稿变更 |
| `/do-not-disturb/{user_id}` | 免打扰状态 |
| `/reviewable_counts/{user_id}` | 待审核计数 |
| `/topic/{topic_id}` | 话题实时更新（新帖、Plugin 自定义事件） |
| `/categories` | 分类增删改 |
| `/client_settings` | SiteSettings 实时同步 |
| `/user-status` | 用户状态 |

---

## 5. Plugin 监听 Topic 消息

Plugin 通过 `registerCustomPostMessageCallback` 响应 `/topic/{id}` 上的自定义 type：

```js
function handleMessages(api) {
  const callback = async (controller, message) => {
    const topic = controller.model;
    if (topic) topic.setAcceptedSolution(message.accepted_answer);
  };

  api.registerCustomPostMessageCallback("accepted_solution", callback);
  api.registerCustomPostMessageCallback("unaccepted_solution", callback);
}
```

callback 参数：`(controller, message)` — controller 是 topic controller，message 是完整 payload。

---

## 6. 后端：publish

### 基本用法

```ruby
# 公开广播（所有订阅者都收到）
MessageBus.publish("/categories", { categories: [category.as_json] })

# 指定用户
MessageBus.publish(
  "/notification/#{user.id}",
  { unread_notifications: 5 },
  user_ids: [user.id]
)

# 指定 group
MessageBus.publish("/reviewable_claimed", data, group_ids: [group_id])
```

### Plugin 自定义 topic 消息

```ruby
# plugin.rb 中
MessageBus.publish(
  "/topic/#{topic.id}",
  {
    type: :accepted_solution,    # 前端用 type 区分
    id: post.id,
    accepted_answer: { post_number: post.post_number },
  }
)
```

### 安全受众（私信 / 受限分类）

```ruby
secure_audience = topic.secure_audience_publish_messages
# → { user_ids: [...], group_ids: [...] }（公开时为空 {}）

if secure_audience[:user_ids] != [] && secure_audience[:group_ids] != []
  MessageBus.publish("/topic/#{topic_id}", message, opts.merge(secure_audience))
end
```

对访问受限的 topic 必须传 secure_audience，否则非授权用户也会收到推送。

---

## 7. 前端初始化流程

```
1. instance-initializer/message-bus（after: "inject-objects"）
   → messageBus.stop()（暂停，等 document.readyState === "complete"）
   → 配置 callbackInterval、baseUrl、ajax
   → messageBus.start()

2. instance-initializer/subscribe-user-notifications（after: "message-bus"）
   → 已登录：订阅通知、草稿、分类等
   → 未登录：跳过

3. Plugin initializer（withPluginApi）
   → registerCustomPostMessageCallback(type, cb)
```

测试环境（`isTesting()`）MessageBus 不启动，避免测试干扰。

---

## 快速参考

```js
// 前端 instance-initializer
constructor(owner) {
  setOwner(this, owner);
  this.messageBus.subscribe("/my-channel", this.onMsg, lastId);
}
teardown() { this.messageBus.unsubscribe("/my-channel", this.onMsg); }
@bind onMsg(data) { /* 更新状态 */ }

// Plugin - topic 自定义消息
api.registerCustomPostMessageCallback("my_type", (controller, message) => {
  controller.model?.handleUpdate(message);
});
```

```ruby
# 后端 Ruby
MessageBus.publish("/channel", data)
MessageBus.publish("/channel", data, user_ids: [user.id])
MessageBus.publish("/channel", data, group_ids: [group.id])
MessageBus.publish("/topic/#{id}", { type: :my_type, payload: ... })
```
