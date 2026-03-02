# Plugin 04 — registerValueTransformer 与 DAG

> 来源：`frontend/discourse/app/lib/dag.js`、`frontend/discourse/app/components/post/menu.gjs`

## 概述

`registerValueTransformer` 是 Discourse 现代插件修改前端 UI 顺序和内容的标准方式。它基于 DAG（有向无环图）实现，允许插件声明式地添加、替换、删除、重排按钮或列表项，同时多个插件之间按依赖关系自动排序，不会互相冲突。

---

## 来源验证

- `frontend/discourse/app/lib/dag.js`（DAG 类，294 行）
- `frontend/discourse/app/components/post/menu.gjs`（`post-menu-buttons` transformer 的完整注册与消费）
- `plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`（solved 按钮注入）
- `plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`（assign 按钮注入，多位置控制）
- `plugins/discourse-reactions/assets/javascripts/discourse/initializers/discourse-reactions.gjs`（replace 替换 like 按钮）

---

## 核心规范

### 1. 调用语法

```javascript
api.registerValueTransformer(
  "transformer-name",         // transformer 的唯一名称（由核心代码定义）
  ({ value: dag, context }) => {
    // 修改 dag 并隐式返回（mutable transformer）
    dag.add("my-key", MyComponent, { before: context.firstButtonKey });
  }
);
```

**DAG 是可变的**：直接修改 `value`（即 dag 对象），不需要显式 return。

---

### 2. post-menu-buttons — 最常用的 transformer

这是插件为帖子菜单添加按钮的唯一官方方式：

```javascript
api.registerValueTransformer(
  "post-menu-buttons",
  ({ value: dag, context: { post, state, buttonKeys, firstButtonKey, lastHiddenButtonKey, secondLastHiddenButtonKey } }) => {

    // 条件：只为符合条件的帖子添加
    if (!post.can_accept_answer) {
      return;
    }

    dag.add(
      "solved",              // 按钮唯一 key（字符串，全局唯一）
      SolvedAcceptButton,    // Glimmer 组件类
      {
        before: firstButtonKey,   // 位置
      }
    );
  }
);
```

**context 字段说明：**

| 字段 | 类型 | 含义 |
|------|------|------|
| `dag` (value) | DAG 实例 | 按钮注册表，可调用 add/replace/delete/reposition |
| `post` | Post model | 当前渲染的帖子对象 |
| `state` | Object | 包含 `currentUser`、`siteSettings`、`collapsed` 等状态 |
| `buttonKeys` | Object | 所有核心按钮的 key 常量（见下表） |
| `firstButtonKey` | string | 当前配置中第一个显示的按钮 key |
| `lastHiddenButtonKey` | string \| null | "更多"菜单中最后一个隐藏按钮的 key |
| `secondLastHiddenButtonKey` | string \| null | "更多"菜单中倒数第二个按钮的 key |
| `lastItemBeforeMoreItemsButtonKey` | string \| null | "显示更多"按钮之前的最后一个按钮 key |
| `buttonLabels` | Object | 控制标签显示/隐藏（`.show(key)`/`.hide(key)`） |
| `collapsedButtons` | Object | 控制按钮折叠状态（`.show(key)`/`.hide(key)`） |

**buttonKeys 常量：**

```javascript
// 核心按钮 key（来自 post/menu.gjs）
buttonKeys.ADMIN       // "admin"
buttonKeys.BOOKMARK    // "bookmark"
buttonKeys.COPY_LINK   // "copyLink"
buttonKeys.DELETE      // "delete"
buttonKeys.EDIT        // "edit"
buttonKeys.FLAG        // "flag"
buttonKeys.LIKE        // "like"
buttonKeys.READ        // "read"
buttonKeys.REPLIES     // "replies"
buttonKeys.REPLY       // "reply"
buttonKeys.SHARE       // "share"
buttonKeys.SHOW_MORE   // "showMore"
```

---

### 3. DAG API 完整方法

```javascript
// 添加（key 不存在时）
dag.add("my-key", MyComponent, { before: "reply", after: "like" });
// 返回 true 成功，false 如果 key 已存在

// 替换（key 必须已存在）
dag.replace("like", ReactionsButton);
// 用 ReactionsButton 替换 like 按钮，位置不变

// 删除
dag.delete("read");

// 重新定位（不改变 value，只改位置）
dag.reposition("bookmark", { before: "reply" });

// 查询是否存在
dag.has("reply");  // => true

// 获取所有条目（含位置信息）
dag.entries();  // => [["reply", ReplyButton, {before: "showMore"}], ...]
```

---

### 4. 位置规则（before/after）

