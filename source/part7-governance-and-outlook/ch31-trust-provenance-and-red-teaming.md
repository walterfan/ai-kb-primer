---
title: "第 31 章 —— 信任、出处与红队演练"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - governance
  - provenance
  - trust
  - ingestion
---

# 第 31 章 —— 信任、出处与红队演练

## Why —— 为什么

第 30 章把成本、隐私、安全当作三条运维约束来整合，但它有一个隐含前提：知识库里的内容是**内生的** —— 由团队成员编写，在仓库内 commit，经评审后发布。然而真实世界的知识库很少只靠自产内容过活。它需要**摄入**来自外部的产出物：

- 团队在 `openspec/` 里写的设计规格书和变更提案；
- AI agent（Superpowers plan、TDS 生成器、Cursor 规则导出）产生的结构化文档；
- 从已有文档站点（Confluence、Notion、内部知识平台、旧 Sphinx 站点、内部 wiki）批量导出的页面。

每一种来源的信任属性不同。一份经过评审的 OpenSpec 规格书和一份 AI 起草的技术方案，不应该在知识库里拥有相同的 `review_status`。一份从 Confluence 导出的页面，其 "最近更新时间" 可能是两年前 —— 放进知识库后如果不标记陈旧度，就成了一颗定时炸弹。

没有出处模型（provenance model）的知识库，无法回答一个根本问题：**这个页面从哪来的？我应该相信它吗？** 本章建立那个模型。

## What —— 是什么

### 三类产出物来源

知识库摄入的外部内容可以按信任属性分成三个层级：

| 来源类型 | 典型代表 | 信任属性 |
|:--|:--|:--|
| **人工编写、结构化** | OpenSpec specs、ADRs、RFCs、设计文档 | 高信任 —— 有明确作者、评审状态、Git 版本控制 |
| **AI 生成、人工评审** | Superpowers plan、TDS、agent 生成的摘要 | 中信任 —— 出处链：agent → 人工评审者 → KB |
| **外部文档站点导出** | Confluence 页面、Notion 导出、内部知识平台 文档、旧 Sphinx 站点 | 可变信任 —— 常常陈旧、无 frontmatter、评审状态未知 |

这三类并非互斥；一份 AI 起草、人工修订的 OpenSpec 变更提案同时属于前两类。关键不是精确分类，而是**每一页都必须带着出处进入知识库**，不能裸入。

### 出处元组

第 25 章定义了文档层元组 $\langle L, U, R, S \rangle$（层级、更新者、评审状态、评分）。摄入场景需要扩展这个元组，加入**来源标签**：

$$
\text{provenance} = \langle \text{source\_class}, \text{source\_path}, \text{review\_status}, \text{staleness}, \text{citation\_integrity} \rangle
$$

- `source_class`：`human-structured`、`ai-generated`、`doc-site-import` 之一
- `source_path`：原始文件路径或 URL（如 `openspec/specs/auth-redesign.md`、`confluence://space/page-id`）
- `review_status`：继承自来源（若来源有评审机制），或默认 `pending`
- `staleness`：基于来源最后修改时间与知识库摄入时间的差值
- `citation_integrity`：页面中的 `file:line` 引用是否在 HEAD 上可解析

## How —— 怎么做

### 1. OpenSpec / RFC 摄入

OpenSpec 是本书仓库已经在用的设计规格书框架。设计规格和变更提案分别存放在 `openspec/specs/` 和 `openspec/changes/` 中，天然就是 Git 版本控制的 Markdown 文件。

摄入规则：

1. **出处标记。** frontmatter 中写入 `source: openspec/<filename>`，`created_by` 取 spec 文件的 Git 作者。
2. **评审状态继承。** OpenSpec spec 是预先评审过的产出物（提交前已经过设计评审），直接写入 `review_status: approved`。变更提案（changes）视其状态而定 —— `accepted` 映射 `approved`，`proposed` 映射 `pending`。
3. **分类。** 第 6 章流水线将 spec 归类为 Diataxis 的 `reference`（如果偏 API 文档）或 `explanation`（如果偏设计推理）。启发式分类即可覆盖大部分情况 —— 文件名或目录路径中含 `spec` 字样。
4. **陈旧检测。** 后续同步中，比较 `openspec/` 文件的 Git 修改时间与知识库页面的 `last_updated`。如果 spec 更新了但知识库页面没有跟进，触发 L1 更新（机械同步内容）或 L2 更新（需要 LLM 解释变更影响）。

ADR 和 RFC 遵循完全相同的逻辑。唯一的差异是 ADR 通常有 `status: accepted/superseded/deprecated` 字段 —— 知识库应当原样保留这个字段，作为一份独立于 `review_status` 的来源端元数据。

### 2. AI 生成的技术方案摄入

AI coding agent 产出物的标准化程度在快速提高。典型的 AI 生成设计文档有两类：

