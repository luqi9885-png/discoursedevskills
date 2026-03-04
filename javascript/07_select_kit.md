# javascript/07 — SelectKit 前端组件规范

> 来源：
> - `plugins/discourse-assign/assets/javascripts/discourse/components/assignment.gjs`（ComboBox + EmailGroupUserChooser）
> - `plugins/discourse-assign/assets/javascripts/discourse/components/remind-assigns-frequency.gjs`（ComboBox + valueProperty）
> - `plugins/discourse-ai/assets/javascripts/discourse/components/ai-tool-selector.gjs`（MultiSelect 最简用法）
> - `plugins/discourse-ai/assets/javascripts/discourse/components/ai-usage.gjs`（ComboBox + none + content 构造）
> - `plugins/discourse-post-voting/assets/javascripts/discourse/initializers/extend-composer-actions.js`（modifySelectKit）
> - `frontend/discourse/select-kit/components/combo-box.js`（ComboBox 源码）
> - `frontend/discourse/select-kit/components/multi-select.gjs`（MultiSelect 源码）
> - `frontend/discourse/select-kit/lib/plugin-api.js`（modifySelectKit API）
> 修订历史：
> - 2026-03-04 初版

## 边界声明

- **本文件负责**：SelectKit 系列组件（ComboBox、MultiSelect、EmailGroupUserChooser）在插件中的使用规范，以及 `modifySelectKit` Plugin API
- **不负责，见其他文件**：
  - Glimmer Component 基础 -> `javascript/01_component_gjs.md`
  - withPluginApi 基础结构 -> `plugins/02_plugin_api_frontend.md`

---

## 概述

SelectKit 是 Discourse 的下拉选择组件体系，插件中最常用的三个：`ComboBox`（单选下拉）、`MultiSelect`（多选标签式）、`EmailGroupUserChooser`（用户/群组搜索选择）。核心接口统一：`@content`（数据）、`@value`（当前值）、`@onChange`（回调）、`@options`（配置项）。

---

## 来源验证

| 验证点 | 文件 | 关键代码 | 差异 |
|--------|------|---------|------|
| ComboBox 基本用法 | `assignment.gjs` | `@content={{this.assignStatusOptions}}` | content 为 `{id, name}` 数组，value 匹配 id |
| ComboBox + valueProperty | `remind-assigns-frequency.gjs` | `@valueProperty="value"` | content 为 `{name, value}` 时需指定 valueProperty |
| ComboBox + none 占位 | `ai-usage.gjs` | `@options={{hash none="i18n.key"}}` | none 选项的 i18n key，选中时 value 为 undefined |
| MultiSelect 最简 | `ai-tool-selector.gjs` | `@options={{hash filterable=true allowAny=false}}` | 仅三个 arg，最简洁写法 |
| EmailGroupUserChooser | `assignment.gjs` | `@options={{hash includeGroups=true maximum=1}}` | 用户+群组混合，maximum=1 限制单选 |
| content 构造模式 | `ai-usage.gjs:401~430` | `features.map((f) => ({ id: f.feature_name, name: f.feature_name }))` | API 数据转为标准格式 |
| modifySelectKit | `extend-composer-actions.js:55` | `api.modifySelectKit("composer-actions").appendContent(...)` | 向已有 SelectKit 追加选项 |

---

## 核心规范

### 1. content 数据格式

所有 SelectKit 组件的 `@content` 默认期望 `{ id, name }` 格式：

```javascript
// 标准格式（默认 valueProperty="id", nameProperty="name"）
get statusOptions() {
  return [
    { id: "open",   name: i18n("my_plugin.status.open") },
    { id: "closed", name: i18n("my_plugin.status.closed") },
  ];
}

// 自定义属性名（需配合 @valueProperty / @nameProperty）
get frequencyOptions() {
  return [
    { value: 0,  name: i18n("my_plugin.frequency.never") },
    { value: 60, name: i18n("my_plugin.frequency.hourly") },
  ];
}

// 从 API 数据构造（来自 ai-usage.gjs 真实代码）
get availableFeatures() {
  return (this.data?.features || []).map((f) => ({
    id: f.feature_name,
    name: f.feature_name,
  }));
}
```

