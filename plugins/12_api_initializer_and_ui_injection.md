# Plugin 12 — apiInitializer、Composer 工具栏与多种 UI 注入点

> 来源：`plugins/discourse-ai/assets/javascripts/discourse/initializers/ai-gist-connector.gjs`、`plugins/discourse-ai/assets/javascripts/discourse/initializers/ai-helper.js`、`plugins/discourse-ai/assets/javascripts/discourse/initializers/ai-bot-replies.js`、`plugins/discourse-ai/assets/javascripts/discourse/initializers/add-ai-bot-to-community-section.gjs`

## 概述

插件有两种 initializer 模式：`withPluginApi`（传统、可拆分多函数）和 `apiInitializer`（简洁、export default 直接传 callback）。两者功能等价，但 `apiInitializer` 推荐用于纯 API 操作、逻辑简单的 initializer。本文还涵盖 Composer 工具栏按钮注入、话题页脚按钮、头部图标、侧边栏社区链接等多种 UI 注入点。

---

## 来源验证

- `plugins/discourse-ai/assets/javascripts/discourse/initializers/ai-gist-connector.gjs`（apiInitializer + addBulkActionButton + addTopicAdminMenuButton + renderInOutlet）
- `plugins/discourse-ai/assets/javascripts/discourse/initializers/ai-helper.js`（onToolbarCreate + addButton）
- `plugins/discourse-ai/assets/javascripts/discourse/initializers/ai-bot-replies.js`（headerIcons.add + registerTopicFooterButton）
- `plugins/discourse-ai/assets/javascripts/discourse/initializers/add-ai-bot-to-community-section.gjs`（addCommunitySectionLink）

---

## 核心规范

### 1. apiInitializer vs withPluginApi

```javascript
// 方式一：withPluginApi（经典，适合有 initialize(container) 逻辑的 initializer）
export default {
  name: "my-plugin-feature",

  initialize(container) {
    const settings = container.lookup("service:site-settings");
    if (!settings.my_plugin_enabled) { return; }

    withPluginApi((api) => {
      api.registerValueTransformer("topic-list-item-class", ...);
    });
  },
};

// 方式二：apiInitializer（现代简洁，推荐用于纯 API 操作）
// container 通过 api.container 访问，无需外层 initialize(container)
import { apiInitializer } from "discourse/lib/api";

export default apiInitializer((api) => {
  const settings = api.container.lookup("service:site-settings");
  if (!settings.my_plugin_enabled) { return; }

  api.registerValueTransformer("topic-list-item-class", ...);
});
```

**选择规则：**
- 需要在 `withPluginApi` 外部先做条件判断（如检查 `currentUser.admin`）→ 用 `withPluginApi`
- 纯粹调用 plugin API、逻辑在 api callback 内就能完成 → 用 `apiInitializer`
- `apiInitializer` 中通过 `api.container.lookup(...)` 获取 service（与 `withPluginApi` 中 `container.lookup` 等价）

---

### 2. Composer 工具栏按钮（onToolbarCreate）

在 Markdown 编辑器工具栏添加自定义按钮：

```javascript
api.onToolbarCreate((toolbar) => {
  const modal = api.container.lookup("service:modal");
  const currentUser = api.getCurrentUser();

  toolbar.addButton({
    id: "my-plugin-trigger",       // 按钮唯一 ID（必须）
    group: "extras",               // 分组：插件按钮放 "extras"，核心按钮有 "fontStyles"/"insert" 等
    icon: "discourse-sparkles",    // SVG 图标名
    title: "my_plugin.toolbar.tooltip",  // i18n 键（工具提示）
    preventFocus: true,            // 点击后不聚焦按钮（防止 textarea 失焦）
    hideShortcutInTitle: true,     // 是否在 title 中隐藏快捷键提示
    shortcut: "ALT+P",             // 快捷键（可选）

    // 快捷键触发的逻辑（与 sendAction 不同）
    shortcutAction: (toolbarEvent) => {
      const text = toolbarEvent.getText();  // 获取编辑器全部内容
      const selected = toolbarEvent.selected.value;  // 获取选中文本

      modal.show(MyModal, {
        model: { selectedText: selected || text },
      });
    },

    // 点击按钮触发的逻辑（优先于 shortcutAction）
    sendAction: async (toolbarEvent) => {
      const menu = api.container.lookup("service:menu");
      menu.show(document.querySelector(".my-trigger"), {
        identifier: "my-composer-menu",
        component: MyComposerMenu,
        modalForMobile: true,
        interactive: true,
        data: { toolbarEvent },
      });
    },

    // 按钮显示条件（同步函数，返回 bool）
    condition: () => {
      const composer = api.container.lookup("service:composer");
      return composer.model?.topic?.archetype !== "private_message";
    },
  });
});
```

**toolbarEvent 常用方法：**

| 方法 | 说明 |
|------|------|
| `toolbarEvent.getText()` | 获取编辑器全部文本内容 |
| `toolbarEvent.selected.value` | 获取当前选中的文本 |
| `toolbarEvent.replaceText(old, new)` | 替换文本 |
| `toolbarEvent.addText(text)` | 在光标位置插入文本 |

