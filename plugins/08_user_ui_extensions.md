# Plugin 08 — 用户菜单、用户偏好、用户活动页扩展

> 来源：`plugins/discourse-assign/assets/javascripts/discourse/initializers/assign-user-menu.js`、`plugins/discourse-assign/assets/javascripts/discourse/connectors/user-preferences-notifications/remind-assigns-frequency.gjs`、`plugins/discourse-assign/assets/javascripts/discourse/connectors/user-activity-bottom/assigned-list.gjs`

## 概述

Discourse 插件可在三个用户相关区域扩展 UI：用户菜单（铃铛下拉面板新增 Tab）、用户偏好设置页（Notification/Tracking 等页面注入自定义控件）、用户活动页（用户主页 Activity 列表注入导航项）。同时可自定义通知类型的渲染方式（图标、标题、描述等）。

---

## 来源验证

- `plugins/discourse-assign/assets/javascripts/discourse/initializers/assign-user-menu.js`（registerUserMenuTab + registerNotificationTypeRenderer）
- `plugins/discourse-assign/assets/javascripts/discourse/connectors/user-preferences-notifications/remind-assigns-frequency.gjs`（用户偏好注入）
- `plugins/discourse-assign/assets/javascripts/discourse/connectors/user-preferences-tracking-topics/notification-level-when-assigned.gjs`（偏好 Tracking 页注入 ComboBox）
- `plugins/discourse-assign/assets/javascripts/discourse/connectors/user-activity-bottom/assigned-list.gjs`（用户活动页导航项）

---

## 核心规范

### 1. 用户菜单新增 Tab（registerUserMenuTab）

在铃铛下拉菜单中增加自定义 Tab（如"我的任务"）：

```javascript
// initializers/my-plugin-user-menu.js
import { withPluginApi } from "discourse/lib/plugin-api";
import MyNotificationsList from "../components/user-menu/my-notifications-list";

export default {
  name: "my-plugin-user-menu",

  initialize(container) {
    withPluginApi((api) => {
      const siteSettings = container.lookup("service:site-settings");
      if (!siteSettings.my_plugin_enabled) { return; }

      const currentUser = api.getCurrentUser();
      if (!currentUser?.can_use_feature) { return; }

      if (api.registerUserMenuTab) {
        api.registerUserMenuTab((UserMenuTab) => {
          return class extends UserMenuTab {
            id = "my-plugin-list";              // Tab 唯一 ID
            panelComponent = MyNotificationsList; // 面板组件
            icon = "user-plus";                  // Tab 图标
            notificationTypes = ["my_notification_type"]; // 关联通知类型

            get count() {
              // 显示在 Tab 上的未读数量
              return this.getUnreadCountForType("my_notification_type");
            }

            get linkWhenActive() {
              // Tab 激活时对应的页面路由
              return `${this.currentUser.path}/activity/my-feature`;
            }
          };
        });
      }
    });
  },
};
```

---

### 2. 自定义通知类型渲染（registerNotificationTypeRenderer）

控制特定通知类型在用户菜单中的外观（图标、标题、描述、链接等）：

```javascript
api.registerNotificationTypeRenderer(
  "assigned",                          // 通知类型名（对应后端 Notification.types[:assigned]）
  (NotificationItemBase) => {
    return class extends NotificationItemBase {
      // 通知链接的 tooltip
      get linkTitle() {
        if (this.notification.data.message === "group_assign") {
          return i18n("user.assigned_to_group.topic", {
            group_name: this.notification.data.display_username,
          });
        }
        return i18n("user.assigned_to_you.topic");
      }

      // 通知图标
      get icon() {
        return "user-plus";
      }

      // 通知标签（显示在用户名旁边，空字符串则不显示）
      get label() {
        if (this.notification.data.is_group) {
          return this.notification.data.display_username;
        }
        return "";
      }

      // 通知描述文字
      get description() {
        return htmlSafe(
          emojiUnescape(
            i18n("user.assignment_description.topic", {
              topic_title: this.notification.fancy_title,
            })
          )
        );
      }
    };
  }
);
```

`NotificationItemBase` 上已有的属性：
- `this.notification` — 通知对象（含 `data`、`topic_id`、`post_number`、`fancy_title` 等）
- `this.currentUser` — 当前用户
- `this.siteSettings` — 站点设置
- `super.linkHref` — 父类默认链接（可 fallback）

---

### 3. 用户偏好设置页注入（Connector）

Connector 目录名对应偏好设置页的插槽名。文件放对位置就自动注入，无需手动注册：

```javascript
// connectors/user-preferences-notifications/my-setting.gjs
// 注入到：用户偏好 → 通知（Notifications）页面

import Component from "@ember/component";
import { classNames, tagName } from "@ember-decorators/component";
import { i18n } from "discourse-i18n";

@tagName("div")
@classNames("user-preferences-notifications-outlet", "my-setting")
export default class MySettingConnector extends Component {
  // 条件渲染：static shouldRender 接收 args 和 context
  static shouldRender(args, context) {
    return context.currentUser?.can_use_feature;
  }

  <template>
    <div class="control-group">
      <label class="control-label">{{i18n "my_plugin.setting_label"}}</label>
      <div class="controls">
        {{! 通过 @outletArgs.model 访问用户数据 }}
        <input
          type="checkbox"
          checked={{@outletArgs.model.user_option.my_setting}}
        />
      </div>
    </div>
  </template>
}
```

