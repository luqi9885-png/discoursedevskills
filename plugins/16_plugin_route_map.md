# Plugin 16 — 插件前端路由：route-map / Route / Controller / Template

> 来源：`plugins/discourse-assign/assigned-group-route-map.js`、`assigned-messages-route-map.js`、`assigns-activity-route-map.js`、`routes/group/assigned.js`、`routes/group/assigned/show.js`、`routes/user-activity/assigned.js`、`routes/user-private-messages/assigned/index.js`、`templates/group/assigned.gjs`、`templates/group/assigned/show.gjs`、`pre-initializers/extend-category-for-assign.js`

## 概述

插件可向 Discourse 现有路由树（用户活动页、私信页、群组页等）挂载全新页面，无需修改核心代码。核心机制：route-map 文件声明挂载点，Route 文件加载数据，Controller 管理状态，Template 渲染 UI。掌握文件位置与路由名对应规律是关键。

---

## 来源验证

| 验证点 | 文件 | 关键代码 |
|--------|------|---------|
| 群组路由映射（嵌套 + 动态参数） | `assigned-group-route-map.js` | `resource: "group"`, `path: "/:filter"` |
| 私信路由映射 | `assigned-messages-route-map.js` | `resource: "user.userPrivateMessages"` |
| 活动页路由映射 | `assigns-activity-route-map.js` | `resource: "user.userActivity"` |
| UserTopicListRoute 继承 | `routes/user-activity/assigned.js` | `extends UserTopicListRoute`, `titleToken()` |
| setupController + redirect | `routes/group/assigned.js` | `modelFor("group")`, `redirect()` 跳默认子路由 |
| findOrResetCachedTopicList | `routes/group/assigned/show.js` | 缓存话题列表防重复请求 |
| 复用私信路由构建器 | `routes/user-private-messages/assigned/index.js` | `createPMRoute(...)` |
| 话题列表模板 | `templates/group/assigned/show.gjs` | `<BasicTopicList>`, `<LoadMore>` |
| pre-initializer | `pre-initializers/extend-category-for-assign.js` | `Category.reopen`, `before: "inject-discourse-objects"` |

---

## 核心规范

### 1. Route Map 文件 — 声明挂载点

Route map 文件放在 `assets/javascripts/discourse/` 根目录，命名格式 `xxx-route-map.js`，Discourse 自动发现并加载：

```javascript
// assigns-activity-route-map.js  ← 挂到用户活动页
export default {
  resource: "user.userActivity",   // 挂载到哪个父路由资源
  map() {
    this.route("assigned");
    // 路由名：userActivity.assigned
    // URL：/u/:username/activity/assigned
  },
};

// assigned-messages-route-map.js  ← 挂到私信页
export default {
  resource: "user.userPrivateMessages",
  map() {
    this.route("assigned", function () {
      this.route("index", { path: "/" }); // index 使父路径同时匹配子路由
    });
  },
};

// assigned-group-route-map.js  ← 挂到群组页（嵌套 + 动态参数）
export default {
  resource: "group",
  map() {
    this.route("assigned", function () {
      this.route("show", { path: "/:filter" }); // :filter 是动态 URL 段
    });
    // 路由名：group.assigned / group.assigned.show
    // URL：/g/:groupname/assigned/:filter
  },
};
```

**常用 resource 挂载点速查：**

| resource | 路由前缀 | 对应 URL 模式 |
|----------|----------|--------------|
| `user.userActivity` | `userActivity.xxx` | `/u/:username/activity/xxx` |
| `user.userPrivateMessages` | `userPrivateMessages.xxx` | `/u/:username/messages/xxx` |
| `group` | `group.xxx` | `/g/:groupname/xxx` |
| `admin.adminPlugins.show` | `adminPlugins.show.xxx` | Admin 插件子页（见 Plugin 15）|

---

### 2. Route 文件 — 加载数据

路由文件目录结构与路由名严格对应（`.` → `/`，camelCase → kebab-case）：

```
路由名 userActivity.assigned              → routes/user-activity/assigned.js
路由名 group.assigned                     → routes/group/assigned.js
路由名 group.assigned.show               → routes/group/assigned/show.js
路由名 userPrivateMessages.assigned.index → routes/user-private-messages/assigned/index.js
```

**方式一：继承 UserTopicListRoute（话题列表页最简形式）**

