# JavaScript 03 — Ember Routes 规范

> 来源：`frontend/discourse/app/routes/`、`frontend/discourse/app/routes/app-route-map.js`

---

## 概述

Route 负责三件事：**加载数据**（model hook）、**权限守卫**（beforeModel/afterModel）、**配置 Controller**（setupController）。模板渲染和交互逻辑不属于 Route。

---

## 1. 基类：DiscourseRoute

所有 Route 继承 `DiscourseRoute` 而非直接继承 Ember 的 `Route`：

```js
import DiscourseRoute from "discourse/routes/discourse";

export default class About extends DiscourseRoute {
  // ...
}
```

`DiscourseRoute` 提供：
- `willTransition` 时自动调用 `seenUser()`（在线状态追踪）
- `titleToken()` / `refreshTitle()` 页面标题管理机制
- `isCurrentUser(user)` 工具方法

---

## 2. 路由定义：app-route-map.js

所有路由在 `app-route-map.js` 中集中声明：

```js
// 简单路由
this.route("about");                              // path: /about

// 自定义 path + 动态段
this.route("post", { path: "/p/:id" });

// 嵌套路由
this.route("topic", { path: "/t/:slug/:id" }, function () {
  this.route("fromParams", { path: "/" });
  this.route("fromParamsNear", { path: "/:nearPost" });
});

// resetNamespace：子路由不加父路由名称前缀
this.route("userActivity", { path: "/activity", resetNamespace: true }, function () {
  this.route("topics");   // route name: userActivity.topics（不是 user.userActivity.topics）
});
```

新增路由在此文件添加。

---

## 3. model hook：加载数据

```js
// 简单：ajax 请求
export default class About extends DiscourseRoute {
  async model() {
    const result = await ajax("/about.json");
    return result.about;  // 返回值成为 controller.model
  }
}

// 复杂：复用已有 model（同 topic 页内导航不重新请求）
export default class TopicRoute extends DiscourseRoute {
  model(params, transition) {
    let topic = this.modelFor("topic");
    if (topic && topic.get("id") === parseInt(params.id, 10)) {
      return topic;
    }
    return this.store.createRecord("topic", params);
  }
}

// 子路由复用父路由 model
export default class Preferences extends RestrictedUserRoute {
  model() {
    return this.modelFor("user");  // 直接复用父路由已加载的 user，避免重复请求
  }
}
```

**规律**：
- `model(params)` 接收 URL 动态段参数
- `model(params, transition)` 还可读 `transition.to.queryParams`
- 子路由常用 `this.modelFor("parentRoute")` 复用父级 model

---

## 4. 生命周期 hooks 顺序

```
beforeModel → model → afterModel → setupController → activate
                                                    ↓（路由激活，页面展示）
                                              deactivate（离开时）
```

### beforeModel：权限守卫（阻断式）

```js
export default class UserRoute extends DiscourseRoute {
  beforeModel() {
    if (this.siteSettings.hide_user_profiles_from_public && !this.currentUser) {
      throw new RouteException({ status: 403, desc: i18n("user.login_to_view_profile") });
    }
  }
}
```

### afterModel：model 加载后的异步操作

```js
export default class UserRoute extends DiscourseRoute {
  afterModel() {
    const user = this.modelFor("user");
    return user
      .findDetails()
      .then(() => user.findStaffInfo())
      .then(() => user.statusManager.trackStatus())
      .catch(() => this.router.replaceWith("/404")); // 找不到则重定向
  }
}
```

### setupController：配置 Controller

```js
export default class TopicRoute extends DiscourseRoute {
  setupController(controller, model) {
    controller.setProperties({
      model,
      editingTopic: false,
      firstPostExpanded: false,
    });
    this.searchService.searchContext = model.get("searchContext");
    this.composer.set("topic", model);
    this.screenTrack.start(model.get("id"), controller);
  }
}
```

### activate / deactivate：订阅与清理

