# RSpec + fab! 测试规范

## 概述
Discourse 使用 RSpec + Fabrication gem 进行测试。`fab!` 宏是核心测试数据创建方式，与 `let` 不同，它在 `before(:context)` 而非每个 example 中创建数据，显著提升套件速度。

## 来源验证

- `spec/models/topic_spec.rb` — Model spec 完整示例
- `spec/models/post_spec.rb` — Model spec + shared examples
- `spec/requests/topics_controller_spec.rb` — Request spec 完整示例
- `spec/fabricators/user_fabricator.rb` — Fabricator 定义示例
- `spec/fabricators/topic_fabricator.rb` — Fabricator 继承示例
- `spec/support/shared_examples/custom_fields.rb` — shared_examples 示例

---

## 核心规范

### 1. fab! vs let vs let!

```ruby
RSpec.describe Topic do
  # ✅ fab! — 在 before(:context) 创建（只创建一次，跨 examples 共享）
  # 适合：不会被修改的背景数据（user、category 等）
  fab!(:user)
  fab!(:admin)
  fab!(:topic)

  # ✅ fab! with block — 自定义 fabricator 参数
  fab!(:user) { Fabricate(:user, refresh_auto_groups: true) }
  fab!(:staff_category) do
    Fabricate(:category).tap do |c|
      c.set_permissions(staff: :full)
      c.save!
    end
  end

  # ✅ fab! shorthand — 第二参数是 fabricator 名
  fab!(:dest_topic, :topic)       # 等价于 fab!(:dest_topic) { Fabricate(:topic) }
  fab!(:pm, :private_message_topic)

  # ✅ let — 每个 example 重新计算（懒加载）
  let(:now) { Time.zone.local(2013, 11, 20, 8, 0) }

  # ✅ let! — 每个 example 都执行（非懒加载）
  let!(:something_that_must_exist) { create(:thing) }
end
```

来源：`spec/models/topic_spec.rb` 顶部

### 2. Fabricator 定义规范

```ruby
# spec/fabricators/user_fabricator.rb

# 基础 Fabricator
Fabricator(:user, class_name: :user) do
  transient refresh_auto_groups: false   # transient: 不写入 DB，只用于回调逻辑
  transient trust_level: nil

  name "Bruce Wayne"
  username { sequence(:username) { |i| "bruce#{i}" } }  # sequence 保证唯一性
  email    { sequence(:email)    { |i| "bruce#{i}@wayne.com" } }
  password "myawesomepassword"
  active true

  after_build  { |user, t| user.trust_level = t[:trust_level] || TrustLevel[1] }
  after_create { |user, t| Group.user_trust_level_change!(user.id, user.trust_level) if t[:trust_level] }
end

# 继承 Fabricator（from:）
Fabricator(:admin, from: :user) do
  admin true
  trust_level TrustLevel[4]
  after_create do |user|
    user.group_users << Fabricate(:group_user, user: user, group: Group[:admins])
  end
end

# 关联对象
Fabricator(:post) do
  user  { Fabricate(:user, refresh_auto_groups: true) }
  topic { |attrs| Fabricate(:topic, user: attrs[:user]) }  # 用 attrs 引用兄弟字段
  raw   "Hello world"
end
```

关键规则：
- **sequence** 用于需要唯一性的字段（username、email）
- **transient** 用于控制 after_create 逻辑但不存入 DB 的参数
- **from:** 用于继承已有 fabricator 并覆盖部分字段
- **after_create block** 中 `|obj, transients|` 第二参数是 transient 值

来源：`spec/fabricators/user_fabricator.rb`, `spec/fabricators/post_fabricator.rb`

### 3. Request Spec 结构

```ruby
RSpec.describe TopicsController do
  # 顶层 fab! 提供通用背景数据
  fab!(:topic)
  fab!(:user) { Fabricate(:user, refresh_auto_groups: true) }
  fab!(:moderator)
  fab!(:admin)

  before { SiteSetting.personal_message_enabled_groups = Group::AUTO_GROUPS[:everyone] }

  describe "#move_posts" do
    # 嵌套 fab! 仅在本 describe 块内有效
    fab!(:p1) { Fabricate(:post, user: moderator) }
    fab!(:p2) { Fabricate(:post, topic: p1.topic, user: moderator) }

    context "when not logged in" do
      it "returns 403" do
        post "/t/111/move-posts.json", params: { title: "blah", post_ids: [1, 2, 3] }
        expect(response.status).to eq(403)
      end
    end

    context "when logged in as moderator" do
      before { sign_in(moderator) }

      it "moves posts successfully" do
        post "/t/#{p1.topic.id}/move-posts.json",
             params: { title: "new topic", post_ids: [p2.id] }
        expect(response.status).to eq(200)
        expect(response.parsed_body["success"]).to eq("OK")
      end
    end
  end
end
```

关键规则：
- HTTP 方法 + 路径 + `params:` 发请求
- 用 `sign_in(user)` 登录，放在 `before` 块
- 用 `response.status`、`response.parsed_body`（自动解析 JSON）断言

来源：`spec/requests/topics_controller_spec.rb`

### 4. Model Spec 结构

```ruby
RSpec.describe Post do
  fab!(:coding_horror) { Fabricate(:coding_horror, refresh_auto_groups: true) }

  before { Oneboxer.stubs :onebox }   # stub 外部依赖

  it_behaves_like "it has custom fields"   # 复用 shared examples

  it { is_expected.to have_many(:reviewables).dependent(:destroy) }   # 单行 matcher

  describe "#types" do
    context "when verifying enum sequence" do
      it "'regular' should be at 1st position" do
        expect(described_class.types[:regular]).to eq(1)
      end
    end
  end
end
```

### 5. shared_examples 用法

```ruby
# spec/support/shared_examples/custom_fields.rb
RSpec.shared_examples_for "it has custom fields" do
  let(:record) { described_class.new }

  it "can't have more than 100 custom fields" do
    record.custom_fields = (1..101).to_a.product(["value"]).to_h
    expect(record).to be_invalid
  end
end

# 在任意 spec 中复用：
RSpec.describe Topic do
  it_behaves_like "it has custom fields"
end

RSpec.describe Post do
  it_behaves_like "it has custom fields"
end
```

来源：`spec/support/shared_examples/custom_fields.rb`，被 `topic_spec.rb` 和 `post_spec.rb` 引用

### 6. 分层 describe / context 规范

```
describe "方法名或功能区块"
  context "when/given 某个条件"
    it "does something specific"
```

- `describe` — 描述方法或功能块（用 `#method_name` 格式描述实例方法）
- `context` — 描述条件（以 "when" 或 "given" 开头）
- `it` — 具体行为，陈述句

## 反模式（避免这样做）

```ruby
# ❌ 在 it 块内直接 Fabricate 大量数据（慢，重复）
it "works" do
  user = Fabricate(:user)
  topic = Fabricate(:topic, user: user)
  # ...
end

# ✅ 用 fab! 提升速度
fab!(:user)
fab!(:topic) { Fabricate(:topic, user: user) }

# ❌ 用 let 创建需要写 DB 的对象（懒加载可能导致顺序问题）
let(:user) { Fabricate(:user) }

# ✅ 用 fab! 或 let!
fab!(:user)
```

## 关联规范

- `ruby/02_models_concerns.md` — Concern 行为可用 shared_examples 测试
- `system_specs/01_page_objects.md` — E2E 测试另见 Page Objects
