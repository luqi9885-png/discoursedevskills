# Plugin 07 — Modal 弹窗与 UI 组件

> 来源：`plugins/discourse-assign/assets/javascripts/discourse/components/modal/assign-user.gjs`、`plugins/discourse-chat-integration/admin/assets/javascripts/admin/components/modal/edit-channel.gjs`、`plugins/discourse-assign/assets/javascripts/discourse/services/task-actions.js`

## 概述

Discourse 插件通过 `DModal` 组件构建弹窗 UI，通过 `modal` Service 的 `show()` 方法触发弹窗。弹窗组件接收 `@model`（数据）和 `@closeModal`（关闭回调）两个标准 arg。插件还可创建自定义 Service 封装业务逻辑，统一管理 Modal 的触发、AJAX 调用和状态。

---

## 来源验证

- `plugins/discourse-assign/assets/javascripts/discourse/components/modal/assign-user.gjs`（DModal 标准骨架）
- `plugins/discourse-chat-integration/admin/assets/javascripts/admin/components/modal/edit-channel.gjs`（DModal 表单提交模式）
- `plugins/discourse-assign/assets/javascripts/discourse/services/task-actions.js`（Service 封装 modal.show + ajax）
- `plugins/discourse-assign/assets/javascripts/discourse/components/modal/edit-topic-assignments.gjs`（复杂 Modal）

---

## 核心规范

### 1. DModal 组件骨架

```javascript
// components/modal/my-feature.gjs
import Component from "@glimmer/component";
import { action } from "@ember/object";
import { service } from "@ember/service";
import DButton from "discourse/components/d-button";
import DModal from "discourse/components/d-modal";
import DModalCancel from "discourse/components/d-modal-cancel";
import { popupAjaxError } from "discourse/lib/ajax-error";
import { ajax } from "discourse/lib/ajax";
import { i18n } from "discourse-i18n";

export default class MyFeatureModal extends Component {
  @service router;

  // this.args.model — 调用 modal.show() 时传入的数据对象
  // this.args.closeModal — 关闭弹窗的回调函数

  @action
  async save() {
    try {
      await ajax("/my_plugin/action", {
        type: "POST",
        data: { target_id: this.args.model.target.id },
      });
      this.args.closeModal();   // 成功后关闭
    } catch (e) {
      popupAjaxError(e);        // 自动弹出错误提示
    }
  }

  <template>
    <DModal
      @title={{i18n "my_plugin.modal.title"}}
      @closeModal={{@closeModal}}
    >
      <:body>
        {{! 表单内容 }}
        <p>{{@model.target.title}}</p>
      </:body>

      <:footer>
        <DButton
          class="btn-primary"
          @action={{this.save}}
          @label="my_plugin.modal.confirm"
        />
        <DModalCancel @close={{@closeModal}} />
      </:footer>
    </DModal>
  </template>
}
```

**DModal 主要参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `@title` | string | 弹窗标题（直接字符串或 i18n 键） |
| `@closeModal` | Function | 关闭回调（必须传递，通常直接 `={{@closeModal}}`）|
| `@tagName` | string | 根元素标签，`"form"` 可让内部按钮触发 submit |
| `class` | string | 自定义 CSS class |

**具名 block：**
- `<:body>` — 弹窗主体内容区
- `<:footer>` — 底部按钮区（通常是 confirm + cancel）

---

### 2. 表单提交模式（DModal + form submit）

```javascript
// 使用 @tagName="form" 让整个 Modal 变成 form 元素
// 用户按 Enter 或点击 type="submit" 按钮即可提交

export default class EditChannelModal extends Component {
  @action
  async save() {
    try {
      await this.args.model.channel.save();
      this.args.closeModal();
    } catch (e) {
      popupAjaxError(e);
    }
  }

  <template>
    <DModal
      {{on "submit" this.save}}
      @title={{i18n "edit_channel_modal.title"}}
      @closeModal={{@closeModal}}
      @tagName="form"
    >
      <:body>
        {{! 表单字段 }}
      </:body>
      <:footer>
        <DButton
          @action={{this.save}}
          @label="save"
          @disabled={{not this.isValid}}
          type="submit"
          class="btn-primary btn-large"
        />
        <DButton @action={{@closeModal}} @label="cancel" class="btn-default btn-large" />
      </:footer>
    </DModal>
  </template>
}
```

---

### 3. 从 Service 触发 Modal（推荐模式）

插件创建自定义 Service 统一管理弹窗触发，避免在组件里到处重复 `modal.show(...)` 代码：

