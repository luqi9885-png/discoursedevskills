# Plugin 17 — topic-list 自定义列与行样式（topic-list-columns / topic-list-item-class）

> 来源：`plugins/discourse-assign/initializers/assignment-list-actions.gjs`、`plugins/discourse-assign/components/assigned-topic-list-column.gjs`、`plugins/discourse-ai/initializers/ai-gist-topic-list-class.js`

## 概述

通过 `registerValueTransformer` 的两个 transformer key，插件可以向话题列表动态添加自定义列（`topic-list-columns`）或为话题行添加 CSS 类（`topic-list-item-class`）。两者常配合使用，均支持按路由名条件生效。

---

## 来源验证

| 验证点 | 文件 | 关键代码 |
|--------|------|---------|
| 按路由条件添加自定义列 | `assignment-list-actions.gjs` | `columns.add("assign-actions", { item, after: "activity" })` |
| 按路由条件添加行 CSS 类 | `assignment-list-actions.gjs` | `classes.push("assigned-list-item")` |
| 按 Service 状态添加行 CSS 类 | `ai-gist-topic-list-class.js` | `context.topic.get("ai_topic_gist")`, `value.push("excerpt-expanded")` |
| 自定义列组件完整实现 | `assigned-topic-list-column.gjs` | `@topic` arg，条件渲染，Service 注入 |

---

## 核心规范

### 1. topic-list-columns — 添加自定义列

```javascript
// initializers/assignment-list-actions.gjs
import { withPluginApi } from "discourse/lib/plugin-api";
import AssignedTopicListColumn from "../components/assigned-topic-list-column";

// 只在特定路由显示的列（在 initialize 里获取 router service）
const ASSIGN_LIST_ROUTES = ["userActivity.assigned", "group.assigned.show"];

// 列的内容组件：必须渲染 <td>，接受 @topic arg
const AssignActionsCell = <template>
  <td class="assign-topic-buttons">
    <AssignedTopicListColumn @topic={{@topic}} />
  </td>
</template>;

export default {
  name: "assignment-list-dropdowns",

  initialize(container) {
    const router = container.lookup("service:router");

    withPluginApi((api) => {
      api.registerValueTransformer(
        "topic-list-columns",
        ({ value: columns }) => {
          if (ASSIGN_LIST_ROUTES.includes(router.currentRouteName)) {
            columns.add("assign-actions", {
              item: AssignActionsCell,  // 列内容组件（渲染为 <td>）
              after: "activity",        // 插到哪一列之后（"activity" | "posters" | "likes" | "views" | "replies"）
            });
          }

          return columns; // 必须 return columns
        }
      );
    });
  },
};
```

**columns.add 参数说明：**

| 参数 | 类型 | 说明 |
|------|------|------|
| key（第一个参数）| string | 列的唯一标识，用于去重和排序 |
| `item` | Glimmer Component | 渲染列内容，必须输出 `<td>` 元素 |
| `after` | string | 放到哪一列之后，核心列：`"replies"`, `"views"`, `"activity"`, `"posters"`, `"likes"` |
| `before` | string | 放到哪一列之前（与 `after` 二选一）|

**列组件接收的 arg：**
- `@topic` — 当前行的 Topic model，有 `assigned_to_user`、`assigned_to_group` 等插件注入属性

---

### 2. topic-list-item-class — 为话题行添加 CSS 类

```javascript
// 方式一：按路由条件添加类（与 topic-list-columns 配合，在同一 transformer 中）
api.registerValueTransformer(
  "topic-list-item-class",
  ({ value: classes }) => {
    if (ASSIGN_LIST_ROUTES.includes(router.currentRouteName)) {
      classes.push("assigned-list-item");
    }
    return classes; // 必须 return classes
  }
);

// 方式二：按 topic 字段 + Service 状态添加类（discourse-ai 模式）
// 使用 apiInitializer（无需 container 注入 router 时更简洁）
import { apiInitializer } from "discourse/lib/api";

export default apiInitializer((api) => {
  const gistService = api.container.lookup("service:gists");

  api.registerValueTransformer(
    "topic-list-item-class",
    ({ value, context }) => {
      const shouldShow =
        gistService.currentPreference === "table-ai" && gistService.showToggle;

      // context.topic：当前行的 Topic model（用 .get() 读取 tracked 属性）
      if (context.topic.get("ai_topic_gist") && shouldShow) {
        value.push("excerpt-expanded");
      }

      return value;
    }
  );
});
```

**transformer 回调参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `value` | Array\<string\> | 当前已有的 CSS 类数组，直接 push 后 return |
| `context.topic` | Topic | 当前行的 Topic 对象，用 `.get("field")` 读取 |
| `context` | object | 也可能包含 `currentUser` 等其他上下文 |

---

### 3. 自定义列组件完整实现

列组件通过 `@topic` 接收当前行 Topic，通常注入 Service 处理操作：

```javascript
// components/my-topic-list-column.gjs
import Component from "@glimmer/component";
import { action } from "@ember/object";
import { service } from "@ember/service";
import MyActionDropdown from "./my-action-dropdown";

export default class MyTopicListColumn extends Component {
  @service myPluginActions; // 注入插件 Service 处理业务逻辑
  @service router;

  @action
  async doAction(topicId) {
    await this.myPluginActions.performAction(topicId);
    this.router.refresh(); // 刷新当前路由，重新加载列表
  }

  @action
  doOtherAction(topic) {
    this.myPluginActions.showModal(topic, {
      onSuccess: () => this.router.refresh(),
    });
  }

  <template>
    {{! 通过 @topic 访问话题字段，包括插件注入的 serializer 字段 }}
    {{#if @topic.my_plugin_field}}
      <MyActionDropdown
        @topic={{@topic}}
        @value={{@topic.my_plugin_field}}
        @onAction={{this.doAction}}
        @onOtherAction={{this.doOtherAction}}
      />
    {{else}}
      {{! 空状态：无操作时显示占位 }}
      <span class="no-value">—</span>
    {{/if}}
  </template>
}
```

