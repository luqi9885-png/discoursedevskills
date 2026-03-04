# Plugin 22 — 插件 Serializer 扩展完整规范

> 来源：discourse-reactions、discourse-assign、discourse-solved、discourse-ai 等主流插件的典型模式

## 概述

插件扩展序列化器有两种方式：
- **`add_to_serializer`**：向已有 Serializer 追加字段，最常用
- **`prepend` module**：修改已有字段的逻辑，适合覆盖核心字段

Discourse 的 Serializer 基于 `ActiveModel::Serializer`，字段用 `attributes` 声明，`include_xxx?` 控制条件渲染。

---

## 核心规范

### 1. add_to_serializer — 追加字段（最常用）

```ruby
# plugin.rb — 始终放在 after_initialize 内
after_initialize do
  # 基本用法
  add_to_serializer(:post, :my_plugin_score) do
    object.custom_fields["my_plugin_score"].to_f
  end

  # 带条件守卫：include_<字段名>?
  add_to_serializer(:post, :my_plugin_data) do
    { score: object.custom_fields["my_plugin_score"].to_f }
  end

  add_to_serializer(:post, :include_my_plugin_data?) do
    SiteSetting.my_plugin_enabled? && object.custom_fields["my_plugin_score"].present?
  end

  # 访问 scope（当前用户 / Guardian）
  add_to_serializer(:topic_view, :my_plugin_user_status) do
    return nil unless scope.current_user
    MyPlugin::Item.find_by(
      topic_id: object.topic.id,
      user_id: scope.current_user.id,
    )&.status
  end
end
```

### 2. 序列化器名速查表

| 序列化器名 | `object` 类型 | 使用场景 |
|-----------|--------------|---------|
| `:post` | `Post` | 帖子详情 |
| `:topic_view` | `TopicView`（`object.topic` 才是 Topic） | 话题详情页 |
| `:topic_list_item` | `Topic` | 话题列表（首页/分类页） |
| `:basic_post` | `Post` | 精简帖子（搜索结果等） |
| `:current_user` | `User` | 当前登录用户（页面初始化） |
| `:user` | `User` | 用户资料页 |
| `:user_card` | `User` | 用户卡片弹窗 |
| `:site` | `Site` | 全局站点配置（页面初始化） |
| `:basic_category` | `Category` | 分类信息 |
| `:notification` | `Notification` | 通知列表 |

### 3. topic_view vs topic_list_item — 最常见混淆

```ruby
after_initialize do
  # topic_view：话题详情页，object 是 TopicView 实例
  add_to_serializer(:topic_view, :my_plugin_status) do
    object.topic.custom_fields["my_plugin_status"]  # 必须 object.topic
  end

  # topic_list_item：话题列表，object 直接是 Topic 实例
  add_to_serializer(:topic_list_item, :my_plugin_status) do
    object.custom_fields["my_plugin_status"]  # 直接 object
  end
end
```

### 4. 预加载 custom_fields（避免列表 N+1）

```ruby
after_initialize do
  # 注册 custom field 类型（让 Discourse 知道要预加载）
  register_post_custom_field_type("my_plugin_score", :integer)
  register_topic_custom_field_type("my_plugin_status", :string)

  # 预加载到话题列表（必须，否则列表每行都查一次 DB）
  TopicList.preloaded_custom_fields << "my_plugin_status" if TopicList.respond_to?(:preloaded_custom_fields)

  # 此后 topic_list_item 序列化器访问 custom_fields 不会触发额外查询
  add_to_serializer(:topic_list_item, :my_plugin_status) do
    object.custom_fields["my_plugin_status"]
  end

  add_to_serializer(:topic_list_item, :include_my_plugin_status?) do
    SiteSetting.my_plugin_enabled?
  end
end
```

### 5. SiteSerializer — 注入全局配置

`SiteSerializer` 在页面初始化时加载一次，适合注入插件级配置供前端 JS 全局使用：

```ruby
after_initialize do
  add_to_serializer(:site, :my_plugin_enabled) do
    SiteSetting.my_plugin_enabled?
  end

  add_to_serializer(:site, :my_plugin_options) do
    MyPlugin::Option.all.map { |o| { id: o.id, name: o.name } }
  end

  add_to_serializer(:site, :include_my_plugin_options?) do
    SiteSetting.my_plugin_enabled?
  end
end
```

### 6. CurrentUserSerializer — 当前用户状态

```ruby
after_initialize do
  add_to_serializer(:current_user, :my_plugin_onboarded) do
    object.custom_fields["my_plugin_onboarded"] == "true"
  end

  add_to_serializer(:current_user, :my_plugin_item_count) do
    MyPlugin::Item.where(user_id: object.id, active: true).count
  end

  add_to_serializer(:current_user, :include_my_plugin_item_count?) do
    SiteSetting.my_plugin_enabled?
  end
end
```