```javascript
// 在 reply 之前
dag.add("my-button", MyBtn, { before: "reply" });

// 在 like 之后
dag.add("my-button", MyBtn, { after: "like" });

// 多个约束（数组）
dag.add("my-button", MyBtn, {
  before: ["reply", "edit"],  // 必须在 reply 和 edit 两者之前
});

// 优先级降级：如果 primary 位置不满足，退而求其次
dag.add("my-button", MyBtn, {
  before: [
    "assign",           // 首选：assign 插件按钮之前（可能不存在）
    firstButtonKey,     // 次选：第一个按钮之前
  ],
});
```

**关键特性：** 当 `before`/`after` 引用的 key 不存在时，DAG 不报错，自动跳过该约束（`throwErrorOnCycle: false` 时处理循环依赖）。

---

### 5. 按钮组件规范（注入 DAG 的组件）

被注入到 `post-menu-buttons` 的组件通过 `@post`、`@buttonActions` 等 args 接收数据：

```javascript
// components/my-action-button.gjs
import Component from "@glimmer/component";
import { action } from "@ember/object";
import DButton from "discourse/components/d-button";
import { ajax } from "discourse/lib/ajax";
import { popupAjaxError } from "discourse/lib/ajax-error";

export default class MyActionButton extends Component {
  // static shouldRender：控制是否渲染（静态方法，可用于性能优化）
  static shouldRender(args) {
    return args.post?.can_do_thing;
  }

  @action
  async doThing() {
    try {
      await ajax("/my_plugin/action", {
        type: "POST",
        data: { post_id: this.args.post.id },
      });
      this.args.post.set("can_do_thing", false);
    } catch (e) {
      popupAjaxError(e);
    }
  }

  <template>
    <DButton
      class="my-action-button"
      @action={{this.doThing}}
      @icon="check"
      @translatedTitle="My Action"
    />
  </template>
}
```

---

### 6. 替换现有按钮（replace）

reactions 插件用 `replace` 把 like 按钮换成 emoji 反应按钮：

```javascript
api.registerValueTransformer(
  "post-menu-buttons",
  ({ value: dag, context: { buttonKeys } }) => {
    // 替换 like 按钮为自定义 reactions 按钮，位置不变
    dag.replace(buttonKeys.LIKE, ReactionsActionButton);

    // 同时在 replies 后添加摘要
    dag.add("discourse-reactions-actions", ReactionsActionSummary, {
      after: buttonKeys.REPLIES,
    });
  }
);
```

---

### 7. 其他常用 transformer 名称

```javascript
// 话题列表行添加 class
api.registerValueTransformer("topic-list-item-class", ({ value, context: { topic } }) => {
  if (topic.has_accepted_answer) {
    value.push("solved");
  }
  return value;   // 数组型 transformer 需要 return
});

// 控制 post-menu 是否折叠
api.registerValueTransformer("post-menu-collapsed", ({ value, context: { post } }) => {
  return post.bookmarked ? false : value;
});

// 标签分隔符
api.registerValueTransformer("tag-separator", "|", { topic, index });
```

**注意区别：** `post-menu-buttons` 是 **mutable**（直接改 dag），列表类是 **immutable**（需 return 新值）。判断方式：看核心代码用的是 `applyMutableValueTransformer` 还是 `applyValueTransformer`。

---

## 反模式（避免这样做）

```javascript
// ❌ 直接修改 DOM 插入按钮
document.querySelector(".post-menu-area").insertAdjacentHTML(...)

// ❌ 用旧版 decorateWidget（已废弃）
api.decorateWidget("post-menu:before", () => { ... });

// ✅ 用 registerValueTransformer + DAG
api.registerValueTransformer("post-menu-buttons", ({ value: dag }) => {
  dag.add("my-button", MyComponent, { before: "reply" });
});

// ❌ 在 add 时依赖一个可能不存在的 key 不做回退
dag.add("my-btn", Btn, { before: "some-other-plugin-button" });
// ✅ 提供多个 before 回退
dag.add("my-btn", Btn, { before: ["some-other-plugin-button", firstButtonKey] });
```

---

## 关联规范

- `plugins/02_plugin_api_frontend.md` — withPluginApi 总体结构与常用 API
- `plugins/05_modify_class.md` — modifyClass 覆盖组件/model/controller

---

## 快速参考

```javascript
// 添加按钮到帖子菜单
api.registerValueTransformer("post-menu-buttons", ({ value: dag, context: { post, buttonKeys, firstButtonKey } }) => {
  if (!post.can_do_thing) { return; }
  dag.add("my-plugin-button", MyButton, { before: firstButtonKey });
});

// 替换现有按钮
dag.replace(buttonKeys.LIKE, MyCustomButton);

// 删除按钮
dag.delete(buttonKeys.READ);

// 重排
dag.reposition("my-button", { after: buttonKeys.REPLY });

// 话题列表添加 CSS class
api.registerValueTransformer("topic-list-item-class", ({ value, context: { topic } }) => {
  if (topic.my_flag) { value.push("my-class"); }
  return value;
});
```
