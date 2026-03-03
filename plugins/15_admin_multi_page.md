# Plugin 15 — Admin 插件多子页面（addAdminPluginConfigurationNav + Route Map）

> 来源：`plugins/discourse-ai/assets/javascripts/discourse/initializers/admin-plugin-configuration-nav.js`、`plugins/discourse-ai/assets/javascripts/discourse/admin-discourse-ai-plugin-route-map.js`、`plugins/discourse-ai/admin/assets/javascripts/discourse/routes/admin-plugins/show/discourse-ai-features.js`、`plugins/discourse-ai/admin/assets/javascripts/discourse/templates/admin-plugins/show/discourse-ai-features/index.gjs`

## 概述

当插件有多个功能区域时，用 `addAdminPluginConfigurationNav` 在插件页面创建子导航 Tab，每个 Tab 对应独立路由。目录结构、路由映射文件（`admin-xxx-plugin-route-map.js`）、路由文件（`routes/admin-plugins/show/`）、模板文件（`templates/admin-plugins/show/`）必须严格对齐，否则路由无法工作。

---

## 来源验证

- `plugins/discourse-ai/assets/javascripts/discourse/initializers/admin-plugin-configuration-nav.js`（addAdminPluginConfigurationNav 完整调用）
- `plugins/discourse-ai/assets/javascripts/discourse/admin-discourse-ai-plugin-route-map.js`（路由映射，嵌套路由）
- `plugins/discourse-ai/admin/assets/javascripts/discourse/routes/admin-plugins/show/discourse-ai-features.js`（Route 加载 model）
- `plugins/discourse-ai/admin/assets/javascripts/discourse/routes/admin-plugins/show/discourse-ai-llms.js`（更简洁的 Route）
- `plugins/discourse-ai/admin/assets/javascripts/discourse/templates/admin-plugins/show/discourse-ai-features/index.gjs`（模板）
- `plugins/discourse-ai/admin/assets/javascripts/discourse/templates/admin-plugins/show/discourse-ai-usage.gjs`（简单模板）

---

## 核心规范

### 1. addAdminPluginConfigurationNav — 子导航配置

在 initializer 中注册插件的子导航 Tab 列表：

```javascript
// assets/javascripts/discourse/initializers/admin-plugin-configuration-nav.js
import { withPluginApi } from "discourse/lib/plugin-api";

const PLUGIN_ID = "my-plugin";  // 必须与 plugin.rb 中的 name 字段完全一致

export default {
  name: "my-plugin-admin-plugin-configuration-nav",

  initialize(container) {
    const currentUser = container.lookup("service:current-user");
    if (!currentUser?.admin) { return; }

    withPluginApi((api) => {
      api.setAdminPluginIcon(PLUGIN_ID, "star");

      // addAdminPluginConfigurationNav：注册多子页面导航
      api.addAdminPluginConfigurationNav(PLUGIN_ID, [
        {
          label: "my_plugin.admin.tabs.features",  // i18n 键（显示在 Tab 上）
          route: "adminPlugins.show.my-plugin-features",  // 路由名（必须与 route map 中一致）
          description: "my_plugin.admin.tabs.features_desc",  // 子标题描述（可选）
        },
        {
          label: "my_plugin.admin.tabs.models",
          route: "adminPlugins.show.my-plugin-models",
          description: "my_plugin.admin.tabs.models_desc",
        },
        {
          label: "my_plugin.admin.tabs.settings",
          route: "adminPlugins.show.my-plugin-settings",
          description: "my_plugin.admin.tabs.settings_desc",
        },
      ]);
    });
  },
};
```

**路由名规律：** `adminPlugins.show.{plugin-id-段}-{子页面名}`，例如 plugin-id 为 `discourse-ai`、子页面为 `features`，则路由名为 `adminPlugins.show.discourse-ai-features`。

---

### 2. 路由映射文件（admin-xxx-plugin-route-map.js）

定义所有子页面的路由树：

