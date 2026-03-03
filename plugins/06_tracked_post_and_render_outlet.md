# Plugin 06 — addTrackedPostProperties + renderAfterWrapperOutlet

> 来源：`plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`、`plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`

## 概述

`addTrackedPostProperties` 将后端 Post 序列化字段声明为响应式，使字段变化能驱动 UI 自动更新。`renderAfterWrapperOutlet` 在帖子 Markdown 内容区域下方注入任意 Glimmer 组件，是 Post 内容区扩展的官方方式。两者几乎总是配合使用。

---

## 来源验证

- `plugins/discourse-solved/assets/javascripts/discourse/initializers/extend-for-solved-button.gjs`（addTrackedPostProperties 3 个字段 + renderAfterWrapperOutlet + shouldRender）
- `plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`（6 个 tracked 属性 + renderAfterWrapperOutlet）

---

## 核心规范

### 1. addTrackedPostProperties

```javascript
api.addTrackedPostProperties(
  "can_accept_answer",     // 对应后端 add_to_serializer(:post, :can_accept_answer)
  "accepted_answer",
  "topic_accepted_answer"
);
```

作用：将后端 Post 序列化器字段标记为 `@tracked`，使 Glimmer 响应式系统监测变化并自动重渲染依赖该字段的 UI。

没有这个声明，字段值变更不会触发 UI 更新，即使后端已通过 `add_to_serializer` 注册。

assign 插件示例（6 个字段一次声明）：

```javascript
api.addTrackedPostProperties(
  "assigned_to_group",
  "assigned_to_group_id",
  "assigned_to_user",
  "assigned_to_user_id",
  "assignment_note",
  "assignment_status"
);
```

---

### 2. renderAfterWrapperOutlet

```javascript
api.renderAfterWrapperOutlet(
  "post-content-cooked-html",  // 固定插槽名，帖子 Markdown 内容区域下方
  MyComponent                   // Glimmer 组件类
);
```

组件通过 `@post` arg 接收当前帖子对象，通过 `@decoratorState` 接收装饰器状态。

---

### 3. static shouldRender（性能过滤）

组件声明 `static shouldRender` 可在渲染前提前过滤，避免对不相关帖子渲染：

```javascript
// 内联匿名类（适合一次性简单情况）
api.renderAfterWrapperOutlet(
  "post-content-cooked-html",
  class extends Component {
    static shouldRender(args) {
      // args.post 为当前帖子对象，只能用同步属性
      return args.post?.post_number === 1 && args.post?.topic?.accepted_answer;
    }

    <template>
      <SolvedAcceptedAnswer @post={{@post}} @decoratorState={{@decoratorState}} />
    </template>
  }
);

// 独立组件文件（可复用，更常见）
// components/post-assignments-display.gjs
export default class PostAssignmentsDisplay extends Component {
  static shouldRender(args) {
    return args.post?.assigned_to_user || args.post?.assigned_to_group;
  }

  <template>{{! 渲染 assign 信息 }}</template>
}

// 在 initializer 中注册：
api.renderAfterWrapperOutlet("post-content-cooked-html", PostAssignmentsDisplay);
```

---

### 4. 组件内操作 @post 字段

```javascript
import Component from "@glimmer/component";
import { action } from "@ember/object";
import { ajax } from "discourse/lib/ajax";
import { popupAjaxError } from "discourse/lib/ajax-error";

export default class MyPostAddon extends Component {
  @action
  async acceptSolution() {
    try {
      await ajax("/solution/accept", {
        type: "POST",
        data: { id: this.args.post.id },
      });
      // 更新 @tracked 属性 → Glimmer 自动重渲染
      this.args.post.set("accepted_answer", true);
      this.args.post.set("can_accept_answer", false);
    } catch (e) {
      popupAjaxError(e);
    }
  }

  <template>
    {{#if @post.can_accept_answer}}
      <DButton @action={{this.acceptSolution}} @icon="check" @label="Accept" />
    {{/if}}
    {{#if @post.accepted_answer}}
      <span class="solved-label">已解决</span>
    {{/if}}
  </template>
}
```

---

### 5. 完整端到端流程

后端 plugin.rb：

```ruby
# 步骤 1：注册序列化字段
add_to_serializer(:post, :can_accept_answer) do
  scope.can_accept_answer?(object, object.topic)
end

add_to_serializer(:post, :accepted_answer) { object.solution? }
```

前端 initializer：

```javascript
// 步骤 2：声明响应式
api.addTrackedPostProperties("can_accept_answer", "accepted_answer");

// 步骤 3：注入 UI
api.renderAfterWrapperOutlet(
  "post-content-cooked-html",
  class extends Component {
    static shouldRender(args) {
      return args.post?.can_accept_answer || args.post?.accepted_answer;
    }
    <template><SolvedAcceptAnswerButton @post={{@post}} /></template>
  }
);

// 步骤 4：MessageBus 实时同步（可选）
api.registerCustomPostMessageCallback("accepted_solution", (controller, message) => {
  const topic = controller.model;
  if (topic) {
    topic.setAcceptedSolution(message.accepted_answer);
    // setAcceptedSolution 内部调用 post.setProperties({accepted_answer: true})
    // 因为 accepted_answer 是 @tracked，UI 自动更新
  }
});
```

---

## 反模式（避免这样做）

```javascript
// 后端字段未声明就读取
this.args.post.my_new_field;  // 不是 @tracked，字段变化不触发重渲染
// 必须先：api.addTrackedPostProperties("my_new_field");

// shouldRender 做异步操作（不支持）
static shouldRender(args) {
  return await fetch("/check");  // 错误！shouldRender 是同步的
}

// 用 DOM 操作插入帖子内容
document.querySelector(".cooked").insertAdjacentHTML("afterend", "<div/>");
// 应使用：api.renderAfterWrapperOutlet(...)
```

---

## 关联规范

- `plugins/04_value_transformer_dag.md` — Post 菜单按钮注入
- `ruby/04_serializers.md` — add_to_serializer 后端实现
- `javascript/06_message_bus.md` — MessageBus 实时推送

---

## 快速参考

```javascript
// 1. 声明后端字段为响应式
api.addTrackedPostProperties("field_a", "field_b");

// 2. 注入帖子内容区组件
api.renderAfterWrapperOutlet("post-content-cooked-html", MyComponent);

// 3. 组件带条件渲染
class MyComponent extends Component {
  static shouldRender(args) { return !!args.post?.my_flag; }
  <template>{{#if @post.my_flag}}<MyUI @post={{@post}} />{{/if}}</template>
}

// 4. 触发 UI 更新
this.args.post.set("my_field", newValue);
```
