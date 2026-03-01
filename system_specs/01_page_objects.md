# Page Objects — 系统测试规范

## 概述
Discourse 系统测试（Capybara）使用 Page Object 模式封装 UI 交互，
让 spec 代码清晰，避免到处散落 CSS 选择器。

## 来源验证（真实代码，多处比对）
- `spec/system/page_objects/pages/base.rb`
- `spec/system/page_objects/pages/topic.rb`
- `spec/system/page_objects/pages/login.rb`
- `spec/system/page_objects/components/composer.rb`
- `spec/system/page_objects/components/base.rb`

---

## 核心规范

### 1. 目录结构

```
spec/system/page_objects/
├── pages/          ← 完整页面（整页路由）
│   ├── base.rb
│   ├── topic.rb
│   ├── login.rb
│   └── ...
├── components/     ← 页面内可复用组件（弹窗、Composer 等）
│   ├── base.rb
│   ├── composer.rb
│   └── ...
└── modals/         ← 模态框（弹出层）
```

### 2. 继承关系

```ruby
# pages/base.rb
module PageObjects
  module Pages
    class Base
      include Capybara::DSL
      include RSpec::Matchers
      include SystemHelpers
    end
  end
end

# pages/topic.rb — 继承 Pages::Base
class Topic < PageObjects::Pages::Base; end

# components/base.rb（同结构）
# components/composer.rb — 继承 Components::Base
class Composer < PageObjects::Components::Base; end
```

### 3. 方法命名约定

| 模式 | 方法名 | 说明 |
|------|--------|------|
| 状态检查（正） | `has_topic_title?(text)` | 使用 Capybara 的 `has_css?` |
| 状态检查（负） | `has_no_topic_title?` | 使用 `has_no_css?` |
| 操作动作 | `click_login`, `open`, `visit_topic` | 动词开头 |
| 获取元素 | `topic_title`, `rich_editor` | 名词，返回元素 |

### 4. 绝对禁止：缓存 find() 结果

```ruby
# ❌ 错误：存储 find() 结果
def setup
  @title = find("#topic-title")   # 页面重渲染后变成 stale element
end

# ✅ 正确：每次调用都重新 find
def topic_title
  find("#topic-title .fancy-title")  # 每次调用都查找新鲜元素
end
```

### 5. 状态检查用 has_x? / has_no_x?

```ruby
# ✅ 用 has_css? / has_no_css? 包装成 has_x? 方法
def open?
  has_css?(".login-fullpage")
end

def closed?
  has_no_css?(".login-fullpage")
end

def has_topic_title?(text)
  has_css?("h1 .fancy-title", text: text)
end

def has_no_topic_title_editor?
  has_no_css?("#topic-title input#edit-title")
end
```

### 6. 操作方法返回 self（支持链式调用）

```ruby
def visit_topic(topic, post_number: nil)
  page.visit(topic.url(post_number))
  self   # ← 返回 self，支持链式
end

def open_composer_actions
  find(".composer-action-title .btn").click
  self
end

# 使用链式：
topic_page.visit_topic(topic).click_reply_button
```

### 7. 常量定义选择器

```ruby
class Composer < PageObjects::Components::Base
  COMPOSER_ID = "#reply-control"
  RICH_EDITOR = ".d-editor-input.ProseMirror"

  def opened?
    page.has_css?("#{COMPOSER_ID}.open")
  end
end
```

### 8. Page 组合 Component

Page Object 持有 Component 对象作为成员：

```ruby
class Topic < PageObjects::Pages::Base
  def initialize
    @composer_component = PageObjects::Components::Composer.new
    @topic_map_component = PageObjects::Components::TopicMap.new
  end

  def composer
    @composer_component
  end
end

# 在 spec 中：
topic_page = PageObjects::Pages::Topic.new
topic_page.visit_topic(topic)
topic_page.composer.opened?
```

---

## 在 Spec 中使用

```ruby
# spec/system/some_feature_spec.rb
describe "some feature" do
  fab!(:user)
  fab!(:topic)

  let(:topic_page) { PageObjects::Pages::Topic.new }
  let(:login_page) { PageObjects::Pages::Login.new }

  it "can do something" do
    sign_in(user)
    topic_page.visit_topic(topic)

    expect(topic_page).to have_topic_title(topic.title)
    topic_page.click_reply_button
    expect(topic_page.composer).to be_opened
  end
end
```

---

## 反模式（避免这样做）

```ruby
# ❌ 在 spec 里直接写 find/CSS
it "can open login" do
  visit("/login")
  expect(page).to have_css(".login-fullpage")
  find("#login-button").click
end

# ✅ 封装到 Page Object
it "can open login" do
  login_page.open
  expect(login_page).to be_open
  login_page.click_login
end
```

```ruby
# ❌ 不要在 page object 里 assert（职责分离）
def click_login
  find("#login-button").click
  expect(page).to have_css(".logged-in")  # 别在 PO 里 assert
end

# ✅ PO 只做操作，assert 在 spec 里
def click_login
  find("#login-button").click
  self
end
```

---

## 关联规范
- `ruby/05_rspec_testing.md` — fab! 和 RSpec 整体规范
