# JavaScript 05 — QUnit 前端测试规范

> 来源：`frontend/discourse/tests/unit/`、`frontend/discourse/tests/integration/`

---

## 1. 测试类型概览

| 类型 | 目录 | 用途 | Setup 函数 |
|------|------|------|------------|
| Unit | `tests/unit/` | 测试纯逻辑（Service、Model、Lib） | `setupTest(hooks)` |
| Integration | `tests/integration/` | 测试组件渲染与交互 | `setupRenderingTest(hooks)` |
| Acceptance | `tests/acceptance/` | 端到端用户流程 | `setupAcceptanceTest(hooks)` |

---

## 2. 文件命名规范

```
tests/unit/services/emoji-store-test.js        # Unit
tests/integration/components/my-comp-test.gjs  # Integration（.gjs 格式）
```

- Unit 测试用 `.js`，Integration 测试用 `.gjs`（需要 JSX/模板语法）
- 文件名格式：`{被测文件名}-test.js(gjs)`

---

## 3. Unit 测试骨架（Service）

```js
import { getOwner } from "@ember/owner";
import { setupTest } from "ember-qunit";
import { module, test } from "qunit";

module("Unit | Service | emoji-store", function (hooks) {
  setupTest(hooks);  // ← Unit 测试必须调用

  hooks.beforeEach(function () {
    // 通过 Ember 容器获取 Service 实例
    this.emojiStore = getOwner(this).lookup("service:emoji-store");
  });

  hooks.afterEach(function () {
    // 清理状态，避免测试间互相污染
    this.emojiStore.reset();
  });

  test("描述此测试做什么", function (assert) {
    // 执行操作
    this.emojiStore.trackEmojiForContext("grinning", "topic");

    // 断言结果
    assert.deepEqual(
      this.emojiStore.favoritesForContext("topic"),
      ["grinning"],
      "描述期望的结果"  // ← 断言消息（可选但推荐）
    );
  });
});
```

---

## 4. Integration 测试骨架（组件）

```gjs
import { click, fillIn, render, select } from "@ember/test-helpers";
import { module, test } from "qunit";
import MyComponent from "discourse/components/my-component";
import { setupRenderingTest } from "discourse/tests/helpers/component-test";

module("Integration | Component | MyComponent", function (hooks) {
  setupRenderingTest(hooks);  // ← Integration 必须调用

  test("renders correctly", async function (assert) {
    this.set("value", "hello");  // 在 this 上设置数据

    // render 使用 .gjs 模板语法（非字符串）
    await render(
      <template>
        <MyComponent @value={{this.value}} />
      </template>
    );

    assert.dom(".my-component").exists("component renders");
    assert.dom(".title").hasText("hello", "displays the value");
  });
});
```

---

## 5. 常用断言

### 5.1 DOM 断言（assert.dom）

```js
// 元素存在性
assert.dom(".selector").exists("element exists");
assert.dom(".selector").doesNotExist("element absent");
assert.dom(".selector").exists({ count: 3 }, "exactly 3 items");

// 文本内容
assert.dom(".title").hasText("Expected Text", "has correct text");
assert.dom("input").hasValue("", "input is empty");

// 属性
assert.dom("input").hasAttribute("placeholder", "Search...");
```

### 5.2 值断言

```js
assert.strictEqual(actual, expected, "message");  // === 比较
assert.deepEqual(actual, expected, "message");     // 深比较（数组/对象）
assert.ok(value, "is truthy");
assert.notOk(value, "is falsy");
```

### 5.3 步骤验证（assert.step / assert.verifySteps）

```js
// 验证回调/事件被调用的顺序
this.set("callback", (value) => {
  assert.step(`filter:${value}`);
});

await fillIn(".filter-input", "test");

assert.verifySteps(["filter:test"], "callback called with correct value");
```

---

## 6. 用户交互 Helpers

```js
import { click, fillIn, render, select, triggerKeyEvent } from "@ember/test-helpers";

// 点击
await click(".btn");

// 输入文本
await fillIn("input", "search term");

// 选择下拉
await select("select", "option-value");

// 键盘事件
await triggerKeyEvent("input", "keydown", "Enter");
```

所有交互 helper 都是 async，必须 await。

---

## 7. module 命名约定

```
"Unit | Service | emoji-store"
"Unit | Model | user"
"Unit | Lib | key-value-store"
"Integration | Component | AdminFilterControls"
"Integration | Helper | format-date"
"Acceptance | Composer | create topic"
```

格式：`"类型 | 类别 | 被测名称"`

---

## 8. hooks 生命周期

```js
module("...", function (hooks) {
  setupTest(hooks);

  hooks.beforeEach(function () {
    // 每个 test 前执行：初始化数据、获取实例
  });

  hooks.afterEach(function () {
    // 每个 test 后执行：清理状态、还原 mock
  });
});
```

---

## 9. this.set 传递数据

Integration 测试中通过 `this.set()` 传数据到模板：

```js
test("filters data", async function (assert) {
  this.set("data", SAMPLE_DATA);                  // 数组
  this.set("callback", (val) => { /* ... */ });   // 函数

  await render(
    <template>
      <MyComponent @array={{this.data}} @onChange={{this.callback}} />
    </template>
  );
});
```

---

## 10. getOwner 获取 Service 实例

Unit 测试中通过 Ember 容器查找 Service：

```js
import { getOwner } from "@ember/owner";

hooks.beforeEach(function () {
  this.myService = getOwner(this).lookup("service:my-service");
});
```

格式：`"service:kebab-case-name"`

---

## 11. 测试文件完整示例（Unit）

```js
import { getOwner } from "@ember/owner";
import { setupTest } from "ember-qunit";
import { module, test } from "qunit";
import { STORE_NAMESPACE, USER_EMOJIS_STORE_KEY } from "discourse/services/emoji-store";

module("Unit | Service | emoji-store", function (hooks) {
  setupTest(hooks);

  hooks.beforeEach(function () {
    this.emojiStore = getOwner(this).lookup("service:emoji-store");
  });

  hooks.afterEach(function () {
    this.emojiStore.reset();
  });

  test(".trackEmojiForContext persists to storage", function (assert) {
    this.emojiStore.trackEmojiForContext("grinning", "topic");

    assert.deepEqual(
      this.emojiStore.favoritesForContext("topic"),
      ["grinning"]
    );
  });

  test("sorts emojis by frequency", function (assert) {
    this.emojiStore.trackEmojiForContext("cat", "topic");
    this.emojiStore.trackEmojiForContext("cat", "topic");
    this.emojiStore.trackEmojiForContext("grinning", "topic");

    assert.deepEqual(
      this.emojiStore.favoritesForContext("topic"),
      ["cat", "grinning"]
    );
  });
});
```

---

## 12. 规范速查

| 规范 | 说明 |
|------|------|
| Unit 测试 | `setupTest(hooks)` + `.js` 文件 |
| Integration 测试 | `setupRenderingTest(hooks)` + `.gjs` 文件 |
| 异步交互 | 所有 DOM helper 必须 `await` |
| 数据传递 | `this.set("key", value)` |
| Service 获取 | `getOwner(this).lookup("service:name")` |
| 断言消息 | 推荐加，说明期望结果 |
| 清理 | `afterEach` 还原状态，避免测试互污 |
| 步骤验证 | `assert.step()` + `assert.verifySteps()` |
