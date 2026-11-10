---
title: "第 5 章 —— 把 frontmatter 当作一份来源记录"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - prose-layer
  - provenance
  - trust
  - frontmatter
---

# 第 5 章 —— 把 frontmatter 当作一份来源记录

本章一页速览 —— 一个**信任层**必须回答的四个问题，把“来源”（frontmatter）和“评审状态”（footer）拆成两块的那次选择，“**AI → unreviewed**”这条文化承诺，以及那些即使 schema 设计得再好、也会让来源记录一路退化成装饰的失败形态：

```{mermaid}
mindmap
  root((Provenance & trust))
    Why
      Who wrote it?
      When, against which commit?
      Has anyone verified it?
      Where did it come from?
    Frontmatter (lineage)
      created, created_by (immutable)
      source (type, uri, ref)
      commit (birth commit)
      verification_status
    Footer (review state)
      last_updated, commit
      updated_by (human, ai, ai+human)
      review_status (pending, approved)
      review_score (0-5)
      reviewed_by (human identity)
    Invariants
      Immutable authorship
      AI -> pending (zero score)
      Only humans approve
      Two-block separation
    Common mistakes
      Mutable created_by
      Single block, conflated rates
      Missing human gate
      Ontology drift
      Re-verification cliff
```

你正和安全团队开视频会议。他们**一边**开着 wiki 上那页描述**认证流程**的页面，**另一边**开着你这边的 Postgres schema。“这上面**写的是**重置 token 是 UUID，”他们说，“但这东西**从三月起就已经是 HMAC 了**。”接着：“**谁写的？什么时候写的？**”你往上翻，你往下翻。wiki 告诉你的是：这页“**四个月前、由系统最后编辑**”，**没有 diff**。它**并不告诉你**：是不是有人类写的？有没有人类**看过**？它**本应对标**的、**后端的哪一次 commit**？

这一章讲的是：**一个软件知识库能做的第二小的改动**，以及从这个改动里**能兑换到的最大的回报** —— 每一份文件**顶上**一段 YAML frontmatter，**底上**一段带六个字段的 HTML 注释。这**两块东西**加在一起，就会**强迫**知识库在**每一次对话开始**时（而不是**在糟糕对话的半截**）回答**四个问题** ——  *谁写的、什么时候、对标的是什么、有没有人核对过* 。

## 为什么

一个知识库有多好用，取决于它有多被信任。而一个软件知识库里的“信任”并不是感觉 —— 它是一串**具体的问题**，知识库每一次都得在**不需要有人在场**的情况下把它们答出来：

1.  **是谁写的？** 一个人类同事，还是一个 LLM？
2.  **什么时候写的？对着哪个版本的软件写的？**
3.  **有人验证过这东西现在还是真的吗？**
4.  **如果它是从别的地方搬过来的，从哪儿搬的？**

一张没有答案的页，是噪声。一张有清晰答案的页，可以被评审、被排序、被下线、被自动升级。Buneman 等人把这件事叫作 *来源问题* —— “对一份数据的**出身**、以及它**到达数据库的过程**的描述”{cite}`buneman2001provenance` —— W3C 的 PROV 数据模型又把它推广到任何信息制品上 {cite}`w3c_prov` 。这一章要展示的是：文稿层参考实现怎么把这些抽象问题变成一小组必填的 YAML 字段（再加上 §“frontmatter 与 footer”会论证的那一段HTML-comment footer），以及代码怎么在它们上面强制不变式，好让信任信号**不会静默腐烂**。

## 是什么

`content/` 底下的每一份 Markdown 页面都以一段 YAML frontmatter 打头，由 `---` 围栏包起来。这一段的**契约**在仓库里被显式声明出来：`wiki-template/metadata/SCHEMA.md` ，而这是整个系统里**唯一** *不是* Go 代码的配置：

```{literalinclude} ../examples/part2-prose-layer/ch05/SCHEMA.md
:language: markdown
:start-after: "## Frontmatter (Required)"
:end-before: "## Diátaxis Documentation Types"
:caption: The frontmatter contract, verbatim from `SCHEMA.md`.
```

其中有三个字段**扛着**整份信任信号：

