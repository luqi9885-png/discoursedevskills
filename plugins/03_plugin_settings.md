# 插件开发规范 03 — settings.yml 与 I18n

## 概述
每个插件通过 `config/settings.yml` 声明 SiteSettings，通过 `config/locales/` 提供多语言文本。设置可在管理后台修改，部分设置可推送到前端。

## 来源验证

- `plugins/discourse-solved/config/settings.yml` — 含多种类型设置
- `plugins/discourse-reactions/config/settings.yml` — enum、emoji_list、validator
- `plugins/discourse-presence/config/settings.yml` — 含 min/max
- `discourse/config/site_settings.yml` — 完整官方类型文档（文件头注释）

---

## 核心规范

### 1. 文件结构

```yaml
# config/settings.yml
my_plugin_namespace:            # 顶层 key（通常是插件名 snake_case）
  my_plugin_enabled:            # 设置名 = SiteSetting 方法名
    default: true
    client: true                # true = 前端 JS 可用
```

**访问方式**：
- Ruby 后端：`SiteSetting.my_plugin_enabled`
- JS 前端（需 `client: true`）：`siteSettings.my_plugin_enabled`

### 2. 所有可用选项

```yaml
my_plugin_namespace:

  # --- 布尔值 ---
  my_bool_setting:
    default: false
    client: true

  # --- 整数（带范围限制）---
  my_int_setting:
    default: 5
    min: 1
    max: 100
    client: true

  # --- 字符串 ---
  my_string_setting:
    default: ""
    min: 0           # 最短字符数
    max: 255         # 最长字符数

  # --- 枚举（固定选项，单选）---
  my_enum_setting:
    type: enum
    default: "always"
    choices:
      - "never"
      - "always"
      - "answered_only"
    client: true

  # --- 枚举（引用 Ruby 类）---
  my_enum_from_class:
    type: enum
    default: "heart"
    enum: "MyEnumClass"    # lib/ 下的类，返回选项列表

  # --- 简单列表（多个值）---
  my_list_setting:
    type: list
    default: "val1|val2"
    client: false

  # --- Group 列表 ---
  my_groups_setting:
    default: "1|2"          # 1=admin, 2=moderators, 14=trust_level_4
    mandatory_values: "1|2" # 这些值不能被管理员删除
    type: group_list
    allow_any: false
    refresh: true           # 修改后客户端自动刷新

  # --- Tag 列表 ---
  my_tags_setting:
    type: tag_list
    default: ""
    client: true

  # --- Emoji 列表 ---
  my_emojis:
    type: emoji_list
    default: "+1|heart"
    client: true
    area: "emojis"

  # --- 密码（前端脱敏）---
  my_api_key:
    default: ""
    secret: true

  # --- 带验证器的设置 ---
  my_validated_setting:
    default: ""
    validator: "MySettingValidator"   # lib/ 下实现 validate(val) 方法的类

  # --- 隐藏设置（不在管理后台显示）---
  my_hidden_setting:
    default: 4
    hidden: true

  # --- TrustLevel 设置 ---
  my_trust_level:
    default: 1
    enum: "TrustLevelSetting"
    client: true

  # --- 分类列表 ---
  my_categories:
    type: category_list
    default: ""
```

### 3. 关键属性说明

| 属性 | 含义 |
|------|------|
| `default` | 默认值 |
| `client: true` | 设置值推送到前端（JS 可读） |
| `refresh: true` | 修改后浏览器自动刷新 |
| `hidden: true` | 管理后台不显示（代码可读写） |
| `secret: true` | 前端显示为密码框，日志脱敏 |
| `mandatory_values` | 列表中不可删除的值（pipe 分隔） |
| `allow_any: false` | 列表只允许预定义值 |
| `validator` | 引用 Ruby 验证器类名 |

### 4. Group 默认值约定

常见 auto group ID：
- `1` = admins
- `2` = moderators
- `3` = staff（含 admin + moderator）
- `10` = trust_level_0
- `11` = trust_level_1
- `12` = trust_level_2
- `13` = trust_level_3
- `14` = trust_level_4

```yaml
my_allowed_groups:
  default: "1|2|14"     # admins + moderators + trust_level_4
  mandatory_values: "1|2"  # admins 和 moderators 必须有权限
  type: group_list
```

### 5. enabled_site_setting（插件开关）

在 `plugin.rb` 顶部（`after_initialize` 外）声明主开关：

```ruby
enabled_site_setting :my_plugin_enabled
```

这个设置必须在 `config/settings.yml` 中定义，且当设置为 `false` 时，插件的所有功能应停止工作（Discourse 会自动处理部分功能，但代码中也要守卫）。

### 6. 在代码中使用设置

```ruby
# 后端 Ruby
SiteSetting.my_plugin_enabled          # 读取
SiteSetting.my_int_setting             # 整数直接返回
SiteSetting.my_groups_setting          # 字符串 "1|2"
SiteSetting.my_groups_setting_map      # 自动转为 Group ID 数组（_map 后缀）

# 守卫检查（推荐写法）
return unless SiteSetting.my_plugin_enabled
return if SiteSetting.disable_my_feature
```

```javascript
// 前端 JS（需要 client: true）
import { service } from "@ember/service";

@service siteSettings;

get showFeature() {
  return this.siteSettings.my_plugin_enabled;
}
```

---

## I18n 规范

### 7. 文件分工

```
config/locales/
├── server.en.yml    ← 后端 I18n（邮件模板、错误、Admin 设置描述）
└── client.en.yml    ← 前端 I18n（JS 使用的所有字符串）
```

### 8. server.en.yml 结构

```yaml
en:
  # 设置的人类可读名称和描述（管理后台显示）
  site_settings:
    my_plugin_enabled: "Enable My Plugin"
    my_int_setting: "Maximum items to show"

  # 邮件模板
  user_notifications:
    my_notification:
      subject_template: "You have a new thing"
      text_body_template: |
        Hi %{username},
        ...

  # 通知文本
  notifications:
    my_type: "My notification"

  # 错误信息
  my_plugin:
    error: "Something went wrong"
```

### 9. client.en.yml 结构

```yaml
en:
  js:
    my_plugin:
      button_label: "Accept Answer"
      button_title: "Mark this as the accepted answer"
      success_message: "Answer accepted!"

    # 通知类型文本
    notifications:
      my_type: "received something"

    # 管理页面
    admin:
      my_plugin:
        title: "My Plugin Settings"
```

### 10. 代码中使用 I18n

```ruby
# 后端 Ruby
I18n.t("my_plugin.error")
I18n.t("user_notifications.my_notification.subject_template", username: user.username)
```

```javascript
// 前端 JS
import { i18n } from "discourse-i18n";

i18n("my_plugin.button_label")                    // 简单字符串
i18n("my_plugin.count", { count: 5 })            // 带变量
```

---

## 反模式（避免这样做）

```yaml
# ❌ 设置名不带插件前缀（全局命名空间污染）
cool_feature_enabled:
  default: true

# ✅ 带插件前缀
my_plugin_cool_feature_enabled:
  default: true

# ❌ 把所有设置都设 client: true（增加页面加载体积）
my_server_only_setting:
  default: "secret-key"
  client: true    # 不需要在前端用，不要暴露

# ❌ 直接在代码里写字符串（不支持多语言）
render json: { message: "Answer accepted!" }

# ✅ 用 I18n
render json: { message: I18n.t("solved.accepted") }
```

---

## 关联规范

- `plugins/01_plugin_rb_and_backend.md` — enabled_site_setting 在 plugin.rb 中的位置
- `plugins/02_plugin_frontend.md` — 前端如何通过 siteSettings service 读取设置
