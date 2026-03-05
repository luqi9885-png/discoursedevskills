# javascript/08 — FormKit 表单框架规范

> 来源：
> - `frontend/discourse/app/form-kit/components/fk/form.gjs`（Form 核心实现）
> - `frontend/discourse/app/form-kit/components/fk/field.gjs`（Field 控件体系）
> - `frontend/discourse/app/form-kit/lib/validator.js`（内建验证规则）
> - `frontend/discourse/app/form-kit/lib/validation-parser.js`（@validation 字符串解析）
> - `frontend/discourse/app/form-kit/lib/constants.js`（validateOn 常量）
> - `plugins/discourse-ai/assets/javascripts/discourse/components/ai-llm-editor-form.gjs`（完整真实用例：Field/Select/Password/Object/InputGroup/Actions/Submit）
> - `frontend/discourse/admin/components/admin-badges-show.gjs`（@onRegisterApi + formApi.reset()）
> 修订历史：
> - 2026-03-05 初版

## 边界声明

- **本文件负责**：FormKit（`discourse/components/form`）在插件 Admin 页面的使用规范
- **不负责，见其他文件**：
  - SelectKit 下拉组件 → `javascript/07_select_kit.md`
  - Admin 页面路由结构 → `plugins/15_admin_multi_page.md`
  - Admin RestModel 数据层 → `plugins/20_admin_rest_model_adapter.md`

---

## 概述

FormKit 是 Discourse 的标准 Admin 表单框架（`discourse/components/form`），117 个核心文件使用它。它处理：数据绑定（`@data`）、内建验证（`@validation`）、错误展示、脏数据检测（离开页面时提示）、submit/reset 生命周期。插件 Admin 页面的编辑表单应优先使用 FormKit 而非手工表单。

---

## 来源验证

| 验证点 | 文件 | 行号 | 差异 |
|--------|------|------|------|
| Form 基本用法 | `ai-llm-editor-form.gjs` | 300 | `@data` 传 POJO，`as \|form data\|` 解构，data 是 draftData（proxy）|
| `@onSubmit` 接收 draftData | `form.gjs` | 241 | submit 后 `await args.onSubmit(formData.draftData)` — 收到的是纯对象快照 |
| `@validation` 字符串格式 | `ai-llm-editor-form.gjs` | 318 | `"required\|length:1,100"` — 管道分隔，冒号传参 |
| `@onSet` 拦截字段变更 | `ai-llm-editor-form.gjs` | 346 | `@onSet={{this.setProvider}}` — 值变化时调用，可联动其他字段 |
| `field.Select` + `select.Option` | `ai-llm-editor-form.gjs` | 347 | 嵌套 block 语法，Option @value 必须是字符串或基本类型 |
| `form.Object` 嵌套对象 | `ai-llm-editor-form.gjs` | 390 | `@name="provider_params" as \|object providerParamsData\|` |
| `form.InputGroup` 横排字段 | `ai-llm-editor-form.gjs` | 438 | 同行多字段，用 `inputGroup.Field` 代替 `form.Field` |
| `form.Actions` + `form.Submit` | `ai-llm-editor-form.gjs` | 628 | Actions 是操作区容器，Submit 自动处理 loading 状态 |
| `@onRegisterApi` 获取命令式 API | `admin-badges-show.gjs` | 208 | `registerApi(api) { this.formApi = api }` 再调用 `formApi.reset()` |

---

## 核心规范

### 1. 基本骨架

```javascript
import Form from "discourse/components/form";
```

```hbs
<Form
  @onSubmit={{this.save}}
  @data={{this.formData}}
  as |form data|
>
  {{! form：表单控件集（Field/Submit/Actions 等） }}
  {{! data：当前 draftData 的 proxy，读取字段当前值 }}

  <form.Field
    @name="display_name"
    @title={{i18n "my_plugin.display_name"}}
    @validation="required|length:1,100"
    as |field|
  >
    <field.Input />
  </form.Field>

  <form.Actions>
    <form.Submit />
  </form.Actions>
</Form>
```

**`@data` 必须是普通 JS 对象**（POJO），不能直接传 Ember Model 实例。通常用 getter 构造：

```javascript
// 来自 ai-llm-editor-form.gjs 真实模式
@cached
get formData() {
  return {
    display_name: this.args.model.displayName,
    url: this.args.model.url,
    provider: this.args.model.provider,
    provider_params: { ...this.args.model.providerParams },
  };
}
```

**`@onSubmit` 接收 draftData 快照（纯对象）**：

```javascript
@action
async save(data) {
  // data 是 formData 的当前值快照（纯对象，不是 proxy）
  try {
    await this.args.model.save(data);
    this.toasts.success({ data: { message: i18n("saved") } });
  } catch (e) {
    popupAjaxError(e);
  }
}
```

### 2. @validation — 内建验证规则

格式：`"rule1|rule2:arg1,arg2"` — 管道分隔，冒号传参：