- `created_by` —— *谁*创建了这一页（一个用户名，或者字面量 `"ai"`）。创建后**不可变**。
- `verification_status` —— 内容*当前有多可信*（`supported` / `unreviewed` / `uncertain` / `contradicted` / `superseded`）。
- `source` —— *从哪来的*，如果不是直接在 wiki 里写的话（`type`、`uri`、`path`、`ref`）。同样**不可变**。

其他字段 —— `created` 、 `updated` 、 `commit` 、 `tags` 、 `summary` 、 `doc_type` 、 `status` —— 用来支撑上面那三个，回答“**什么时候**”、“**对着哪个版本**”、“**关于什么**”这类问题。序列化这块由完整的 YAML-1.2 规范 {cite}`yaml_spec` 负责；而 frontmatter 块这个约定本身，是被 Pandoc {cite}`pandoc` 推广开的。

为什么是 YAML，不是旁挂一份 JSON、也不是一行数据库记录？三条理由：

1.  **同处一地。** 元数据和内容住在同一份文件里。读者 `cat` 一下页面就能看到溯源信息，不需要另开工具。
2.  **可 diff。** 元数据变更在 Git 里就是一行 diff，可以和正文变更并排评审。旁挂文件会把评审分裂到两个文件名上；数据库行则分裂到两个系统上。
3.  **人可写。** 第一个作者在创建页面时手写 YAML。机器后续读取和打补丁，但 schema 本身对人来说仍然是一目了然的，这防止了信任契约消失在 ORM 后面。

## 怎么做

### 解析与序列化

代码全都在 `frontmatter.go` 里，短到可以**一坐读完**。解析逻辑就是在 `---` 分隔符上把文件切开，再把 YAML 块反序列化：

```{literalinclude} ../examples/part2-prose-layer/ch05/frontmatter.go
:language: go
:lines: 16-42
:caption: `ParseFrontmatter` — `---` is the fence, YAML does the rest.
```

序列化就是反向操作：把 YAML 块围着正文重新拼回来：

```{literalinclude} ../examples/part2-prose-layer/ch05/frontmatter.go
:language: go
:lines: 44-57
:caption: `SerializeFrontmatter` — round-trip with the standard YAML marshaller.
```

**对称性**很重要。一个人写出来的那段字节，能被服务器读进来、打补丁、再写回去，中间**不会**经过任何次级 schema 的有损投影。这就是为什么“用 `vim` 编辑、用 `git` 提交”是一个**被正式支持的工作流**，而不是一个紧急逃生门。

### 为外来文件推断 frontmatter

并不是所有掉进 `content/` 里的 Markdown 文件都是 wiki 自己创建的。从 `raw/` 导入的、用 `cp` 扔进来的、从另一个仓库同步进来的 —— 这些**全都要**变成一等公民页面。`InferFrontmatter` 从文件名和 mtime 里合成一份最小但看起来合理的 frontmatter：

```{literalinclude} ../examples/part2-prose-layer/ch05/frontmatter.go
:language: go
:lines: 59-86
:caption: `InferFrontmatter` — a sane default for files that arrived without a header.
```

有两条默认值里藏着**政策**：

- `CreatedBy: "human"`。导入的文件*不会*被默认当作 AI 生成的，除非导入器显式说明。第 6 章会展示流水线在知道更多信息时如何覆盖这个默认值。
- `VerificationStatus: VerificationSupported`。已经存在于仓库里的文件至少获得了初步信任；不会被自动降级为 `unreviewed`。

### AI 内容的信任默认值

整个 frontmatter 子系统里**后果最大**的一块代码，长度只有三行：

```{literalinclude} ../examples/part2-prose-layer/ch05/frontmatter.go
:language: go
:lines: 88-95
:caption: `DefaultVerificationStatus` — one branch, one cultural commitment.
```

如果作者是 AI，默认的验证状态就是 `unreviewed` 。如果作者是人，就是 `supported` 。这就是 `SCHEMA.md` 在文稿层宣布的那条政策 （“AI 写的页面默认 `verification_status: unreviewed`”）的**可执行形态**。没有哪个人类评审需要专门记住“要把 AI 页面降级”；默认值已经替他们做了。

