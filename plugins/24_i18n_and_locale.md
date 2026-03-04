# Plugin 24 — 插件国际化（i18n / locale）完整规范

> 来源：discourse-assign、discourse-reactions、discourse-solved、discourse-ai 等主流插件的国际化实现

## 概述

插件国际化分两套完全独立的文件：
- **`config/locales/server.xx.yml`**：后端（Ruby `I18n.t()`），用于 Job、Mailer、通知文本
- **`config/locales/client.xx.yml`**：前端（JS `i18n()`），用于 UI 文字、按钮、错误提示

两套文件独立加载，**不共享命名空间**。`en` 是必须提供的基础语言，其他语言按需添加。

---

## Part 1 — 文件结构

### 1. 目录布局

```
my-plugin/
└── config/
    └── locales/
        ├── server.en.yml      # 后端英文（必须）
        ├── server.zh_CN.yml   # 后端简体中文（可选）
        ├── client.en.yml      # 前端英文（必须）
        └── client.zh_CN.yml   # 前端简体中文（可选）
```

### 2. server.en.yml — 后端翻译

```yaml
# config/locales/server.en.yml
en:
  # 通知文本（复数形式）
  notifications:
    my_plugin_reaction:
      one:   "%{username} reacted to your post"
      other: "%{username} and %{count} others reacted to your posts"

  # 系统私信模板
  my_plugin_mailer:
    digest_subject: "Your activity summary from %{site_name}"
    greeting: "Hello %{username},"

  # Job / 业务逻辑使用的文本
  my_plugin:
    errors:
      not_found:       "Item not found"
      already_done:    "This has already been processed"
      quota_exceeded:  "You have reached the limit of %{limit} items"

    system_messages:
      welcome_title: "Welcome to My Plugin!"
      welcome_body: |
        Thank you for joining! Here's how to get started:
        1. Visit your profile page
        2. Enable notifications
```

**后端使用：**

```ruby
I18n.t("my_plugin.errors.quota_exceeded", limit: 10)
# => "You have reached the limit of 10 items"

I18n.t("notifications.my_plugin_reaction", username: "alice", count: 3)
# count > 1 → "alice and 3 others reacted to your posts"
# count == 1 → "alice reacted to your post"
```

### 3. client.en.yml — 前端翻译

前端所有 key 必须在 `en.js.` 命名空间下：

```yaml
# config/locales/client.en.yml
en:
  js:
    # 插件 UI 文字（用插件名作前缀避免冲突）
    my_plugin:
      add_reaction:    "Add Reaction"
      remove_reaction: "Remove"
      loading:         "Loading..."

      # 复数形式
      reaction_count:
        one:   "1 reaction"
        other: "%{count} reactions"

      # 错误提示
      errors:
        generic:    "Something went wrong. Please try again."
        not_found:  "Item not found"
        forbidden:  "You don't have permission"

      # 确认对话框
      confirm_delete: "Are you sure you want to delete this?"

      # 状态标签
      status:
        pending:   "Pending"
        active:    "Active"
        completed: "Completed"

    # 通知前端渲染文本（固定命名空间）
    notifications:
      my_plugin_reaction:             "<b>%{username}</b> reacted to your post"
      my_plugin_reaction_consolidated: "<b>%{username}</b> and %{count} others reacted"

  # Admin 设置页描述（key 名 = plugin.rb 中 setting 名称）
  site_settings:
    my_plugin_enabled:       "Enable My Plugin"
    my_plugin_max_items:     "Maximum items per user (0 = unlimited)"
    my_plugin_notify_user:   "Send notification when item is created"
```

---

## Part 2 — 前端使用

### 4. JavaScript 中访问翻译

```javascript
import { i18n } from "discourse-i18n";

i18n("my_plugin.add_reaction")
// => "Add Reaction"

i18n("my_plugin.reaction_count", { count: 1 })   // => "1 reaction"
i18n("my_plugin.reaction_count", { count: 5 })   // => "5 reactions"
```

**Glimmer 组件中：**

