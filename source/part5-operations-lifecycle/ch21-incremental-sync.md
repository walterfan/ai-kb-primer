---
title: "第 21 章 —— 增量同步"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - operations
  - deepwiki
  - drift
---

# 第 21 章 —— 增量同步

## Why —— 为什么

第 8 章的运维模型抛出了一个 C3 本身回答不了的问题： * 到底哪些页面需要更新？ * 如果这个问题没有好答案，“增量”同步就只是个说法 —— 要么流水线在每次提交时把所有页面都重跑一遍（今天看成本很低，但它会产出一个 400 个文件的 PR，没有任何人类会去评审），要么它什么都不做，于是知识库开始腐烂。

增量同步就是夹在“有东西变了”和“有具体某几个页面需要关注”之间的那层机制。正是这一层接口让 C3 的三级更新策略（第 23 章）真正用得起，也是 C2 的 metadata footer（第 5 章）最直接依赖的那一层。做得好，同步能把一个 1000 次提交的周，收敛成寥寥几次“可评审的页面更新”。做不好，它就把一周的 commit 变成噪声。

本书所扩展的那篇博客 {cite}`fanyamin2026deepwiki` 把代码层（第四部分）的增量同步状态机阐述为：“在 tree-sitter 实体 ID 上做 Merkle diff”。本章把同一模式推广到文稿层，并讲清楚这两层是如何共用一套变更检测底座的。

RepoAgent 从文档层面独立地得出了同样的运维结论：仓库级文档如果不自带一个持久的**更新循环**，而只是一次性生成，就不算真正有用 {cite}`luo2024repoagentllmpoweredopensourceframework`。RepoAgent 命名了三个模块 —— 全局结构分析、文档生成和文档更新 —— 本章把更新阶段显式化到足够可推理的程度：先检测出最小安全候选集，再让第 23 章来决定修复手段是机械式的、受限 LLM 的，还是人工主导的。

## What —— 是什么

增量同步针对一对 commit $(c_\text{prev}, c_\text{now})$ 回答一个问题：

$$
\text{affected pages} = f(\text{diff}(c_\text{prev}, c_\text{now}))
$$

难点在 $f$ 。最朴素的实现把 $f$ 定义成“每一个页面” —— 正确但没用。过于聪明的实现把 $f$ 定义成“只挑那些正文里字面提到了变更文件名的页面” —— 便宜但不安全，因为一次重构完全可能在不改动任何页面正文字面内容的前提下，让页面里的 `file:line` 引用失效。

运维模型里 $f$ 被拆成三层、从最便宜的一层开始 —— 这个拆分是作者在一个私有的 project-knowledge-base 工具包 {cite}`fanyamin_pkb_skill` 中，跨多个仓库反复打磨出来的。每一层都以递增的精度和递增的成本回答“哪些页面受影响”这个问题；第一层能解决的变更，永远不会落到第二层。

| Layer | Input | Output | Cost | Catches |
|:--|:--|:--|--:|:--|
| **L-git** | `git diff --name-only $prev..HEAD` | candidate files | O(1) per commit | renames, additions, deletions |
| **L-entity** | code-layer entity IDs that moved (Chapter 11) | pages whose `file:line` citations point at moved code | O(entities × citations) | refactors that preserve filenames |
| **L-link** | every `](...md)` and `file:line` anchor in every page | pages whose outgoing links no longer resolve | O(links) | link rot from any cause |

只有在三层都被依次咨询过之后，系统才会把候选页面交给第 23 章的 L1/L2/L3 派发器。同步本身 * 不 * 修改任何页面；它的职责就是产出一份可评审、已去重、“最小而安全”的候选页面集合。

这种分离 —— 候选在这里检测、更新在那边做 —— 是第 8 章那套模型能规模化的原因。检测便宜、确定性强；更新贵，且有时要调 LLM。把两件事并成一步，会把两种成本模型耦合在一起，结果往往是两边的缺点都被占满。

## How —— 怎么做

### L-git 层

最便宜的一层。每个页面都在 footer 中带一个 `commit:` 字段（第 5 章 C2）。在每次同步时，作业会计算“该页面 `commit:` 与 `HEAD` 之间发生过变化的文件集合”：

```text
for each page P in content/**/*.md:
    prev := P.footer.commit
    changed := `git diff --name-only prev..HEAD`
    if changed == []:
        # page is up to date at the commit-SHA level
        continue
    add P to candidates
```

