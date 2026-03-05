# plugins/29 — Plugin API 杂项方法速查

> 来源：
> - `plugins/discourse-assign/assets/javascripts/discourse/initializers/extend-for-assigns.js`（全文）
> 修订历史：
> - 2026-03-05 初版

## 边界声明

- **本文件负责**：`withPluginApi` 中不属于其他专题文件的 api 方法，按使用场景分组整理
- **不负责，见其他文件**：
  - `modifyClass` → `plugins/05_modify_class.md`
  - `modifySelectKit` → `javascript/07_select_kit.md`
  - `addComposerToolbarPopupMenuOption` → `plugins/12_api_initializer_and_ui_injection.md`
  - `addTopicListItem` / `registerValueTransformer` → `plugins/17_topic_list_columns.md`
  - `addUserMenuItems` → `plugins/08_user_ui_extensions.md`
  - `addAdvancedSearchOptions` → `plugins/27_search_filter.md`
  - `renderAfterWrapperOutlet` → `plugins/06_tracked_post_and_render_outlet.md`

---

## 来源验证

| 方法 | 行号 | 用途一句话 |
|------|------|-----------|
| `registerTopicFooterButton` | 46 | 话题底部操作栏加按钮（含动态 icon/label/displayed） |
| `addNavigationBarItem` | 276 | 话题列表左侧导航栏加项目 |
| `addDiscoveryQueryParam` | 355 | 将 URL 参数绑定到话题列表路由（刷新 model） |
| `addTagsHtmlCallback` | 357 | 在话题标签区域注入自定义 HTML |
| `replaceIcon` | 523 | 替换系统图标（通知图标等） |
| `addKeyboardShortcut` | 532 | 注册全局键盘快捷键 |
| `addPostSmallActionIcon` | 580 | 为 small action post 的 action_code 注册图标 |
| `addPostSmallActionClassesCallback` | 572 | 为 small action post 动态追加 CSS class |
| `addGroupPostSmallActionCode` | 690 | 声明某 action_code 是群组操作（影响渲染） |
| `addSearchSuggestion` | 687 | 在搜索框下拉补全列表中追加建议词 |
| `addBulkActionButton` | 702 | 话题批量操作菜单加按钮 |
| `addSaveableUserOption` | 698 | 将用户选项字段标记为可保存 |
| `addUserSearchOption` | 696 | 向用户搜索 API 追加自定义参数 |

---

## 1. registerTopicFooterButton — 话题底部操作按钮

话题页底部的操作栏（书签、分享旁边），插件可以追加按钮：

```javascript
// 来自 extend-for-assigns.js:46 真实代码（简化）
api.registerTopicFooterButton({
  id: "assign",                      // 唯一 ID，不能与其他插件冲突

  // 动态属性：this 绑定到 { topic, currentUser, site, siteSettings }
  icon() {
    return this.topic.isAssigned() ? null : "user-plus";
  },
  priority: 250,                     // 数字越大越靠左，默认 0

  translatedTitle() {                // tooltip 文字
    return i18n("my_plugin.assign.title");
  },
  translatedLabel() {                // 按钮文字（移动端显示）
    return i18n("my_plugin.assign.label");
  },
  translatedAriaLabel() {            // 无障碍标签
    return i18n("my_plugin.assign.title");
  },

  async action() {                   // 点击回调
    const modal = getOwner(this).lookup("service:modal");
    await modal.show(MyModal, { model: { topic: this.topic } });
  },

  dropdown() {                       // 返回 true 时折叠到下拉菜单
    return this.site.mobileView;
  },

  classNames: ["my-plugin-btn"],     // 追加 CSS class

  displayed() {                      // 控制是否显示
    return !!this.currentUser?.can_do_thing;
  },

  dependentKeys: ["topic.my_plugin_field"],  // 这些属性变化时重新计算动态属性
});
```

