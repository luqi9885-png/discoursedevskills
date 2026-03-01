# Model Concerns

## 概述
通过 ActiveSupport::Concern 将 Model 的通用行为拆分为独立模块，保持 Model 干净，实现行为复用。

## 来源验证

- `app/models/concerns/trashable.rb` → 被 `Post`, `Topic`, `Invite`, `PostAction` 使用
- `app/models/concerns/positionable.rb` → 被 `Category` 使用
- `app/models/concerns/has_custom_fields.rb` → 被 `User`, `Topic`, `Post`, `Group`, `Category` 使用
- `app/models/concerns/roleable.rb` → 被 `User` 使用
- `app/models/concerns/searchable.rb` → 被 `User`, `Topic`, `Post`, `Category`, `Tag` 使用
- `app/models/concerns/has_sanitizable_fields.rb` → 被 `Tag`, `Badge`, `UserField`, `TranslationOverride` 使用

## 核心规范

### 1. 标准 Concern 骨架

每个 Concern 必须遵守固定结构：

```ruby
# frozen_string_literal: true

module Trashable
  extend ActiveSupport::Concern

  included do
    # 在这里写 has_many / belongs_to / scope / validate / callback
    default_scope { where(deleted_at: nil) }
    scope :with_deleted, -> { unscope(where: :deleted_at) }
    belongs_to :deleted_by, class_name: "User"
  end

  # 实例方法
  def trashed?
    deleted_at.present?
  end

  def trash!(trashed_by = nil)
    trash_update(DateTime.now, trashed_by.try(:id))
  end

  # 私有方法
  private

  def trash_update(deleted_at, deleted_by_id)
    self.update_columns(deleted_at: deleted_at, deleted_by_id: deleted_by_id)
  end
end
```

关键点：
- `extend ActiveSupport::Concern` — 必须
- `included do ... end` — 所有 ActiveRecord 宏（关联、scope、callback、验证）放这里
- 实例方法直接定义在模块顶层
- 私有方法用 `private`

### 2. class_methods 块

如需类方法，用 `class_methods do ... end`（由 ActiveSupport::Concern 提供）：

```ruby
module HasCustomFields
  extend ActiveSupport::Concern

  class_methods do
    def custom_fields_for_ids(ids, allowed_fields)
      # ...
    end

    def preload_custom_fields(objects, fields)
      # ...
    end
  end

  included do
    has_many :_custom_fields, dependent: :destroy, class_name: "#{name}CustomField"
    after_save { save_custom_fields(run_validations: false) }
  end
end
```

来源：`app/models/concerns/has_custom_fields.rb`

### 3. Concern 命名规范

| 命名模式 | 含义 | 示例 |
|---------|------|------|
| `Has + 名词` | 提供某类能力 | `HasCustomFields`, `HasSearchData` |
| `形容词able` | 状态/行为特征 | `Trashable`, `Searchable`, `Positionable` |
| `动词able` | 可被某种操作 | `Roleable` |
| `描述名词` | 功能描述 | `CachedCounting`, `SecondFactorManager` |

### 4. 在 Model 中 include

```ruby
class Topic < ActiveRecord::Base
  include HasCustomFields
  include Searchable
  include Trashable
  # ...
end
```

来源：`app/models/topic.rb`, `app/models/user.rb`, `app/models/post.rb`

### 5. Concern 中的数据库交互

小型 Concern 只做行为封装，不要包含过复杂的跨 Model 查询。重型的 Redis/DB 操作可以放在 Concern 里但需清晰注释：

```ruby
# app/models/concerns/cached_counting.rb
module CachedCounting
  extend ActiveSupport::Concern

  LUA_HGET_DEL = DiscourseRedis::EvalHelper.new <<~LUA
    local result = redis.call("HGET", KEYS[1], KEYS[2])
    redis.call("HDEL", KEYS[1], KEYS[2])
    return result
  LUA

  # ...
end
```

来源：`app/models/concerns/cached_counting.rb`

## 反模式（避免这样做）

```ruby
# ❌ 不要直接在模块顶层写 ActiveRecord 宏（没有 included do 包裹）
module Trashable
  extend ActiveSupport::Concern
  scope :with_deleted, -> { unscope(where: :deleted_at) }  # 报错！
end

# ❌ 不要用普通 module 代替 Concern（会失去 included do 支持）
module Trashable
  def self.included(base)
    base.scope :with_deleted, -> { unscope(where: :deleted_at) }
  end
end

# ✅ 正确
module Trashable
  extend ActiveSupport::Concern

  included do
    scope :with_deleted, -> { unscope(where: :deleted_at) }
  end
end
```

## 关联规范

- `ruby/01_service_objects.md` — Service 也用 include 模式但目的不同
- `ruby/05_rspec_testing.md` — `it_behaves_like` 可测试 Concern 行为