```javascript
// routes/user-activity/assigned.js
import { service } from "@ember/service";
import UserTopicListRoute from "discourse/routes/user-topic-list";
import { i18n } from "discourse-i18n";

export default class UserActivityAssigned extends UserTopicListRoute {
  @service currentUser;
  @service store;

  // 当模板/controller 路径与路由名不一致时，显式声明
  templateName = "user-activity.assigned";
  controllerName = "user-activity.assigned";

  userActionType = 16;  // 对应后端 UserAction.types 值（用于过滤用户动作）
  noContentHelpKey = "discourse_assigns.no_assigns"; // 空列表提示文案 key

  beforeModel() {
    if (!this.currentUser) {
      this.send("showLogin"); // 未登录则触发登录弹窗
    }
  }

  model(params) {
    return this.store.findFiltered("topicList", {
      filter: `topics/messages-assigned/${this.modelFor("user").username_lower}`,
      params: {
        exclude_category_ids: [-1],
        order: params.order,
        ascending: params.ascending,
        search: params.search,
      },
    });
  }

  titleToken() {
    return i18n("discourse_assign.assigned"); // 设置浏览器 <title> 后缀
  }
}
```

**方式二：继承 DiscourseRoute（需要 setupController 或 redirect）**

```javascript
// routes/group/assigned.js（父路由：加载列表，默认跳子路由）
import { service } from "@ember/service";
import { ajax } from "discourse/lib/ajax";
import DiscourseRoute from "discourse/routes/discourse";

export default class GroupAssigned extends DiscourseRoute {
  @service router;

  model() {
    // modelFor("group") 获取当前群组页的 model
    return ajax(`/assign/members/${this.modelFor("group").name}`);
  }

  setupController(controller, model) {
    controller.setProperties({
      model,
      members: [],
      group: this.modelFor("group"), // 将父路由 model 注入 controller
    });
    // 也可向父 model 写入属性（影响父层 UI，如 tab 计数）
    controller.group.setProperties({
      assignment_count: model.assignment_count,
      group_assignment_count: model.group_assignment_count,
    });
    controller.findMembers(true);
  }

  redirect(model, transition) {
    // redirect：model 加载后、setupController 之前运行
    // 用途：将无参数访问的父路由自动跳转到默认子路由
    if (!transition.to.params.hasOwnProperty("filter")) {
      this.router.transitionTo("group.assigned.show", "everyone");
    }
  }
}
```

```javascript
// routes/group/assigned/show.js（子路由：:filter 参数决定加载内容）
import { findOrResetCachedTopicList } from "discourse/lib/cached-topic-list";
import DiscourseRoute from "discourse/routes/discourse";

export default class GroupAssignedShow extends DiscourseRoute {
  model(params) {
    // params.filter 来自 URL 动态段 /:filter
    let filter;
    if (["everyone", this.modelFor("group").name].includes(params.filter)) {
      filter = `topics/group-topics-assigned/${this.modelFor("group").name}`;
    } else {
      filter = `topics/messages-assigned/${params.filter}`;
    }

    // findOrResetCachedTopicList：读缓存 → 无则发请求（避免重复加载）
    return (
      findOrResetCachedTopicList(this.session, filter) ||
      this.store.findFiltered("topicList", {
        filter,
        params: {
          order: params.order,
          ascending: params.ascending,
          search: params.search,
          direct: params.filter !== "everyone",
        },
      })
    );
  }

  setupController(controller, model) {
    controller.setProperties({
      model,
      search: this.currentModel.params.search,
    });
  }
}
```

**方式三：用核心构建器复用私信路由逻辑**

```javascript
// routes/user-private-messages/assigned/index.js
import createPMRoute from "discourse/routes/build-private-messages-route";

// 最简形式：复用内置私信路由构建器
// 参数：(类型名, filter路径片段, controller名)
export default createPMRoute("assigned", "private-messages-assigned", "assigned");
```

---

### 3. Controller — 管理页面状态

Controller 文件路径与 Route 完全平行（`routes/` → `controllers/`）：