footer 已经指向 `HEAD` 的页面会被完全跳过 —— 不读正文、不调 LLM。这就是 L1 机械化场景最纯粹的形态：“这个页面没事”的成本 = 一次 git 命令。

所以 footer 里的 `commit:` 字段不是装饰。它是 L-git 真正赖以工作的承重数据。一个知识库如果丢掉这个字段（或者因为“评审时只看没写回”让它脱节），就等于丢掉了 L-git，从此只能落到 L-entity 和 L-link —— 这两层都要贵得多。

### L-entity 层

L-git 只能检测到 * 哪些文件 * 变了。一次重命名了导出函数的重构并不会重命名任何文件 —— 但它会打破每一个按名字引用该函数的页面。L-entity 正是通过以第 18 章定义的代码层实体 ID 为 key，来捕捉这类情况。

每个代码实体都有一个稳定 ID：`sha256(repoID | filePath | entityType | name | startLine)`。代码层同步跑完后（第 18 章），ID 发生变化的实体会落入一个 `moved_entities` 集合。L-entity 接着问： * 哪些文稿页面引用了这些实体？ * 答案来自 verify 步骤（第 7 章）构建的反向索引 —— 它会扫描每个页面的 `file:line` 锚点，并记录一条 `(page, entity_id)` 边。

```text
moved := code_sync.moved_entities(prev, HEAD)
for each entity_id in moved:
    for each page P in reverse_index[entity_id]:
        add P to candidates
```

L-entity 是让 * 文稿层与代码层保持同步 * 的那一层。没有它，一次重命名函数的重构会让文稿继续引用一个已死的实体，而一个检索 agent（第 29 章）会照样欢快地把这条引用引出来。有了它，页面会被打上标记，走第 23 章的 L2 有界 LLM 路径，重新针对当前代码引用一次。

### L-link 层

捕获前两层漏掉的一切：另一页中重排的章节导致 `ch05-frontmatter#foo` 锚点移位；一个被移动的示例文件把每一条 `literalinclude` 路径都打断；`_static/` 中一张被删除的图片。L-link 是暴力扫描 —— 每个页面里的每一个外链都会对当前 commit 重新测试一次。

```text
for each page P in content/**/*.md:
    for each link L in outgoing_links(P):
        if not resolves_at(HEAD, L):
            add P to candidates
```

L-link 看起来贵，其实不贵：这些检查都是纯文件系统操作，对 1000 个页面的知识库而言会在数秒内完成。它在每次同步时都整体跑一遍 —— 这里没有增量捷径，因为一条链接可能因 * 目标 * 的变更而断（L-git 和 L-entity 已经知道这类变更），也可能因为 * 源 * 的变更而断，而后者这两层谁都没看到（例如有人直接用 `vim` 改了一个页面）。

### 产出候选集合

三层发射出来的候选页面会有一定重叠。同步阶段会去重，并给每个候选打上“是哪一层捕获到它的”标签，好让第 23 章的派发器据此选择级别：

| Caught by | Suggested level (Chapter 16) |
|:--|:--|
| L-git only, and only the footer `commit:` drifted | **L1** re-stamp |
| L-git only, and real body-citing files changed | **L2** bounded-LLM |
| L-entity | **L2** bounded-LLM, with the moved-entity ID in context |
| L-link (target moved) | **L1** link fix if possible; else **L2** |
| Any layer + page is in `architecture/` or `adr/` | **L3** human-led |

最后一行很关键。架构页和 ADR 都是安全敏感的文稿；运维模型拒绝让 LLM 在无人知晓的情况下悄悄重新生成它们 —— 不管前面几层怎么判。这是一条策略，在派发器配置里只声明一次，而不是在每个页面上反复争论。

## Example —— 范例

针对一个把 Go 包从 `internal/worker` 改名为 `internal/job` 的 PR，跑一次具体的同步：

**输入 diff。** 12 个文件变了。其中 4 个原本在 `internal/worker/`（现在在 `internal/job/`）下；8 个是此前导入旧包的调用方。

**L-git。** 产出 12 个候选文件。在知识库的页面中，有三个的 `commit:` 值落在 PR 之前：`data-and-api.md`、`repo-map.md`、`runbook.md`。三个都被加入候选集。