```javascript
// services/my-plugin-actions.js
import Service, { service } from "@ember/service";
import { ajax } from "discourse/lib/ajax";
import { popupAjaxError } from "discourse/lib/ajax-error";
import MyFeatureModal from "../components/modal/my-feature";

export default class MyPluginActions extends Service {
  @service modal;

  // 封装"打开弹窗"的行为
  showMyModal(target, options = {}) {
    return this.modal.show(MyFeatureModal, {
      model: {
        target,
        isReassign: options.isReassign || false,
        onSuccess: options.onSuccess,
      },
    });
  }

  // 封装不需要弹窗的直接操作
  async doDirectAction(targetId) {
    try {
      await ajax("/my_plugin/action", {
        type: "PUT",
        data: { target_id: targetId },
      });
    } catch (e) {
      popupAjaxError(e);
    }
  }
}
```

**在其他组件中注入并使用：**

```javascript
// components/my-button.gjs
import Component from "@glimmer/component";
import { action } from "@ember/object";
import { service } from "@ember/service";

export default class MyButton extends Component {
  @service myPluginActions;  // 注入自定义 Service

  @action
  async handleClick() {
    await this.myPluginActions.showMyModal(this.args.post.topic);
  }

  <template>
    <DButton @action={{this.handleClick}} @icon="user-plus" />
  </template>
}
```

**Service 注册（plugin.rb 不需要）：** Ember 会自动通过文件路径发现 `services/` 目录下的 Service，无需手动注册。

---

### 4. 从 container 获取 Service（在 initializer 中）

```javascript
// initializer 的 initialize(container) 内
initialize(container) {
  withPluginApi((api) => {
    // 通过 container.lookup 获取 service，或通过 getOwner 获取
    const router = container.lookup("service:router");
    const modal = container.lookup("service:modal");

    api.registerValueTransformer("post-menu-buttons", ({ value: dag }) => {
      dag.add("my-button", class extends Component {
        @service myPluginActions;  // 在 gjs 组件内直接用 @service 更简洁

        @action
        open() { this.myPluginActions.showMyModal(this.args.post.topic); }

        <template><DButton @action={{this.open}} @icon="user-plus" /></template>
      });
    });
  });
}
```

---

### 5. Modal 中传递可变数据（TrackedObject）

当 modal model 数据需要在弹窗内被修改并响应式更新时，用 `TrackedObject` 包装：

```javascript
import { TrackedObject } from "@ember-compat/tracked-built-ins";

export default class AssignUser extends Component {
  // 创建可追踪的数据副本，避免直接修改原始 model
  model = new TrackedObject(this.args.model);

  @action
  async onSubmit() {
    this.args.closeModal();
    await this.taskActions.assign(this.model);  // 使用副本
  }
}
```

---

### 6. DButton 常用参数

```javascript
<DButton
  @action={{this.doSomething}}   // 点击回调
  @icon="check"                   // SVG 图标名
  @label="my_plugin.btn_label"   // i18n 键（自动翻译）
  @translatedLabel="直接字符串"   // 直接字符串（不经过 i18n）
  @disabled={{this.isSaving}}    // 禁用状态
  @isLoading={{this.isSaving}}   // 加载动画
  type="submit"                   // HTML type（配合 form 使用）
  class="btn-primary btn-large"  // CSS class
/>
```

---

## 反模式（避免这样做）

```javascript
// 在组件中直接用原生 alert/confirm
if (confirm("确定？")) { ... }
// 应使用 DModal 或 api.dialog（Discourse 内置对话框服务）

// 在多个组件中重复 modal.show(MyModal, { model: ... })
// 应封装进自定义 Service 的方法

// Modal 不传 @closeModal
<DModal @title="..." />  // 用户无法关闭弹窗！
// 必须传：<DModal @title="..." @closeModal={{@closeModal}} />
```

---

## 关联规范

- `plugins/02_plugin_api_frontend.md` — withPluginApi 整体结构
- `plugins/04_value_transformer_dag.md` — 触发 Modal 的按钮注入
- `plugins/08_user_ui_extensions.md` — 用户菜单/偏好扩展

---

## 快速参考

```javascript
// Modal 组件骨架
export default class MyModal extends Component {
  @action async save() {
    await ajax("/...", { type: "POST", data: {...} });
    this.args.closeModal();
  }
  <template>
    <DModal @title="..." @closeModal={{@closeModal}}>
      <:body>{{! 内容 }}</:body>
      <:footer>
        <DButton class="btn-primary" @action={{this.save}} @label="..." />
        <DModalCancel @close={{@closeModal}} />
      </:footer>
    </DModal>
  </template>
}

// 触发 Modal
const modal = container.lookup("service:modal");
modal.show(MyModal, { model: { target, data } });

// Service 中封装触发
showModal(target) {
  return this.modal.show(MyModal, { model: { target } });
}
```