```javascript
// assets/javascripts/discourse/admin-my-plugin-route-map.js
// 文件名格式：admin-{plugin-id}-route-map.js（使用 kebab-case）

export default {
  resource: "admin.adminPlugins.show",  // 固定值，挂到 admin.adminPlugins.show 资源下

  path: "/plugins",  // 固定值

  map() {
    // 简单子页面（无嵌套）
    this.route("my-plugin-settings", { path: "my-plugin-settings" });
    this.route("my-plugin-usage", { path: "my-plugin-usage" });

    // 有 index/new/edit 的子页面（CRUD 套路）
    this.route("my-plugin-models", { path: "my-plugin-models" }, function () {
      this.route("new");                          // /admin/plugins/my-plugin/my-plugin-models/new
      this.route("edit", { path: "/:id/edit" }); // /admin/plugins/my-plugin/my-plugin-models/:id/edit
    });

    // 有 index/edit 的子页面（无 new，edit 通过 id）
    this.route("my-plugin-features", { path: "my-plugin-features" }, function () {
      this.route("edit", { path: "/:id/edit" });
    });
  },
};
```

---

### 3. 目录结构与文件对应关系

Admin 插件多子页面需要在 `admin/assets/` 下建立严格的目录结构：

```
admin/
└── assets/
    └── javascripts/
        └── discourse/
            ├── routes/
            │   └── admin-plugins/
            │       └── show/
            │           ├── my-plugin-features.js       ← features 主路由
            │           ├── my-plugin-features/
            │           │   └── edit.js                 ← features/:id/edit 路由
            │           ├── my-plugin-models.js         ← models 主路由（index）
            │           ├── my-plugin-models/
            │           │   ├── new.js                  ← models/new 路由
            │           │   └── edit.js                 ← models/:id/edit 路由
            │           └── my-plugin-settings.js       ← settings 路由
            └── templates/
                └── admin-plugins/
                    └── show/
                        ├── my-plugin-features/
                        │   ├── index.gjs               ← features 列表模板
                        │   └── edit.gjs                ← features 编辑模板
                        ├── my-plugin-models/
                        │   ├── index.gjs               ← models 列表模板
                        │   ├── new.gjs                 ← models 新建模板
                        │   └── edit.gjs                ← models 编辑模板
                        └── my-plugin-settings.gjs      ← settings 模板（简单页面）
```

**关键规律：**
- 路由文件目录 = `routes/admin-plugins/show/`
- 模板文件目录 = `templates/admin-plugins/show/`
- 嵌套路由（new/edit）对应子目录下的文件
- 简单页面（无嵌套）直接是一个 `.gjs` 文件

---

### 4. Route 文件（加载数据）

```javascript
// admin/assets/javascripts/discourse/routes/admin-plugins/show/my-plugin-models.js

import DiscourseRoute from "discourse/routes/discourse";

// 简单 Route：只加载列表数据
export default class DiscourseAiMyModelsRoute extends DiscourseRoute {
  model() {
    return this.store.findAll("my-plugin-model");  // 通过 store 加载
  }
}
```

```javascript
// admin/assets/javascripts/discourse/routes/admin-plugins/show/my-plugin-features.js
// 需要注入 service 的 Route

import { service } from "@ember/service";
import DiscourseRoute from "discourse/routes/discourse";

export default class AdminPluginsShowMyPluginFeaturesRoute extends DiscourseRoute {
  @service store;

  async model() {
    const features = await this.store.findAll("my-plugin-feature");
    return features.content;  // .content 取出数组（去掉 ResultSet 包装）
  }
}
```

```javascript
// admin/assets/javascripts/discourse/routes/admin-plugins/show/my-plugin-models/edit.js
// 编辑路由：通过 id 加载单个记录

import DiscourseRoute from "discourse/routes/discourse";

export default class MyPluginModelsEditRoute extends DiscourseRoute {
  async model({ id }) {
    return this.store.find("my-plugin-model", id);
  }
}
```

---

### 5. 模板文件（渲染页面）

简单模板（单页面，无嵌套）：

```javascript
// admin/assets/javascripts/discourse/templates/admin-plugins/show/my-plugin-settings.gjs

import MySettingsForm from "../../../../discourse/components/my-settings-form";

// @controller.model 是路由 model() 的返回值
export default <template>
  <MySettingsForm @model={{@controller.model}} />
</template>
```

有子路由的 index 模板：

```javascript
// admin/assets/javascripts/discourse/templates/admin-plugins/show/my-plugin-models/index.gjs

import MyModelList from "../../../../../discourse/components/my-model-list";

export default <template>
  <MyModelList @models={{@controller.model}} />
</template>
```

edit 模板：

