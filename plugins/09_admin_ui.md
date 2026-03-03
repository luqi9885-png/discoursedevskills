# Plugin 09 — 管理后台 UI 扩展

> 来源：`plugins/discourse-assign/assets/javascripts/discourse/initializers/assign-admin-plugin-configuration-nav.js`、`plugins/discourse-chat-integration/admin/assets/javascripts/admin/templates/admin-plugins/chat-integration.gjs`、`plugins/discourse-reactions/plugin.rb`

## 概述

Discourse 插件可以在三个管理后台区域扩展 UI：插件配置页（Admin → Plugins 下的专属页面）、管理后台 Connector（在现有 Admin 页面注入内容）、以及后端 `add_report` 注册数据报表（在 Admin → Reports 展示图表）。

---

## 来源验证

- `plugins/discourse-assign/assets/javascripts/discourse/initializers/assign-admin-plugin-configuration-nav.js`（setAdminPluginIcon）
- `plugins/discourse-reactions/assets/javascripts/discourse/initializers/reactions-admin-plugin-configuration-nav.js`（setAdminPluginIcon）
- `plugins/discourse-chat-integration/admin/assets/javascripts/admin/templates/admin-plugins/chat-integration.gjs`（Admin 插件页模板）
- `plugins/discourse-reactions/plugin.rb`（add_report 完整示例）
- `plugins/discourse-user-notes/assets/javascripts/discourse/connectors/admin-user-controls-after/add-user-notes-button.gjs`（Admin 用户页注入）

---

## 核心规范

### 1. 插件配置页图标（setAdminPluginIcon）

在 Admin → Plugins 列表中为插件设置图标：

```javascript
// initializers/my-plugin-admin-nav.js
import { withPluginApi } from "discourse/lib/plugin-api";

const PLUGIN_ID = "my-plugin-name";  // 与 plugin.rb 中 name 字段一致

export default {
  name: "my-plugin-admin-plugin-configuration-nav",

  initialize(container) {
    const currentUser = container.lookup("service:current-user");
    if (!currentUser?.admin) {
      return;  // 非 admin 直接跳过，不加载
    }

    withPluginApi((api) => {
      api.setAdminPluginIcon(PLUGIN_ID, "user-plus");  // 第二个参数为 SVG 图标名
    });
  },
};
```

---

### 2. 插件配置页模板（Admin Plugins 专属页面）

在 `admin/assets/javascripts/admin/templates/admin-plugins/` 目录下建立模板文件，Discourse 自动将其挂载到 `/admin/plugins/your-plugin-name`：

```javascript
// admin/assets/javascripts/admin/templates/admin-plugins/my-plugin.gjs
import DButton from "discourse/components/d-button";
import NavItem from "discourse/components/nav-item";
import { i18n } from "discourse-i18n";

export default <template>
  <div id="admin-plugin-my-plugin">
    {{! 顶部控制栏 }}
    <div class="admin-controls">
      <ul class="nav nav-pills">
        <NavItem
          @route="adminPlugins.my-plugin.settings"
          @label="my_plugin.admin.tabs.settings"
        />
        <NavItem
          @route="adminPlugins.my-plugin.logs"
          @label="my_plugin.admin.tabs.logs"
        />
      </ul>

      <DButton
        @icon="gear"
        @label="my_plugin.admin.configure"
        @action={{@controller.openSettings}}
        class="btn-default"
      />
    </div>

    {{outlet}}
  </div>
</template>
```

**目录结构：**

```
admin/
└── assets/
    └── javascripts/
        └── admin/
            ├── templates/
            │   └── admin-plugins/
            │       └── my-plugin.gjs     ← 主页面（路由：adminPlugins.my-plugin）
            └── components/
                └── modal/
                    └── my-admin-modal.gjs
```

---

### 3. Admin 页面 Connector 注入

在现有 Admin 页面插入自定义内容，与普通 Connector 一样，目录名即插槽名：

```javascript
// connectors/admin-user-controls-after/my-plugin-button.gjs
// 注入到：Admin → Users → 某用户页面的控制按钮区域后

import Component from "@glimmer/component";
import { action } from "@ember/object";
import { service } from "@ember/service";
import DButton from "discourse/components/d-button";

export default class MyPluginAdminButton extends Component {
  @service modal;

  static shouldRender(args, context) {
    // args.model — 当前被查看的用户
    return context.currentUser?.admin;
  }

  @action
  openModal() {
    this.modal.show(MyAdminModal, {
      model: { user: this.args.outletArgs.model },
    });
  }

  <template>
    <DButton
      @action={{this.openModal}}
      @icon="note-sticky"
      @label="my_plugin.add_note"
      class="btn-default"
    />
  </template>
}
```

**常用 Admin Connector 插槽：**

| 插槽名 | 位置 |
|--------|------|
| `admin-user-controls-after` | Admin 用户详情页控制按钮区 |
| `admin-dashboard-moderation-bottom` | Admin 管理面板底部 |
| `above-review-filters` | 审核列表过滤器上方 |
| `admin-nav-additional-root-item` | Admin 侧边栏额外导航项 |

---

### 4. 管理报表（add_report）

