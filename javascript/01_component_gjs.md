# Glimmer Component (.gjs) 规范

## 概述
Discourse 前端使用 Ember + Glimmer，组件文件后缀为 `.gjs`，
将 JavaScript 类和 `<template>` 标签合并在同一文件中（单文件组件格式）。

## 来源验证（真实代码，多处比对）
- `frontend/discourse/app/components/d-button.gjs`（完整类组件）
- `frontend/discourse/app/components/copy-button.gjs`（旧式 @ember/component，对比用）
- `frontend/discourse/app/components/conditional-loading-spinner.gjs`（模板组件，无类）

---

## 核心规范

### 1. 两种组件形态

#### 形态 A：模板组件（无 class，简单组件首选）

```js
// conditional-loading-spinner.gjs
import helper from "discourse/helpers/some-helper";

const MyComponent = <template>
  <div class={{helper @someArg}}>
    {{yield}}
  </div>
</template>;

export default MyComponent;
```

特点：
- 没有 JS 类，直接 `const X = <template>...</template>`
- 适合只做展示、无内部状态的组件
- `@arg` 访问父级传入参数

#### 形态 B：类组件（有状态/行为的组件）

```js
// d-button.gjs
import Component from "@glimmer/component";
import { action } from "@ember/object";
import { service } from "@ember/service";

export default class DButton extends Component {
  @service router;         // 注入服务

  get computedLabel() {    // getter 计算属性
    return this.args.label ? i18n(this.args.label) : this.args.translatedLabel;
  }

  @action
  click(event) {           // 事件处理用 @action 装饰
    // ...
  }

  <template>
    <button {{on "click" this.click}}>
      {{this.computedLabel}}
    </button>
  </template>
}
```

### 2. `<template>` 写在类内部

类组件的 `<template>` **必须写在 class 主体内**，紧跟最后一个方法之后。
这是 `.gjs` 单文件组件格式的核心特征。

```js
export default class MyComponent extends Component {
  // ... JS 逻辑 ...

  <template>
    {{! 模板在这里，类的最后 }}
  </template>
}
```

### 3. 访问参数和状态

| 语法 | 含义 |
|------|------|
| `@argName` | 父级传入的参数（模板中） |
| `this.args.argName` | 父级参数（JS 中） |
| `this.propName` | 类属性或 getter |
| `{{on "event" this.handler}}` | 绑定事件 |
| `...attributes` | 传递 HTML 属性给根元素 |

### 4. 服务注入

```js
import { service } from "@ember/service";

export default class MyComponent extends Component {
  @service router;
  @service capabilities;
  @service siteSettings;   // 访问站点设置
}
```

### 5. 动作（@action）

```js
@action
click(event) {
  event.preventDefault();
  // 处理逻辑
}
```

模板中绑定：`{{on "click" this.click}}`

### 6. 计算属性（getter）

```js
get isDisabled() {
  return this.forceDisabled || this.args.disabled;
}
```

模板中：`disabled={{this.isDisabled}}`

### 7. 条件类名

```js
import concatClass from "discourse/helpers/concat-class";

// 模板中：
class={{concatClass
  @class
  (if @isLoading "is-loading")
  (if this.noText "no-text")
}}
```

### 8. 不要写 JSDoc

根据 CLAUDE.md 规范：**新代码不写 JSDoc**。
d-button.gjs 里的 JSDoc 是历史遗留，维护即可。

---

## 导入路径规范

```js
// 组件
import DButton from "discourse/components/d-button";

// helper
import concatClass from "discourse/helpers/concat-class";
import icon from "discourse/helpers/d-icon";

// 服务（通过 @service 注入，不直接 import）

// 国际化
import { i18n } from "discourse-i18n";

// truth-helpers（条件判断）
import { or, eq, not } from "discourse/truth-helpers";
```

---

## 反模式（避免这样做）

```js
// ❌ .js 组件（旧格式，不要新建）
// copy-button.gjs 里仍然使用 @ember/component，是历史包袱
import Component from "@ember/component";
import { tagName } from "@ember-decorators/component";
@tagName("")
export default class OldComponent extends Component { ... }

// ✅ 新代码应使用 @glimmer/component
import Component from "@glimmer/component";
export default class NewComponent extends Component { ... }
```

```js
// ❌ 模板放在类外部
const template = <template>...</template>;
class MyComp extends Component {}
MyComp.template = template;  // 错误做法

// ✅ 模板写在 class 内部
export default class MyComp extends Component {
  <template>...</template>
}
```

---

## 关联规范
- `javascript/02_services.md` — Service 的使用方式
- `javascript/05_qunit_testing.md` — 组件测试