```js
export default class UserRoute extends DiscourseRoute {
  activate() {
    super.activate(...arguments);
    const user = this.modelFor("user");
    this.messageBus.subscribe(`/u/${user.username_lower}`, this.onUserMessage);
  }

  deactivate() {
    super.deactivate(...arguments);
    const user = this.modelFor("user");
    // 离开路由时必须取消订阅，防止内存泄漏
    this.messageBus.unsubscribe(`/u/${user.username_lower}`, this.onUserMessage);
    this.searchService.searchContext = null;  // 清理全局状态
  }
}
```

---

## 5. queryParams

```js
export default class TopicRoute extends DiscourseRoute {
  queryParams = {
    filter: { replace: true },           // replace: true = 不产生浏览器历史记录
    username_filters: { replace: true },
  };
}
```

`replace: true` 适用于过滤、分页等不需要浏览器回退的状态变化。

---

## 6. @action：Route 级别的 Actions

Route 中的 `@action` 方法可被子路由和 Controller 通过 `this.send()` 冒泡调用：

```js
export default class TopicRoute extends DiscourseRoute {
  @service modal;

  @action
  showFlags(model) {
    // 打开 Modal 是 Route 的职责（而非 Component）
    this.modal.show(FlagModal, {
      model: { flagModel: model },
    });
  }

  @action
  willTransition() {
    super.willTransition(...arguments);
    cancel(this.scheduledReplace);  // 离开前清理定时器
    return true;  // 返回 true 允许继续跳转
  }

  @action
  didTransition() {
    setTopicId(this.controllerFor("topic").get("model.id"));
    return true;
  }
}
```

---

## 7. titleToken：页面标题

```js
// 返回字符串
export default class About extends DiscourseRoute {
  titleToken() { return i18n("about.simple_title"); }
}

// 返回数组时组合成 "标题 - 分类" 格式
export default class TopicRoute extends DiscourseRoute {
  titleToken() {
    const model = this.modelFor("topic");
    const cat = model.get("category");
    return [model.get("title"), cat?.get("name")];
  }
}
```

---

## 8. serialize：model → URL 参数

```js
export default class UserRoute extends DiscourseRoute {
  // 控制 model 如何序列化为 URL 参数（link-to 等场景使用）
  serialize(model) {
    if (!model) return {};
    return { username: (model.username || "").toLowerCase() };
  }
}
```

---

## 9. 专用基类模式

Discourse 提供多个可继承的专用基类，封装重复的权限逻辑：

```js
// routes/restricted-user.js - 需要编辑权限才能访问
export default class RestrictedUser extends DiscourseRoute {
  afterModel() {
    if (!this.modelFor("user").get("can_edit")) {
      this.router.replaceWith("userActivity");
    }
  }
}

// 继承使用（子路由自动获得权限检查）
export default class Preferences extends RestrictedUserRoute {
  model() { return this.modelFor("user"); }
}
```

类似的还有 `build-topic-route.js`、`build-private-messages-route.js` 等工厂函数，用于批量生成结构相似的路由。

---

## 快速参考

```js
// 完整骨架
import DiscourseRoute from "discourse/routes/discourse";
export default class XxxRoute extends DiscourseRoute {
  @service router;

  beforeModel() { /* 权限守卫，throw 或 replaceWith */ }
  model(params) { return ajax("/xxx.json"); }
  afterModel() { return this.modelFor("xxx").loadExtra(); }
  setupController(controller, model) { controller.set("model", model); }
  activate() { super.activate(...arguments); /* 订阅 */ }
  deactivate() { super.deactivate(...arguments); /* 取消订阅，清理全局状态 */ }
  titleToken() { return i18n("xxx.title"); }
  @action willTransition() { return true; }
  @action didTransition() { return true; }
}

// 常用片段
model() { return this.modelFor("parentRoute"); }        // 复用父路由 model
queryParams = { filter: { replace: true } };            // 不产生历史记录的 queryParam
this.route("foo", { resetNamespace: true }, fn);        // 子路由不加父前缀
this.router.replaceWith("/404");                        // 重定向（不产生历史记录）
```