---

### 3. 话题脚注按钮（registerTopicFooterButton）

在话题页面底部按钮区域（Reply 按钮旁边）添加自定义按钮：

```javascript
api.registerTopicFooterButton({
  id: "my-feature-button",            // 按钮唯一 ID
  icon: "share-nodes",                // 图标
  label: "my_plugin.footer_btn.name",  // i18n 键（按钮文字）
  title: "my_plugin.footer_btn.title", // i18n 键（tooltip）

  // 点击 action（this 指向 topic controller context）
  action() {
    // this.topic 为当前话题
    showMyModal(modal, this.topic.id);
  },

  classNames: ["my-feature-button"],   // CSS class 数组

  // 响应式依赖：列出影响 displayed/disabled 计算的 key
  dependentKeys: ["topic.my_custom_field"],

  // 显示条件（this = topic controller context）
  displayed() {
    return (
      currentUser?.can_use_feature &&
      this.topic.my_custom_field       // 使用 dependentKeys 中列出的字段
    );
  },

  // 禁用条件（可选）
  disabled() {
    return this.topic.closed;
  },
});
```

---

### 4. 头部图标（headerIcons.add）

在 Discourse 顶部导航栏右侧图标区域（铃铛旁边）添加图标：

```javascript
// 方式一：api.headerIcons.add（推荐，ai-bot-replies.js 中使用）
import AiBotHeaderIcon from "../components/ai-bot-header-icon";

api.headerIcons.add("ai", AiBotHeaderIcon);
// 第一个参数：图标唯一 ID
// 第二个参数：Glimmer 组件类

// AiBotHeaderIcon 组件样例：
// 组件接收无特殊 args，通过 @service 获取数据
export default class AiBotHeaderIcon extends Component {
  @service currentUser;

  <template>
    {{#if this.currentUser.ai_enabled_chat_bots}}
      <li class="header-dropdown-toggle ai-bot-button">
        <DButton @icon="robot" @href="/ai-bot" class="icon btn-flat" />
      </li>
    {{/if}}
  </template>
}
```

---

### 5. 侧边栏社区区域链接（addCommunitySectionLink）

在 Discourse 侧边栏"社区"板块添加自定义导航链接：

```javascript
api.addCommunitySectionLink((baseSectionLink) => {
  return class extends baseSectionLink {
    name = "my-feature";                        // 唯一名称（用于 CSS class）
    route = "my-plugin-route";                  // Ember 路由名
    text = i18n("my_plugin.sidebar.link_text"); // 显示文字
    title = i18n("my_plugin.sidebar.link_title"); // tooltip
    defaultPrefixValue = "star";                // 默认图标名（SVG）
  };
});
```

---

### 6. renderInOutlet（话题列表 Outlet）

`renderInOutlet` 用于注入到话题列表的特定 outlet，与 `renderAfterWrapperOutlet`（帖子内容区）不同：

```javascript
import { apiInitializer } from "discourse/lib/api";

export default apiInitializer((api) => {
  // shouldRender 接收三个参数：args、context、owner
  // owner 用于 lookup service（比 context 更通用）
  api.renderInOutlet(
    "topic-list-topic-cell-link-bottom-line__before",  // outlet 名
    class extends Component {
      static shouldRender(args, context, owner) {
        // owner.lookup 获取任意服务
        const router = owner.lookup("service:router");
        return router.currentRouteName?.startsWith("discovery.");
      }

      // @topic 是 topic-list 系列 outlet 传入的 arg
      <template><MyGistComponent @topic={{@topic}} /></template>
    }
  );
});
```

**`renderInOutlet` vs `renderAfterWrapperOutlet`：**

| 特性 | renderInOutlet | renderAfterWrapperOutlet |
|------|---------------|--------------------------|
| 使用场景 | 话题列表 / 通用 outlet | 帖子内容区域 |
| 固定插槽名 | 多种 outlet 名 | `"post-content-cooked-html"` |
| arg 来源 | 各 outlet 不同（@topic 等） | `@post`、`@decoratorState` |
| shouldRender 参数 | `(args, context, owner)` | `(args)` |

**常用 renderInOutlet outlet 名：**

| outlet 名 | 位置 |
|-----------|------|
| `topic-list-topic-cell-link-bottom-line__before` | 话题标题下方（桌面） |
| `topic-list-main-link-bottom` | 话题标题下方（移动） |
| `topic-map-expanded-after` | 话题 Map 展开后 |
| `topic-map-participants-after` | 话题 Map 参与者后 |

---

### 7. 话题批量操作按钮（addBulkActionButton）

在话题列表批量选择后的操作菜单中添加自定义按钮：