写入路径也**遵守同一条规则**。`service_write.go` 里的 `CreatePage` 在 `wikiAuthor` 参数上分支，然后调用辅助函数：

```{literalinclude} ../examples/part2-prose-layer/ch05/service_write.go
:language: go
:lines: 33-80
:caption: `CreatePage` — author identity is a first-class argument, not a late inference.
```

有两件事值得留意：

1.  `createdBy` 在创建时从已认证用户*或*一个显式的 AI 标记中选取，然后冻结在 `fm.Created` / `fm.CreatedBy` 里。service 中没有任何代码会再修改这两个字段。
2.  页面写入之后，service 向 Git 层请求当前 commit hash，并把它写回 `fm.Commit`。这一页在它的头部里*字面地*携带着它的出生 commit。这就是 Buneman 意义上的 `where-provenance` {cite}`buneman2001provenance`，也是让读者或 agent 能够在不离开文件的情况下追问"这份文档描述的是代码的哪个版本？"的关键所在。

### 在更新路径上强制不可变

更新路径通常是**来源泄漏**的地方。一个朴素的“把整段 frontmatter 覆盖掉”的 handler ，会在每次有人修个错别字的时候把 `created` 、 `created_by` 、 `source` 全擦掉。`MergeFrontmatterUpdate` 在**数据结构这一层**就拦住它：

```{literalinclude} ../examples/part2-prose-layer/ch05/frontmatter.go
:language: go
:lines: 97-113
:caption: `MergeFrontmatterUpdate` — immutable fields are physically copied back from the existing record.
```

更新结构体是合并结果的 *起始值* ，所以调用方可以**随便传**。但 `Created` 、 `CreatedBy` 、 `Source` 随后会被**已有记录里的值**覆盖回去，而 `Updated` 会被顶到 `time.Now().UTC()` 。一个恶意或者粗心的调用方，**就算想**也改不了历史。**这是那个让审计轨迹在页面演化时仍然诚实的唯一不变式。**

### frontmatter 与 footer：两块，不是一块

到目前为止，本章都把 frontmatter 当成 *那份* 元数据记录。但实践中，有**两种截然不同的问题**一直挤向同一个块，互相排挤：

1.  **这一页从哪来的？** —— `created_by`、`source`、`commit`。血缘。创建之后很少变。
2.  **这一页现在还可信吗？被谁信任？信任程度多高？** —— `verification_status` 一个字段试图回答所有这些。它不够。

作者的一个私人 project-knowledge-base 工具集 {cite}`fanyamin_pkb_skill` 是这样解决这个挤占问题的：把**评审状态**放进第二块 —— 页面最底部的一段 HTML-comment **footer** ，有六个**每一页都必须带**的字段。本书把这种拆分作为**规范形态**。frontmatter 仍然是一份 *来源* 记录；footer 是一份 *评审* 记录：

```markdown
<!-- PKB-metadata
last_updated: 2026-04-14
commit: 4a1c92b
updated_by: ai
review_status: pending
review_score: 0
reviewed_by:
-->
```

六个字段，每一个回答一个很窄的问题：

| 字段 | 类型 | 它回答什么 |
|-------|------|-----------------|
| `last_updated` | `YYYY-MM-DD` | 这一页的正文最近一次被 *任何人* 动过是什么时候？ |
| `commit` | 短 SHA | 对着哪个代码 commit 写的？（语义和 frontmatter 里的 `commit` 一致；在这里共处一地，是为了让过时检查便宜。） |
| `updated_by` | `human` / `ai` / `ai+human` | 最近一次改动是谁做的？和 YAML 里的 `updated_by` 相对应，但**这里用的是一套封闭的三值词汇表**，让闸可以精确地匹配。 |
| `review_status` | `pending` / `approved` | 有没有人为当前这份内容**签过字**？ |
| `review_score` | `0`–`5` | 在评审人看来，这页有多好？ `0` 表示还没被评审过。 |
| `reviewed_by` | 用户名，或空 | 是哪一个人评审的？评审之前为空。 |