- **Superpowers plan** —— 存放在 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`，有结构化的 header（目标、文件结构、任务分解、测试计划）。
- **TDS（Technical Design Specification）** —— 由 `generate-design-spec` 工具从 Jira issue 生成，输出结构化 Markdown。

摄入规则：

1. **AI 永远 pending。** 这是第 8 章 C3 的核心规则：AI 生成的内容，`review_status` 一律设为 `pending`。知识库**绝不**自动将 AI 生成的页面提升为 `approved` —— 必须由人工评审。
2. **出处标记。** `created_by: ai`，`source: superpowers/plan/<name>` 或 `source: tds/<jira-key>`。这条出处链让后续审计可以追溯："这个页面是哪个 agent、在什么上下文下生成的？"
3. **引用校验。** AI 生成的方案常常引用代码文件或模块。这些引用必须通过第 22 章 H4 门禁（`file:line` 可解析）才能发布。AI 幻觉出的模块名在此被拦截。
4. **冻结 vs. 活文档。** 一份 AI 方案一旦被人工评审通过并执行完毕，就应转为 `frozen` 状态 —— 后续代码变更不应触发对该方案的重新生成。方案是历史记录，不是活文档。

### 3. 文档站点导出摄入

从已有文档站点批量导入是成本最高、风险最大的摄入场景。来源包括 Confluence 空间导出（HTML/XML）、Notion 数据库导出（Markdown/CSV）、内部知识平台 页面（API/Markdown）、以及旧 Sphinx 站点的 `_build/html/`。

摄入规则：

1. **默认不信任。** `review_status: pending`，无论来源站点上的评审状态如何 —— 因为来源端的评审语义与知识库不同，且无法机械校验。
2. **以导出时间戳为准。** `last_updated` 设为导出时间，**不是**原始创建日期。原始日期保存在 `source_created_at` 扩展字段中，作为参考但不参与陈旧计算。来源站点上"最近更新：3 年前"的页面，如果不做这个转换，会在知识库里安静地待着而不被陈旧检测捕获。
3. **LLM 分类。** 文档站点导出页面通常没有 Diataxis 分类信息。第 6 章分类流水线的 LLM 模式（而非启发式模式）是必须的 —— 这也是第 30 章提到的"文档站点导出分类成本高"的原因。
4. **去重。** 导入前检查是否已有 `source` 字段匹配的页面。同一 Confluence 页面 ID 导入两次，应被拦截而非产生重复。
5. **分批导入。** 第 30 章建议的"先算再导"干跑策略在此适用。一个 500 页的 Confluence 空间不应一次性导入 —— 按空间或标签分批，每批跑完检查分类分布和成本，再决定是否继续。

### 4. 出处链全景

三类来源的摄入路径汇入同一条流水线：

```{mermaid}
flowchart TD
    subgraph sources ["外部来源"]
        OS["openspec/specs/\n(结构化, 已评审)"]
        SP["superpowers/plans/\n(AI 生成, 待评审)"]
        DS["文档站点导出\n(非结构化, 信任未知)"]
    end
    subgraph pipeline ["第 6 章 摄入流水线"]
        CL["分类\n(Diataxis)"]
        FM["分配 frontmatter\n+ 出处标签"]
        VR["校验\n(链接, 引用, 去重)"]
    end
    subgraph kb ["知识库"]
        PG["带出处的 KB 页面"]
        GT["门禁 (第 22 章)"]
        PB["已发布"]
    end
    OS -->|"source_class:\nhuman-structured"| CL
    SP -->|"source_class:\nai-generated"| CL
    DS -->|"source_class:\ndoc-site-import"| CL
    CL --> FM --> VR --> PG --> GT --> PB
