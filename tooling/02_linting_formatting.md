# Tooling 02 — Lint 与格式化规范

> 来源：`.rubocop.yml`、`eslint.config.mjs`、`.prettierrc.cjs`、`lefthook.yml`、`package.json`
> 命令速查见 `tooling/01_commands.md`

---

## 概述

Discourse 使用多套 lint/格式化工具，覆盖 Ruby、JavaScript/TypeScript、CSS、模板、YAML：

| 工具 | 语言/文件 | 配置文件 |
|------|-----------|---------|
| RuboCop + rubocop-discourse | `.rb`, `.rake`, `.thor` | `.rubocop.yml` |
| Syntax Tree (stree) | `.rb`（格式化） | 继承自 `rubocop-discourse` |
| ESLint + discourse rules | `.js`, `.gjs` | `eslint.config.mjs` |
| Prettier | `.js`, `.gjs`, `.scss`, `.css` | `.prettierrc.cjs` |
| ember-template-lint | `.gjs`（Glimmer 模板部分） | 默认配置 |
| Stylelint | `.scss`, `.css` | 默认配置 |
| yaml-lint | `.yml`, `.yaml` | 默认配置 |
| i18n-lint | `client.en.yml`, `server.en.yml` | `script/i18n_lint.rb` |

所有工具通过 **Lefthook** 在 `pre-commit` 时自动运行，也可手动通过 `bin/lint` 触发。

---

## 1. RuboCop：Ruby 规范

### 1.1 配置继承

```yaml
# .rubocop.yml
inherit_gem:
  rubocop-discourse: stree-compat.yml   # 继承 Discourse 官方规则集
```

所有规则都来自 `rubocop-discourse` gem，`.rubocop.yml` 只做项目级覆盖。

### 1.2 Discourse 自定义 Cops（plugin 相关）

这些 cop 只对 `plugins/**/*` 生效，写 plugin 时必须遵守：

| Cop | 规则 | 违规示例 | 正确写法 |
|-----|------|---------|---------|
| `Discourse/Plugins/CallRequiresPlugin` | Controller 必须调用 `requires_plugin` | `class MyController < ApplicationController` | 加 `requires_plugin PLUGIN_NAME` |
| `Discourse/Plugins/UsePluginInstanceOn` | Plugin 钩子必须用 `on` DSL | 直接调 `DiscourseEvent.on` | `on(:event) { ... }` |
| `Discourse/Plugins/NamespaceMethods` | Plugin 方法必须在命名空间内 | 直接 `def my_method` | `module MyPlugin; def self.my_method` |
| `Discourse/Plugins/NamespaceConstants` | Plugin 常量必须在命名空间内 | `MY_CONST = "x"` | `module MyPlugin; MY_CONST = "x"` |
| `Discourse/Plugins/UseRequireRelative` | Plugin 内 require 必须用相对路径 | `require "discourse_solved/engine"` | `require_relative "engine"` |
| `Discourse/Plugins/NoMonkeyPatching` | Plugin 不能随意猴子补丁核心类 | `Topic.class_eval { ... }` | 用 `prepend` 或 `reloadable_patch` |

```ruby
# 合规示例（plugins/ 下的 controller）
class DiscourseSolved::AnswerController < ::ApplicationController
  requires_plugin DiscourseSolved::PLUGIN_NAME   # ← CallRequiresPlugin 要求
end

# 合规示例（plugins/ 下定义常量和方法）
module ::DiscourseSolved
  PLUGIN_NAME = "discourse-solved"               # ← NamespaceConstants 要求

  def self.accept_answer!(post, acting_user)     # ← NamespaceMethods 要求
    # ...
  end
end
```

### 1.3 其他关键规则

```yaml
# 迁移相关
Discourse/NoResetColumnInformationInMigrations:
  Enabled: true    # 迁移文件中不能调用 reset_column_information（除非特殊情况）

# RSpec
RSpec/InstanceVariable:
  Enabled: true
  Include:
    - spec/models/**/*    # models spec 中不允许用 @instance_variable（用 let 替代）
```

### 1.4 禁用特定规则