**为什么是两块、不是一块。** frontmatter 和 footer 的**变化频率不同**，**动它们的人也不同**。frontmatter 记录的是 *来源* ：创建时写一次，更新时小心打补丁（见 `MergeFrontmatterUpdate` ），在 `git log` 里应当看起来像一个**几乎不变**的干净头部。footer 记录的是 *评审状态* ：正文一改，它就被重置为 `pending` ；只有**人出手**，它才翻到 `approved` 。把两者挤在一个 frontmatter 里，会混淆这两种变化速率 —— 每一次评审状态翻转都会呈现为一次来源 diff —— 还会让**过时检测**更难机械化，因为工具必须一字段一字段地判断“这个是血缘、那个是评审”。

**“AI 永远设回 pending” 这条规则。** 当一个 LLM —— 或**任何**自动化路径，包括第 23 章那个 L1 机械改写器 —— 改动一张页时，`review_status` **必须**被设为 `pending` ，`review_score` **必须**重置为 `0` ，`reviewed_by` **必须**被清空。**只有** *人类* 评审可以把 `approved` 或非零分数写上去。这就是那条让人类注意力变得 *可审计* 的唯一规则：一旦知识库的闸系统（第 22 章）看到一个 `approved` 状态却没有对应的人类 `reviewed_by` ，就说明有东西绕过了这条规则，闸就会失败。**没有这条纪律，“approved”这个词**在第一次让 LLM 写它的时候**就失去了意义**。**

**footer *不是* 什么。** 它**不是** `verification_status` 的替代品。frontmatter 里的那个字段记录的是 *编辑层面的信任信号* （“这份内容是 supported，还是 contradicted，还是 superseded？”），关心的是**内容正确性**。footer 记录的是 *评审信号* （“有没有人看过当前这个版本？”），关心的是**评审节奏**。一页可以同时带着 `verification_status: supported` 和 `review_status: pending` —— 意思是：“这一页被写下来时我们是信的；昨天一个 LLM 把它重写了之后，还没人看过。”两种信号都重要；它们回答的是**不同的问题**。

第 22 章会把 footer 做成一道**硬构建闸**；第 23 章会用它把变更**路由**进三级更新策略；第 25 章会把单轴的文档分层（L0–L4）扩展成一个四元组 `(layer, updated_by, review_status, review_score)` ，让一个检索系统可以在**不丢失血缘这根轴**的前提下按评审状态筛选。**footer 是这一章与上面三章之间的承重铰链。**

## 示例

下面用一张页的 frontmatter 把它的完整生命周期讲一遍。

**阶段 1 —— 一个人通过 API 创建了这一页。** frontmatter 和 footer 意见一致：Alice 写的，她是这一页的评审人，这一页在**诞生 commit** 上就是 approved 。

```yaml
---
title: How to enable TLS on the wiki
slug: how-to-enable-tls-on-the-wiki
created: 2026-04-10T14:03:11Z
created_by: alice
updated: 2026-04-10T14:03:11Z
updated_by: alice
status: published
doc_type: how-to
verification_status: supported
commit: a1b2c3d4e5f6
---
```

```markdown
<!-- PKB-metadata
last_updated: 2026-04-10
commit: a1b2c3d4
updated_by: human
review_status: approved
review_score: 3
reviewed_by: alice
-->
```

**阶段 2 —— 一个 LLM agent 为了让正文更清晰，把它重写了一遍。** frontmatter 按第 5 章原有规则把 `verification_status` 降为 `unreviewed` 。footer 按“AI 永远设回 pending”规则：`review_status` 重置为 `pending` ，`review_score` 清零，`reviewed_by` 清空。**两块互相印证：**

```yaml
---
title: How to enable TLS on the wiki
slug: how-to-enable-tls-on-the-wiki
created: 2026-04-10T14:03:11Z      # preserved
created_by: alice                   # preserved
updated: 2026-04-14T09:27:00Z      # bumped
updated_by: ai                      # new signal
status: published
doc_type: how-to
verification_status: unreviewed     # demoted by default
commit: 9f8e7d6c5b4a
---
```

```markdown
<!-- PKB-metadata
last_updated: 2026-04-14
commit: 9f8e7d6c
updated_by: ai                 # matches frontmatter
review_status: pending         # reset: human has not seen this
review_score: 0                # reset
reviewed_by:                   # cleared
-->
```

