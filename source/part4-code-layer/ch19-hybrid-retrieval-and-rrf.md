---
title: "第 19 章 —— 混合检索与 RRF"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - code-layer
  - search
  - ir
  - deepwiki
---

# 第 19 章 —— 混合检索与 RRF

本章一图览 —— 三层结构、一条升级路径，以及四类会把整套设计一脚踢翻的典型错误：

```{mermaid}
mindmap
  root((Hybrid Retrieval))
    Three tiers
      Vector primary
      Keyword fallback
      Graph expansion
    Query routing
      Semantic goes vector
      Identifier goes keyword
      Structural goes graph
    Theory anchor
      Ranks not scores
      RRF formula
      Upgrade path
    Common mistakes
      Score-level fusion
      Run all three always
      Graph as primary
      Caching at wrong layer
```

三个工程师，三种抱怨。Alice 说搜索太“字面”了 —— 她输入了 * “how does the sync retry” * ，拿到一份 500 行、名叫 `retry_utils.go` 的文件，和 sync 半点关系都没有。Bob 说它又太“模糊”了 —— 他原样输入 `runSync`，真正同名的那个函数却排在 * 第四 * 。Carol 说它根本拒绝回答 * “谁调用了 `runSync`” * —— 而 Alice 和 Bob 都没说错，Carol 也没说错，因为他们三人问的是三个不同的问题，每个问题需要不同的检索器。**本章讲的就是那套三层策略 —— 把每条查询路由到最适合的栈上 —— 以及 Reciprocal Rank Fusion 如何最终把这套 fallback 机制换成一份始终稳定好用的答案。**

## Why —— 为什么

第 17 章给了我们一份向量索引 —— 它在模糊的自然语言查询上非常出色，在标识符查询上只能算勉强够用。第 18 章给了我们一张图 —— 它在结构化查询上非常出色，在自然语言查询上几乎毫无用处。第 19 章的检索器，就是那一薄层：把每条查询路由到最适合回答它的那个栈；当拿不准时，再把它们的答案组合起来。

三个实证事实驱动着这份设计：

1.  *标识符查询胜过向量召回。* 当开发者输入 `entityID` 时，BM25 风格 {cite}`robertson2009bm25` 的打分器以显著优势碾压稠密模型。CodeSearchNet {cite}`husain2019codesearchnet` 记录了这一点；此后每篇代码 RAG 综述 {cite}`gao2023retrieval` 都在重复它。
2.  *结构化查询胜过两者。* 任何形如 *"谁调用了 X"*、*"什么实现了 Y"*、*"Z 返回什么"* 的查询，从图（第 18 章）上直接取出，recall = 1。
3.  *模糊查询受益于融合。* Reciprocal Rank Fusion {cite}`cormack2009rrf` 仅通过排名把异构检索器的有序列表组合起来 —— 不需要分数校准。在文稿上的混合搜索已被证明优于任何单一系统；GraphRAG {cite}`edge2024graphrag` 把这一原则扩展到了以图遍历作为第三个检索器。

最近的仓库级研究阐明了这对代码意味着什么。Zhu 等人关于 LLM 用于 KG 构建与推理的综述指出，前沿模型作为**结构化证据上的推理器**通常比作为从原始输入中 few-shot 提取证据的提取器更强 {cite}`zhu2024llmsknowledgegraphconstruction`。用第 19 章的话说，检索器的工作因此不仅仅是"找到相似的文本"，而是"组装一个小型的、有类型的证据集，让模型可以在上面推理而不用猜"。CGM 在仓库级规模上锐化了同一主张：一旦代码图成为模型输入或 attention 的一部分，仓库级任务就会提升 {cite}`tao2025codegraphmodelcgm`。我们的混合检索器是同一押注的外部化、运维友好版本。

这一系列工作中，还有三条同样重要的线索。Shrivastava 等人（RLPG）是 repo-level RAG 的概念起点：即使不微调模型，仅靠从仓库中智能选择跨文件上下文（import、使用示例、类型定义），就能显著提升代码补全效果 {cite}`shrivastava2023rlpg`。这个结论直接支撑了本章"上下文组装是检索的一等问题"的立场。RepoCoder 接着问：如果第一轮检索信号不够好怎么办？它的回答是"检索-生成-再检索"的迭代循环 —— LLM 的初步草稿变成了下一轮检索的更好锚点 {cite}`zhang2023repocoder`。迭代收敛通常在 2–3 轮就完成，但对跨文件复杂查询的命中率提升显著。Repoformer 则从另一个方向切入：不是每次查询都需要检索；它训练了一个轻量分类器来预测检索是否有益，无益时跳过 {cite}`li2024repoformer`。这条"选择性检索"的思路对应了本章检索路由层的设计 —— 当标识符查询在 BM25 上已经有高置信度命中时，跳过向量检索不仅省延迟，也降低噪声。