```javascript
// assets/javascripts/discourse/components/my-plugin-button.gjs
import { i18n } from "discourse-i18n";
import DButton from "discourse/components/d-button";

export default class MyPluginButton extends Component {
  get label() {
    return this.args.reacted
      ? i18n("my_plugin.remove_reaction")
      : i18n("my_plugin.add_reaction");
  }

  <template>
    <DButton @label={{this.label}} @action={{@onToggle}} />
  </template>
}
```

**hbs 模板中（旧式组件）：**

```handlebars
<button>{{i18n "my_plugin.add_reaction"}}</button>
<span>{{i18n "my_plugin.reaction_count" count=this.reactionCount}}</span>
```

### 5. 通知文本渲染（两层）

通知文本分后端和前端两层，分别负责不同场景：

```
server.en.yml  notifications.my_plugin_reaction.one/other
  → 用于：邮件通知、系统消息、后端日志

client.en.yml  js.notifications.my_plugin_reaction
  → 用于：页面内通知列表显示（HTML，支持 <b> 标签）
```

---

## Part 3 — 后端进阶

### 6. 用户语言切换（Job / Mailer 中必须做）

```ruby
# 错误：使用服务器默认语言
def send_digest(user)
  msg = I18n.t("my_plugin.digest_body")  # 可能不是用户的语言！
  send_message(user, msg)
end

# 正确：切换到用户语言
def send_digest(user)
  I18n.with_locale(user.effective_locale) do
    msg = I18n.t("my_plugin.digest_body")
    send_message(user, msg)
  end
end

# Job 中批量处理多个用户
class Jobs::MyPluginDigest < ::Jobs::Scheduled
  every 1.day

  def execute(args)
    User.where(active: true).find_each do |user|
      I18n.with_locale(user.effective_locale) do
        MyPlugin::DigestSender.new(user).send!
      end
    end
  end
end
```

### 7. Admin 设置描述

`site_settings.xxx` 不在 `js.` 命名空间下，是 `client.en.yml` 的顶层特殊区域：

```yaml
en:
  site_settings:
    my_plugin_enabled: "Enable My Plugin"
```

对应 `plugin.rb` 中的 setting 定义：

```ruby
setting(:my_plugin_enabled, false)
enabled_site_setting :my_plugin_enabled
```

Admin 设置页会自动显示 `site_settings.my_plugin_enabled` 作为 label。

---

## 反模式

```yaml
# 错误：server 和 client 混用同一个文件
# server.en.yml 里写了 en.js.my_plugin.xxx —— 前端读不到！
# client.en.yml 里写了 en.my_plugin.errors —— Ruby 读不到！

# 错误：前端 key 不在 en.js. 命名空间下
en:
  my_plugin:          # 错！前端读不到
    react: "React"
# 正确
en:
  js:
    my_plugin:
      react: "React"
```

```ruby
# 错误：Job 中忘记切换用户语言
def notify(user)
  I18n.t("my_plugin.msg")  # 服务器语言，不是用户语言
end

# 错误：后端硬编码字符串
notification_text = "#{username} reacted to your post"
# 正确
notification_text = I18n.t("notifications.my_plugin_reaction.one", username: username)
```

```javascript
// 错误：JS 中硬编码字符串
<button>Add Reaction</button>

// 正确
<button>{{i18n "my_plugin.add_reaction"}}</button>
```

---

## 快速参考

```
文件：
  config/locales/server.en.yml   → Ruby I18n.t()，后端使用
  config/locales/client.en.yml   → JS i18n()，必须在 en.js. 下

Ruby：
  I18n.t("my_plugin.key")
  I18n.t("my_plugin.key", var: value)
  I18n.with_locale(user.effective_locale) { I18n.t(...) }

JavaScript：
  import { i18n } from "discourse-i18n";
  i18n("my_plugin.key")
  i18n("my_plugin.key", { count: n })

特殊命名空间：
  server: notifications.xxx.one / other   → 通知后端文本（复数）
  client: js.notifications.xxx            → 通知前端渲染（HTML）
  client: site_settings.xxx              → Admin 设置页描述（不在 js. 下）
```