注意：`created_by` 仍然是 `alice` 。这次更新**没有**改写历史。但 `verification_status` 自动掉到了 `unreviewed` ， *同时* `review_status` 掉到了 `pending` ，所以任何**在意信任**的读者或者检索器，可以用这两个信号里的**任何一个** —— 或者两个一起 —— 来筛选。

**阶段 3 —— 一个人批准了这次重写。** frontmatter 把最近的 `updated_by` 记成这个评审人；footer 则单独记录了 *批准* 这一动作 —— 它**和“编辑”这个动作不是一件事**：

```yaml
verification_status: supported
updated_by: alice
updated: 2026-04-14T10:15:42Z
# created_by: alice  (still)
```

```markdown
<!-- PKB-metadata
last_updated: 2026-04-14
commit: 9f8e7d6c
updated_by: ai+human           # AI wrote the body; a human approved it
review_status: approved
review_score: 4
reviewed_by: alice
-->
```

`updated_by` 上的 `ai+human` 这个值是**故意**的：它记录的是 —— 当前这一页是一个 LLM 的草稿、一个人验证过。这和**单独的 `ai`**（LLM 写的，没人看过）以及**单独的 `human`**（人自己写的）是**真正不同的来源声明**。第 25 章那个文档分层元组在对一页分类时，会读这个值。

三个状态、一份文件、一条完整的**来源轨迹** *以及* 一条完整的**评审轨迹** —— **没有任何附表**。`git log -p content/how-to/enable-tls.md` 就能把两条轨迹一次讲完。

## 其他做法

| 系统 | 来源模型 | 作者身份的不可变性 | 评审状态 |
|--------|------------------|----------------------------|--------------|
| 文稿层参考实现 | YAML frontmatter + Git commit SHA + `PKB-metadata` footer | 由 `MergeFrontmatterUpdate` 强制执行。 | 一段显式的六字段 footer（见 §“frontmatter 与 footer”）。 |
| Pandoc / MyST {cite}`pandoc,myst_parser` | 一段 YAML 元数据块 | **只是约定**；没有强制。 | 没有。 |
| Notion AI {cite}`notion_ai` | 专有审计日志 | 不透明；由厂商控制。 | 逐块评论。 |
| Confluence | 版本历史表 | 做得不错；和专有数据库耦合在一起。 | 页面状态 / 标签。 |
| 纯 Markdown + Git | **只有** Git 历史 | *git blame* 很强，但**没有**显式的“这是 AI 写的”字段。 | 没有。 |
| W3C PROV-O {cite}`w3c_prov` | 完整的 RDF / OWL 本体 | 学术上严谨；**在文档工具里极少被采用**。 | `prov:wasInvalidatedBy` 用来建模“被废弃”。 |

**参考实现的立场是故意窄的：** 只取 PROV 词汇表里**很小**的一个子集（ `wasGeneratedBy` 、 `wasDerivedFrom` 、 `generatedAtTime` ），把它们编码成 frontmatter 里的几条**普通 YAML 键**，再加上一段六字段 HTML-comment footer 来承载评审状态 —— 这样**作者和 agent 都不用学一个新框架**，就能把两块读懂。

## 常见错误，以及每一种怎样**悄无声息地**把信任层搞坏

**来源类 schema 的失败方式有一种很特别的共性：它们 *静默地* 退化。** 一个坏掉的搜索框会**主动自报家门**；一个坏掉的信任层却会继续把**恰好是错的**页面端给你。在几乎每一个真实部署里都能看到五种失败形态，每一种都配有一条值得**显式写出来**的纠正规则。

1.  **把 `created_by` 当可变字段。** 一次出发点是好的批量迁移（"现在有了真实用户账号，让我们把作者字段都重新盖章"）会悄无声息地改写每一页的 `created_by`，整条审计轨迹就蒸发了。规则是：作者身份是*历史性的*，不是*行政性的*。一经写入，`created`、`created_by` 和 `source` 就被冻结，唯一合法的修改方式是用一个*在 Git 中记录了此次修正的 PR* 来纠正真正的错误。`MergeFrontmatterUpdate` 之所以在数据结构层面强制执行这一点，恰恰是因为出发点是好的代码路径不能被信任来记住这条不变式 {cite}`buneman2001provenance`。