### 7. prepend module — 修改已有字段

```ruby
# lib/my_plugin/post_serializer_extension.rb
module MyPlugin
  module PostSerializerExtension
    # 覆盖核心已有字段
    def cooked
      result = super
      MyPlugin::HtmlProcessor.process(result, object)
    end

    # 新增字段（配合下面的 attributes 声明）
    def my_plugin_metadata
      object.custom_fields["my_plugin_metadata"]
    end
  end
end

# plugin.rb
after_initialize do
  PostSerializer.prepend(MyPlugin::PostSerializerExtension)
  PostSerializer.attributes(:my_plugin_metadata)  # 声明新字段
end
```

### 8. 完整示例 — 话题评分插件序列化层

```ruby
after_initialize do
  register_topic_custom_field_type("topic_score", :float)
  register_post_custom_field_type("post_vote_count", :integer)
  TopicList.preloaded_custom_fields << "topic_score" if TopicList.respond_to?(:preloaded_custom_fields)

  # 话题列表显示分数
  add_to_serializer(:topic_list_item, :topic_score) do
    object.custom_fields["topic_score"].to_f
  end
  add_to_serializer(:topic_list_item, :include_topic_score?) do
    SiteSetting.topic_scoring_enabled?
  end

  # 话题详情页显示投票状态
  add_to_serializer(:topic_view, :user_voted) do
    return false unless scope.current_user
    MyPlugin::Vote.exists?(topic_id: object.topic.id, user_id: scope.current_user.id)
  end
  add_to_serializer(:topic_view, :include_user_voted?) do
    SiteSetting.topic_scoring_enabled? && scope.current_user.present?
  end

  # 帖子级投票数
  add_to_serializer(:post, :post_vote_count) do
    object.custom_fields["post_vote_count"].to_i
  end
  add_to_serializer(:post, :include_post_vote_count?) do
    SiteSetting.topic_scoring_enabled?
  end

  # 全局开关注入前端
  add_to_serializer(:site, :topic_scoring_enabled) do
    SiteSetting.topic_scoring_enabled?
  end

  # 当前用户总投票数
  add_to_serializer(:current_user, :total_votes_cast) do
    MyPlugin::Vote.where(user_id: object.id).count
  end
  add_to_serializer(:current_user, :include_total_votes_cast?) do
    SiteSetting.topic_scoring_enabled?
  end
end
```

---

## 反模式

```ruby
# 错误：topic_view 中直接用 object 访问 Topic 字段
add_to_serializer(:topic_view, :my_data) do
  object.custom_fields["my_data"]  # object 是 TopicView，不是 Topic！
end
# 正确
add_to_serializer(:topic_view, :my_data) { object.topic.custom_fields["my_data"] }

# 错误：没有 include_xxx? 守卫时访问可能为 nil 的 scope.current_user
add_to_serializer(:post, :user_reaction) do
  MyPlugin::Reaction.find_by(post_id: object.id, user_id: scope.current_user.id)
  # 匿名用户 scope.current_user 为 nil，抛 NoMethodError！
end
# 正确：用 include_xxx? 提前过滤
add_to_serializer(:post, :include_user_reaction?) do
  scope.current_user.present? && SiteSetting.my_plugin_enabled?
end

# 错误：列表序列化器中做 N+1 查询
add_to_serializer(:topic_list_item, :reaction_count) do
  MyPlugin::Reaction.where(topic_id: object.id).count  # 每个 topic 都查一次！
end
# 正确：预加载到 custom_fields 或用 TopicList preload 机制

# 错误：add_to_serializer 放在 after_initialize 外
add_to_serializer(:post, :my_data) { object.id }  # Serializer 可能尚未加载
# 正确：始终在 after_initialize 内
after_initialize { add_to_serializer(:post, :my_data) { object.id } }
```

---

## 快速参考

```ruby
after_initialize do
  # 追加字段
  add_to_serializer(:post, :field_name) { expression }
  add_to_serializer(:post, :include_field_name?) { condition }

  # scope 用法
  scope.current_user      # User 或 nil（匿名）
  scope.is_admin?
  scope.can_see?(object)

  # object 类型速记
  # :topic_view       → object.topic（才是 Topic）
  # :topic_list_item  → object（直接是 Topic）
  # :post             → object（直接是 Post）
  # :current_user     → object（直接是 User）

  # 预加载
  register_post_custom_field_type("field", :integer)
  register_topic_custom_field_type("field", :string)
  TopicList.preloaded_custom_fields << "field"
end
```