在后端 plugin.rb 中注册，报表会自动出现在 Admin → Reports 页面：

```ruby
add_report("my_plugin_events") do |report|
  report.icon = "star"        # 报表图标
  report.modes = [:table]     # 展示模式：:table（表格）、:chart（图表）、两者皆可

  # 定义表格列
  report.labels = [
    { type: :date,   property: :day,        title: I18n.t("reports.my_plugin_events.labels.day") },
    { type: :number, property: :event_count, title: I18n.t("reports.my_plugin_events.labels.count") },
  ]

  # 查询数据（report.start_date / report.end_date 由 Discourse 自动设置）
  results = DB.query(<<~SQL, start_date: report.start_date.to_date, end_date: report.end_date.to_date)
    SELECT
      date_trunc('day', created_at)::date AS day,
      count(*) AS event_count
    FROM my_plugin_events
    WHERE created_at::DATE >= :start_date::DATE
      AND created_at::DATE <= :end_date::DATE
    GROUP BY day
    ORDER BY day
  SQL

  report.data = results.map { |r| { day: r.day, event_count: r.event_count } }
end
```

**report 常用属性：**

| 属性 | 说明 |
|------|------|
| `report.icon` | SVG 图标名 |
| `report.modes` | `[:table]`、`[:chart]`、`[:table, :chart]` |
| `report.labels` | 列定义数组，每列含 `type`、`property`、`title` 或 `html_title` |
| `report.data` | 数据数组（每项是 Hash，key 对应 `property`）|
| `report.start_date` | 报表开始日期（框架自动设置）|
| `report.end_date` | 报表结束日期（框架自动设置）|

**labels 的 type 可选值：** `:date`、`:number`、`:string`、`:link`、`:user`、`:topic`、`:post`。

**reactions 插件报表示例（动态列）：**

```ruby
# 根据 SiteSettings 动态添加每种 emoji 的统计列
add_report("reactions") do |report|
  main_id = DiscourseReactions::Reaction.main_reaction_id
  report.modes = [:table]

  # 固定列
  report.labels = [
    { type: :date, property: :day, title: I18n.t("reports.reactions.labels.day") },
    {
      type: :number,
      property: :like_count,
      html_title: PrettyText.unescape_emoji(CGI.escapeHTML(":#{main_id}:")),
    },
  ]

  # 根据启用的 reactions 动态添加列
  reactions = SiteSetting.discourse_reactions_enabled_reactions.split("|") - [main_id]
  reactions.each do |reaction|
    report.labels << {
      type: :number,
      property: "#{reaction}_count",
      html_title: PrettyText.unescape_emoji(CGI.escapeHTML(":#{reaction}:")),
    }
  end

  # ... 填充 report.data
end
```

---

### 5. Admin 侧边栏导航扩展

通过 Connector 在 Admin 侧边栏添加自定义导航项：

```javascript
// connectors/admin-nav-additional-root-item/my-plugin-nav.gjs
import NavItem from "discourse/components/nav-item";

export default <template>
  <NavItem
    @route="adminPlugins.my-plugin"
    @label="my_plugin.admin.nav_label"
  />
</template>
```

---

## 反模式（避免这样做）

```javascript
// 非 admin 用户加载 Admin 相关资源
initialize(container) {
  withPluginApi((api) => {
    api.setAdminPluginIcon(PLUGIN_ID, "gear");  // 错误！非 admin 也执行
  });
}

// 正确：先检查权限
initialize(container) {
  const currentUser = container.lookup("service:current-user");
  if (!currentUser?.admin) { return; }  // 非 admin 直接返回
  withPluginApi((api) => { api.setAdminPluginIcon(PLUGIN_ID, "gear"); });
}
```

```ruby
# report 中直接用 ActiveRecord 查询（性能差）
report.data = MyPluginEvent.where(created_at: report.start_date..report.end_date)
  .group("date_trunc('day', created_at)")
  .count

# 应用原生 SQL via DB.query（与 Discourse 其他报表保持一致）
report.data = DB.query("SELECT ... GROUP BY day")
```

---

## 关联规范

- `plugins/01_plugin_rb_and_backend.md` — plugin.rb 基础结构
- `plugins/07_modal_and_ui_components.md` — Admin Modal 弹窗
- `plugins/02_plugin_api_frontend.md` — Connector 基础

---

## 快速参考

```javascript
// 设置插件图标（仅 admin）
initialize(container) {
  if (!container.lookup("service:current-user")?.admin) { return; }
  withPluginApi((api) => { api.setAdminPluginIcon("my-plugin", "star"); });
}

// Admin Connector 注入（文件放对位置即自动注入）
// connectors/admin-user-controls-after/my-button.gjs
static shouldRender(args, context) { return context.currentUser?.admin; }
```

```ruby
# 后端报表注册
add_report("my_events") do |report|
  report.icon = "star"
  report.modes = [:table]
  report.labels = [
    { type: :date, property: :day, title: "日期" },
    { type: :number, property: :count, title: "数量" },
  ]
  report.data = DB.query("SELECT day, count FROM ...").map { |r| { day: r.day, count: r.count } }
end
```
