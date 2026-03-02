# Ruby 11 — PostCreator & TopicCreator 核心创建流程

> 来源：`lib/post_creator.rb`、`lib/topic_creator.rb`、`app/models/post.rb`

---

## 概述

PostCreator 是 Discourse 中帖子和话题创建的核心协调类。它封装了验证、保存、事件触发、Job 入队等全部逻辑，Controller 只需调用 `PostCreator.create(user, opts)` 即可。

---

## 来源验证

- `lib/post_creator.rb`（678 行，主要实现）
- `lib/topic_creator.rb`（后端话题创建，PostCreator 内部调用）
- `app/models/post.rb`（Post 模型钩子 + 方法）
- `app/controllers/posts_controller.rb`（调用入口）

---

## 核心规范

### 1. 标准调用方式

```ruby
# Controller 中的标准模式
creator = PostCreator.new(current_user, opts)

if creator.valid?
  post = creator.create
  if post && creator.errors.blank?
    # 成功
    render_serialized(post, PostSerializer, topic: post.topic, add_raw: true)
  end
else
  render_json_error(creator)
end

# 快捷类方法（返回 nil 不抛异常）
post = PostCreator.create(user, opts)

# 严格版本（失败抛 ActiveRecord::RecordNotSaved）
post = PostCreator.create!(user, opts)
```

### 2. 常用 opts 参数

```ruby
# 回复帖子
opts = {
  topic_id: 123,
  raw: "回复内容",
  reply_to_post_number: 2,   # 可选，引用哪楼
}

# 新建话题（同时创建 OP）
opts = {
  raw: "正文内容",
  title: "话题标题",
  category: category.id,     # 分类 ID
  archetype: Archetype.default,
  tags: ["tag1", "tag2"],
}

# 私信
opts = {
  raw: "私信内容",
  title: "私信标题",
  archetype: Archetype.private_message,
  target_usernames: "alice,bob",   # 收件人用户名，逗号分隔
}

# 高级选项
opts.merge!(
  skip_validations: true,          # 跳过内容验证（管理员操作用）
  skip_jobs: true,                 # 不入队 Job（需手动 enqueue_jobs）
  import_mode: true,               # 导入模式：跳过通知/事件触发
  cook_method: :raw_html,          # 不经 Markdown 解析，直接存 HTML
  via_email: true,                 # 标记为邮件来源
  created_at: Time.zone.now,       # 指定创建时间（导入用）
  no_bump: true,                   # 不 bump 话题
  custom_fields: { "key" => "val" }, # 自定义字段
)
```

### 3. create 方法执行顺序

```ruby
def create
  if valid?
    transaction do
      build_post_stats           # 记录打字时长、草稿次数
      create_topic               # 若新话题则调 TopicCreator
      create_post_notice         # 新用户/回归用户提示
      save_post                  # 保存到 DB
      UserActionManager.post_created(@post)
      extract_links              # TopicLink.extract_from
      track_topic                # 更新 TopicUser（posted: true）
      update_topic_stats         # 更新 last_post_user_id、word_count 等
      update_topic_auto_close    # 检查是否触发自动关闭
      update_user_counts         # 更新 user_stat（post_count 等）
      @post.link_post_uploads    # 绑定上传文件
      delete_owned_bookmarks     # on_owner_reply 时删书签
      @post.save_reply_relationships
    end
  end

  if @post && errors.blank? && !@opts[:import_mode]
    store_unique_post_key        # Redis 防重复
    @post.topic.reload

    publish                      # MessageBus 通知前端（reply_created）
    track_latest_on_category     # 更新 Category.latest_post_id
    trigger_after_events         # DiscourseEvent.trigger(:post_created)
    enqueue_jobs                 # ProcessPost Job
    BadgeGranter.queue_badge_grant(...)
    auto_close                   # 检查 max posts 自动关闭
  end

  @post
end
```

### 4. before_create 钩子（Post 模型）

```ruby
# post.rb
before_create { PostCreator.before_create_tasks(self) }

def self.before_create_tasks(post)
  # 分配 post_number（递增，用 DistributedMutex 保证无间隙）
  post.post_number ||= Topic.next_post_number(post.topic_id, ...)

  # 渲染 Markdown
  post.cooked ||= post.cook(post.raw, cooking_options.symbolize_keys)

  post.sort_order = post.post_number
  post.last_version_at ||= Time.now
end
```

### 5. 事务与并发安全

```ruby
def transaction(&blk)
  if new_topic?
    Post.transaction { blk.call }
  else
    # 非新话题：DistributedMutex 确保 post_number 单调递增无间隙
    DistributedMutex.synchronize("topic_id_#{@opts[:topic_id]}") do
      Post.transaction { blk.call }
    end
  end
end
```