```hbs
{{! 必填 }}
<form.Field @name="name" @validation="required" ...>

{{! 必填 + 长度限制 }}
<form.Field @name="title" @validation="required|length:1,100" ...>

{{! 数字范围 }}
<form.Field @name="count" @validation="number|between:0,100" ...>

{{! URL 格式 }}
<form.Field @name="endpoint" @validation="url" ...>

{{! 整数 }}
<form.Field @name="tokens" @validation="integer" ...>
```

**所有内建规则：**

| 规则 | 格式 | 说明 |
|------|------|------|
| `required` | `required` 或 `required:trim` | 必填，加 trim 则空白字符串也视为空 |
| `length` | `length:min,max` | 字符串长度范围 |
| `between` | `between:min,max` | 数值范围 |
| `number` | `number` | 必须是数字 |
| `integer` | `integer` | 必须是整数 |
| `url` | `url` | 有效 URL 格式 |
| `accepted` | `accepted` | 必须是 true/yes/on/1 |
| `dateBeforeOrEqual` | `dateBeforeOrEqual:2025-12-31` | 日期上限 |
| `dateAfterOrEqual` | `dateAfterOrEqual:2024-01-01` | 日期下限 |

**自定义跨字段验证**（`@validate`）：

```javascript
// 整个表单级别的校验，在 field 级 @validation 之后执行
@action
async validateForm(data, { addError, removeError }) {
  if (data.max_tokens < data.min_tokens) {
    addError("max_tokens", {
      title: i18n("my_plugin.max_tokens"),
      message: i18n("my_plugin.max_must_exceed_min"),
    });
  }
}
```

```hbs
<Form @validate={{this.validateForm}} @onSubmit={{this.save}} @data={{this.formData}} as |form|>
```

### 3. Field 控件类型速查

所有控件都在 `as |field|` 块里使用：

```hbs
{{! 文本输入 }}
<form.Field @name="name" @title="Name" as |field|>
  <field.Input />
  {{! 带类型：}}
  <field.Input @type="number" step="any" min="0" lang="en" />
</form.Field>

{{! 密码 }}
<form.Field @name="api_key" @title="API Key" as |field|>
  <field.Password autocomplete="off" />
</form.Field>

{{! 多行文本 }}
<form.Field @name="description" @title="Description" as |field|>
  <field.Textarea />
</form.Field>

{{! 勾选框 }}
<form.Field @name="enabled" @title="Enabled" as |field|>
  <field.Checkbox />
</form.Field>

{{! 开关 }}
<form.Field @name="active" @title="Active" as |field|>
  <field.Toggle />
</form.Field>

{{! 下拉选择（来自 ai-llm-editor-form.gjs 真实代码）}}
<form.Field @name="provider" @title="Provider" @validation="required" as |field|>
  <field.Select as |select|>
    {{#each this.providers as |provider|}}
      <select.Option @value={{provider.id}}>{{provider.name}}</select.Option>
    {{/each}}
  </field.Select>
</form.Field>

{{! 颜色选择器 }}
<form.Field @name="color" @title="Color" as |field|>
  <field.Color />
</form.Field>

{{! 图标选择器 }}
<form.Field @name="icon" @title="Icon" as |field|>
  <field.Icon />
</form.Field>

{{! 自定义控件（嵌入 SelectKit 等） }}
<form.Field @name="assignee" @title="Assignee" as |field|>
  <field.Custom>
    <EmailGroupUserChooser
      @value={{field.value}}
      @onChange={{field.set}}
      @options={{hash maximum=1}}
    />
  </field.Custom>
</form.Field>
```

### 4. @onSet — 字段变更联动

当某字段值改变时联动更新其他字段（来自 ai-llm-editor-form.gjs 真实模式）：

```javascript
@action
setProvider(provider) {
  // provider 是新值
  // 联动清除 provider_params
  this.formData = {
    ...this.formData,
    provider,
    provider_params: this.defaultParamsFor(provider),
  };
}
```

```hbs
<form.Field
  @name="provider"
  @title={{i18n "provider"}}
  @onSet={{this.setProvider}}
  as |field|
>
  <field.Select as |select|>
    {{#each this.providers as |p|}}
      <select.Option @value={{p.id}}>{{p.name}}</select.Option>
    {{/each}}
  </field.Select>
</form.Field>
```

### 5. 嵌套结构：form.Object + form.InputGroup

**form.Object** — 绑定到 data 中的子对象：

```hbs
{{! 来自 ai-llm-editor-form.gjs 真实代码 }}
<form.Object @name="provider_params" as |object providerParamsData|>
  {{! providerParamsData 是 data.provider_params 的当前值 }}
  <object.Field @name="api_version" @title="API Version" as |field|>
    <field.Input />
  </object.Field>
  <object.Field @name="region" @title="Region" as |field|>
    <field.Input />
  </object.Field>
</form.Object>
```

**form.InputGroup** — 同行横排多个字段：

```hbs
{{! 来自 ai-llm-editor-form.gjs 真实代码 }}
<form.InputGroup as |inputGroup|>
  <inputGroup.Field
    @name="input_cost"
    @title={{i18n "input_cost"}}
    @helpText={{i18n "cost_measure"}}
    as |field|
  >
    <field.Input @type="number" step="any" min="0" />
  </inputGroup.Field>
  <inputGroup.Field
    @name="output_cost"
    @title={{i18n "output_cost"}}
    as |field|
  >
    <field.Input @type="number" step="any" min="0" />
  </inputGroup.Field>
</form.InputGroup>
```