2.  **把评审状态塞进 frontmatter。** 现实中最常见的形态是 frontmatter 里一个孤零零的 `status: reviewed | pending` 字段，别无其他。它之所以失败，原因就是 §"frontmatter 与 footer"那一节已经讲过的：评审状态在 LLM 每次改动正文时都会翻转，而 `git log` 里不断翻转的评审字段会把*真正的*血缘变更淹没在评审状态噪声之下。两块拆分不是风格偏好，而是让 `git log` 在知识库老化时仍然可读的关键。

3.  **`approved` 没有硬闸。** 一个允许任何角色写入 `review_status: approved` 的 schema，在一年之内就会出现由 LLM 批准的页面。"AI 永远设回 pending" 这条规则是必要的，但不够 —— 还需要一道构建闸，拒绝发布任何 `approved` 状态却没有匹配的人类 `reviewed_by` 的页面。第 22 章会接入这道闸；常见的失败是只上了 schema 没上闸，然后*假设*规则会靠约定来维持。

4.  **`source.type`、`doc_type` 和状态枚举的本体漂移。** 团队一开始用了一个小小的封闭词表（`tutorial | how-to | reference | explanation`），六个月后索引里已经悄悄长出了 `guide`、`docs`、`ref`、`manual` 和一个空字符串 —— 因为没有人拒绝过引入它们的 PR。规则是：封闭词表必须在*语法层面*封闭，也就是说 frontmatter 校验器必须拒绝未知值。只是发出警告的软闸，在几周之内就会被评审队列吞没。

5.  **重新验证悬崖。** 一份四月创建时标为 `verification_status: supported` 的页面，到了十月仍然是 `supported`，尽管它描述的代码已经被重写了两次。frontmatter 看起来是诚实的，但内容已经过期了。这正是第 8 章运营模型要去暴露的那种失败：`verification_status` 是一个绑定到 `commit` 的*时间点断言*，自动过期检测器必须机械地降级那些 `commit` 已经和它们引用的代码不再匹配的页面（第 21 章）。不会自动衰减的信任，最终会变成虚构。

6.  **`source` 字段的遗漏式谎言。** 一份从外部 URL 导入的页面，`source.uri` 指向那个 URL —— 直到那个 URL 404 了，或者源页面悄悄改了措辞。规则是：`source.ref`（一个类似 commit 的标识符）在来源有此概念时必须伴随 `source.uri`：对代码导入，这是代码的 commit；对文章导入，这是 Wayback Machine 的快照时间戳；对 API spec，这是 OpenAPI 文件的内容哈希。一个没有 `ref` 的 `source` 是一个移动靶，从审计的角度看，并不比完全没有来源好多少。

六条的共同线索是：*溯源需要强制执行，而不仅仅是表达*。一个没有校验器的信任 schema 只是装饰；一个没有硬闸的校验器只是一盏没人看的警示灯。本章的全部论点是：frontmatter + footer + 合并不变式 + 构建闸合在一起构成一台机器，抽掉任何一个角，另外三个都会塌掉。

## 结论

Frontmatter 和 footer 不是装饰品。它们合在一起，就是文稿层与所有下游消费者 —— 检索、发布、代码图谱、AI 编程助手 —— 之间关于*一页从哪来的*以及*此刻该信任它多少*的契约。`SCHEMA.md` 声明契约；`frontmatter.go` 和 `service_write.go` 强制执行血缘那一半；`PKB-metadata` footer 承载评审那一半；Git 把二者都记录为物理 diff。它们一起赋予软件知识库一样传统 wiki 很难说清楚的东西：一个显式的、机器可检查的答案 —— "这是谁写的、对着哪个 commit、现在还可信吗、被谁信任的？"

下一章把这份契约拿过来，走一遍那条流水线：**即使输入是一整个乱糟糟的、被随手倾倒进来的文件夹，**它也要产出**规格良好的 frontmatter** 和**一份诚实的 `pending` footer**。

## 参考文献

```{bibliography}
:filter: keywords % "provenance" or keywords % "prose-layer"
```
