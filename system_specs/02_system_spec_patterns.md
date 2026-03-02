# System Specs 02 — 系统测试通用模式

> 来源：`spec/system/`（about_page_spec、flagging_post_spec、topic_page_spec、admin_filter_controls_spec）
> `spec/support/system_helpers.rb`
> `spec/system/page_objects/pages/topic.rb`、`base.rb`
> `plugins/discourse-solved/spec/system/solved_spec.rb`

---

## 概述

System Spec 是端到端测试，用真实浏览器（Playwright）驱动，测试完整用户流程。与 Request Spec 的区别：

| | System Spec | Request Spec |
|---|---|---|
| 层次 | 浏览器 + JS + UI | HTTP 请求/响应 |
| 速度 | 慢（秒级） | 快（毫秒级） |
| 验证 | 用户看到的 DOM | JSON/状态码 |
| 适用 | 交互流程、JS 行为 | API 正确性 |

---

## 1. 基本结构

```ruby
# frozen_string_literal: true

describe "功能描述", type: :system do
  # 1. 数据准备（fab! 在测试前一次性创建）
  fab!(:admin)
  fab!(:topic)
  fab!(:post_to_flag) { Fabricate(:post, topic: topic) }

  # 2. Page Object（let 懒加载）
  let(:topic_page) { PageObjects::Pages::Topic.new }
  let(:flag_modal) { PageObjects::Modals::Flag.new }

  # 3. 设置（before 在每个 it 前运行）
  before do
    SiteSetting.some_setting = true
    sign_in(admin)
  end

  # 4. 测试用例
  it "does the thing" do
    topic_page.visit_topic(topic)
    topic_page.expand_post_actions(post_to_flag)
    topic_page.click_post_action_button(post_to_flag, :flag)

    expect(topic_page).to have_css(".flag-modal")
  end
end
```

---

## 2. fab! 与 Fabricate

```ruby
# fab! 在 describe 块开始前运行一次（共享，速度快）
fab!(:admin)                                    # 等价于 Fabricate(:admin)
fab!(:current_user, :admin)                     # 指定 fabricator 类型
fab!(:topic) { Fabricate(:topic, closed: true) } # 自定义属性

# 在 context 内的 fab! 只属于该 context
context "with extra groups" do
  fab!(:extra_groups) { Fabricate.times(3, :group) }
end

# Fabricate 在 before/it 内按需创建（每次 it 都重新创建）
before do
  Fabricate(:post, topic: topic, created_at: 2.days.ago)
  Fabricate(:post, topic: topic, created_at: 8.days.ago)
end
```

**规律**：稳定的背景数据用 `fab!`；需要控制时序或每次不同的数据用 `Fabricate` 放在 `before`/`it` 内。

---

## 3. 登录：sign_in

```ruby
# sign_in 是 SystemHelpers 提供的方法，访问特殊路由完成登录
def sign_in(user)
  visit "/session/#{user.encoded_username}/become.json?redirect=false"
  expect(page).to have_content("Signed in to #{user.encoded_username} successfully")
end

# 用法
before { sign_in(admin) }

# 或在 it 内按需登录
it "allows admins to edit" do
  sign_in(admin)
  # ...
end

# 测试未登录状态：不调用 sign_in 即可（默认匿名）
it "hides edit link for anonymous" do
  about_page.visit
  expect(about_page).to have_no_edit_link
end
```

---

## 4. SiteSetting 与 Stubs

```ruby
before do
  SiteSetting.title = "My Forum"        # 直接赋值（测试结束后自动还原）
  SiteSetting.solved_enabled = true
end

# Stubs（使用 Mocha 风格）
Discourse.stubs(:site_creation_date).returns(5.years.ago)
Discourse.stubs(:plugins_sorted_by_name).returns([poll_plugin])

# 用于 context 分支
context "when the setting is disabled" do
  before { SiteSetting.display_eu_visitor_stats = false }
  it "doesn't show the row" { ... }
end

context "when the setting is enabled" do
  before { SiteSetting.display_eu_visitor_stats = true }
  it "shows the row" { ... }
end
```

