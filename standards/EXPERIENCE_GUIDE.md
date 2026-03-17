# Experience 编写指南

> **本文件供 AI（开发者角色）阅读执行。**
> 每次开发任务结束后，AI 必须读取本文件，按规范将开发过程中的问题和决策写入 `experience/`。
> 目标：命名唯一、格式规范、内容可升级、不重复。

---

## 一、何时写 experience

以下情形必须触发写入，不得跳过：

| 触发条件 | 目标目录 |
|---------|---------|
| 遇到 bug 并找到根因 | `experience/mistakes/` |
| 卡住超过两轮对话后找到解法 | `experience/mistakes/` |
| 做了一个非显而易见的设计决策（选 A 不选 B，有理由） | `experience/decisions/` |
| 发现现有主文档有遗漏或描述不准确 | `experience/mistakes/` + 同时开 GitHub Issue |
| 有观察但尚未确认是否值得记录 | `experience/inbox/` |

**不需要写 experience 的情形：**
- 完全按主文档执行，无偏差，无意外
- 遇到的问题在主文档中已有明确记载

---

## 二、去重检查（写入前必须执行）

写入任何 experience 文件前，先执行：

```bash
# 检查 mistakes/ 已有条目关键词
ls experience/mistakes/
grep -l "guardian" experience/mistakes/*.md 2>/dev/null

# 检查 decisions/ 已有条目
ls experience/decisions/
```

**去重判断规则：**
- 同一根因、同一场景 → **追加到现有文件**，不新建
- 同一根因、不同场景 → **追加"变体场景"章节**到现有文件
- 不同根因 → 新建文件

---

## 三、文件命名规范

### mistakes/ 和 decisions/

```
YYYY-MM-DD_<slug>.md
```

- `YYYY-MM-DD`：发生日期（非写入日期，若不确定用写入日期）
- `slug`：2-4 个英文单词，用连字符连接，描述核心主题
- 全部小写，不含空格

**命名示例：**
```
2026-03-17_guardian-scope-nil-in-serializer.md
2026-03-17_db-exec-vs-activerecord-bulk-update.md
2026-03-20_serializer-preload-missing-for-custom-field.md
```

**命名禁忌：**
```
fix.md                    ← 无日期无描述
2026-03-17_bug1.md        ← slug 无意义
2026-03-17_Guardian.md    ← 大写
```

### inbox/

```
YYYY-MM-DD_raw-<slug>.md
```

inbox 是临时暂存，写入后需在 7 天内处理（移至 mistakes/decisions/ 或删除）。


---

## 四、文件内容格式

### mistakes/ 格式模板

```markdown
# [根因简述]（≤10 个字）

**日期**：YYYY-MM-DD
**插件/项目**：插件名或任务名
**主文档关联**：`plugins/xx_xxx.md` 或 `ruby/xx_xxx.md`（相关主文档路径）
**状态**：🔴 已记录 / 🟡 已升级至主文档

---

## 场景

[2-3 句话描述当时在做什么，触发了什么现象]

## 根因

[明确的技术根因，一句话]

## 解法

[最终生效的解法，附代码片段（简短）]

```ruby
# 错误做法
xxx

# 正确做法
xxx
```

## 教训

[这个问题说明了什么原则或认知盲区，1-2 句]

## 升级评估

- [ ] 主文档中尚未覆盖此点 → 应升级
- [ ] 主文档已有相关内容但描述不够清晰 → 应补充
- [x] 主文档已完整覆盖 → 无需升级，仅作记录
```

---

### decisions/ 格式模板

```markdown
# [决策主题]（≤10 个字）

**日期**：YYYY-MM-DD
**插件/项目**：插件名或任务名
**主文档关联**：`patterns/xx_xxx.md` 或相关文档路径
**状态**：🔴 已记录 / 🟡 已升级至主文档

---

## 背景

[为什么需要做这个决策，当时面临哪些选项]

## 选项对比

| 选项 | 优势 | 劣势 |
|------|------|------|
| 选项 A | ... | ... |
| 选项 B | ... | ... |

## 决策

选择了 **选项 X**，理由：[1-2 句核心理由]

## 后续验证

[这个决策后来证明是否正确，有无副作用]

## 升级评估

- [ ] 这个决策对应的权衡在主文档中未被讨论 → 应升级至 patterns/
- [x] 主文档已有类似内容 → 仅作记录
```

---

## 五、升级到主文档的标准

满足以下任意一条，应将 experience 条目升级写入主文档：

1. **复现性**：同一根因在不同插件/不同任务中出现 ≥2 次
2. **主文档盲点**：现有主文档完全未覆盖此知识点
3. **反模式价值**：这个错误具有典型性，写入"反模式"章节能防止再次踩坑
4. **决策可泛化**：此决策背后的原则适用于所有同类场景

**升级操作流程：**

```bash
# 1. 在主文档对应章节追加内容（来源注明）
# 2. 在 exp 文件顶部状态改为 🟡 已升级至主文档，并注明目标文件
# 3. 提交两个 commit（先 refactor 主文档，再 exp 标注状态）

git add plugins/xx.md
git commit -m "refactor(plugins/xx): add guardian scope anti-pattern from exp"

git add experience/mistakes/2026-03-17_guardian-scope-nil.md
git commit -m "exp(mistakes): mark guardian-scope-nil as upgraded to plugins/27"
```

---

## 六、开发者会话结束检查流程

每次开发任务完成后，AI 按以下顺序执行：

```
1. 回顾本次任务：是否遇到任何意外、卡点、决策？
   └─ 有 → 执行步骤 2
   └─ 无 → 跳至步骤 5

2. 执行去重检查（见第二节）
   └─ 已有相关文件 → 追加
   └─ 无 → 新建

3. 按模板写入 experience/（见第四节）

4. exp commit：
   git add experience/
   git commit -m "exp(mistakes|decisions): <描述>"

5. 评估升级（见第五节）
   └─ 符合升级标准 → 修改主文档 + refactor commit

6. chore commit（更新 LEARNING_LOG 如适用）
```

---

## 七、quality 检查（AI 自检清单）

写完 experience 条目后，对照以下清单逐项确认：

- [ ] 文件名格式正确（`YYYY-MM-DD_slug.md`，全小写，无空格）
- [ ] 去重检查已执行，无重复文件
- [ ] 场景描述包含"在做什么 + 触发了什么"
- [ ] 根因是技术性的明确表达，不是"不小心写错了"
- [ ] 解法包含可直接参考的代码片段
- [ ] 升级评估已填写（不得留空）
- [ ] 主文档关联字段已填写（不得为空）
- [ ] 状态字段已标注