DeepWiki 方法论 {cite}`fanyamin2026deepwiki`（§7）规定了我们今天实现的这套策略： **向量优先，关键词兜底，图扩展** ；基于融合的混合检索被明确标注为“升级路径”。

## What —— 是什么

检索器有三层。

**1 —— 向量优先。** 如果配置了嵌入服务、且向量存储里已经有这个仓库的条目，我们就在“用户 query 的嵌入”上做 KNN（第 17 章）。按相似度顺序返回最多 `TopK` 个实体 ID。

**2 —— 关键词兜底。** 如果（a）enricher 不可用（没有嵌入 API key），（b）向量 KNN 调用失败，或（c）向量结果为空，我们就退回到进程内的关键词排序器。实体 * 名字 * 上的命中按正文命中的 3 倍加权。

**3 —— 图扩展。** 当向量/关键词那一层返回了种子集合后，可选地再从 KB 图上扩展 1–2 跳，这样生成器看到的不只是最匹配的实体，还包括它的调用方、它的被调方，以及（对 class 来说）它的方法。这是图在查询时的红利，不只是在构建时的红利。

第三层就是第 19 章真正触及 **graph-RAG** 的地方。图不是向量搜索上面的一层装饰；它是把一个语义匹配的种子变成一个*查询形状的局部子图*的机制。文档 QA 中的 GraphRAG 使用实体和社区结构来组装上下文 {cite}`edge2024graphrag`；在代码知识库里，同样的模式变得更严格、更局部：从一个实体出发，沿类型化的边（如 `CALLS`、`IMPLEMENTS`、`RETURNS`）扩展，然后把结果作为证据（而不仅仅是文稿）传给下游。

升级路径在代码里已经明确标注：用 Reciprocal Rank Fusion {cite}`cormack2009rrf` 把向量列表和关键词列表合并起来 —— 让检索器同时拥有 BM25 的“对标识符敏感”和稠密检索的“能泛化”。我们今天还没上线；第 22 章会（用实测数据）解释：为什么在我们当前语料规模下，fallback 策略仍然是 Pareto 最优。

### 理论锚点：RRF 融合的是排名，不是分数

2009 年以来每一款正经的混合检索器背后，都压着同一个承重思想。

**按分数加权融合的问题。** 把一个余弦相似度（范围 $[-1, 1]$）乘 $0.7$，再把一个 BM25 分数（范围 $[0, \infty)$，并会随 IDF 和语料长度漂移）乘 $0.3$，相加，得到的这个数 * 在不同查询之间并不可比 * 。对同一次查询的同一份文档，它在一个检索器上的分数可能是 0.82，在另一个上是 14.7；加权和反映的是 * 检索器的分数分布 * ，不是文档的相关性。干过这事的人最后都会发现自己在偷偷按语料、按模型版本调权重，然后一升级，所有调优瞬间蒸发。

**为什么排名就不一样。** Cormack、Clarke、Büttcher {cite}`cormack2009rrf` 的观察是：一份文档在某个检索器输出里占的 * 位置 * —— 也就是它的 rank —— 是一个无量纲的、被 $k$（top-k 截断）界定的量。因此，排名在不同检索器之间 * 无需任何按检索器的校准 * 就可以比较。Reciprocal Rank Fusion 只是简单地把倒数加起来： $$\text{RRF}(d) = \sum_{r \in R} \frac{1}{k_0 + \text{rank}_r(d)}$$ 其中 $k_0 \approx 60$ 是一个用来压低“非常靠后名次”权重的小常数。原论文已经证明 RRF 能匹敌甚至超过它那个时代的每一种基于学习的 rank-fusion 方法；之后十五年，学界也并没有产出一种稳定地胜过它的 * 简单 * 融合方式。

第 19 章的实操立场是：一旦参考实现的语料规模超过“fallback 仍然胜出”的那个阈值（第 22 章会测出这个阈值），RRF 就是那一刀精准的升级。把 `vector_empty ? keyword : vector` 换成 `RRF(vector.top(50), keyword.top(50)).top(TopK)`。实现只有十五行；那十五行之所以能跑赢一周的权重调优，恰恰来自上面那条理论保证。

## How —— 怎么做

关键词排序器是整个知识库里最简单、最乏味的组件 —— 然而它熬过了每一次重构。完整代码来自 `reference-impl/code-kg/retriever/keyword.go`：