### 2. ComboBox（单选下拉）

```javascript
import ComboBox from "discourse/select-kit/components/combo-box";
```

基本用法（来自 assignment.gjs 真实代码）：

```hbs
<ComboBox
  @id="assign-status"
  @content={{this.assignStatusOptions}}
  @value={{this.status}}
  @onChange={{this.setStatus}}
/>
```

带 none 占位项（来自 ai-usage.gjs 真实代码）：

```hbs
<ComboBox
  @value={{this.selectedFeature}}
  @content={{this.availableFeatures}}
  @onChange={{this.onFeatureChanged}}
  @options={{hash none="discourse_ai.usage.all_features"}}
/>
```

带自定义 valueProperty（来自 remind-assigns-frequency.gjs 真实代码）：

```hbs
<ComboBox
  @id="remind-assigns-frequency"
  @valueProperty="value"
  @content={{this.availableFrequencies}}
  @value={{this.selectedFrequency}}
  @onChange={{fn (mut this.user.custom_fields.remind_assigns_frequency)}}
/>
```

常用 @options：

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `none` | — | 空选项的 i18n key，选中时 value 为 null |
| `filterable` | 条目>=10 自动开启 | 是否显示搜索框 |
| `clearable` | `false` | 是否显示清除按钮 |
| `disabled` | `false` | 禁用整个组件 |
| `translatedNone` | — | 空选项的直接文字（不走 i18n） |

### 3. MultiSelect（多选）

```javascript
import MultiSelect from "discourse/select-kit/components/multi-select";
```

最简用法（来自 ai-tool-selector.gjs 真实代码）：

```hbs
<MultiSelect
  @value={{@value}}
  @onChange={{@onChange}}
  @content={{@content}}
  @options={{hash filterable=true allowAny=false disabled=@disabled}}
/>
```

value 是数组，onChange 也接收数组：

```javascript
@tracked selectedToolIds = [];

@action
onToolsChanged(newIds) {
  this.selectedToolIds = newIds;  // newIds 是数组
}
```

常用 @options：

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `filterable` | `true` | 是否显示搜索框 |
| `allowAny` | `false` | 是否允许输入任意值 |
| `maximum` | — | 最多选几个，达到后隐藏加号 |
| `disabled` | `false` | 禁用 |

### 4. EmailGroupUserChooser（用户/群组搜索）

```javascript
import EmailGroupUserChooser from "discourse/select-kit/components/email-group-user-chooser";
```

完整用法（来自 assignment.gjs 真实代码）：

```hbs
<EmailGroupUserChooser
  autocomplete="off"
  @id="assignee-chooser"
  @value={{this.assignee}}
  @onChange={{this.setAssignee}}
  @showUserStatus={{true}}
  @options={{hash
    includeGroups=true
    groupMembersOf=this.taskActions.allowedGroups
    maximum=1
    expandedOnInsert=(not this.assignee)
    caretUpIcon="magnifying-glass"
    caretDownIcon="magnifying-glass"
  }}
/>
```

onChange 接收数组，maximum=1 时用解构（来自 assignment.gjs 真实代码）：

```javascript
@action
setAssignee([newAssignee]) {  // 解构取第一个
  if (this.taskActions.allowedGroupsForAssignment.includes(newAssignee)) {
    this.args.assignment.username = null;
    this.args.assignment.group_name = newAssignee;
  } else {
    this.args.assignment.username = newAssignee;
    this.args.assignment.group_name = null;
  }
}
```

常用 @options：

| 选项 | 说明 |
|------|------|
| `includeGroups` | 搜索结果包含群组 |
| `includeMessageableGroups` | 只包含可发私信的群组 |
| `groupMembersOf` | 限定候选人必须是这些群组的成员 |
| `maximum` | 最多选几个（1 = 单选） |
| `excludeCurrentUser` | 排除当前登录用户 |
| `expandedOnInsert` | 渲染后立即展开下拉 |