**L-entity。** 代码层同步报告了 9 个移动实体（该包的 9 个导出符号 `name` 保持不变、但 `filePath` 变了，所以它们的稳定 ID 也跟着变了）。反向索引显示 `data-and-api.md` 引用了其中 3 个，`repo-map.md` 引用了 1 个。两个页面本来就在候选集里；L-entity 没有新增候选 —— 但它把这些“移动实体 ID”挂上来，好让 L2 在上下文里能拿到。

**L-link。** 知识库中所有 `internal/worker/...` 字面路径都已失效。`repo-map.md` 里有 4 条这样的路径；`data-and-api.md` 里有 2 条。两个页面也都已经在候选集里。

**派发器输入。** 三个候选页面，每个都带着“是哪一层抓到它的”标签：

```text
repo-map.md       {L-git, L-entity, L-link}   suggest L1 (regenerate from git ls-files)
data-and-api.md   {L-git, L-entity, L-link}   suggest L2 (body cites moved entities)
runbook.md        {L-git}                     suggest L1 (commit drift only)
```

接力棒交到第 23 章：L1 用 `git ls-files` 以零 token 重新生成 `repo-map.md`；L2 拿着“移动实体 diff + 现有正文”对 `data-and-api.md` 调一次 LLM；L1 给 `runbook.md` 的 footer 重打一次时间戳。总成本：一次 LLM 调用，约 3 kB 上下文，不到一美分。人工评审范围：一份 diff（`data-and-api.md`）。对比一下朴素的“全部重嵌入”基线：那种做法会让评审者在几十个页面的 diff 里蹚来蹚去。

## 14.5   大型仓库与性能基准

### 规模很重要

本书的参考实现基于一个 19 kLOC 的 Go 仓库。对于小型项目而言，全量同步在十几秒内就能跑完，增量同步更是眨眼之间。但现实中的生产代码库远不止于此 —— 100k LOC 的中型服务、500k LOC 的大型平台、甚至 1M+ LOC 的 mono-repo 都很常见。当仓库规模跨越量级时，知识库的 entity 数量、graph edge 数量、向量索引体积以及查询延迟都会发生本质变化。

本节基于参考实现的实测数据，结合 RepoAgent {cite}`luo2024repoagentllmpoweredopensourceframework`、RepoBench 以及 Long Code Arena 等公开基准的外推，给出一张粗略但实用的规模预期表。

### 性能基准表

| Metric | 19k LOC | 100k LOC | 500k LOC | 1M LOC (mono-repo) |
|:--|--:|--:|--:|--:|
| Entities | ~850 | ~4,500 | ~25,000 | ~55,000 |
| Graph edges | ~3,200 | ~18,000 | ~100,000 | ~230,000 |
| Full sync time | 12 s | ~65 s | ~6 min | ~15 min |
| Incremental sync (10 files) | 1.2 s | 1.5 s | 2.5 s | 5 s |
| Vector index size | 3 MB | 16 MB | 90 MB | 200 MB |
| Graph DB memory | 50 MB | 250 MB | 1.2 GB | 2.8 GB |
| Query latency (p95) | 45 ms | 60 ms | 120 ms | 250 ms |

几个值得注意的趋势：

- **Full sync 是超线性增长的**：entity 解析、embedding 生成和 graph 写入三者都有各自的瓶颈。在 500k LOC 以上，全量同步的时间成本已经使其不适合在 CI 中作为常规步骤运行。
- **Incremental sync 是亚线性增长的**：由于增量同步只关心 diff 所涉及的文件和实体，其耗时主要取决于变更集大小而非仓库总量。即使在 1M LOC 的 mono-repo 中，10 个文件的增量同步也仅需约 5 秒。
- **Query latency 在 1M LOC 时依然可控**：p95 在 250 ms 以内，对于交互式 agent 来说仍在可接受范围。

### 规模化策略

当仓库突破特定规模门槛时，以下五种策略可以组合使用：

**(a) Mono-repo 分区（Monorepo partitioning）**

按顶层目录将仓库拆分为独立的 KB namespace。每个目录（通常对应一个 service 或 module）拥有自己的 entity 集、graph 子图和向量索引。跨 namespace 的依赖通过 `IMPORTS` 类型的 edge 进行链接。这种方式使每个分区的规模保持在"甜区"——通常 50k–100k LOC —— 从而让全量同步在分钟级完成，增量同步在秒级完成。

**(b) 超过 500k LOC 后只做增量同步（Incremental-only beyond 500k LOC）**