```ruby
# 单行禁用
Post.reset_column_information # rubocop:disable Discourse/NoResetColumnInformationInMigrations

# 区块禁用
# rubocop:disable Discourse/Plugins/NamespaceMethods
def some_method_that_must_be_here
end
# rubocop:enable Discourse/Plugins/NamespaceMethods
```

**原则**：禁用注释是最后手段，必须加注释说明原因。

### 1.5 Syntax Tree（stree）

Discourse 用 Syntax Tree 做 Ruby 格式化（而不是 RuboCop 的 Layout cops）。两者在 `lefthook.yml` 中分别运行：

```
rubocop → 检查代码质量（逻辑、风格）
stree   → 检查代码格式（缩进、换行、括号）
```

**实践**：运行 `bin/lint --fix` 会同时执行两者的自动修复（rubocop -A + stree write）。

---

## 2. ESLint：JavaScript 规范

### 2.1 配置结构

```js
// eslint.config.mjs（Flat Config 格式）
import DiscourseRecommended from "@discourse/lint-configs/eslint";

export default [
  ...DiscourseRecommended,      // 继承 Discourse 官方规则集
  {
    rules: {
      // 项目级额外规则
      "qunit/no-assert-equal": "error",      // QUnit 测试中禁用 assert.equal（用 assert.strictEqual）
      "qunit/no-loose-assertions": "error",  // 禁用宽松断言
      "ember/no-classic-components": "error", // 禁用 Ember 经典组件
      "discourse/no-route-template": "error", // 禁用旧式 route template
      "discourse/moved-packages-import-paths": "error", // 强制使用新的包路径
    },
  },
  {
    ignores: [
      "plugins/**/lib/javascripts/locale",  // 本地化文件忽略
      "public/", "vendor/", "spec/",        // 非源码目录忽略
      "**/node_modules/",
    ],
  },
  {
    // themes 有额外全局变量
    files: ["themes/**/*.{js,gjs}"],
    languageOptions: {
      globals: {
        settings: "readonly",
        themePrefix: "readonly",
      },
    },
  },
];
```

### 2.2 常见规则含义

**`qunit/no-assert-equal`**：`assert.equal` 用 `==`（宽松比较），必须用 `assert.strictEqual`（`===`）。

```js
// ❌
assert.equal(result, "hello");

// ✅
assert.strictEqual(result, "hello");
```

**`ember/no-classic-components`**：不能用 `Component.extend({...})`，必须用 Glimmer 组件。

```js
// ❌
export default Component.extend({ tagName: "div" });

// ✅
export default class MyComponent extends Component { ... }
```

**`discourse/moved-packages-import-paths`**：某些包已迁移到新路径，必须用新路径。

```js
// ❌（旧路径）
import { i18n } from "I18n";

// ✅（新路径）
import { i18n } from "discourse-i18n";
```

### 2.3 ember-template-lint

`.gjs` 文件中的 Glimmer 模板部分（`<template>...</template>`）单独由 `ember-template-lint` 检查，常见规则：

- 禁止内联事件处理（`onclick="..."` → 用 `@action`）
- 必须使用具名参数（`@arg`）而非位置参数
- 模板内 `{{this.xxx}}` vs `{{@xxx}}` 的正确用法

---

## 3. Prettier：代码格式化

### 3.1 配置

```js
// .prettierrc.cjs
module.exports = require("@discourse/lint-configs/prettier");
// 完全继承 Discourse 官方 prettier 配置，无项目级覆盖
```

Prettier 负责格式化：JS/GJS 文件的缩进、分号、引号、行长度；SCSS/CSS 的格式。

### 3.2 与 ESLint 的分工

```
ESLint  → 代码质量（逻辑错误、禁用 API、最佳实践）
Prettier → 代码格式（缩进、换行、括号、引号）
```

两者不冲突（`@discourse/lint-configs` 内已处理 `eslint-config-prettier` 禁用冲突规则）。

---

## 4. Lefthook：Pre-commit 自动化

`lefthook.yml` 定义三个 hook 组：

### 4.1 pre-commit（提交时自动运行）

