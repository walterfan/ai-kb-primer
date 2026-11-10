---
title: "第 24 章 —— 文稿与代码相遇"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - hybrid
  - graph-rag
---

# 第 24 章 —— 文稿与代码相遇

## Why —— 为什么

第二部分建设了一个文稿层：Markdown 页面、Diataxis 分类、frontmatter schema、发布门禁。第四部分建设了一个代码层：解析器提取实体、向量索引接住模糊查询、图谱承载结构关系。两层各自运转正常。问题是：它们**互相不知道对方**。

一个开发者问 *"为什么 `runSync` 要恢复 panic？"* —— 答案一半在代码里（`recover()` 调用的位置），一半在文稿里（ADR-0014 对 worker-pool panic 策略的论述）。纯代码检索找到了函数体，但不知道有一份 ADR 在解释它；纯文稿检索找到了 ADR，但不知道它关联的是哪个函数。两条检索链路各自返回的 top-5 结果，可能都不包含另一层的关键证据。

这不是检索质量的问题，而是**结构缺失的问题**。文稿和代码之间缺少显式的跨层链接 —— 一种能让检索器同时看到两岸的桥梁。本章建设那座桥。

近期的仓库级研究正在从不同角度验证同一需求。RepoGraph 在代码层构建了类型化的结构图（calls、imports、inherits、defines），并用图遍历扩展检索种子，使上下文不再只是语义相似的文本片段，而是结构完整的邻域 {cite}`ouyang2025repograph`。本章把 RepoGraph 的思路推向跨层：不仅在代码节点之间建图边，还在代码节点和文稿节点之间建图边。LLM 与 KG 的协同问答研究也印证了这一点 —— 将软件仓库建模为包含 Developer、Commit、Issue 等节点的专用知识图谱后，"谁改了这个模块？相关的 ADR 是什么？"这类多跳问题才能被机械回答 {cite}`xu2024llmkgsynergy`。

## What —— 是什么

**跨层链接**（cross-layer link）是连接一个文稿节点和一个代码节点的有向边。它不是一条松散的超链接，而是一条**被解析过的、有类型的、可校验的**关系。

三种核心的跨层链接类型：

| 链接类型 | 方向 | 语义 | 典型例子 |
|:--|:--|:--|:--|
| `CITES` | prose → code | 文稿引用了一段代码 | ADR 中的 `file:line` 锚点 |
| `EXPLAINS` | prose → code | 文稿解释了一个代码实体的设计理据 | 架构文档解释了某个模块的存在理由 |
| `DOCUMENTS` | prose → code | 文稿是某个代码实体的用户手册 | runbook 描述了某个服务的操作流程 |

反向遍历同样有意义。从一个代码实体出发，沿反向 `EXPLAINS` 可以到达解释它的 ADR；沿反向 `CITES` 可以到达引用它的所有页面。这正是"为什么 `runSync` 恢复 panic"这类问题需要的信息流。

### 与纯超链接的区别

很多文档系统支持从 Markdown 页面链接到 GitHub 文件。那是一个**URL**，不是一条**图边**。区别在于：

1. **URL 不可遍历。** 检索器不能从代码实体反向走到引用它的 ADR。跨层链接是双向的。
2. **URL 不可校验。** 文件移动后 URL 断掉了，但没有自动化手段能发现。跨层链接锚定在稳定 ID 上，第 22 章门禁 H4 会机械地校验每一条。
3. **URL 不带类型。** 检索器不知道一条链接是"引用"还是"解释"。跨层链接的类型标签让上下文组装器（第 26 章）可以按问题形状选择扩展哪类边。

## How —— 怎么做

### 1. 从现有锚点提取 `CITES` 边

第 5 章的 frontmatter schema 要求文稿页面中的代码引用使用 `file:line` 格式。第 16 章为每个代码实体分配了稳定 ID。两者的交汇点就是 `CITES` 边的提取：

```text
对每个文稿页面 P:
  扫描 P 的正文和 frontmatter
  对每个匹配 `filepath:startLine-endLine` 模式的锚点 A:
    在代码实体索引中查找包含该行范围的实体 E
    如果找到:
      创建边 (P) --CITES--> (E)
      记录锚点的精确行范围
    如果没找到:
      标记为悬空引用，交给第 22 章门禁 H4
```

这个过程是**完全机械的** —— 不需要 LLM，不需要人工标注。它在每次 `make build` 时重新执行，成本为 O(页面数 × 平均引用数)。

### 2. 从命名约定推断 `EXPLAINS` 边

`EXPLAINS` 边比 `CITES` 复杂一些。并非每份 ADR 都含有 `file:line` 锚点 —— 有些 ADR 只用自然语言提到了模块名。推断规则分两层：

**L1：精确匹配。** 如果 ADR 的 frontmatter 中有 `relates_to: [pkg/auth, internal/pipeline]` 字段，直接创建 `EXPLAINS` 边到对应路径下的所有实体。这是启发式的黄金通道 —— 零歧义。