**注意**：`@topic.my_plugin_field` 的值来自后端 serializer（需要用 `add_to_serializer(:topic_list_item, :my_plugin_field)` 注入，并在 `TopicList.on_preload` 里预加载避免 N+1）。

---

### 4. 后端配合：TopicList.on_preload 预加载

列组件需要的 topic 字段通过 serializer 注入后，必须在 `TopicList.on_preload` 里批量预加载，否则每个 topic 会触发一次独立查询（N+1）：

```ruby
# plugin.rb

# 1. 注入 serializer 字段
add_to_serializer(:topic_list_item, :my_plugin_data) do
  # 不能在这里做查询！要在 on_preload 里预先设置
  object.my_preloaded_data  # 读取 topic 上预先挂载的值
end

add_to_serializer(:topic_list_item, :include_my_plugin_data?) do
  SiteSetting.my_plugin_enabled?
end

# 2. on_preload：批量预加载，避免 N+1
TopicList.on_preload do |topics, topic_list|
  next unless SiteSetting.my_plugin_enabled?

  # 批量查询
  data_map = MyPluginRecord
    .where(topic_id: topics.map(&:id))
    .index_by(&:topic_id)

  topics.each do |topic|
    # 将数据挂到 topic 对象上供 serializer 读取
    # 注意：即使没有数据也要赋 nil，避免后续 N+1
    topic.my_preloaded_data = data_map[topic.id]&.value
  end
end
```

---

### 5. 组合模式：同时注册 columns + item-class

两个 transformer 在同一 initializer 里注册是推荐做法，共享 router service：

```javascript
export default {
  name: "my-plugin-topic-list",

  initialize(container) {
    const router = container.lookup("service:router");
    const myService = container.lookup("service:my-plugin");

    withPluginApi((api) => {
      // 1. 添加自定义列
      api.registerValueTransformer(
        "topic-list-columns",
        ({ value: columns }) => {
          if (router.currentRouteName === "userActivity.myFeature") {
            columns.add("my-column", {
              item: MyColumnCell,
              after: "activity",
            });
          }
          return columns;
        }
      );

      // 2. 为行添加 CSS 类
      api.registerValueTransformer(
        "topic-list-item-class",
        ({ value: classes, context }) => {
          if (router.currentRouteName === "userActivity.myFeature") {
            classes.push("my-feature-row");
          }
          // 也可以按 topic 字段条件添加
          if (context.topic.get("my_urgent_field")) {
            classes.push("my-feature-urgent");
          }
          return classes;
        }
      );
    });
  },
};
```

---

## 反模式

```javascript
// 忘记 return columns / return classes（transformer 返回 undefined，列表消失）
api.registerValueTransformer("topic-list-columns", ({ value: columns }) => {
  columns.add("my-col", { item: MyCell, after: "activity" });
  // ❌ 忘记 return columns
});

// ✅ 必须 return
api.registerValueTransformer("topic-list-columns", ({ value: columns }) => {
  columns.add("my-col", { item: MyCell, after: "activity" });
  return columns; // ✅
});

// 列组件用 <div> 而不是 <td>（会破坏 table 结构）
const MyCell = <template>
  <div class="my-col">...</div>   // ❌ 应该是 <td>
</template>;

const MyCell = <template>
  <td class="my-col">...</td>     // ✅
</template>;

// 在 serializer block 里做查询（N+1）
add_to_serializer(:topic_list_item, :my_data) do
  MyRecord.find_by(topic_id: object.id)&.value  // ❌ 每个 topic 一次查询！
end

# ✅ 在 on_preload 批量预加载，serializer 只读预加载值
TopicList.on_preload do |topics, _|
  map = MyRecord.where(topic_id: topics.map(&:id)).index_by(&:topic_id)
  topics.each { |t| t.my_preloaded_data = map[t.id]&.value }
end
add_to_serializer(:topic_list_item, :my_data) { object.my_preloaded_data }
```

---

## 关联规范

- `plugins/04_value_transformer_dag.md` — registerValueTransformer 与 DAG 原理
- `plugins/06_tracked_post_and_render_outlet.md` — addTrackedPostProperties（post 级别的 tracked 属性）
- `plugins/11_register_modifier_backend.md` — 后端 register_modifier（与 on_preload 配合）

---

## 快速参考

```javascript
// 添加自定义列
const MyCell = <template><td class="my-col"><MyComponent @topic={{@topic}} /></td></template>;

api.registerValueTransformer("topic-list-columns", ({ value: columns }) => {
  if (router.currentRouteName === "myRoute") {
    columns.add("my-key", { item: MyCell, after: "activity" });
  }
  return columns;
});

// 添加行 CSS 类（按路由 + 按 topic 字段）
api.registerValueTransformer("topic-list-item-class", ({ value, context }) => {
  if (router.currentRouteName === "myRoute") value.push("my-route-class");
  if (context.topic.get("my_flag")) value.push("my-flag-class");
  return value;
});

// 后端预加载（避免 N+1）
TopicList.on_preload do |topics, _|
  map = MyRecord.where(topic_id: topics.map(&:id)).index_by(&:topic_id)
  topics.each { |t| t.my_data = map[t.id]&.value }
end
add_to_serializer(:topic_list_item, :my_data) { object.my_data }
```