```

流水线是统一的，但**信任默认值不同**。这正是出处元组的价值：同一条流水线、同一套门禁，但每个来源进入时的"起跑线"不一样 —— OpenSpec 带着 `approved` 进来，AI 方案带着 `pending` 进来，文档站点导出带着 `pending` 加高陈旧风险进来。

### 5. 红队演练

知识库的治理模型需要用对抗性场景来测试。以下四个红队场景分别攻击出处链的不同环节：

**场景 1：陈旧的 Confluence 导入。** 一份六个月前从 Confluence 导入的页面，描述的是旧版 API 的行为。新版 API 已上线但知识库页面未更新。一个 agent 引用了这个页面来回答开发者问题 —— 答案是错的。

- **攻击面：** 陈旧度 —— 来源站点更新了但知识库没有跟上。
- **检测手段：** 第 23 章的漂移检查 + `last_updated` 与当前日期的差值。超过阈值的页面应被标记为 `stale` 并降低搜索权重。
- **防御纪律：** 文档站点导入不是一次性的；要么建立定期重新导入的 cron，要么在导入时就设定 `max_staleness_days` 并在超期后自动降权。

**场景 2：AI 方案中的幻觉架构。** 一份 TDS 声称系统有一个 `AuthorizationService` 模块，但这个模块其实不存在 —— AI 基于命名惯例推断出了一个虚构的组件。开发者信以为真，花了半天去找这个模块。

- **攻击面：** 引用完整性 —— AI 生成的引用指向不存在的实体。
- **检测手段：** 第 22 章 H4 门禁 —— 每个 `file:line` 引用在 HEAD 上解析。H4 会拦截这份 TDS 的发布。
- **防御纪律：** AI 生成的方案在发布前必须通过引用解析门禁。对于没有 `file:line` 引用但包含模块名的方案，额外跑一次"模块名存在性检查"（在代码图中查找对应的 entity ID）。

**场景 3：被污染的 OpenSpec。** 攻击者通过一个合法的 PR 提交了一份 OpenSpec 变更提案，表面上是一个性能优化，实际上修改了认证逻辑的关键路径。因为它走的是 OpenSpec 流程，知识库以 `approved` 状态导入了这份变更。

- **攻击面：** 作者信任 —— 攻击者利用了 "OpenSpec 是预评审的" 这个假设。
- **检测手段：** `created_by` + Git 签名 commit + `source: openspec/` 出处审计。团队应定期审查 `source_class: human-structured` 且 `source_path` 含 `security/` 或 `auth/` 的页面。
- **防御纪律：** 安全敏感路径的 OpenSpec 不应自动以 `approved` 进入 —— 在派发器中设定 `openspec/specs/security-*` 的覆盖规则，强制 `review_status: pending`，走 L3。

**场景 4：重复导入冲突。** 同一个 Confluence 页面被两个人在不同时间导入，各自做了少量编辑。知识库中现在有两个同源页面，内容略有不同。agent 搜索时两个都命中，给出自相矛盾的答案。

- **攻击面：** 去重失败 —— 同源内容的多副本。
- **检测手段：** `source` 字段去重检查。摄入流水线在写入 `content/` 前查询是否已有 `source` 匹配的页面。
- **防御纪律：** 去重检查是摄入流水线的**硬闸**，不是软建议。如果 `source` 匹配已有页面，流水线应拒绝导入并报告冲突，让人类决定是更新已有页面还是创建独立副本。

## Example —— 范例

一个团队将 Confluence 空间 "Platform Architecture" 导入知识库的具体步骤：

```text
Step 1: Export
  $ confluence-export --space PLAT --format markdown --output ./raw/confluence-plat/
  Exported 347 pages to ./raw/confluence-plat/

Step 2: Dry run
  $ kb-cli import dir ./raw/confluence-plat/ --dry-run
  Classification distribution:
    tutorial:     12   (3.5%)
    how-to:       89  (25.6%)
    reference:   156  (45.0%)
    explanation:  73  (21.0%)
    unclassified: 17   (4.9%)
  Estimated LLM tokens:  ~520k  (347 pages × avg 1.5k tokens/classify)
  Estimated cost:         ~$0.26
  Duplicates detected:    4 (matching existing source URLs)
  Staleness warnings:     23 pages older than 180 days

Step 3: Review dry-run report
  Team decides:
  - Skip 17 unclassified pages (manual triage later)
  - Skip 23 stale pages (will import after source owners confirm currency)
  - Proceed with remaining 307 pages in 3 batches

Step 4: Batch import
  $ kb-cli import dir ./raw/confluence-plat/ --batch 1 --skip-stale --skip-unclassified
  Imported 107 pages:
    source_class: doc-site-import
    review_status: pending (all)
    source: confluence://PLAT/<page-id>
  Frontmatter example:
    ---
    title: "Platform Auth Flow"
    source: "confluence://PLAT/123456"
    source_class: doc-site-import
    source_created_at: 2024-03-15
    created_by: confluence-import
    review_status: pending
    keywords: [platform, auth]
    ---

Step 5: Gate check
  $ make check
  Hard-gate failures: 3
    - 2 citation-resolve failures (file:line anchors point to deleted files)
    - 1 frontmatter-schema failure (missing 'keywords' on 1 page)
  → Fix and re-run, or route to L3 for manual review.
```

这个例子中最重要的数字不是成本（$0.26 很便宜），而是**23 个陈旧页面被拦在门外、4 个重复被检测到、3 个门禁失败被捕获** —— 全部发生在内容进入知识库之前。

## Conclusion —— 小结

信任不是二值的。一个页面的可信度是一个多维元组 —— 来源类型、评审状态、陈旧度、引用完整性 —— 知识库按页面跟踪这个元组，而不是假设所有内容生而平等。

本章的核心主张是：**摄入路径不同，信任默认值就不同，但流水线和门禁是统一的。** OpenSpec 带着高信任进入，AI 方案带着中信任进入，文档站点导出带着低信任进入 —— 但所有来源都走同一条第 6 章流水线、同一套第 22 章门禁。差异不在流水线本身，而在每个来源进入流水线时背上的出处标签。

红队演练的价值不在于发现知识库今天有多少漏洞（它一定有很多），而在于**用对抗性场景验证出处链的每一个环节是否在机械层面被覆盖** —— 陈旧检测能不能捕获过期页面？去重能不能拦截重复导入？引用门禁能不能拦截幻觉引用？作者审计能不能追溯可疑提交？如果其中任何一个的答案是"只靠人记着"，那就是下一个要自动化的环节。

下一章 —— 也是全书最后一章 —— 讨论仍然开放的问题：当知识库不再只从一个仓库生长、而是从多个来源汇聚时，治理模型将面临什么新的挑战。

## 参考文献

```{bibliography}
:filter: keywords % "governance" or keywords % "provenance" or keywords % "self-citation"
```
