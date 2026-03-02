# Plugin 05 — modifyClass 规范

> 来源：`plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`、`plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`

## 概述

`api.modifyClass` 是插件扩展 Discourse 核心 model、component、route、controller 的官方方式。它使用 ES class 继承，比旧版 `reopen` 更安全，支持 TypeScript 类型推断，且不会破坏其他插件。

---

## 来源验证

- `plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`（扩展 model:topic、model:post、model:group、controller:topic、component:topic-notifications-button 共 5 处）
- `plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`（扩展 model:topic）
- `plugins/discourse-reactions/assets/javascripts/discourse/initializers/discourse-reactions.gjs`（扩展 component:emoji-value-list）
- `frontend/discourse/app/lib/plugin-api.gjs`（modifyClass 实现）

---

## 核心规范

### 1. 基本语法

```javascript
api.modifyClass(
  "type:name",               // "model:topic", "component:my-comp", "controller:topic", etc.
  (Superclass) =>            // 工厂函数，接收原始类，返回新类
    class extends Superclass {
      // 扩展内容
    }
);
```

**工厂函数模式**是必须的（不能直接传类），因为 Discourse 需要在运行时决定继承哪个版本（可能已被其他插件修改过）。

---

### 2. 扩展 Model（常用：添加属性和方法）

```javascript
// model:topic — 添加 @tracked 属性和自定义方法
api.modifyClass(
  "model:topic",
  (Superclass) =>
    class extends Superclass {
      @tracked accepted_answer;    // 响应式属性（需 import { tracked }）
      @tracked has_accepted_answer;

      setAcceptedSolution(acceptedAnswer) {
        // 更新 postStream 中所有相关 post
        this.postStream?.posts?.forEach((post) => {
          post.setProperties({ accepted_answer: !!acceptedAnswer });
        });
        this.accepted_answer = acceptedAnswer;
        this.has_accepted_answer = !!acceptedAnswer;
      }
    }
);
```

```javascript
// model:post — 覆盖 getter（注意同时要覆盖 setter）
api.modifyClass(
  "model:post",
  (Superclass) =>
    class extends Superclass {
      // 覆盖 getter
      get can_edit() {
        return isSpecialAction(this.action_code) ? true : super.can_edit;
      }

      // ⚠️ 覆盖 @tracked getter 时必须同时覆盖 setter，否则报错
      set can_edit(value) {
        super.can_edit = value;
      }

      get isSmallAction() {
        return isSpecialAction(this.action_code) ? true : super.isSmallAction;
      }
    }
);
```

```javascript
// model:bookmark — 添加 @discourseComputed 计算属性
import discourseComputed from "discourse/lib/decorators";

api.modifyClass(
  "model:bookmark",
  (Superclass) =>
    class extends Superclass {
      @discourseComputed("assigned_to_user")
      assignedToUserPath(assignedToUser) {
        return userPath(assignedToUser?.username);
      }
    }
);
```

---

### 3. 扩展 Controller（添加 MessageBus 订阅/取消）

```javascript
// controller:topic — 扩展 subscribe/unsubscribe
api.modifyClass(
  "controller:topic",
  (Superclass) =>
    class extends Superclass {
      subscribe() {
        super.subscribe(...arguments);    // ⚠️ 必须调用 super，否则核心订阅失效

        this.messageBus.subscribe("/staff/topic-assignment", (data) => {
          const topic = this.model;
          if (data.topic_id === topic.id) {
            topic.set("assigned_to_user", data.assigned_to);
          }
          this.appEvents.trigger("header:update-topic", topic);
        });
      }

      unsubscribe() {
        super.unsubscribe(...arguments);  // ⚠️ 必须调用 super

        if (!this.model?.id) { return; }
        this.messageBus.unsubscribe("/staff/topic-assignment");
      }
    }
);
```

---

### 4. 扩展 Component（覆盖 getter/方法）

```javascript
// component:topic-notifications-button — 覆盖 reasonText getter
api.modifyClass(
  "component:topic-notifications-button",
  (Superclass) =>
    class extends Superclass {
      get reasonText() {
        // 自定义条件
        if (this.currentUser.never_auto_track_topics &&
            this.args.topic.get("assigned_to_user.username") === this.currentUser.username) {
          return i18n("notification_reason.user");
        }
        return super.reasonText;   // 其他情况走父类逻辑
      }
    }
);
```

```javascript
// component:emoji-value-list — 覆盖 @cached getter，带 ignoreMissing
api.modifyClass(
  "component:emoji-value-list",
  (Superclass) =>
    class extends Superclass {
      @service siteSettings;

      @cached
      get collection() {
        const result = super.collection;
        // 插入额外项
        if (this.args.setting.setting === "my_setting_key") {
          result.unshift({ value: "default", isEditable: false });
        }
        return result;
      }
    },
  { ignoreMissing: true }  // 第三个参数：组件不一定存在（如 admin 组件）
);
```

