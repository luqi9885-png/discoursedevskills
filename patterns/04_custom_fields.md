# Patterns 04 — Custom Fields

> 来源：`app/models/concerns/has_custom_fields.rb`、`lib/plugin/instance.rb`、
> `plugins/discourse-solved/plugin.rb`、
> `connectors/category-custom-settings/solved-settings.gjs`

---

## 概述

Custom Fields 是 Discourse 给核心模型（Topic、Post、Category、User、Group）附加任意键值对的机制。Plugin 注册字段类型 → 后端读写 → 前端通过 connector 暴露 UI 或 serializer 输出。

**支持类型**：`:string`（默认）、`:integer`、`:boolean`、`:json`

---

## 1. 后端注册（plugin.rb）

```ruby
# 在 after_initialize 外（全局注册，多租户环境下所有 site 生效）
register_topic_custom_field_type("my_field", :string)
register_topic_custom_field_type("my_count", :integer)
register_topic_custom_field_type("my_flag", :boolean)
register_topic_custom_field_type("my_data", :json)

register_post_custom_field_type("my_post_field", :string)
register_category_custom_field_type("enable_feature", :string)
register_user_custom_field_type("user_setting", :boolean)
register_group_custom_field_type("group_config", :json)
```

**注意**：`register_xxx_custom_field_type` 放在 `after_initialize` **外面**，因为它通过 `reloadable_patch` 注册到 Class 级别的元数据，不依赖 DB。

---

## 2. 后端读写

### 读

```ruby
topic = Topic.find(id)

# 直接读（会触发 DB 查询）
value = topic.custom_fields["my_field"]  # 字符串 key 或 Symbol 均可

# 预加载（N+1 防护，批量时使用）
Topic.preload_custom_fields(topics, ["my_field", "other_field"])
topics.each { |t| t.custom_fields["my_field"] }  # 不再 N+1
```

### 写（方式一：修改 hash + save）

```ruby
topic.custom_fields["my_field"] = "new_value"
topic.save!  # after_save 钩子自动调用 save_custom_fields
```

### 写（方式二：upsert，并发安全）

```ruby
# 只更新/插入，不删除其他字段，并发更安全
topic.upsert_custom_fields("my_field" => "value", "my_count" => 42)
```

### 删除

```ruby
topic.custom_fields.delete("my_field")
topic.save_custom_fields(true)  # force=true 强制保存
```

---

## 3. 类型序列化行为

```ruby
# 注册为 :boolean
register_topic_custom_field_type("enabled", :boolean)

topic.custom_fields["enabled"] = true
topic.save!
# DB 存储：字符串 "t"
# 读取时自动反序列化为 true（Ruby boolean）

# 注册为 :integer
register_topic_custom_field_type("count", :integer)
topic.custom_fields["count"] = "5"   # 传字符串也行
topic.save!
# 读取时自动变为整数 5

# 注册为 :json
register_topic_custom_field_type("metadata", :json)
topic.custom_fields["metadata"] = { key: "value", list: [1, 2] }
topic.save!
# DB 存储：JSON 字符串
# 读取时自动 parse 为 Hash/Array
```

---

## 4. Category Custom Fields 完整模式

Category 是最常见的 custom field 场景（per-category 功能开关）。

### 4.1 后端注册 + 预加载

```ruby
# plugin.rb
ENABLE_ANSWERS = "enable_accepted_answers"

# 注册类型（after_initialize 外）
register_category_custom_field_type(ENABLE_ANSWERS, :string)

after_initialize do
  # 让 Site.categories 端点预加载这个字段（前端可见）
  Site.preloaded_category_custom_fields << ENABLE_ANSWERS
end
```

### 4.2 后端读取

```ruby
def allow_accepted_answers?(category_id)
  category = Category.find_by(id: category_id)
  return false unless category
  category.custom_fields[ENABLE_ANSWERS] == "true" ||
    SiteSetting.allow_solved_on_all_topics
end
```