```{literalinclude} ../examples/part3-code-layer/ch12/retriever_keyword.go
:language: go
:lines: 1-50
:caption: RankByKeyword — name hits count 3× body hits, then bubble-sort to topK.
```

这段代码里有两件事变得具体：

- 第 27 行（name 命中）加 **3 分**，第 30 行（body 命中）加 **1 分**。这个 3:1 比例 —— 而非某个 BM25 校准值 —— 就是在不增加实现负担的情况下模拟 IDF 风格标识符加权效果的方式。项目在第 7 章的文稿索引和第 19 章的代码兜底中都用同一比例；*一致性比逐层调参更有价值*。
- 第 38–44 行是冒泡排序，因为 `topK` 很小（在所有实测代码路径中 ≤ 50）。我们故意保持 O(n²)；换成 `sort.Slice` 能在一个 50 元素的 slice 上省下几微秒，但要多一个 import。这是本书反复做出的那个典型的工程级取舍：当规模上限很低时，选最明显、最无聊的实现。

### 总体编排

参考实现中 `Service` 的 `Search` 方法，把三层缝合在一起：

```{literalinclude} ../examples/part3-code-layer/ch12/service_search.go
:language: go
:lines: 486-596
:caption: Search orchestration — vector primary, keyword fallback, answer generation.
```

其中最关键的是四行：

- **第 494 行** —— 层选择条件：enricher 和向量存储必须同时可用。任何一个为 nil，我们就根本不尝试向量路径；直接跳到兜底（第 505 行）。这就是让一个没有任何 API key 的本地运行知识库仍然可用的关键。
- **第 498 行** —— 任何向量错误都降级到关键词兜底。降级以 Warn 级别记录日志，运维可以通过日志速率检测到慢性嵌入失败。
- **第 517–562 行** —— 向量路径本身：嵌入 query，`vectorStore.Search(queryEmb, TopK)`（一次 KNN 调用），按 ID 拉取匹配的 `Entity` 行，保持相似度顺序（`idOrder` map）。数据库过滤在 KNN *之后*应用；我们用少量的 KNN recall 换取按 `repo_id` 或 `entity_type` 过滤的能力。
- **第 564–575 行** —— 兜底：从数据库拉取所有候选（已过滤），跑上面的关键词排序器。这在大语料上不言而喻地慢；下面 §*升级路径* 一节解释我们计划如何改变。

结果集流入 `generateAnswer` —— 这是第 20 章的主题 —— 它会用一个强制“每条结论都带 `file:line` 引用”的提示词去调用 LLM。

### 图扩展（独立入口）

今天的图扩展并没有被折进默认的 `Search` 调用；它是通过服务方法 `AnalyzeEntity` / `GraphQuery` 显式发起的，这些方法需要一个种子实体 ID 和一个 `MaxHops`。分开的理由：图查询比向量 KNN * 贵得多 * （一次带 `shortestPath` 的 Cypher vs. 一次 HNSW 查找），所以我们把“什么时候付这笔成本”的决定权交给调用方。CLI 前端（`code-kg callers <name>`、`code-kg callees <name>`）只是围绕它的一层薄包装。

### 升级路径：RRF

当语料规模超过（比如）5 万个实体后，兜底关键词排序器的全表扫描就不再可接受了。升级路径如下：

1.  在同步阶段对 `(name, body)` 构建 token 级倒排索引（第 7 章 wiki 索引中已有的 `searchTerms` map 的自然延伸）。
2.  用一个查询倒排索引的 BM25 {cite}`robertson2009bm25` 打分器替换 `rankByKeyword`。
3.  用 Reciprocal Rank Fusion {cite}`cormack2009rrf` 融合向量和关键词的排名：`score(d) = Σ_{retriever} 1/(k + rank(d))`，`k = 60` 作为标准起始值。

Gao 等人 {cite}`gao2023retrieval` 综述了混合检索的全景；GraphRAG {cite}`edge2024graphrag` 把图谱作为第三个有序列表也加进同一份融合里。HippoRAG {cite}`gutierrez2024hipporag` 则展示同一模式可以通过联想记忆扩展到长程问答。retriever 包里的接口故意设计得很窄，让这些升级能直接替换进来，而不必触动 service 或 generator。

### Graph-RAG：通向仓库级任务的桥梁

把图扩展仅仅看作对 *"谁调用了 X？"* 的小众回答，是低估了它。一旦检索器能返回一个排过序的种子集加上一个有边标签的有界邻域，它就变成了一个通用的**仓库任务底座**：