### 6. 操作区：form.Actions + form.Submit + form.Button

```hbs
{{! 来自 ai-llm-editor-form.gjs 真实代码 }}
<form.Actions>
  {{! 自定义按钮（不 submit） }}
  <form.Button
    @action={{fn this.test data}}
    @disabled={{this.testRunning}}
    @label="my_plugin.test"
  />

  {{! 主提交按钮（自动处理 loading 状态） }}
  <form.Submit />

  {{! 危险操作按钮 }}
  <form.Button
    @action={{this.delete}}
    @label="my_plugin.delete"
    class="btn-danger"
  />
</form.Actions>
```

### 7. @onRegisterApi — 命令式操作 Form

当需要在组件逻辑里（而非模板里）触发 submit/reset/set 时：

```javascript
// 来自 admin-badges-show.gjs 真实代码
@tracked formApi = null;

@action
registerApi(api) {
  this.formApi = api;
}

@action
async handleDelete() {
  // 删除前先 reset 表单（清除脏数据标记，避免导航确认弹窗）
  await this.formApi.reset();
  await this.args.badge.destroy();
}
```

```hbs
<Form
  @data={{this.formData}}
  @onSubmit={{this.save}}
  @onRegisterApi={{this.registerApi}}
  as |form|
>
  ...
</Form>
```

**formApi 可用方法：**

| 方法 | 说明 |
|------|------|
| `api.submit()` | 以编程方式触发 submit（含验证） |
| `api.reset()` | 重置表单到初始状态，清除脏数据标记 |
| `api.set(name, value)` | 以编程方式设置字段值 |
| `api.setProperties(obj)` | 批量设置多个字段 |
| `api.get(name)` | 读取字段当前值 |
| `api.addError(name, {title, message})` | 手动添加错误（后端返回错误时用） |
| `api.removeError(name)` | 清除某字段错误 |

### 8. @validateOn — 控制触发时机

```hbs
{{! 默认：提交时验证 }}
<Form @validateOn="submit" ...>

{{! 输入时实时验证 }}
<Form @validateOn="input" ...>

{{! 失焦时验证 }}
<Form @validateOn="focusout" ...>

{{! 值变化时验证（适合 Select/Toggle） }}
<Form @validateOn="change" ...>
```

---

## 反模式

```javascript
// 错误1：@data 直接传 Ember Model 实例
get formData() {
  return this.args.model;  // Model 不是 POJO，FormKit 无法追踪变更
}
// 正确：map 成 POJO
get formData() {
  return { name: this.args.model.name, url: this.args.model.url };
}

// 错误2：@onSubmit 里直接修改 this.args.model（绕过 FormKit 的 draftData 机制）
@action
async save() {
  this.args.model.name = this.currentName;  // 绕过了 FormKit
  await this.args.model.save();
}
// 正确：用 @onSubmit 接收 data 参数
@action
async save(data) {
  await this.args.model.update(data);
}

// 错误3：用 field.Custom 但忘记绑定 field.value 和 field.set
<form.Field @name="assignee" as |field|>
  <field.Custom>
    <EmailGroupUserChooser
      @value={{this.assignee}}   // 没用 field.value，FormKit 无法追踪
      @onChange={{this.setAssignee}}  // 没用 field.set，表单数据不更新
    />
  </field.Custom>
</form.Field>
// 正确：
<form.Field @name="assignee" as |field|>
  <field.Custom>
    <EmailGroupUserChooser
      @value={{field.value}}
      @onChange={{field.set}}
    />
  </field.Custom>
</form.Field>

// 错误4：验证规则格式错误
@validation="required, length:1:100"   // 分隔符错误（应用 | 和 ,）
// 正确：
@validation="required|length:1,100"
```

---

## 快速参考

```javascript
import Form from "discourse/components/form";

// @data：POJO（不是 Model 实例）
// @onSubmit(data)：data 是 draftData 快照
// @validate(data, {addError})：跨字段校验
// @onRegisterApi(api)：获取命令式 API（api.reset/submit/set）
// @validateOn：submit（默认）/ input / focusout / change
```

```hbs
<Form @data={{this.formData}} @onSubmit={{this.save}} as |form data|>
  {{! 文本 }}
  <form.Field @name="x" @title="X" @validation="required|length:1,100" as |field|>
    <field.Input />
  </form.Field>

  {{! 下拉 }}
  <form.Field @name="y" @title="Y" as |field|>
    <field.Select as |select|>
      {{#each this.items as |item|}}
        <select.Option @value={{item.id}}>{{item.name}}</select.Option>
      {{/each}}
    </field.Select>
  </form.Field>

  {{! 自定义控件 }}
  <form.Field @name="z" @title="Z" as |field|>
    <field.Custom><MyWidget @value={{field.value}} @onChange={{field.set}} /></field.Custom>
  </form.Field>

  <form.Actions>
    <form.Submit />
  </form.Actions>
</Form>
```