---

## 5. 直接操作 DOM vs Page Object

### 5.1 直接用 Capybara DSL

```ruby
# 简单断言直接用 page / expect(page)
visit "/t/#{topic.slug}/#{topic.id}"
find("#toc-h2-testing .anchor", visible: :all).click
expect(current_url).to match("/t/#{topic.slug}/#{topic.id}#toc-h2-testing")

# CSS 存在性
expect(page).to have_css(".admin-plugins-list__row", count: 2)
expect(page).to have_no_css(".topic-admin-menu-content")

# 文本
expect(page).to have_content("Signed in successfully")

# send_keys（键盘快捷键）
send_keys([:shift, "a"])
send_keys(:end)
```

### 5.2 通过 Page Object（推荐）

```ruby
# Page Object 封装选择器和操作，避免 spec 直接写 CSS
topic_page.visit_topic(topic)
topic_page.expand_post_actions(post_to_flag)
topic_page.click_post_action_button(post_to_flag, :flag)

expect(topic_page).to have_css(UNACCEPTED_BUTTON_SELECTOR)
expect(topic_page).to have_no_css(ACCEPTED_BUTTON_SELECTOR)

# 使用 within 缩小查找范围
within("aside.accepted-answer.quote blockquote") do
  expect(page).to have_css("pre code.lang-ruby")
end
```

---

## 6. Page Object 实现规范

```ruby
module PageObjects
  module Pages
    class Topic < PageObjects::Pages::Base   # 继承 Base（包含 Capybara::DSL）
      def initialize
        @composer_component = PageObjects::Components::Composer.new  # 组合 Component
      end

      # 导航方法
      def visit_topic(topic, post_number: nil)
        page.visit(topic.url(post_number))
        self   # 返回 self 支持链式调用
      end

      # has_xxx? → 正向断言（Capybara 自动等待）
      def has_topic_title?(text)
        has_css?("h1 .fancy-title", text: text)
      end

      # has_no_xxx? → 负向断言
      def has_no_edit_link?
        has_no_css?(".edit-about-page")
      end

      # 操作方法
      def expand_post_actions(post)
        post_by_number(post).find(".show-more-actions").click
      end

      def click_post_action_button(post, button)
        find_post_action_button(post, button).click
      end

      # 私有：选择器映射（集中管理，避免重复）
      private

      def selector_for_post_action_button(button)
        case button
        when :flag   then ".post-controls .post-action-menu__flag"
        when :edit   then ".post-controls .post-action-menu__edit"
        when :delete then ".post-controls .post-action-menu__delete"
        else raise "Unknown button: #{button}"
        end
      end

      def within_post(post)
        within(post_by_number(post)) { yield }
      end
    end
  end
end
```

**Base 提供**：`Capybara::DSL`（`find`、`has_css?`、`visit`等）、`RSpec::Matchers`、`SystemHelpers`。

---

## 7. 私有辅助方法（减少 it 块复杂度）

Plugin spec 中常见的模式：将步骤提取为 `private` 方法：

```ruby
describe "Solved", type: :system do
  UNACCEPTED_BUTTON_SELECTOR = ".post-action-menu__solved-unaccepted"
  ACCEPTED_BUTTON_SELECTOR   = ".post-action-menu__solved-accepted"

  it "accepts and unaccepts post as solution" do
    sign_in(accepter)
    visit_solver_post         # ← 私有方法

    verify_solution_unaccepted_state
    accept_solution
    verify_solution_accepted_state

    unaccept_solution
    verify_solution_unaccepted_state
  end

  private

  def visit_solver_post
    topic_page.visit_topic(topic, post_number: 2)
  end

  def accept_solution
    find(UNACCEPTED_BUTTON_SELECTOR).click
  end

  def verify_solution_accepted_state
    expect(topic_page).to have_css(ACCEPTED_BUTTON_SELECTOR)
    expect(topic_page).to have_css("aside.accepted-answer.quote[data-expanded='false']")
  end
end
```