- 一个使用工具的 IDE agent 可以调用检索器，然后在返回的邻域上追问；
- 一个无 agent 的批处理求解器可以在单次 prompt 或模型调用中消费同样的组装上下文；
- 一个图集成的阅读器（如 CGM）可以把同一份邻域当作结构化先验来使用，而不是从原始文件中重建 {cite}`tao2025codegraphmodelcgm`。

这就是本章把图扩展放在生成器外面、检索器里面的原因。图不仅仅是为了生成更好回答的后处理；它是那个让仓库上下文从"更多文本"变成*有类型的证据*的控制点。

## Example —— 范例

在同一份已同步知识库（第 17 章范例）上跑三条查询 —— 每条都被设计来命中一层：

```bash
# Tier 1 — vector primary. Fuzzy, natural-language.
code-kg search --repo-id demo "where do we retry embedding calls"
# → top hit: enricher.Service.GenerateEmbeddings
#   reference-impl/code-kg/enricher/service.go:67

# Tier 2 — keyword fallback. Identifier-heavy.
code-kg search --repo-id demo "RankByKeyword bubble sort"
# → top hit: retriever.RankByKeyword
#   reference-impl/code-kg/retriever/keyword.go:16

# Tier 3 — graph expansion (separate command). Structural.
code-kg callers --repo-id demo --name runSync
# → [TriggerSync, runIncrementalSync (recursion guard)] as in ch18.
```

第 1 层即使在 query 与结果几乎没有共享 token 的情况下也命中成功 —— 这就是稠密检索的价值。在模糊查询上第 2 层会被第 1 层拉开明显差距，但在标识符查询上它反过来胜出。第 3 层是唯一能以完美召回率回答结构化问题的那一层。

## 常见错误

以下四个检索层反模式，即使在前面几个阶段都做对了的团队身上也照样出现。

**1 —— 跨栈的分数级融合。** 症状是一个写成 `final_score = 0.7*cosine + 0.3*bm25` 的函数。它看起来很有章法，其实毫无意义。余弦相似度在 [−1, 1] 区间，BM25 分数上不封顶、且会随语料统计量漂移。* 同一份文档 * 的余弦可能是 0.82，而它的 BM25 可能是 14.7 也可能是 1.4，仅取决于它所在语料。修法是排名级融合（RRF）：`score = sum_r 1/(k + rank_r(d))`。它融合的是 * 排名 * ，排名在不同栈之间是可比的；不是 * 分数 * ，分数不可比。

**2 —— 把三层当同一类查询对待。** 症状是：流水线对每条 query 都并行跑 vector + keyword + graph，然后从它们的并集里取 top-k。你花了 3 倍的延迟去给三种本该得到不同答案的问题排序。修法是第 19 章的两段式设计：一个分类器（* 这是不是结构化查询？ * ）把结构化查询路由到“图优先”；其他所有查询都走向量路径，* 只有当向量结果为空时 * 才落到关键词兜底。

**3 —— 把图扩展当成首轮检索。** 症状是： * “how does the sync retry work” * 这条查询返回了一段从人工挑选的种子节点出发的图游走 —— 然后因为种子本身就挑错了，所有合法命中全被漏掉。图是用来回答“已经拿到种子之后”的追问（* “还有谁也调用了它” * ），不是用来做首轮语义匹配的。把图扩展放在一个独立的端点上，并且把“种子实体 ID”作为 * 必填 * 参数。这一处设计选择就能挡住 80% 的误用。

**4 —— 在错误的层缓存。** 症状是一个 `query → top-k entityIDs` 的进程内 LRU 缓存，在一个 commit 落地的瞬间就失效，因为缓存没有挂钩到索引器。如果你一定要缓存，缓存 query 字符串的*嵌入*（稳定的）或者按 repo-ID + commit-SHA 粒度缓存图的*邻接关系*（也是稳定的）。永远不要缓存最终的排名列表，除非你能在每次写入时都做失效处理。

## Conclusion —— 小结

对于一个必须同时回答自然语言、标识符和结构化查询的知识库，三层检索器是最小可行设计。代码层参考实现能保持小巧，是因为每一层要么是一个单屏幕大小的组件（关键词），要么是对一种专用存储的薄薄包装（向量 / 图）。RRF 和 BM25 是写进文档的升级路径；两者我们今天都还没上线，第 22 章会测出那是否仍然是正确的决定。

第 20 章将以最后一段来收束第四部分：把检索器给出的结构化上下文变成一份可被引用的答案。

## 参考文献

```{bibliography}
:filter: keywords % "search" or keywords % "ir" or keywords % "code-layer"
```
