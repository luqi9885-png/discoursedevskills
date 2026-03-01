# JavaScript 04 — Ember Store 与 Model 规范

> 来源：`frontend/discourse/app/services/store.js`、`frontend/discourse/app/models/rest.js`、`frontend/discourse/app/models/topic.js`、`frontend/discourse/app/models/user.js`

---

## 概述

Discourse 使用自定义 Store（不是 Ember Data），基于 `RestModel` 基类，配合 identity map 实现单例缓存。所有前端模型都通过 `this.store` 操作，不直接 `new` 或 `fetch`。

---

## 1. 架构层次

```
store（service）           ← 操作入口：find / createRecord / update / destroyRecord
  ↓ 读写
identity map（内存缓存）    ← 相同 type+id 永远是同一个对象实例
  ↓ 实例化
RestModel（基类）           ← 所有 model 的父类，提供 save / update / destroyRecord
  ↓ 继承
Topic / User / Post / ...  ← 具体 model，扩展属性和方法
```

Store 是一个 **Service**，在 Route / Component 中用 `@service store` 注入：

```js
import { service } from "@ember/service";
export default class MyRoute extends DiscourseRoute {
  @service store;
  model() { return this.store.find("topic", 42); }
}
```

---

## 2. Store 核心方法

```js
// 按 id 查找（优先读 identity map，再发请求）
this.store.find("topic", 42);
this.store.find("topic", { slug: "my-topic", id: 42 });

// 查找全部（返回 ResultSet）
this.store.findAll("badge");

// 创建内存记录（有 id → hydrate 进 identity map；无 id → isNew=true 的新记录）
this.store.createRecord("topic", { id: 42, title: "Hello" });  // 有 id
this.store.createRecord("topic", {});                           // 新记录

// 更新（发 PUT，更新 identity map）
this.store.update("topic", 42, { title: "New Title" });

// 删除（发 DELETE，从 identity map 移除）
this.store.destroyRecord("topic", topicInstance);

// 分页加载更多（修改 ResultSet.content）
this.store.appendResults(resultSet, "topic", nextPageUrl);
```

---

## 3. RestModel 基类

### 3.1 状态属性

```js
@tracked isSaving = false;
@equal("__state", "new") isNew;        // createRecord 无 id 时为 true
@equal("__state", "created") isCreated;
```

### 3.2 必须重写的方法

```js
export default class MyModel extends RestModel {
  // 必须！save() 新记录时调用此方法收集要发送的属性
  createProperties() {
    return this.getProperties("title", "category_id", "body");
  }

  // 可选：update() 时调用
  updateProperties() {
    return this.getProperties("title", "category_id");
  }
}
```

### 3.3 保存记录

```js
// save() 自动判断 isNew：new → _saveNew()，created → update()
const topic = this.store.createRecord("topic", {});
topic.set("title", "My Topic");
await topic.save();           // POST 创建

const existing = await this.store.find("topic", 42);
await existing.save({ title: "Updated" });   // PUT 更新

// 删除
await record.destroyRecord();  // DELETE
```

### 3.4 static munge：预处理 JSON

```js
export default class Topic extends RestModel {
  // Server JSON 进入 create() 之前会先过这里
  static munge(json) {
    delete json.category;   // 避免覆盖 computed property
    json.bookmarks = json.bookmarks || [];
    return json;
  }
}
```

---

## 4. Identity Map：单例缓存

Store 内部维护一个 identity map：同一 type + id 的对象在内存中**只存在一个实例**。

```js
// 两次 find 返回同一个对象
const a = await this.store.find("topic", 42);
const b = await this.store.find("topic", 42);
a === b; // true（从 identity map 返回）

// hydrate 时若 id 已存在，只更新属性，不创建新实例
this.store.createRecord("topic", { id: 42, title: "New" });
// → 找到已有实例，调用 setProperties 更新，不 new 一个新对象
```

这意味着对同一个 model 的修改，所有引用它的地方会自动响应更新。

---

## 5. Model 中的计算属性

### 5.1 @discourseComputed（依赖追踪）

```js
import discourseComputed from "discourse/lib/decorators";

export default class Topic extends RestModel {
  // 依赖 "fancy_title"，fancy_title 变化时重新计算
  @discourseComputed("fancy_title")
  fancyTitle(title) {
    return fancyTitle(title, this.siteSettings.support_mixed_text_direction);
  }

  // 依赖多个属性
  @discourseComputed("bumped_at", "createdAt")
  bumpedAt(bumped_at, createdAt) {
    return bumped_at ? new Date(bumped_at) : createdAt;
  }

  // 无参数：只计算一次（类似 getter）
  @discourseComputed
  postStream() {
    return this.store.createRecord("postStream", { id: this.id, topic: this });
  }
}
```