### 5. modifySelectKit — 插件扩展已有组件

向 Discourse 核心或其他插件的 SelectKit 追加/替换选项（来自 discourse-post-voting 真实代码）：

```javascript
withPluginApi("1.0", (api) => {
  api.modifySelectKit("composer-actions")
    .appendContent((component, content) => {
      // 条件不满足时返回 undefined，不追加
      if (!component.selectKit.options.showPostVoting) {
        return;
      }
      return {
        id: "toggle_post_voting",
        name: i18n("post_voting.composer.toggle"),
        icon: "chart-bar",
      };
    })
    .onChange((component, value) => {
      // 链式：响应选择变化
      if (value === "toggle_post_voting") {
        component.composerModel.toggleProperty("post_voting");
      }
    });
});
```

modifySelectKit 四个方法：

| 方法 | 回调参数 | 用途 |
|------|---------|------|
| `appendContent(cb)` | `(component, content)` | 在列表末尾追加选项 |
| `prependContent(cb)` | `(component, content)` | 在列表开头插入选项 |
| `replaceContent(cb)` | `(component, content)` | 完全替换选项列表 |
| `onChange(cb)` | `(component, value, item)` | 拦截/响应选择变化 |

pluginApiIdentifiers 速查（可 modifySelectKit 的目标）：

| identifier | 对应组件 | 典型用途 |
|-----------|---------|---------|
| `"composer-actions"` | Composer 操作下拉 | 添加自定义发帖类型 |
| `"topic-notifications-button"` | 话题通知级别 | 自定义通知级别 |
| `"category-notifications-button"` | 分类通知级别 | 同上 |

---

## 反模式

```javascript
// 错误1：content 用 label 而非 name
get options() {
  return [{ id: "open", label: "Open" }];  // label 不识别，显示为空
}
// 正确：{ id, name }，或指定 @nameProperty="label"

// 错误2：MultiSelect onChange 当单值处理
@action
onChanged(value) {
  this.selectedId = value;  // value 实际是数组！
}
// 正确：
@action
onChanged(values) { this.selectedIds = values; }

// 错误3：EmailGroupUserChooser onChange 忘记解构
@action
setAssignee(newAssignee) {    // 实际收到 ["username"]
  this.username = newAssignee;  // 存的是数组
}
// 正确：
@action
setAssignee([newAssignee]) { this.username = newAssignee; }

// 错误4：modifySelectKit 回调无条件追加
api.modifySelectKit("composer-actions").appendContent(() => {
  return { id: "my_action", name: "My Action" };  // 永远追加
});
// 正确：条件不满足时 return（不返回值）

// 错误5：用原生 <select> 代替 SelectKit
// 导致样式不一致、无法被其他插件通过 modifySelectKit 扩展
```

---

## 快速参考

```javascript
// 导入
import ComboBox from "discourse/select-kit/components/combo-box";
import MultiSelect from "discourse/select-kit/components/multi-select";
import EmailGroupUserChooser from "discourse/select-kit/components/email-group-user-chooser";

// content 格式：{ id, name }（默认）或自定义 + @valueProperty

// ComboBox — 单选
// <ComboBox @content={{this.items}} @value={{this.val}} @onChange={{this.onChanged}}
//           @options={{hash none="i18n.key"}} />

// MultiSelect — 多选，value/onChange 均为数组
// <MultiSelect @content={{this.items}} @value={{this.vals}} @onChange={{this.onChanged}}
//              @options={{hash filterable=true allowAny=false}} />

// EmailGroupUserChooser — 用户/群组，onChange 解构
// <EmailGroupUserChooser @value={{this.name}} @onChange={{this.setName}}
//                        @options={{hash includeGroups=true maximum=1}} />
// @action setName([name]) { this.name = name; }

// modifySelectKit
withPluginApi("1.0", (api) => {
  api.modifySelectKit("composer-actions").appendContent((component) => {
    if (!shouldShow(component)) { return; }
    return { id: "my_id", name: "My Label", icon: "star" };
  });
});
```
