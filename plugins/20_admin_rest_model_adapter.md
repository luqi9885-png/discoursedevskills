# Plugin 20 — Admin RestModel + Adapter 数据层规范

> 来源验证：
> - `plugins/discourse-ai/assets/javascripts/discourse/admin/models/ai-persona.js`（完整 168 行）
> - `plugins/discourse-ai/assets/javascripts/discourse/admin/adapters/ai-persona.js`（完整 21 行）
> - `frontend/discourse/app/adapters/rest.js`（RestAdapter 基类）

## 概述

Admin 插件页面的数据层由两个类组成：
- **RestModel**（`admin/models/my-plugin-item.js`）：定义字段映射、序列化、自定义操作
- **RestAdapter**（`admin/adapters/my-plugin-item.js`）：定义 API 路径和请求格式

文件名用 kebab-case，与 `store.findAll("my-plugin-item")` 中的类型字符串对应。

---

## 核心规范

### 1. RestAdapter — 路径和格式配置

必须继承 `RestAdapter` 并覆盖关键方法（来自 discourse-ai 真实代码）：

```javascript
// admin/adapters/ai-persona.js
import RestAdapter from "discourse/adapters/rest";

export default class Adapter extends RestAdapter {
  jsonMode = true;  // 发送 JSON body，而非默认的 form-encoded

  basePath() {
    return "/admin/plugins/discourse-ai/";
  }

  // 必须覆盖：默认实现会把 kebab-case 转成 underscore，导致路径错误
  // 默认：ai-persona → ai_persona（错误）
  // 覆盖后：直接返回正确的 kebab-case 名称
  pathFor(store, type, findArgs) {
    let path =
      this.basePath(store, type, findArgs) +
      store.pluralize(this.apiNameFor(type));
    return this.appendQueryParams(path, findArgs);
  }

  apiNameFor() {
    return "ai-persona";   // 精确返回 API 路径中的资源名
  }
}
```

**生成的 URL：**
```
GET    /admin/plugins/discourse-ai/ai-personas       findAll
GET    /admin/plugins/discourse-ai/ai-personas/:id   find
POST   /admin/plugins/discourse-ai/ai-personas       save (新建)
PUT    /admin/plugins/discourse-ai/ai-personas/:id   save (更新)
DELETE /admin/plugins/discourse-ai/ai-personas/:id   destroyRecord
```

### 2. RestModel — 字段定义与序列化

```javascript
// admin/models/ai-persona.js
import { ajax } from "discourse/lib/ajax";
import RestModel from "discourse/models/rest";

// 创建时发送的字段
const CREATE_ATTRIBUTES = [
  "name", "description", "system_prompt", "enabled",
  "allowed_group_ids", "tools", "temperature", "top_p",
  "default_llm_id", "vision_enabled",
];

// 更新时发送的字段（可能是 CREATE_ATTRIBUTES 的子集）
const UPDATE_ATTRIBUTES = [
  "id",          // 必须包含！
  "name", "description", "enabled", "allowed_group_ids",
  "tools", "temperature",
];

export default class AiPersona extends RestModel {
  // createProperties()：POST 新建时调用，返回要发送的字段
  createProperties() {
    return this.getProperties(CREATE_ATTRIBUTES);
  }

  // updateProperties()：PUT 更新时调用，必须包含 id
  updateProperties() {
    const attrs = this.getProperties(UPDATE_ATTRIBUTES);
    attrs.id = this.id;  // 确保 id 存在（即使已在 UPDATE_ATTRIBUTES 中也显式保证）
    return attrs;
  }

  // populateTools()：wire 格式转换，把服务端的数组格式转为前端易用的结构
  // 服务端格式：[["ToolName", {option: val}, forceFlag], "OtherTool"]
  // 前端格式：{ tools: ["ToolName", "OtherTool"], toolOptions: {ToolName: {option: val}}, forcedTools: ["ToolName"] }
  populateTools(attrs) {
    const forcedTools = [];
    const toolOptions = {};

    const flatTools = (attrs.tools || []).map((tool) => {
      if (typeof tool === "string") {
        return tool;
      }
      const [toolId, options, force] = tool;
      if (options && Object.keys(options).length > 0) {
        toolOptions[toolId] = options;
      }
      if (force) {
        forcedTools.push(toolId);
      }
      return toolId;
    });

    attrs.tools = flatTools;
    attrs.forcedTools = forcedTools;
    attrs.toolOptions = toolOptions;
  }

  // fromPOJO()：从服务端 JSON 构建前端 Model 实例
  fromPOJO(data) {
    const persona = AiPersona.create(data);
    persona.populateTools(data);  // 转换 wire 格式
    return persona;
  }

  // toPOJO()：把前端结构转回 wire 格式，用于发送给服务端
  toPOJO() {
    const attrs = this.getProperties(CREATE_ATTRIBUTES);
    this.populateTools(attrs);  // 把 toolOptions/forcedTools 重新打包回数组格式
    return attrs;
  }

  // 自定义 AJAX 操作（不走 store CRUD 的操作）
  async createUser() {
    const result = await ajax(
      `/admin/plugins/discourse-ai/ai-personas/${this.id}/create-user.json`,
      { type: "POST" }
    );
    this.user = result.user;
    this.user_id = this.user.id;
    return this.user;
  }

  // 静态方法：不依赖实例的操作
  static async import(json) {
    const result = await ajax("/admin/plugins/discourse-ai/ai-personas/import.json", {
      type: "POST",
      data: json,
    });
    return result;
  }
}
```