### 5.2 Ember 内置 computed macros

```js
import { alias, equal, notEmpty, or, and, fmt } from "@ember/object/computed";

@alias("details.allowed_groups") allowedGroups;    // 别名
@notEmpty("deleted_at") deleted;                   // deleted_at 有值则为 true
@equal("archetype", "private_message") isPrivateMessage;
@or("details.can_edit", "details.can_edit_tags") canEditTags;
@and("pinned", "readLastPost") canClearPin;
@fmt("url", "%@/print") printUrl;                  // 字符串格式化
```

### 5.3 @tracked：响应式属性

```js
import { tracked } from "@glimmer/tracking";
import { trackedArray } from "discourse/lib/tracked-tools";

export default class Topic extends RestModel {
  @tracked posts_count;
  @tracked deleted_at;
  @trackedArray bookmarks;   // 数组变化也能触发响应式更新
}
```

### 5.4 @cached + @dependentKeyCompat：惰性计算

```js
import { cached } from "@glimmer/tracking";
import { dependentKeyCompat } from "@ember/object/compat";

@dependentKeyCompat
@cached
get relatedMessages() {
  // 计算结果缓存，依赖变化时清除
  return this.get("related_messages")?.map((data) => Topic.create(data));
}
```

---

## 6. Model 上的静态方法（类方法）

不走 Store 的情况下，Model 可以定义静态方法直接调用 ajax：

```js
export default class Topic extends RestModel {
  // 批量操作：不需要单个 model 实例
  static bulkOperation(topics, operation) {
    return ajax("/topics/bulk", {
      type: "PUT",
      data: { topic_ids: topics.map((t) => t.id), operation },
    });
  }

  // 按条件查找（不走 store identity map）
  static find(topicId, opts) {
    return ajax(`/t/${topicId}.json`, { data: opts });
  }

  // Server JSON 转换钩子
  static munge(json) { ... }

  // 模型变换（异步，Plugin 可扩展）
  static async applyTransformations(topics) {
    await applyModelTransformations("topic", topics);
  }
}
```

**规律**：需要操作单个已加载 model 实例 → 实例方法；操作多个或不需要实例 → 静态方法直接 ajax。

---

## 7. @singleton：全局单例 Model

```js
import singleton from "discourse/lib/singleton";

@singleton
export default class User extends RestModel.extend(Evented) {
  // User.current() → 返回 PreloadStore 里的当前登录用户（全局唯一实例）
  static createCurrent() {
    const userJson = PreloadStore.get("currentUser");
    if (userJson) {
      const store = getOwnerWithFallback(this).lookup("service:store");
      return store.createRecord("user", userJson);
    }
    return null;
  }
}
```

`@singleton` 装饰器使 `User.current()` 永远返回同一个实例，适合「当前登录用户」这类全局状态。

---

## 8. ResultSet：分页列表

`findAll` 和 `find`（带对象参数）返回 `ResultSet`，不是普通数组：

```js
const badges = await this.store.findAll("badge");
badges.content;      // 实际数据数组
badges.totalRows;    // 服务端总条数
badges.loadMoreUrl;  // 下一页 URL
badges.canLoadMore;  // 还有更多？

// 加载更多（追加到 content）
await this.store.appendResults(badges, "badge", badges.loadMoreUrl);
```

---

## 快速参考

```js
// 注入 store
@service store;

// 查询
this.store.find("topic", 42);
this.store.find("topic", { username: "alice" });
this.store.findAll("badge");
this.store.findStale("topic", findArgs, opts);   // 先返回缓存，后台刷新

// 创建 / 保存 / 删除
const r = this.store.createRecord("post", {});
r.set("raw", "hello");
await r.save();          // POST（isNew=true）
await r.save({ raw: "updated" });  // PUT（isCreated）
await r.destroyRecord(); // DELETE

// Model 定义
export default class MyModel extends RestModel {
  static munge(json) { return json; }          // 预处理 JSON
  createProperties() { return this.getProperties("title"); }  // 必须实现
  updateProperties() { return this.getProperties("title"); }  // 可选

  @tracked title;
  @trackedArray tags;
  @discourseComputed("title") titleUpper(t) { return t?.toUpperCase(); }
  @alias("details.can_edit") canEdit;
}

// identity map：同 type+id 永远同一实例，setProperties 自动广播更新
```