**优点**：`it` 块读起来像用户故事，细节隐藏在私有方法里。常量集中定义选择器，修改时只改一处。

---

## 8. SystemHelpers 常用工具

```ruby
# 等待直到成功（带重试）
try_until_success(timeout: 5) do
  expect(element["data-status"]).to eq("done")
end

# 等待动画停止再操作
wait_for_animation(element)
element.click

# 等待属性变为特定值
wait_for_attribute(element, :class, "active")

# 调整窗口大小（自动还原）
resize_window(width: 375, height: 812) do
  # 测试移动端布局
end

# 使用指定时区
using_browser_timezone("Asia/Tokyo") do
  # 测试时区相关逻辑
end

# 滚动到底部（测试无限加载等）
fake_scroll_down_long

# 暂停测试（调试用，不提交）
pause_test

# 获取元素背景色（CSS 测试）
color = get_rgb_color(element, "backgroundColor")

# 键盘平台修饰键（Mac: Meta, Linux/Win: Ctrl）
send_keys([PLATFORM_KEY_MODIFIER, "a"])   # Cmd+A / Ctrl+A
```

---

## 9. 典型场景速查

```ruby
# 访问页面
visit "/admin/plugins"
topic_page.visit_topic(topic)

# 点击
find(".btn-primary").click
topic_page.click_post_action_button(post, :flag)

# 填写表单
fill_in "search", with: "poll"
find("input[type='text']").set("hello")

# 等待元素出现（Capybara 自动等待）
expect(page).to have_css(".modal", wait: 5)

# 元素数量
expect(page).to have_css(".row", count: 3)

# 文本内容
expect(page).to have_content("Success")
expect(find(".title")).to have_text("My Topic")

# 不存在
expect(page).to have_no_css(".error-message")

# within 缩小范围
within("#post_2") { expect(page).to have_css(".accepted") }

# 当前 URL
expect(current_url).to end_with("/admin/config/about")
expect(current_url).to match("/t/.+/#{topic.id}")

# JS 执行
page.execute_script("document.getElementById('x').scrollIntoView()")
value = page.evaluate_script("window.getSelection().toString()")

# Playwright 直接操作（高级）
page.driver.with_playwright_page do |pw_page|
  pw_page.mouse.click(x, y, clickCount: 3)
end
```

---

## 10. Plugin System Spec 特殊点

Plugin 的 system spec 放在 `plugins/xxx/spec/system/` 下，使用方式与核心完全一致：

```ruby
# 启用 plugin 设置
before do
  SiteSetting.solved_enabled = true
  SiteSetting.allow_solved_on_all_topics = true
end

# 使用核心 Page Object（可直接复用）
let(:topic_page) { PageObjects::Pages::Topic.new }

# Plugin 自己的 Fabricator
fab!(:solver_post) { Fabricate(:solved_topic, topic:, answer_post: post, accepter:) }
```

Plugin 无需创建自己的 Page Object，尽量复用核心已有的。

---

## 快速参考

```ruby
# 结构骨架
describe "Feature", type: :system do
  fab!(:admin)
  fab!(:topic) { Fabricate(:topic, ...) }

  let(:page_obj) { PageObjects::Pages::MyPage.new }

  before do
    SiteSetting.xxx = true
    sign_in(admin)
  end

  it "does something" do
    page_obj.visit
    page_obj.do_action
    expect(page_obj).to have_result
  end

  context "when condition" do
    before { SiteSetting.yyy = false }
    it "behaves differently" { ... }
  end

  private

  def helper_step
    find(".selector").click
    expect(page).to have_css(".result")
  end
end
```
