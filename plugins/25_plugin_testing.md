# plugins/25 — 插件 RSpec 测试规范

> 来源：
> - `plugins/discourse-assign/spec/`（完整目录结构）
> - `plugins/discourse-reactions/spec/`（独立插件对比验证）
> 修订历史：
> - 2026-03-04 初版

## 边界声明

- **本文件负责**：插件 spec 目录结构、Fabricator 定义、request spec、model spec、job spec、shared_context 的插件专属用法
- **不负责，见其他文件**：
  - 核心 RSpec 模式（`fab!`、`let`、`before`）→ `ruby/05_rspec_testing.md`
  - System spec / Page Objects → `system_specs/01_page_objects.md`
- **与以下文件有交叉，已在 TOPIC_MAP 协调**：
  - `ruby/05_rspec_testing.md`（本文只写插件特有的组织方式和 Fabricator 定义，`fab!` 基础用法不重复）

---

## 概述

插件测试与核心代码测试的主要区别在于：需要在 `before` 中开启插件的 `SiteSetting`、Fabricator 定义在插件自己的 `spec/fabricators/` 目录、共享测试上下文用 `spec/support/` 中的 `shared_context`。

---

## 来源验证

| 验证点 | 文件路径 | 关键行 / 方法名 | 与其他验证点的差异 |
|--------|---------|---------------|-----------------|
| Fabricator 定义（带关联） | `discourse-assign/spec/fabricators/assignment_fabricator.rb` | `transient :post`、`target { \|attrs\| }` | 带 transient 属性和动态关联 |
| Fabricator 定义（简单） | `discourse-reactions/spec/fabricators/reaction_fabricator.rb` | `Fabricator(:reaction, class_name: "DiscourseReactions::Reaction")` | 明确指定 class_name，避免命名空间歧义 |
| Request spec 结构 | `discourse-assign/spec/requests/assign_controller_spec.rb` | `fab!` + `sign_in` + `put/get` + `response.status` | 含 `require_relative "../support/..."` 引入 shared_context |
| Request spec（reactions） | `discourse-reactions/spec/requests/custom_reactions_controller_spec.rb` | `fab!` + `before` SiteSetting + `response.parsed_body` | 大量 fab! 预建数据，before 中设多个 SiteSetting |
| shared_context | `discourse-assign/spec/support/assign_allowed_group.rb` | `shared_context "with group..."` + `fab!` + helper 方法 | 用 `include_context` 在多个 spec 中复用 |
| Model spec | `discourse-assign/spec/models/assignment_spec.rb` | `described_class`、`subject`、`contain_exactly` | 用 `described_class` 而非硬编码类名 |
| Job spec | `discourse-assign/spec/jobs/regular/assign_notification_spec.rb` | `stub/stubs/expects`、`raise_error` | Mocha stub 风格，不用 RSpec double |

---

## 核心规范

### 1. 插件 spec 目录结构

```
my-plugin/
└── spec/
    ├── fabricators/               # Fabricator 定义（插件自己的 factory）
    │   └── my_item_fabricator.rb
    ├── models/                    # Model spec
    │   └── my_item_spec.rb
    ├── requests/                  # Controller / API spec（integration）
    │   └── my_items_controller_spec.rb
    ├── jobs/
    │   ├── regular/               # Jobs::Base 子类
    │   └── scheduled/             # Jobs::Scheduled 子类
    ├── serializers/               # Serializer spec
    ├── services/                  # Service spec
    ├── lib/                       # lib/ 下的类 spec
    ├── support/                   # shared_context、helper 方法
    │   └── my_plugin_context.rb
    ├── system/                    # System spec（Capybara）
    └── plugin_spec.rb             # 插件顶层集成测试
```

### 2. Fabricator 定义

插件的 Fabricator 放在 `spec/fabricators/`，Discourse 会自动加载。关键规则：