> **关键**：回复时用 `DistributedMutex` 串行化，保证 `post_number` 不重不漏。

### 6. DiscourseEvent 触发点

```ruby
# 验证阶段
DiscourseEvent.trigger(:before_create_post, @post, @opts)
DiscourseEvent.trigger(:validate_post, @post)

# 创建成功后（trigger_after_events）
DiscourseEvent.trigger(:topic_created, @post.topic, @opts, @user)  # 新话题时
DiscourseEvent.trigger(:post_created, @post, @opts, @user)
```

Plugin 监听这些事件做扩展，不修改 PostCreator 源码。

### 7. skip_jobs 模式（事务内安全）

```ruby
# 若 PostCreator 被包在事务内调用，必须 skip_jobs:
ActiveRecord::Base.transaction do
  post = PostCreator.create!(user, opts.merge(skip_jobs: true))
  # ...其他 DB 操作
end
# 事务提交后手动入队
creator.enqueue_jobs
```

原因：Sidekiq Job 可能在事务 COMMIT 前 dequeue，导致查不到刚创建的数据。

### 8. Post#cook — Markdown 渲染

```ruby
def cook(raw, opts = {})
  # raw_html 模式跳过 pipeline
  return raw if cook_method == Post.cook_methods[:raw_html]

  options = opts.dup
  options[:user_id] = self.last_editor_id  # 用于 hashtag 权限检查
  options[:omit_nofollow] = true if omit_nofollow?

  # secure_uploads 下替换 URL
  # ...

  cooked = post_analyzer.cook(raw, options)

  # Plugin Filter 钩子（after_post_cook）
  Plugin::Filter.apply(:after_post_cook, self, cooked)
end
```

### 9. 唯一帖子检查（防刷）

```ruby
# Redis key：unique[-pm]-post-{user_id}:{raw_hash}
def unique_post_key
  "unique#{topic&.private_message? ? "-pm" : ""}-post-#{user_id}:#{raw_hash}"
end

# SiteSetting.unique_posts_mins 分钟内相同内容被拒
def store_unique_post_key
  Discourse.redis.setex(unique_post_key, SiteSetting.unique_posts_mins.minutes.to_i, id)
end

def matches_recent_post?
  post_id = Discourse.redis.get(unique_post_key)
  post_id != nil && post_id.to_i != id
end
```

### 10. auto_close 逻辑

```ruby
# create 完成后触发
def auto_close
  topic_posts_count = @post.topic.posts_count

  # 私信超过 max 楼数自动关闭
  if is_private_message && SiteSetting.auto_close_messages_post_count > 0 &&
     SiteSetting.auto_close_messages_post_count <= topic_posts_count
    @post.topic.update_status(:closed, true, Discourse.system_user, message: ...)
  # 公开话题超过 max 楼数自动关闭，且可创建 linked topic
  elsif SiteSetting.auto_close_topics_post_count > 0 &&
        SiteSetting.auto_close_topics_post_count <= topic_posts_count
    topic.update_status(:closed, true, ...)
    Jobs.enqueue_in(5.seconds, :create_linked_topic, post_id: @post.id) if SiteSetting.auto_close_topics_create_linked_topic?
  end
end
```

---

## 反模式（避免这样做）

```ruby
# ❌ 直接 Post.create! — 跳过 PostCreator 的所有协调逻辑
Post.create!(raw: "...", topic_id: 1, user: user)

# ❌ 在事务内使用 PostCreator 而不 skip_jobs
ActiveRecord::Base.transaction do
  PostCreator.create!(user, opts)  # Job 可能在 commit 前执行！
end

# ✅ 正确做法
ActiveRecord::Base.transaction do
  creator = PostCreator.new(user, opts.merge(skip_jobs: true))
  post = creator.create!
end
creator.enqueue_jobs
```

---

## 关联规范

- `ruby/08_guardian.md` — can_create?(Post, topic) 权限检查
- `ruby/07_jobs.md` — ProcessPost Job
- `ruby/10_rate_limiter.md` — Post model 中的 rate_limit DSL
- `patterns/03_discourse_event.md` — post_created / validate_post 事件

---

## 快速参考

```ruby
# 基本创建
post = PostCreator.create(user, topic_id: 1, raw: "内容")

# 新建话题
post = PostCreator.create(user, title: "标题", raw: "内容", category: cat.id)

# 私信
post = PostCreator.create(user,
  title: "私信",
  raw: "内容",
  archetype: Archetype.private_message,
  target_usernames: "alice"
)

# 严格模式（失败抛异常）
post = PostCreator.create!(user, opts)

# 带事务安全
creator = PostCreator.new(user, opts.merge(skip_jobs: true))
post = creator.create!
creator.enqueue_jobs

# 检查创建者错误
creator.errors.full_messages  # => ["raw 不能为空"]
```