```javascript
// admin/assets/javascripts/discourse/templates/admin-plugins/show/my-plugin-models/edit.gjs

import MyModelEditor from "../../../../../discourse/components/my-model-editor";

export default <template>
  <MyModelEditor @model={{@controller.model}} />
</template>
```

---

### 6. plugin.rb 中注册 Admin 路由

```ruby
after_initialize do
  add_admin_route(
    "my_plugin.admin.title",    # i18n 键（Admin 侧边栏显示的名称）
    "my-plugin",                # plugin ID（与 addAdminPluginConfigurationNav 第一个参数一致）
    { use_new_show_route: true }  # 必须加，使用新式路由系统
  )
end
```

---

### 7. Admin Store Model（前端数据模型）

Admin 页面常用 `RestModel` 作为前端数据模型，通过 `store.findAll / store.find` 加载：

```javascript
// assets/javascripts/discourse/admin/models/my-model.js
import RestModel from "discourse/models/rest";

export default class MyPluginModel extends RestModel {
  // 存储层 key = 类名 kebab-case 去 "my-plugin-" 前缀
  // 即 store.findAll("my-plugin-model") → GET /admin/discourse-ai/my-models.json
}
```

对应后端 Adapter（告诉 store 请求哪个 URL）：

```javascript
// assets/javascripts/discourse/admin/adapters/my-model.js
import DiscourseAjaxAdapter from "discourse/adapters/rest";

export default class MyModelAdapter extends DiscourseAjaxAdapter {
  get basePath() {
    return "/admin/discourse-ai/";
  }
  // store.findAll("my-plugin-model") → GET /admin/discourse-ai/my-plugin-models.json
  // store.find("my-plugin-model", 1) → GET /admin/discourse-ai/my-plugin-models/1.json
}
```

---

## 反模式（避免这样做）

```javascript
// addAdminPluginConfigurationNav 中 route 名与路由映射文件不一致
api.addAdminPluginConfigurationNav(PLUGIN_ID, [
  {
    route: "adminPlugins.show.myPlugin-features",  // 错误！驼峰 vs kebab 不一致
  },
]);

// 路由映射文件中：
this.route("my-plugin-features", ...);  // 对应路由名为 adminPlugins.show.my-plugin-features

// 正确：全部用 kebab-case
route: "adminPlugins.show.my-plugin-features"

// 路由文件放错目录
// 错误：routes/admin-plugins/my-plugin-features.js（缺少 show/ 层级）
// 正确：routes/admin-plugins/show/my-plugin-features.js

// 模板文件路径错误（import 路径计算错误）
// 正确做法：从 templates/admin-plugins/show/ 到 discourse/components/
// 每深一级目录就多一个 ../
import MyComp from "../discourse/components/my-comp";     // 从 show/ 上去
import MyComp from "../../discourse/components/my-comp";  // 从 show/sub/ 上去
import MyComp from "../../../../../discourse/components/my-comp";  // 从 show/sub/ 上去
```

---

## 关联规范

- `plugins/09_admin_ui.md` — setAdminPluginIcon + add_report（Admin UI 基础）
- `plugins/13_backend_advanced_patterns.md` — add_admin_route + plugin.rb 高级 DSL

---

## 快速参考

```javascript
// 1. initializer 注册
api.addAdminPluginConfigurationNav("my-plugin", [
  { label: "my_plugin.admin.tab1", route: "adminPlugins.show.my-plugin-tab1", description: "..." },
  { label: "my_plugin.admin.tab2", route: "adminPlugins.show.my-plugin-tab2" },
]);

// 2. 路由映射（admin-my-plugin-route-map.js）
export default {
  resource: "admin.adminPlugins.show",
  path: "/plugins",
  map() {
    this.route("my-plugin-tab1", { path: "tab1" });
    this.route("my-plugin-tab2", { path: "tab2" }, function () {
      this.route("new");
      this.route("edit", { path: "/:id/edit" });
    });
  },
};

// 3. Route（routes/admin-plugins/show/my-plugin-tab1.js）
export default class extends DiscourseRoute {
  model() { return this.store.findAll("my-plugin-item"); }
}

// 4. 模板（templates/admin-plugins/show/my-plugin-tab1.gjs）
import MyComp from "../../../../discourse/components/my-comp";
export default <template><MyComp @items={{@controller.model}} /></template>

// 5. plugin.rb
after_initialize do
  add_admin_route("my_plugin.admin.title", "my-plugin", { use_new_show_route: true })
end
```