```ruby
# spec/fabricators/assignment_fabricator.rb
# 来自 discourse-assign 真实代码

# 基本关联：target 动态从 attrs 取
Fabricator(:topic_assignment, class_name: :assignment) do
  topic                                          # 自动 Fabricate(:topic)
  target { |attrs| attrs[:topic] }              # 引用同一个 topic（不新建）
  target_type "Topic"
  assigned_by_user { Fabricate(:user) }
  assigned_to { Fabricate(:user) }
end

# transient 属性：不映射到 AR 字段，只用于内部逻辑
Fabricator(:post_assignment, class_name: :assignment) do
  transient :post                                # 接收 post: xxx 参数但不存到 DB
  topic { |attrs| attrs[:post]&.topic || Fabricate(:topic) }
  target { |attrs| attrs[:post] || Fabricate(:post, topic: attrs[:topic]) }
  target_type "Post"
  assigned_by_user { Fabricate(:user) }
  assigned_to { Fabricate(:user) }
end

# 明确指定 class_name（有命名空间时必须）
# 来自 discourse-reactions 真实代码
Fabricator(:reaction, class_name: "DiscourseReactions::Reaction") do
  post { |attrs| attrs[:post] }  # 依赖调用方传入 post:
  reaction_type 0
  reaction_value "otter"
end

# Notification 扩展：继承已有 Fabricator
Fabricator(:assignment_notification, from: :notification) do
  transient :group
  notification_type Notification.types[:assigned]
  post_number 1
  data do |attrs|
    {
      message: attrs[:group] ? "assign_group_notification" : "assign_notification",
      display_username: attrs[:group] ? "group" : "user",
      assignment_id: rand(1..100),
    }.to_json                    # data 字段存 JSON 字符串
  end
end
```

**Fabricator 命名规则：**
- 名称用 snake_case，与 `Fabricate(:my_item)` 调用对应
- 有命名空间的类必须指定 `class_name: "MyPlugin::MyItem"`（字符串）或 `class_name: :my_item`（推断）
- 用 `from: :base_fabricator` 继承并覆盖字段，避免重复定义

### 3. Request Spec（Controller / API 测试）

```ruby
# spec/requests/my_items_controller_spec.rb
# 来自 discourse-assign/spec/requests/assign_controller_spec.rb 真实代码

require_relative "../support/my_plugin_context"  # 引入 shared_context

RSpec.describe MyPlugin::ItemsController do
  # 1. before 中开启插件 SiteSetting（每个 spec 文件都要）
  before do
    SiteSetting.my_plugin_enabled = true
    SiteSetting.my_plugin_allowed_groups = "#{allowed_group.id}"
  end

  # 2. fab! 预建持久化数据（整个 describe 共享，只建一次）
  fab!(:allowed_group, :group)
  fab!(:admin)
  fab!(:allowed_user) { Fabricate(:user, groups: [allowed_group]) }
  fab!(:topic)
  fab!(:post)

  describe "权限控制" do
    it "未登录用户返回 403" do
      post "/my-plugin/topics/#{topic.id}/items.json",
           params: { data: "test" }
      expect(response.status).to eq(403)
    end

    it "无权限用户返回 403" do
      sign_in(Fabricate(:user))  # 非 allowed_user
      post "/my-plugin/topics/#{topic.id}/items.json",
           params: { data: "test" }
      expect(response.status).to eq(403)
    end

    it "有权限用户可以创建" do
      sign_in(allowed_user)
      expect do
        post "/my-plugin/topics/#{topic.id}/items.json",
             params: { data: "test" }
      end.to change { MyPlugin::Item.count }.by(1)

      expect(response.status).to eq(200)
      # 解析 JSON 响应
      body = response.parsed_body
      expect(body["item"]["data"]).to eq("test")
    end
  end

  describe "数据验证" do
    before { sign_in(allowed_user) }

    it "缺少必填参数返回 400" do
      post "/my-plugin/topics/#{topic.id}/items.json", params: {}
      expect(response.status).to eq(400)
    end
  end
end
```

**关键 matchers：**

```ruby
expect(response.status).to eq(200)
expect(response.parsed_body["key"]).to eq("value")   # 自动解析 JSON
expect(response.parsed_body["items"].length).to eq(3)

# 检查数据库变化
expect { action }.to change { Model.count }.by(1)
expect { action }.to change { Model.count }.by(1).and change { OtherModel.count }.by(1)
expect { action }.not_to change { Model.count }

# 集合匹配（不关心顺序）
expect(results).to contain_exactly(item1, item2)
expect(ids).to match_array([1, 2, 3])
```

### 4. Model Spec