### 3. Store API 速查

```javascript
// Route 中加载数据
export default class AdminPluginsDiscourseAiPersonasRoute extends DiscourseRoute {
  async model() {
    // findAll 返回 ResultSet，不是数组！
    const personas = await this.store.findAll("ai-persona");
    // 必须用 .content 取数组
    return personas.content;
  }
}

// 同时加载多个资源
async model() {
  const [personas, llms] = await Promise.all([
    this.store.findAll("ai-persona"),
    this.store.findAll("ai-llm"),
  ]);
  return { personas: personas.content, llms: llms.content };
}

// find 单条
const persona = await this.store.find("ai-persona", id);

// 新建记录
const record = this.store.createRecord("ai-persona", {
  name: "New Persona",
  enabled: false,
});
await record.save();  // 调用 createProperties() 发 POST

// 更新记录
record.name = "Updated Name";
await record.save();  // 调用 updateProperties() 发 PUT

// 删除记录
await this.store.destroyRecord("ai-persona", record);
```

### 4. 文件位置约定

```
my-plugin/
└── assets/javascripts/discourse/
    └── admin/
        ├── adapters/
        │   └── my-plugin-item.js      # kebab-case，与 store 类型字符串对应
        └── models/
            └── my-plugin-item.js      # 同上
```

`store.findAll("my-plugin-item")` → 自动加载 `admin/adapters/my-plugin-item.js` + `admin/models/my-plugin-item.js`

---

## 反模式

```javascript
// 错误：updateProperties 忘记包含 id
updateProperties() {
  return this.getProperties(["name", "enabled"]);  // PUT /personas/undefined !
}
// 正确：始终包含 id
updateProperties() {
  return { ...this.getProperties(["name", "enabled"]), id: this.id };
}

// 错误：不覆盖 pathFor，用默认实现
// 默认实现用 underscore()：my-plugin-item → my_plugin_item（URL 错误！）
// 正确：始终在 Adapter 中覆盖 pathFor 和 apiNameFor

// 错误：忘记 jsonMode = true
// 默认发 form-encoded，Rails 无法正确解析 JSON body
// 正确：Adapter 中声明 jsonMode = true

// 错误：把 findAll 结果直接当数组用
const personas = await this.store.findAll("ai-persona");
personas.forEach(...)         // 报错！findAll 返回 ResultSet，不是数组
personas.map(...)             // 报错！
// 正确：取 .content
personas.content.forEach(...)
personas.content.map(...)
```

---

## 快速参考

```javascript
// Adapter 最小实现
export default class MyAdapter extends RestAdapter {
  jsonMode = true;
  basePath() { return "/admin/plugins/my-plugin/"; }
  pathFor(store, type, findArgs) {
    return this.appendQueryParams(
      this.basePath() + store.pluralize(this.apiNameFor(type)),
      findArgs
    );
  }
  apiNameFor() { return "my-plugin-item"; }
}

// Model 最小实现
export default class MyModel extends RestModel {
  createProperties() { return this.getProperties(CREATE_ATTRS); }
  updateProperties() { return { ...this.getProperties(UPDATE_ATTRS), id: this.id }; }
}

// Store 使用
const items = (await this.store.findAll("my-plugin-item")).content;
const item  = await this.store.find("my-plugin-item", id);
const rec   = this.store.createRecord("my-plugin-item", attrs);
await rec.save();
await this.store.destroyRecord("my-plugin-item", rec);
```