```javascript
api.addBulkActionButton({
  label: "my_plugin.bulk_action.label",   // i18n 键
  icon: "arrows-rotate",                  // 图标
  class: "btn-default",                   // CSS class

  // 显示条件
  visible: ({ topics, siteSettings, currentUser }) => {
    if (topics?.length > 30) { return false; }
    return siteSettings.my_plugin_enabled && currentUser.staff;
  },

  // 执行逻辑
  async action(opts) {
    const topics = opts.model.bulkSelectHelper.selected;
    const topicIds = topics.map((t) => t.id);

    try {
      await ajax("/my_plugin/bulk_action", {
        type: "PUT",
        data: { topic_ids: topicIds },
      });
      // 刷新列表并关闭 modal
      opts.model.refreshClosure?.().then(() => {
        opts.args.closeModal();
        opts.model.bulkSelectHelper.toggleBulkSelect();
      });
    } catch {
      // 错误处理
    }
  },

  actionType: "performAndRefresh",  // 固定值（需要刷新列表时）
});
```

---

### 8. 话题管理菜单按钮（addTopicAdminMenuButton）

在话题管理员菜单中添加自定义操作项：

```javascript
api.addTopicAdminMenuButton((topic) => {
  // 返回 null/undefined 表示不显示
  if (!SiteSetting.my_feature_enabled) { return; }

  return {
    action: async () => {
      await ajax("/my_plugin/topic_action", {
        type: "PUT",
        data: { topic_id: topic.id },
      });
      window.location.reload();  // 或使用更优雅的刷新方式
    },
    icon: "arrows-rotate",
    className: "my-action-button",
    label: "my_plugin.topic_admin.action_label",  // i18n 键
  };
});
```

---

### 9. Admin 报表自定义渲染组件（registerReportModeComponent）

覆盖特定报表的渲染方式（用自定义 Glimmer 组件代替默认图表）：

```javascript
import { optionalRequire } from "discourse/lib/utilities";

// optionalRequire：可选加载，模块不存在时返回 null 而不报错
// 适合跨插件依赖、或环境特定组件
const MyCustomChart = optionalRequire(
  "discourse/plugins/my-plugin/discourse/components/my-custom-chart"
);

api.registerReportModeComponent("my_report_type", MyComponent);
// 第一个参数：report 的 mode 名称（对应后端 add_report 中 report.modes 的值）
// 第二个参数：用于渲染的 Glimmer 组件
```

---

## 反模式（避免这样做）

```javascript
// apiInitializer 中忘记使用 api.container，直接用外部变量
import SiteSetting from "discourse/models/site-setting";  // 错误！不可能这样 import
export default apiInitializer((api) => {
  // 正确：
  const settings = api.container.lookup("service:site-settings");
});

// onToolbarCreate 的 condition 做异步操作
condition: async () => await checkPermission(),  // 错误！condition 是同步的
// 正确：在 initialize 时预先计算并缓存结果
const canUse = currentUser?.some_sync_field;
condition: () => canUse,

// registerTopicFooterButton 的 dependentKeys 缺少响应式字段
dependentKeys: [],  // displayed() 读了 this.topic.my_field 但没在 dependentKeys 里
// 正确：列出所有 displayed/disabled 中使用的字段
dependentKeys: ["topic.my_field"],
```

---

## 关联规范

- `plugins/04_value_transformer_dag.md` — post-menu-buttons / topic-list-item-class
- `plugins/05_modify_class.md` — modifyClass controller:topic 扩展 subscribe/unsubscribe
- `plugins/06_tracked_post_and_render_outlet.md` — renderAfterWrapperOutlet（帖子区域注入）

---

## 快速参考

```javascript
// apiInitializer（简洁模式）
import { apiInitializer } from "discourse/lib/api";
export default apiInitializer((api) => {
  const settings = api.container.lookup("service:site-settings");
  // ...
});

// Composer 工具栏按钮
api.onToolbarCreate((toolbar) => {
  toolbar.addButton({
    id: "my-btn", group: "extras", icon: "star",
    title: "my_plugin.toolbar.title",
    sendAction: (toolbarEvent) => { /* ... */ },
    condition: () => true,
  });
});

// 话题脚注按钮
api.registerTopicFooterButton({
  id: "my-btn", icon: "star",
  label: "my_plugin.footer.label",
  action() { /* this.topic */ },
  dependentKeys: ["topic.my_field"],
  displayed() { return !!this.topic.my_field; },
});

// 头部图标
api.headerIcons.add("my-icon", MyHeaderIconComponent);

// 侧边栏链接
api.addCommunitySectionLink((base) =>
  class extends base {
    name = "my-link"; route = "my-route";
    text = i18n("..."); title = i18n("...");
    defaultPrefixValue = "star";
  }
);

// 话题列表 outlet 注入
api.renderInOutlet("topic-list-main-link-bottom",
  class extends Component {
    static shouldRender(args, context, owner) { return true; }
    <template><MyComponent @topic={{@topic}} /></template>
  }
);

// 批量操作按钮
api.addBulkActionButton({
  label: "...", icon: "star", class: "btn-default",
  visible: ({ siteSettings, currentUser }) => currentUser.staff,
  async action(opts) { /* opts.model.bulkSelectHelper.selected */ },
  actionType: "performAndRefresh",
});
```