**用户偏好页常用插槽：**

| 插槽名（目录名） | 位置 |
|----------------|------|
| `user-preferences-notifications` | 偏好 → 通知页 |
| `user-preferences-tracking-topics` | 偏好 → 追踪主题页 |
| `user-preferences-account` | 偏好 → 账户页 |
| `user-preferences-profile` | 偏好 → 个人信息页 |
| `user-preferences-emails` | 偏好 → 邮件页 |

**ComboBox（下拉选择）注入示例（assign 插件）：**

```javascript
// connectors/user-preferences-tracking-topics/notification-level-when-assigned.gjs
import Component from "@glimmer/component";
import { fn } from "@ember/helper";
import { service } from "@ember/service";
import ComboBox from "discourse/select-kit/components/combo-box";
import { i18n } from "discourse-i18n";

export default class NotificationLevelWhenAssigned extends Component {
  @service siteSettings;

  get options() {
    return [
      { name: i18n("user.my_plugin.watch"), value: "watch" },
      { name: i18n("user.my_plugin.track"), value: "track" },
      { name: i18n("user.my_plugin.mute"), value: "mute" },
    ];
  }

  <template>
    {{#if this.siteSettings.my_plugin_enabled}}
      <div class="controls controls-dropdown">
        <label>{{i18n "user.my_plugin.label"}}</label>
        <ComboBox
          @content={{this.options}}
          @value={{@outletArgs.model.user_option.my_setting}}
          @valueProperty="value"
          @onChange={{fn (mut @outletArgs.model.user_option.my_setting)}}
        />
      </div>
    {{/if}}
  </template>
}
```

---

### 4. 用户活动页（Activity）注入导航项

在用户主页 Activity 列表下方注入链接，引导到插件自定义的活动列表：

```javascript
// connectors/user-activity-bottom/my-feature-list.gjs
/* eslint-disable ember/no-classic-components */
import Component from "@ember/component";
import { LinkTo } from "@ember/routing";
import { classNames, tagName } from "@ember-decorators/component";
import icon from "discourse/helpers/d-icon";
import { i18n } from "discourse-i18n";

@tagName("li")
@classNames("user-activity-bottom-outlet", "my-feature-list")
export default class MyFeatureList extends Component {
  <template>
    {{#if this.currentUser.can_use_feature}}
      <LinkTo @route="userActivity.myFeature">
        {{icon "user-plus"}}
        {{i18n "my_plugin.activity_label"}}
      </LinkTo>
    {{/if}}
  </template>
}
```

对应后端需要注册路由（route map）：

```javascript
// discourse/my-feature-route-map.js
export default {
  resource: "user.userActivity",
  path: "/activity",
  map() {
    this.route("myFeature", { path: "/my-feature" });
  },
};
```

---

### 5. 后端配合：可编辑用户自定义字段

如果偏好设置页的新字段需要存储，后端 plugin.rb 中需声明：

```ruby
# 声明字段可被用户编辑
frequency_field = "my_plugin_frequency"
register_editable_user_custom_field frequency_field
register_user_custom_field_type(frequency_field, :integer, max_length: 10)

# 将字段序列化给前端
DiscoursePluginRegistry.serialized_current_user_fields << frequency_field

# 或用 UserOption 扩展（更推荐，字段在 user_options 表而不是 custom_fields）
UserUpdater::OPTION_ATTR.push(:my_plugin_setting)
add_to_serializer(:user_option, :my_plugin_setting) { object.my_plugin_setting }
```

---

## 反模式（避免这样做）

```javascript
// Connector 文件名和目录不一致导致注入失败
// 目录：connectors/user-preferences-notifications/
// 文件：my-setting.js（内容是 Glimmer 组件但用了旧的 .js）
// 应用 .gjs 并正确使用 <template> 语法

// 在 Tab 的 count getter 里做 AJAX 请求（每次渲染都触发）
get count() {
  return ajax("/api/count").then(r => r.count);  // 错误！getter 是同步的
}
// 正确：只用 getUnreadCountForType 统计本地已有的通知数
get count() {
  return this.getUnreadCountForType("my_notification_type");
}
```

---

## 关联规范

- `plugins/02_plugin_api_frontend.md` — withPluginApi 和 Connector 基础
- `plugins/07_modal_and_ui_components.md` — 弹窗 UI 组件
- `ruby/09_notification.md` — 后端通知注册

---

## 快速参考

```javascript
// 用户菜单 Tab
api.registerUserMenuTab((UserMenuTab) =>
  class extends UserMenuTab {
    id = "my-tab";
    panelComponent = MyPanel;
    icon = "star";
    notificationTypes = ["my_type"];
    get count() { return this.getUnreadCountForType("my_type"); }
    get linkWhenActive() { return `${this.currentUser.path}/activity/my-feature`; }
  }
);

// 自定义通知渲染
api.registerNotificationTypeRenderer("my_type", (Base) =>
  class extends Base {
    get icon() { return "star"; }
    get label() { return this.notification.data.username; }
    get description() { return i18n("my_plugin.notification_desc"); }
  }
);

// 偏好页 Connector（放到正确目录即自动注入）
// connectors/user-preferences-notifications/my-setting.gjs
static shouldRender(args, context) { return context.currentUser?.can_feature; }

// 活动页导航 Connector
// connectors/user-activity-bottom/my-list.gjs
// @tagName="li" + <LinkTo @route="userActivity.myFeature">
```