```ruby
# spec/models/assignment_spec.rb
# 来自 discourse-assign 真实代码

RSpec.describe Assignment do
  before { SiteSetting.assign_enabled = true }

  describe ".active_for_group" do
    # subject 封装被测方法调用
    subject(:assignments) { described_class.active_for_group(group) }

    # let! 立即求值（bang！），适合需要在 before 前存在的数据
    let!(:group) { Fabricate(:group) }
    let!(:assignment1) { Fabricate(:topic_assignment, assigned_to: group) }
    let!(:assignment2) { Fabricate(:post_assignment, assigned_to: group) }

    before do
      # 建立干扰数据，验证过滤逻辑
      Fabricate(:post_assignment, assigned_to: group, active: false)
      Fabricate(:post_assignment, assigned_to: Fabricate(:user))
    end

    it "只返回该 group 的 active assignments" do
      expect(assignments).to contain_exactly(assignment1, assignment2)
    end
  end

  describe "#assigned_users" do
    # described_class 自动引用 RSpec.describe 的类，重构时无需修改
    let(:assignment) { Fabricate(:topic_assignment) }

    it "returns assigned user" do
      expect(assignment.assigned_users).to include(assignment.assigned_to)
    end
  end
end
```

### 5. Job Spec

```ruby
# spec/jobs/regular/my_job_spec.rb
# 来自 discourse-assign/spec/jobs/regular/assign_notification_spec.rb 真实代码

RSpec.describe Jobs::MyPluginJob do
  describe "#execute" do
    subject(:execute_job) { described_class.new.execute(args) }

    # 缺少必填参数时报错
    context "when required param is missing" do
      let(:args) { {} }

      it "raises InvalidParameters" do
        expect { execute_job }.to raise_error(Discourse::InvalidParameters, "item_id")
      end
    end

    context "when params are valid" do
      let(:item) { Fabricate(:my_plugin_item) }
      let(:args) { { item_id: item.id } }

      # Mocha stub 风格（Discourse 使用 Mocha，不用 RSpec double）
      before { MyPlugin::Item.stubs(:find).with(item.id).returns(item) }

      it "executes main logic" do
        item.expects(:process!)    # 断言 process! 被调用一次
        execute_job
      end

      it "skips if disabled" do
        SiteSetting.my_plugin_enabled = false
        item.expects(:process!).never
        execute_job
      end
    end
  end
end
```

**Mocha vs RSpec double（Discourse 统一用 Mocha）：**

```ruby
# Mocha stub（允许任意次调用，返回固定值）
MyModel.stubs(:find).with(id).returns(record)
object.stubs(:method_name).returns(value)

# Mocha expect（断言调用次数）
object.expects(:method_name)           # 期望调用 1 次
object.expects(:method_name).never     # 期望不被调用
object.expects(:method_name).twice     # 期望调用 2 次
object.expects(:method_name).with(arg) # 期望以特定参数调用
```

### 6. shared_context — 跨 spec 复用

```ruby
# spec/support/my_plugin_context.rb
# 来自 discourse-assign/spec/support/assign_allowed_group.rb 真实代码

shared_context "with my plugin allowed group" do
  # fab! 在 shared_context 中也可用
  fab!(:my_plugin_allowed_group) do
    Fabricate(:group, assignable_level: Group::ALIAS_LEVELS[:everyone])
  end

  before do
    SiteSetting.my_plugin_allowed_groups += "|#{my_plugin_allowed_group.id}"
  end

  # Helper 方法：在 it 块中调用
  def add_to_allowed_group(user)
    my_plugin_allowed_group.add(user)
  end

  def allowed_group_name
    my_plugin_allowed_group.name
  end
end

# 在 spec 中使用
RSpec.describe MyPlugin::ItemsController do
  include_context "with my plugin allowed group"

  it "allows group members" do
    add_to_allowed_group(user)
    sign_in(user)
    get "/my-plugin/items.json"
    expect(response.status).to eq(200)
  end
end
```

### 7. plugin_spec.rb — 顶层集成测试