```yaml
pre-commit:
  parallel: true          # 并行执行，速度快
  skip: [merge, rebase]   # merge/rebase 时跳过
  commands:
    rubocop:   glob: "**/*.{rb,rake,thor}"   run: bundle exec rubocop {staged_files}
    stree:     glob: "**/*.{rb,rake,thor}"   run: bundle exec stree check {staged_files}
    prettier:  glob: "**/*.{js,gjs,scss}"    run: pnpm pprettier --list-different {staged_files}
    eslint:    glob: "**/*.{js,gjs}"         run: pnpm eslint --quiet {staged_files}
    ember-template-lint: glob: "**/*.gjs"    run: pnpm ember-template-lint {staged_files}
    stylelint: glob: "**/*.scss"             run: pnpm stylelint {staged_files}
    yaml-syntax: glob: "**/*.{yaml,yml}"     run: bundle exec yaml-lint {staged_files}
    i18n-lint: glob: "**/{client,server}.en.yml" run: bundle exec ruby script/i18n_lint.rb {staged_files}
```

**关键**：只检查 `{staged_files}`（已 `git add` 的文件），不全量扫描，速度快。

### 4.2 fix-staged（手动修复暂存文件）

```bash
lefthook run fix-staged    # 自动修复暂存文件（等价于 bin/lint --fix）
```

### 4.3 fix-all（全量修复）

```bash
lefthook run fix-all       # 修复所有文件（等价于 bin/lint --fix --all）
```

---

## 5. 实践：开发流程中的 lint

### 5.1 日常开发

```bash
# 1. 改完文件后 lint（必须！）
bin/lint path/to/changed/file.rb path/to/changed/file.js

# 2. 如果有格式问题，自动修复
bin/lint --fix path/to/changed/file.rb

# 3. 修复最近改动的文件（最常用）
bin/lint --fix --recent
```

### 5.2 仅需 lint 特定类型时

```bash
# 只跑 RuboCop
bundle exec rubocop plugins/my-plugin/app/controllers/

# 只跑 ESLint
pnpm eslint plugins/my-plugin/assets/javascripts/

# 只跑 Prettier 检查（不修复）
pnpm pprettier --list-different plugins/my-plugin/assets/

# 只跑 Prettier 修复
pnpm prettier --write plugins/my-plugin/assets/
```

### 5.3 CI 全量检查

```bash
# package.json 中定义的 lint 脚本（CI 用）
pnpm lint          # 并行运行所有 JS lint
pnpm lint:js       # 仅 ESLint
pnpm lint:css      # 仅 Stylelint
pnpm lint:hbs      # 仅 ember-template-lint
pnpm lint:prettier # 仅 Prettier 检查
pnpm lint:types    # TypeScript 类型检查（ember-tsc）
```

---

## 6. i18n Lint 特殊规则

`i18n_lint.rb` 对英文 locale 文件有额外检查：

- 不允许使用 HTML 实体（`&amp;` 等），用真实字符
- 插值变量必须用 `%{var}` 格式，不是 `{{var}}`
- `client.en.yml` 和 `server.en.yml` 的 key 层级必须正确

```yaml
# ❌
my_key: "Hello &amp; World"
my_other: "Hello {{name}}"

# ✅
my_key: "Hello & World"
my_other: "Hello %{name}"
```

---

## 快速参考

```bash
# 日常 lint（必须执行）
bin/lint path/to/file               # 检查
bin/lint --fix path/to/file         # 检查 + 修复
bin/lint --fix --recent             # 修复最近改动

# Ruby 分别执行
bundle exec rubocop file.rb         # 检查
bundle exec rubocop -A file.rb      # 自动修复（Autocorrect）
bundle exec stree check file.rb     # 格式检查
bundle exec stree write file.rb     # 格式修复

# JavaScript 分别执行
pnpm eslint --fix file.js
pnpm prettier --write file.js
pnpm ember-template-lint --fix file.gjs
pnpm stylelint --fix file.scss

# 全量（CI 级别）
pnpm lint
bundle exec rubocop
```

```
Plugin 开发 lint 重点：
- Controller 必须有 requires_plugin
- 常量和方法必须在命名空间模块内
- require 用 require_relative（不用 require）
- 不能直接猴子补丁核心类（用 prepend / reloadable_patch）
- 提交前 bin/lint --fix 处理 staged files
```