### 4.3 前端 UI（category-custom-settings connector）

```js
// connectors/category-custom-settings/solved-settings.gjs
import Component from "@ember/component";
import { action } from "@ember/object";

export default class SolvedSettings extends Component {
  @action
  onChangeSetting(value) {
    // category.custom_fields 是前端可读写的对象
    this.set(
      "category.custom_fields.enable_accepted_answers",
      value ? "true" : "false"
    );
  }

  <template>
    <section class="field">
      <input
        {{on "change" (withEventValue this.onChangeSetting "target.checked")}}
        checked={{this.category.enable_accepted_answers}}
        type="checkbox"
      />
    </section>

    {{! 数字字段直接绑定 mut }}
    <input
      {{on "input" (withEventValue (fn (mut this.category.custom_fields.auto_close_hours)))}}
      value={{this.category.custom_fields.auto_close_hours}}
      type="number"
    />
  </template>
}
```

前端 category 对象的 `custom_fields` 对应后端存储，保存分类时自动持久化。

---

## 5. Topic Custom Fields

### 5.1 注册 + Serializer 输出

```ruby
# plugin.rb
register_topic_custom_field_type("my_feature_data", :json)

after_initialize do
  # 让 TopicView 预加载（topic page 可用）
  TopicView.add_post_custom_fields_allowlister { |user, topic| ["my_feature_data"] }

  # 通过 add_to_serializer 输出给前端
  add_to_serializer(:topic_view, :my_feature_data) do
    object.topic.custom_fields["my_feature_data"]
  end
end
```

### 5.2 批量预加载（TopicList）

```ruby
# plugin.rb
after_initialize do
  # 在 topic list 中预加载
  if TopicList.respond_to?(:preloaded_custom_fields)
    TopicList.preloaded_custom_fields << "my_field"
  end
end
```

---

## 6. User Custom Fields

```ruby
# plugin.rb
register_user_custom_field_type("my_user_pref", :boolean)

after_initialize do
  # 让用户可以编辑（需要前端配合）
  DiscoursePluginRegistry.register_editable_user_custom_field("my_user_pref", plugin: self)

  add_to_serializer(:user, :my_user_pref) do
    object.custom_fields["my_user_pref"]
  end
end
```

---

## 7. 常见模式汇总

```ruby
# 读取（注意判空）
value = topic.custom_fields["my_field"].to_s.presence
count = topic.custom_fields["my_count"].to_i
flag  = topic.custom_fields["my_flag"]  # :boolean 类型返回 true/false

# 写入（attach 方式，最安全）
topic.upsert_custom_fields("my_field" => "value")

# 写入（修改后 save，适合已加载的对象）
topic.custom_fields["my_field"] = "value"
topic.save_custom_fields(true)   # force=true 不管 clean 状态都写

# 让 category 字段在 API 中可见
Site.preloaded_category_custom_fields << "my_field"

# 让 topic 字段在 topic page serializer 中可见
add_to_serializer(:topic_view, :my_field) { object.topic.custom_fields["my_field"] }
```

---

## 快速参考表

| 场景 | 注册方法 | 读写对象 | 预加载 |
|------|----------|----------|--------|
| 分类开关 | `register_category_custom_field_type` | `category.custom_fields[]` | `Site.preloaded_category_custom_fields << field` |
| 话题数据 | `register_topic_custom_field_type` | `topic.custom_fields[]` | `TopicList.preloaded_custom_fields << field` |
| 帖子数据 | `register_post_custom_field_type` | `post.custom_fields[]` | `TopicView.add_post_custom_fields_allowlister` |
| 用户偏好 | `register_user_custom_field_type` | `user.custom_fields[]` | `add_to_serializer(:user, :field)` |
| Group 配置 | `register_group_custom_field_type` | `group.custom_fields[]` | `add_to_serializer(:basic_group, :field)` |