```ruby
# spec/plugin_spec.rb
# 来自 discourse-assign/spec/plugin_spec.rb 真实代码

RSpec.describe DiscourseAssign do
  before { SiteSetting.assign_enabled = true }

  # 测试 register_modifier 效果
  describe "topics_filter_options modifier" do
    let(:user) { Fabricate(:user) }

    before do
      SiteSetting.assign_allowed_on_groups = Group::AUTO_GROUPS[:staff]
      user.update!(admin: true)
    end

    it "有权限的用户可以看到 assigned 过滤选项" do
      options = TopicsFilter.option_info(user.guardian)
      assigned_option = options.find { |o| o[:name] == "assigned:" }
      expect(assigned_option).to be_present
      expect(assigned_option).to include(type: "username_group_list")
    end

    it "无权限的用户看不到 assigned 过滤选项" do
      regular_user = Fabricate(:user)
      options = TopicsFilter.option_info(regular_user.guardian)
      expect(options.find { |o| o[:name] == "assigned:" }).to be_nil
    end
  end
end
```

---

## 反模式

```ruby
# 错误：before 中没有开启插件 SiteSetting，测试行为与生产不一致
RSpec.describe MyPlugin::ItemsController do
  fab!(:user)
  it "creates item" do
    sign_in(user)
    post "/my-plugin/items.json", params: { data: "x" }
    # 插件默认禁用，实际上返回 404，但测试者以为是业务逻辑问题
  end
end
# 正确：每个 spec 文件的 before 中开启 SiteSetting
before { SiteSetting.my_plugin_enabled = true }

# 错误：Fabricator 没有指定 class_name，命名空间类找不到
Fabricator(:reaction) do   # 找 Reaction 而非 DiscourseReactions::Reaction
  reaction_value "heart"
end
# 正确
Fabricator(:reaction, class_name: "DiscourseReactions::Reaction") do
  reaction_value "heart"
end

# 错误：用 let 代替 let! 导致数据未被建立（let 是懒加载）
let(:assignment) { Fabricate(:topic_assignment, assigned_to: group) }
before { Fabricate(:post_assignment, assigned_to: group, active: false) }
# assignment 可能在 before 之后才建立，干扰数据已存在但基准数据还没有
# 正确：需要在 before 前存在的数据用 let!
let!(:assignment) { Fabricate(:topic_assignment, assigned_to: group) }

# 错误：用 RSpec double 代替 Mocha stub（Discourse 用 Mocha）
allow(MyModel).to receive(:find).and_return(record)  # RSpec 风格，与 Mocha 混用会出问题
# 正确：统一用 Mocha
MyModel.stubs(:find).with(id).returns(record)

# 错误：require_relative 路径错误（spec/ 内的相对路径要小心）
require_relative "support/my_context"     # 从当前文件位置算，可能找不到
# 正确（从当前文件目录算）
require_relative "../support/my_context"  # requests/ 下的 spec 需要往上一级
```

---

## 快速参考

```ruby
# spec/fabricators/my_item_fabricator.rb
Fabricator(:my_item, class_name: "MyPlugin::Item") do
  user
  topic
  data "default value"
end

Fabricator(:my_item_with_post, class_name: "MyPlugin::Item") do
  transient :post
  user
  topic { |attrs| attrs[:post]&.topic || Fabricate(:topic) }
  post  { |attrs| attrs[:post] || Fabricate(:post, topic: attrs[:topic]) }
end

# spec/requests/my_items_controller_spec.rb
RSpec.describe MyPlugin::ItemsController do
  before { SiteSetting.my_plugin_enabled = true }   # 必须
  fab!(:user)
  fab!(:topic)

  it "creates" do
    sign_in(user)
    expect { post "/my-plugin/items.json", params: { topic_id: topic.id } }
      .to change { MyPlugin::Item.count }.by(1)
    expect(response.status).to eq(200)
    expect(response.parsed_body["item"]["topic_id"]).to eq(topic.id)
  end
end

# spec/models/my_item_spec.rb
RSpec.describe MyPlugin::Item do
  before { SiteSetting.my_plugin_enabled = true }
  subject(:item) { described_class.create!(user:, topic:) }
  let!(:user)  { Fabricate(:user) }
  let!(:topic) { Fabricate(:topic) }

  it "does something" do
    expect(item).to be_valid
  end
end

# spec/jobs/regular/my_job_spec.rb
RSpec.describe Jobs::MyPluginJob do
  describe "#execute" do
    subject(:execute) { described_class.new.execute(args) }
    let(:args) { { item_id: Fabricate(:my_item).id } }
    before { SiteSetting.my_plugin_enabled = true }

    it "runs" do
      expect { execute }.not_to raise_error
    end
  end
end
```