**L2：模糊匹配。** 如果 ADR 正文中提到了一个与代码实体名称匹配的标识符（例如出现了 `AuthService`，代码图中恰好有一个同名 type），创建一条候选 `EXPLAINS` 边，标记 `confidence: medium`。这条边在检索时可用，但在 `kb.cite` 输出中需要附注"推断链接"以区别于精确链接。

**不做 L3。** 用 LLM 做自由文本到代码实体的链接推断成本高且不可靠。文稿中说 *"认证模块"* 不一定对应名为 `auth` 的包。宁愿少链接，也不要造出一堆噪声边。

### 3. 从文档结构推断 `DOCUMENTS` 边

`DOCUMENTS` 边表示"这份文档是某个代码组件的用户手册"。推断依赖目录约定：

- `runbooks/ingest-pipeline.md` → 关联 `internal/pipeline/` 下的实体
- `architecture/auth-flow.md` → 关联 `pkg/auth/` 下的实体
- `api-docs/sync-api.md` → 关联导出了 HTTP handler 的实体

映射规则通过一个配置文件 `docs-to-code.yaml` 声明：

```yaml
mappings:
  - docs_glob: "runbooks/*.md"
    code_path: "internal/"
    link_type: DOCUMENTS
    strategy: name_prefix_match
  - docs_glob: "architecture/*.md"
    code_path: ""
    link_type: EXPLAINS
    strategy: frontmatter_relates_to
```

这比"扫描全部文本做 NER"要廉价得多，且可审计 —— 映射规则本身就是知识库的一部分。

### 4. 在仓库图中物化跨层边

第 18 章的代码知识图谱已经有了代码层的节点和边。跨层链接作为**新的边类型**加入同一张图，不是单独的存储：

```text
(:ProsePage {id: "adr-0014", title: "Panic-safety in worker pools"})
  -[:EXPLAINS {confidence: "high", source: "frontmatter.relates_to"}]->
(:Function {id: "abc123", name: "runSync", file: "service.go", start: 201})
```

这意味着第 19 章的图检索和第 26 章的图扩展可以无缝地跨越文稿-代码边界。当 `runSync` 是种子时，沿 `EXPLAINS` 反向一跳就能到达 ADR-0014。

### 5. 增量维护

跨层链接不是一次性产物。代码变了、文稿变了，链接可能断掉或过期。维护策略继承第 21 章的增量同步模式：

- **代码侧变更：** 一个函数重命名，所有指向旧名称的 `CITES` 和 `EXPLAINS` 边变成悬空。增量同步在"影响分析"阶段检测到实体 ID 变化，将受影响的文稿页面标记为候选（第 23 章三级更新策略的入口）。
- **文稿侧变更：** 一份 ADR 新增了 `file:line` 引用。下次构建时 `CITES` 提取器重新扫描该页面，创建新边。
- **悬空边清理：** 每次全量构建时，校验所有跨层边的两端是否仍然存在。不存在的边被移除。这是第 22 章门禁 H4 的批量版本。

## Example —— 范例

以下是一个跨层链接从提取到使用的完整流程：

**阶段 1：构建时提取。** `make build` 执行 cross-link 提取器。输出日志：

```text
Cross-layer link extraction:
  Scanned 47 prose pages
  CITES edges created:       128
  EXPLAINS edges (exact):     34
  EXPLAINS edges (fuzzy):     12
  DOCUMENTS edges:            19
  Dangling references:         3  → routed to gate H4
  Total cross-layer edges:   193
```

**阶段 2：检索时跨越。** 开发者问 *"`AuthService.Validate` 为什么接受过期 token？"* 混合检索以 `AuthService.Validate` 函数为种子，然后沿 `EXPLAINS` 反向边找到 *"ADR-0021: Token grace period for clock skew"*。没有跨层链接，这份 ADR 不会出现在 top-5 中 —— 因为它在文本上与 `AuthService.Validate` 几乎没有重叠。

**阶段 3：agent 引用。** agent 回答时同时引用了代码位置和 ADR：

```text
`AuthService.Validate` (pkg/auth/service.go:89-112) accepts tokens
up to 30 seconds past expiry. This grace period is a deliberate
design choice documented in ADR-0021, which accounts for clock skew
between client and server.
```

两条引用 —— 一条代码锚点、一条文稿锚点 —— 都是从知识库的跨层图中机械获取的，不是 LLM 编造的。

## Conclusion —— 小结

文稿层和代码层分别建好之后，它们之间的桥梁不会自动出现。本章建设的跨层链接——`CITES`、`EXPLAINS`、`DOCUMENTS` —— 把两岸连成一张可遍历的图。提取是机械的，成本低；维护继承增量同步；校验由门禁 H4 保证。

这座桥改变的不仅是检索质量。它改变的是知识库的**回答结构**。没有桥的知识库回答代码问题时只能给代码，回答文稿问题时只能给文稿。有桥的知识库回答任何问题时都可以同时给出 *what*（代码在哪里）和 *why*（文稿怎么解释）。这是第 26 章图引导上下文组装的前提条件：组装器需要一张包含两种节点的图，而本章就是那张图的接缝层。

## 参考文献

```{bibliography}
:filter: keywords % "hybrid" or keywords % "graph-rag"
```