```javascript
// controllers/group/assigned.js
import { tracked } from "@glimmer/tracking";
import Controller, { inject as controller } from "@ember/controller";
import { action } from "@ember/object";
import { service } from "@ember/service";
import { ajax } from "discourse/lib/ajax";
import discourseDebounce from "discourse/lib/debounce";
import discourseComputed from "discourse/lib/decorators";
import { INPUT_DELAY } from "discourse/lib/environment";
import { trackedArray } from "discourse/lib/tracked-tools";

export default class GroupAssigned extends Controller {
  @service router;
  @controller application;  // inject as controller：注入另一个 Controller

  @tracked filter = "";
  @tracked loading = false;
  @tracked offset = 0;
  @trackedArray members = []; // Discourse 封装的响应式数组（等价于 TrackedArray）

  // @discourseComputed：声明式计算属性，依赖 queryParams 或其他路径
  @discourseComputed("router.currentRoute.queryParams.order")
  order(order) {
    return order || "";
  }

  @discourseComputed("site.mobileView")
  isDesktop(mobileView) {
    return !mobileView;
  }

  @action
  onChangeFilterName(value) {
    // 防抖：INPUT_DELAY (400ms) 后才触发实际搜索
    discourseDebounce(this, this._setFilter, value, INPUT_DELAY);
  }

  @action
  async loadMore() {
    if (this.loading) return;
    this.offset += 50;
    this.loading = true;
    const result = await ajax(`/assign/members/${this.group.name}`, {
      data: { filter: this.filter, offset: this.offset },
    });
    this.members = [...this.members, ...result.members];
    this.loading = false;
  }

  _setFilter(filter) {
    this.loading = true;
    this.offset = 0;
    this.filter = filter;
    ajax(`/assign/members/${this.group.name}`, {
      data: { filter, offset: 0 },
    }).then((result) => {
      this.members = result.members;
      this.loading = false;
    });
  }
}
```

---

### 4. Template — 渲染页面

**父模板（左导航 + `{{outlet}}` 挂载子路由）：**

```javascript
// templates/group/assigned.gjs
import { Input } from "@ember/component";
import { on } from "@ember/modifier";
import LoadMore from "discourse/components/load-more";
import MobileNav from "discourse/components/mobile-nav";
import bodyClass from "discourse/helpers/body-class";
import withEventValue from "discourse/helpers/with-event-value";
import { i18n } from "discourse-i18n";
import GroupAssignedFilter from "../../components/group-assigned-filter";

export default <template>
  <section class="user-secondary-navigation group-assignments">
    {{bodyClass "group-assign"}}
    <MobileNav @desktopClass="action-list nav-stacked" class="activity-nav">
      {{#if @controller.isDesktop}}
        <div class="search-div">
          <Input
            {{on "input" (withEventValue @controller.onChangeFilterName)}}
            @value={{readonly @controller.filterName}}
            placeholder={{i18n "discourse_assign.sidebar_name_filter_placeholder"}}
          />
        </div>
      {{/if}}
      <LoadMore @selector=".activity-nav li" @action={{@controller.loadMore}}>
        <GroupAssignedFilter @filter="everyone" @assignmentCount={{@controller.group.assignment_count}} />
        {{#each @controller.members as |member|}}
          <GroupAssignedFilter @filter={{member}} />
        {{/each}}
      </LoadMore>
    </MobileNav>
  </section>

  <section class="user-content">
    {{outlet}}
  </section>
</template>
```

**子模板（BasicTopicList + LoadMore + 搜索框）：**

```javascript
// templates/group/assigned/show.gjs
import { Input } from "@ember/component";
import { on } from "@ember/modifier";
import BasicTopicList from "discourse/components/basic-topic-list";
import ConditionalLoadingSpinner from "discourse/components/conditional-loading-spinner";
import LoadMore from "discourse/components/load-more";
import withEventValue from "discourse/helpers/with-event-value";
import { i18n } from "discourse-i18n";

export default <template>
  <div class="topic-search-div">
    <Input
      {{on "input" (withEventValue @controller.onChangeFilter)}}
      @value={{readonly @controller.search}}
      @type="search"
      placeholder={{i18n "discourse_assign.topic_search_placeholder"}}
      autocomplete="off"
    />
  </div>

  <LoadMore
    @selector=".paginated-topics-list .topic-list tr"
    @action={{@controller.loadMore}}
    class="paginated-topics-list"
  >
    <BasicTopicList
      @topicList={{@controller.model}}
      @hideCategory={{@controller.hideCategory}}
      @changeSort={{@controller.changeSort}}
      @bulkSelectEnabled={{@controller.bulkSelectEnabled}}
      @selected={{@controller.selected}}
      @hasIncoming={{@controller.hasIncoming}}
      @showInserted={{@controller.showInserted}}
    />
    <ConditionalLoadingSpinner @condition={{@controller.model.loadingMore}} />
  </LoadMore>
</template>
```