---

### 5. 扩展 Route（生命周期扩展）

```javascript
// route:some-route — 扩展 model hook 或生命周期
api.modifyClass(
  "route:discovery.latest",
  (Superclass) =>
    class extends Superclass {
      async model(params, transition) {
        const result = await super.model(params, transition);
        // 附加额外数据
        result.myPluginData = await this.loadExtraData();
        return result;
      }
    }
);
```

---

### 6. 扩展 search-advanced-options Component

一个特殊场景——扩展搜索界面的高级选项：

```javascript
api.modifyClass(
  "component:search-advanced-options",
  (Superclass) =>
    class extends Superclass {
      // 添加新的搜索条件处理方法
      updateSearchTermForAssignedUsername() {
        const match = this.filterBlocks(REGEXP_USERNAME_PREFIX);
        const userFilter = this.searchedTerms?.assigned;
        let searchTerm = this.searchTerm || "";

        if (userFilter?.length !== 0) {
          if (match.length !== 0) {
            searchTerm = searchTerm.replace(match[0], `assigned:${userFilter}`);
          } else {
            searchTerm += ` assigned:${userFilter}`;
          }
          this._updateSearchTerm(searchTerm);
        } else if (match.length !== 0) {
          this._updateSearchTerm(searchTerm.replace(match[0], ""));
        }
      }
    }
);
```

---

### 7. modifyClass 的第三个参数（options）

```javascript
api.modifyClass("component:xyz", factory, {
  ignoreMissing: true,  // 组件不存在时不报错（用于可选的 admin 或条件加载的组件）
});
```

---

### 8. 类型名称约定（type:name）

| 前缀 | 示例 | 对应文件路径 |
|------|------|------------|
| `model:` | `model:topic` | `frontend/.../models/topic.js` |
| `model:` | `model:post` | `frontend/.../models/post.js` |
| `model:` | `model:bookmark` | `frontend/.../models/bookmark.js` |
| `model:` | `model:group` | `frontend/.../models/group.js` |
| `controller:` | `controller:topic` | `frontend/.../controllers/topic.js` |
| `component:` | `component:emoji-value-list` | `frontend/.../components/emoji-value-list.gjs` |
| `component:` | `component:topic-notifications-button` | `frontend/.../components/topic-notifications-button.gjs` |
| `route:` | `route:discovery.latest` | `frontend/.../routes/discovery/latest.js` |

**命名规则**：`type:kebab-case-name`（文件名中 `/` 用 `.` 替代，去掉 `.js`/`.gjs` 后缀）。

---

## 反模式（避免这样做）

```javascript
// ❌ 旧版 reopen（已废弃）
Topic.reopen({
  myMethod() { ... }
});

// ❌ 直接修改原型
Topic.prototype.myMethod = function() { ... };

// ✅ 正确：modifyClass 工厂函数
api.modifyClass("model:topic", (Superclass) => class extends Superclass {
  myMethod() { ... }
});

// ❌ 覆盖 @tracked getter 时漏写 setter
get can_edit() { return true; }
// 运行时：Error: You attempted to set can_edit but...

// ✅ getter 和 setter 必须成对
get can_edit() { return isSpecial(this) ? true : super.can_edit; }
set can_edit(value) { super.can_edit = value; }

// ❌ 在 subscribe/unsubscribe 里忘记调用 super
subscribe() {
  this.messageBus.subscribe(...)  // 核心订阅全部失效！
}

// ✅ 必须 super
subscribe() {
  super.subscribe(...arguments);
  this.messageBus.subscribe(...);
}
```

---

## 关联规范

- `plugins/04_value_transformer_dag.md` — 按钮 UI 扩展（与 modifyClass 互补）
- `plugins/02_plugin_api_frontend.md` — withPluginApi 整体结构
- `javascript/02_services.md` — @service 注入模式

---

## 快速参考

```javascript
// Model 添加属性 + 方法
api.modifyClass("model:topic", (S) => class extends S {
  @tracked my_prop;
  myMethod() { this.my_prop = "done"; }
});

// 覆盖 getter（必须配对 setter）
api.modifyClass("model:post", (S) => class extends S {
  get can_edit() { return myCondition(this) || super.can_edit; }
  set can_edit(v) { super.can_edit = v; }
});

// Controller 扩展 subscribe
api.modifyClass("controller:topic", (S) => class extends S {
  subscribe() { super.subscribe(...arguments); this.messageBus.subscribe(...); }
  unsubscribe() { super.unsubscribe(...arguments); this.messageBus.unsubscribe(...); }
});

// 可选组件（不存在时不报错）
api.modifyClass("component:optional-component", factory, { ignoreMissing: true });
```