**注意**：`icon`、`translatedTitle`、`displayed` 等都可以是函数（动态）或静态值。`this` 的上下文是框架注入的对象，不是 Component。

### 2. addNavigationBarItem — 话题列表左侧导航

```javascript
// 来自 extend-for-assigns.js:276 真实代码
api.addNavigationBarItem({
  name: "unassigned",               // URL 路径中的名称：/latest?order=unassigned

  // 自定义过滤条件：只在特定分类页显示
  customFilter: (category) => {
    return category?.custom_fields?.enable_unassigned_filter === "true";
  },

  // 自定义链接
  customHref: (category) => {
    if (category) {
      return getURL(category.url) + "/l/latest?status=open&assigned=nobody";
    }
  },

  // 自定义高亮状态
  forceActive: (category, args) => {
    const queryParams = args.currentRouteQueryParams;
    return queryParams?.assigned === "nobody" && queryParams?.status === "open";
  },

  before: "top",                    // 插入位置：在 "top" 项之前
});
```

### 3. addDiscoveryQueryParam — URL 参数绑定路由

让自定义 URL 参数参与路由刷新（否则参数变化不会重新请求话题列表）：

```javascript
// 来自 extend-for-assigns.js:355 真实代码
api.addDiscoveryQueryParam("assigned", {
  replace: true,       // 用 replaceState 而非 pushState（不增加历史记录）
  refreshModel: true,  // 参数变化时重新加载 model（触发 API 请求）
});
```

### 4. addKeyboardShortcut — 全局键盘快捷键

```javascript
// 来自 extend-for-assigns.js:532 真实代码
api.addKeyboardShortcut(
  "g a",                           // 快捷键（Discourse 风格：连续按 g 再按 a）
  "",                              // 描述（空字符串 = 不显示在快捷键帮助列表）
  { path: "/my/activity/assigned" } // 导航到指定路径
);

// 也可以用 action 而不是 path
api.addKeyboardShortcut("g a", i18n("my_plugin.shortcut_desc"), {
  action: () => {
    // 自定义操作
  },
});
```

### 5. addPostSmallActionIcon + addPostSmallActionClassesCallback

Small action post 是话题时间线中的系统事件（如「已关闭」「已分配」），这两个方法控制其图标和样式：

```javascript
// 来自 extend-for-assigns.js:580 真实代码
// 为 action_code 注册图标（后端 PostCustomField action_code 对应）
api.addPostSmallActionIcon("assigned", "user-plus");
api.addPostSmallActionIcon("unassigned", "user-xmark");
api.addPostSmallActionIcon("reassigned", "user-plus");

// 动态追加 CSS class（来自 extend-for-assigns.js:572 真实代码）
api.addPostSmallActionClassesCallback((post) => {
  // post 是 Post model 实例
  if (post.action_code.includes("assigned") && !siteSettings.assigns_public) {
    return ["private-assign"];   // 返回 class 数组
  }
  // 返回 undefined 或空数组 = 不追加
});
```

### 6. addGroupPostSmallActionCode — 声明群组 action_code

```javascript
// 来自 extend-for-assigns.js:690 真实代码
// 声明这些 action_code 是群组操作，影响 UI 渲染（显示群组名而非用户名）
api.addGroupPostSmallActionCode("assigned_group");
api.addGroupPostSmallActionCode("reassigned_group");
api.addGroupPostSmallActionCode("unassigned_group");
```

### 7. addSearchSuggestion — 搜索框补全建议

在搜索框输入时的下拉补全列表中追加建议词：

```javascript
// 来自 extend-for-assigns.js:687 真实代码
api.addSearchSuggestion("in:assigned");    // 补全词（完整字符串）
api.addSearchSuggestion("in:unassigned");

// 通常只在有权限时注册
if (currentUser?.can_assign) {
  api.addSearchSuggestion("in:assigned");
}
```

### 8. addBulkActionButton — 批量操作菜单

话题列表勾选多个话题后出现的批量操作菜单：