---

### 5. Pre-Initializer — 在路由启动前扩展 Ember 类

```javascript
// pre-initializers/extend-category-for-assign.js
import { computed } from "@ember/object";
import Category from "discourse/models/category";

export default {
  name: "extend-category-for-assign",
  before: "inject-discourse-objects", // 必须在 Ember 初始化前运行

  initialize() {
    // Category.reopen：向所有 Category 实例添加计算属性
    // 用于读取 custom_fields 中的插件自定义字段
    Category.reopen({
      enable_unassigned_filter: computed(
        "custom_fields.enable_unassigned_filter",
        {
          get() {
            return this?.custom_fields?.enable_unassigned_filter === "true";
          },
        }
      ),
    });
  },
};
```

何时用 pre-initializer vs initializer：
- pre-initializer（`before: "inject-discourse-objects"`）：需要在 Ember app 初始化前完成的 Model/Class 扩展
- initializer（withPluginApi / apiInitializer）：需要 Plugin API 的功能，必须在 Ember app 启动后运行

---

### 6. 完整目录结构

```
plugins/my-plugin/
└── assets/javascripts/discourse/
    ├── group-my-feature-route-map.js      ← route-map（根目录，自动注册）
    ├── activity-my-feature-route-map.js
    ├── pre-initializers/
    │   └── extend-my-model.js
    ├── routes/
    │   ├── group/
    │   │   ├── my-feature.js              ← group.myFeature（父路由）
    │   │   └── my-feature/
    │   │       └── show.js                ← group.myFeature.show（:id 动态段）
    │   └── user-activity/
    │       └── my-feature.js              ← userActivity.myFeature
    ├── controllers/
    │   └── group/
    │       └── my-feature.js
    └── templates/
        ├── group/
        │   ├── my-feature.gjs             ← 父模板（含 {{outlet}}）
        │   └── my-feature/
        │       └── show.gjs
        └── user-activity/
            └── my-feature.gjs
```

---

## 反模式

```javascript
// resource 写错
resource: "userActivity",       // ❌ 缺少 "user." 前缀
resource: "user_activity",      // ❌ 下划线而非 camelCase
resource: "user.userActivity",  // ✅

// 路由文件路径名 —— camelCase 路由名必须转成 kebab-case 目录名
// userActivity.assigned → routes/userActivity/assigned.js  // ❌
// userActivity.assigned → routes/user-activity/assigned.js // ✅

// setupController 直接赋值不触发响应式
controller.model = model;            // ❌
controller.setProperties({ model }); // ✅

// titleToken 忘记 return
titleToken() { i18n("key"); }       // ❌ 返回 undefined，<title> 不更新
titleToken() { return i18n("key"); } // ✅
```

---

## 关联规范

- `plugins/15_admin_multi_page.md` — Admin 子页面路由（`admin-xxx-plugin-route-map.js`）
- `javascript/03_routes.md` — Ember Routes 核心规范
- `javascript/04_models.md` — store.findFiltered / findOrResetCachedTopicList

---

## 快速参考

```javascript
// 1. route-map（放在 assets/javascripts/discourse/ 根目录，命名 xxx-route-map.js）
export default {
  resource: "user.userActivity",
  map() { this.route("my-tab"); }
};

// 2. Route（UserTopicListRoute 最简形式）
export default class extends UserTopicListRoute {
  templateName = "user-activity.my-tab";
  titleToken() { return i18n("my_plugin.tab_title"); }
  model(params) {
    return this.store.findFiltered("topicList", {
      filter: `topics/my-filter/${this.modelFor("user").username_lower}`,
      params: { order: params.order, ascending: params.ascending },
    });
  }
}

// 3. Template（BasicTopicList + LoadMore）
export default <template>
  <LoadMore @selector=".topic-list tr" @action={{@controller.loadMore}}>
    <BasicTopicList @topicList={{@controller.model}} @changeSort={{@controller.changeSort}} />
    <ConditionalLoadingSpinner @condition={{@controller.model.loadingMore}} />
  </LoadMore>
</template>

// 4. pre-initializer（扩展 Ember 基础类）
export default {
  name: "my-plugin-pre-init",
  before: "inject-discourse-objects",
  initialize() {
    Category.reopen({ myField: computed("custom_fields.my_field", { get() { ... } }) });
  },
};
```