一旦仓库规模超过 500k LOC，全量重建的成本已经高到不值得定期执行。此时应完全依赖 L-git 和 L-entity 两层增量检测机制，仅在知识库初始化或灾难恢复时才执行全量同步。这要求 `commit:` footer 和反向索引始终保持准确 —— 一旦它们脱节，就失去了"跳过未变化页面"的能力。

**(c) Graph sharding**

将 graph 按 service 边界进行分片，每个 service 维护自己的子图。查询时通过 federated Cypher 在多个子图之间进行联合查询。这种方式的优势在于：单个子图的 memory footprint 可控（通常在 200–500 MB），且不同 service 的 graph 可以独立更新而互不阻塞。

**(d) Embedding 批处理流水线（Embedding batch pipeline）**

对于大型仓库，embedding 生成是最耗时的步骤之一。采用 queue + worker pool 模式：变更检测阶段将待嵌入的 entity 推入消息队列，多个 worker 并行消费并生成 embedding。对于非紧急的全量重建，可以安排在低峰时段（如凌晨）运行，避免与开发者的交互式查询争抢 GPU/API 配额。

**(e) Vector index 的分级选型**

不同规模的仓库适合不同的向量存储后端：

- **< 50k entities**：sqlite-vec 即可满足需求，部署简单，无需额外基础设施。
- **50k – 500k entities**：切换到 pgvector，利用 PostgreSQL 的成熟生态获得更好的并发性能和索引管理能力。
- **> 500k entities**：考虑 Milvus 或 Qdrant 等专用向量数据库，获得分布式索引、GPU 加速检索和水平扩展能力。

### Mono-repo 特有的挑战

Mono-repo 带来的不只是"仓库更大"这么简单，它在知识库同步层面引入了三类独特的复杂性：

**Vendor 目录过滤。** `node_modules/`、`vendor/`、`third_party/` 等目录可能包含数十万行代码，但它们不属于项目自身的知识。同步流水线必须在最早阶段就将这些目录排除在外 —— 不仅是为了性能，更是为了避免向量索引被第三方代码"污染"，导致检索结果偏离项目本身的语义。

**跨服务依赖的稠密图区域。** 当多个 service 共享一个内部 library 时，该 library 的 entity 会与大量 service 级 entity 产生 `IMPORTS` 和 `CALLS` edge，形成 graph 中的"稠密区"。这些区域会拖慢 graph traversal 查询，需要通过 edge pruning 或 materialized view 进行优化。

**多服务 PR 的增量同步。** 在 mono-repo 中，一个 PR 可能同时修改 5 个 service 的代码。增量同步必须能够正确识别出所有受影响的 service 分区，并在各分区内独立执行 L-git → L-entity → L-link 的三层检测。如果分区策略实现得不好，一次跨服务 PR 可能触发所有分区的全量扫描 —— 这就失去了分区的意义。

### 实践建议

> **不要在第一天就为 1M LOC 做优化。** 从一个 service 开始，证明知识库的价值，
> 然后再扩展到更多 service。过早地引入 graph sharding、分布式向量索引等重型
> 基础设施，只会增加运维成本而不会带来对应的收益。知识库的核心价值在于"让正确
> 的信息在正确的时间出现在正确的人面前" —— 这个价值在 19k LOC 上就能验证。
> 规模化是一个渐进的过程，而增量同步机制（本章）正是让这个过程成为可能的关键。

## Conclusion —— 小结

增量同步是代码层（第四部分）与文稿层（第二部分）之间的接口。它上游的所有东西都在谈 * 正确性 * ——“这个实体 ID 是否仍然指向同一件事？”“这个链接是否还能解析？”；它下游的所有东西都在谈 * 成本 * ——“这是一次免费的重打时间戳、一次便宜的重生成，还是必须交给人？”把这道接口做对，本书模型的两半就都能跑起来；做错了，便宜的层就会被迫干贵的活。

下一章会把各种门禁显式讲清楚：不管一个页面被哪一种更新级别处理过，它在发布前依然要通过同一套硬性检查（footer 存在、引用能解析、没有坏掉的 Mermaid）。检测与派发是第 14、16 章的事；门禁则属于第 22 章。

## 参考文献

```{bibliography}
:filter: keywords % "operations" or keywords % "deepwiki" or keywords % "drift" or keywords % "self-citation"
```