```javascript
// 来自 extend-for-assigns.js:702 真实代码
// 模式 A：展示自定义组件（复杂交互）
api.addBulkActionButton({
  id: "assign-topics",
  label: "topics.bulk.assign",     // i18n key
  icon: "user-plus",
  class: "btn-default assign-topics",
  actionType: "setComponent",
  action({ setComponent }) {
    setComponent(BulkActionsAssignUser);  // 点击后渲染此 Component
  },
});

// 模式 B：直接执行操作（简单操作）
api.addBulkActionButton({
  id: "unassign-topics",
  label: "topics.bulk.unassign",
  icon: "user-xmark",
  class: "btn-default unassign-topics",
  actionType: "performAndRefresh",
  action({ performAndRefresh }) {
    performAndRefresh({ type: "unassign" });  // 发请求后刷新列表
  },
});
```

**actionType 两种模式：**

| actionType | action 回调参数 | 适用场景 |
|-----------|----------------|---------|
| `"setComponent"` | `{ setComponent }` | 需要显示 UI 让用户进一步输入 |
| `"performAndRefresh"` | `{ performAndRefresh }` | 直接执行后端操作，无需额外交互 |

### 9. replaceIcon — 替换系统图标

```javascript
// 来自 extend-for-assigns.js:523 真实代码
// 将通知类型图标替换为自定义图标
api.replaceIcon("notification.assigned", "user-plus");
api.replaceIcon(
  "notification.discourse_assign.assign_group_notification",
  "group-plus"
);

// 通用格式：replaceIcon(原始图标名, 替换图标名)
// 原始图标名可以是 icon ID 或 "notification.类型名"
```

### 10. addTagsHtmlCallback — 话题标签区域注入 HTML

在话题列表每行的标签区域（tags 旁边）追加自定义 HTML：

```javascript
// 来自 extend-for-assigns.js:357 真实代码（简化）
api.addTagsHtmlCallback((topic, params = {}) => {
  const assignedTo = topic.get("assigned_to_user");
  if (!assignedTo) {
    return "";   // 返回空字符串 = 不注入
  }

  // 返回 HTML 字符串，会被插入到 tags 后面
  return `<a class="assigned-to" href="/u/${assignedTo.username}">
    ${renderAvatar(assignedTo, { imageSize: "small" })}
  </a>`;
});
```

### 11. addSaveableUserOption + addUserSearchOption

```javascript
// 来自 extend-for-assigns.js:698 真实代码
// 将用户选项字段标记为可通过 PUT /u/:username 保存
api.addSaveableUserOption("notification_level_when_assigned", {
  page: "tracking",   // 出现在哪个设置页（tracking = 通知设置页）
});

// 来自 extend-for-assigns.js:696 真实代码
// 向 /users/search 请求追加自定义参数
api.addUserSearchOption("assignableGroups");
// 后端需对应处理这个参数（在 UsersController search action 中）
```

---

## 快速参考索引

| 需求 | 方法 |
|------|------|
| 话题页底部加按钮 | `registerTopicFooterButton` |
| 话题列表左侧导航加项 | `addNavigationBarItem` |
| URL 参数绑定路由刷新 | `addDiscoveryQueryParam` |
| 搜索框补全建议 | `addSearchSuggestion` |
| 话题标签区域追加 HTML | `addTagsHtmlCallback` |
| 系统小操作事件的图标 | `addPostSmallActionIcon` |
| 系统小操作事件的 CSS class | `addPostSmallActionClassesCallback` |
| 声明 action_code 是群组操作 | `addGroupPostSmallActionCode` |
| 键盘快捷键 | `addKeyboardShortcut` |
| 批量操作菜单按钮 | `addBulkActionButton` |
| 替换通知/系统图标 | `replaceIcon` |
| 用户选项可保存 | `addSaveableUserOption` |
| 用户搜索附加参数 | `addUserSearchOption` |
