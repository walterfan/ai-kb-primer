---
title: "第 26 章 —— 图引导的上下文组装"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: partial
keywords:
  - hybrid
  - graph-rag
---

# 第 26 章 —— 图引导的上下文组装

## Why —— 为什么

一个返回"最相关的 5 页"的检索器还不是一个仓库助手。仓库问题很少想要五条孤立的命中。它想要的是一份**小型的、有连接的解释**：函数本身、它的调用方、那份为此行为提供理据的 ADR、那份指出了故障模式的 runbook —— 并且其中只包含已评审过的页面。原始的 top-k 对于一个搜索框够用了；对于一个必须回答 *why*、*where else* 和 *what changed* 而不能瞎猜的 LLM，它不够。

本章恰好坐落在这个边界上。第 19 章检索了种子：向量用于模糊意图、关键词用于标识符、图用于结构。第 25 章给每个文稿节点一个元组 $\langle L, U, R, S \rangle$ 来表示它有多值得信任。第 29 章将把最终的知识库以工具的形式暴露给 agent。缺失的那一步，就是把一份排了序的种子列表变成一个 LLM 可以安全推理的**有界的、有类型的上下文包**。

近期文献让这一步更容易命名。GraphRAG 表明图结构不仅对存储有用，而且对**以查询为中心的上下文构造**有用 {cite}`edge2024graphrag`。HippoRAG 把同一直觉推向长期记忆 {cite}`gutierrez2024hipporag`。Zhu 等人的综述解释了为什么这不仅仅是一个检索技巧：LLM 在**对结构化证据进行推理**方面，往往比按需从原始语料中提取该结构要强得多 {cite}`zhu2024llmsknowledgegraphconstruction`。CGM 是同一思路在仓库级上的模型端延续 {cite}`tao2025codegraphmodelcgm`。本章是本书的检索端回答：保持图的显式性，但用它来组装模型应该看到的那个精确的证据包。

## What —— 是什么

**图引导上下文组装**是检索与生成之间的那一步，它把：

$$
(\text{query},\ \text{seed hits},\ \text{graphs},\ \text{tuple labels},\ \text{budget})
\rightarrow \text{context packet}
$$

上下文包是生成器或 agent 真正消费的东西。它不是"top-k 文档"。它是一个**被渲染为证据的有界子图**。

三项性质定义了这个包：

1.  **有种子的。** 组装从第 19 章检索到的种子出发；它不会盲目地遍历仓库图。
2.  **有类型的。** 扩展保留了关系标签，如 `CALLS`、`IMPLEMENTS`、`RETURNS`、页面到页面的链接和页面到代码的引用。包应该解释每个节点*为什么*出现在这里。
3.  **按权威性和预算过滤的。** 来自第 25 章的元组元数据和一个 token 预算，决定了什么能存活进最终的包。

本章刻意把问题范围限制得很窄。它**不是**关于：

- 构建新的图数据库 schema；
- 替换第 19 章的检索器；
- 把整个仓库总结成一个巨型 prompt；
- 学习一个像 CGM 那样的图集成模型。

它关注的是那个运维层的中间步骤 —— 使用工具的 agent 和无 agent 的仓库阅读器都需要的：把检索到的种子转化为一个与问题匹配的小型邻域。

### The heterogeneous graph the packet comes from

组装层推理的对象是一张**异构的仓库图**，不仅仅是第 18 章的代码图。它至少包含：

| Node kind | Examples | Main metadata |
|:--|:--|:--|
| Code entity | function, method, type, file | stable ID, path, signature, lines |
| Prose page | ADR, runbook, architecture note, chapter page | tuple $\langle L,U,R,S\rangle$, frontmatter, footer |
| Citation anchor | `file:line`, page section, include target | target path, line span, resolution status |

它还包含至少三个边族：

| Edge family | Examples | Purpose |
|:--|:--|:--|
| Code structure | `CALLS`, `IMPLEMENTS`, `RETURNS`, `CONTAINS` | structural neighbourhood |
| Prose structure | page links, section references, includes | documentation neighbourhood |
| Cross-layer links | page cites entity, ADR explains file, runbook points to path | prose ↔ code bridge |

重要的不是每条边是否都物化在一个物理存储中。重要的是组装层能像它们构成了一张有类型的图一样去遍历它们。

## How —— 怎么做

### Step 1 — Retrieve seeds, do not answer yet

第 19 章返回一个排序后的种子集合。这个种子集合是一份*提议*，还不是上下文。不同的查询形状产生不同的种子特征：

- **语义查询**：种子通常是稠密检索找到的一两个实体或页面。
- **标识符查询**：种子通常是关键词搜索返回的一个精确符号。
- **结构化查询**：种子通常是一个符号加上它的显式图邻域。
- **Why 查询**：种子通常是混合的 —— 一个代码实体加上一个文稿页面或 ADR。

本章的第一条纪律因此很简单：**不要让生成器直接消费种子列表**。一条 top hit 只是进入本地仓库邻域的入口。

### Step 2 — Expand locally, with relation policy

组装然后围绕种子执行一次有界的图扩展。有界意味着同时是**跳数受限的**和**关系受限的**。沿所有可能的边扩展两跳通常太多了；沿正确的边扩展一跳通常就够了。

一个有用的默认策略是：

| Query shape | Preferred expansion |
|:--|:--|
| "where / who calls / what implements" | code edges first (`CALLS`, `IMPLEMENTS`, `CONTAINS`) |
| "why / how / what is the rationale" | cross-layer links first (ADR, architecture page, reviewed runbook) |
| "what changed / what breaks" | code neighbours plus pages citing the changed entities |

正是这套关系策略，让图引导组装不同于"把图邻居追加上去"。图不只是增加 recall。它选择的是让问题*可回答*的那*类*周围证据。

### Step 3 — Filter and weight with the tuple

此时包中往往包含了超出上下文窗口承受能力的相关材料。第 25 章的元组在这里变成了承重件。一个被图引用的页面，如果是 AI 起草的且待评审，喂给模型仍然可能是坏事。

一个实用的打分草案：

```text
final_score(node) =
    seed_score(node)
  * relation_weight(edge_to_node)
  * hop_decay(hops_from_seed)
  * tuple_weight(node.tuple)
  * freshness_weight(node.commit_age)
```

Where:

- `seed_score` comes from Chapter 12 (vector / keyword / graph / RRF);
- `relation_weight` 对 "why" 问题偏好解释性边，对 "where" 问题偏好结构性边；
- `hop_decay` 惩罚离种子远的节点；
- `tuple_weight` 降权 `pending` 或低分的文稿；
- `freshness_weight` 是可选的，但在两页同等相关而其中一页明显更新时很有用。

关键思想不是精确的公式。而是**上下文组装是检索相关性与文档权威性最终会合的地方**。

### Step 4 — Render the packet as evidence, not as a dump

最终的包应该保留谱系和关系线索。一条好的包条目不仅要说*包含了什么*文本，还要说*为什么*：

```text
[seed] Function runSync
  file: reference-impl/code-kg/service.go:201-248
  reason: keyword seed

[neighbor via CALLS<-] Function TriggerSync
  file: reference-impl/code-kg/service.go:161-184
  reason: caller of runSync

[neighbor via EXPLAINS] ADR-0014 Panic-safety in worker pools
  tuple: ⟨L1, human, approved, 4⟩
  reason: cited by runSync doc comment and mentions panic recovery
```

这就是通往第 29 章的桥梁。一个像 `kb.search` 这样的 MCP 工具不需要暴露原始图遍历；它可以暴露一个现成的包，其条目已经包含了关系标签、元组标签和可解析的锚点。

### Step 5 — Reuse the same packet for both agents and agentless tasks

保持组装层显式化的一个原因是，它同等地服务于两种下游消费者。

**使用工具的 agent。** 一个 IDE agent 问一个问题、收到一个包、读一两页、然后追问。第 29 章的 `kb.search` 和 `kb.cite` 就是这种模式。

**无 agent 的仓库任务。** 批量评估器、补丁生成器或 CGM 风格的阅读器可能更倾向于对预组装好的邻域做一次性推理 {cite}`tao2025codegraphmodelcgm`。包的格式仍然有用：它是模型应该关心的本地子图在检索时的显式表示。

RepoAgent 在这里是互补的、而非竞争的工作。RepoAgent 改善的是仓库知识的**生产端** —— 通过持续循环地做全局结构分析、文档生成和文档更新 {cite}`luo2024repoagentllmpoweredopensourceframework`。图引导组装改善的是**消费端** —— 通过在运行时让那些被持续维护的仓库记忆变成查询形状的。

### The operational rule: assemble locally, never globally

graph-RAG 系统中最大的失败模式是把图访问变成过度抓取的许可证。仓库图足够稠密，几次不小心的跳转就能让包爆炸。最安全的运维规则是：

> **全局检索，局部组装。**

全局搜索找到正确的种子。局部图组装构建可回答的邻域。颠倒顺序，就是系统把半个仓库展示给模型然后管它叫"上下文"的由来。

## Example —— 范例

假设开发者问：

> *"`runSync` 为什么恢复 panic 而不是让 worker 崩溃？"*

### 1. Seeds from Chapter 12

混合检索返回：

1.  `service.go` 中的 `runSync` 函数
2.  ADR 页面 `ADR-0014 Panic-safety in worker pools`
3.  `runIncrementalSync` 作为排名较低的结构性邻居

### 2. Local graph expansion

Assembly follows:

- `CALLS` / reverse-`CALLS` edges around `runSync`
- page-to-code citation edges from `ADR-0014`
- 从 `ADR-0014` 到事故 runbook 的页面到页面链接

这产生了一个原始的 11 节点邻域。

### 3. Tuple and budget filtering

两个节点被丢弃：

- 一个 AI 生成的摘要页面，元组为 `⟨L3, ai, pending, 0⟩`
- 一份超出 token 预算的过期事故复盘

五个节点存活，排列为最终的包。下面的包是示意性的；精确的行范围随仓库和 commit 而异：

```text
1. [seed] runSync
   file: reference-impl/code-kg/service.go:<runSync span>
   why included: exact function under discussion

2. [code neighbor] TriggerSync
   file: reference-impl/code-kg/service.go:161-184
   why included: caller of runSync

3. [prose bridge] ADR-0014
   tuple: ⟨L1, human, approved, 4⟩
   why included: rationale for panic recovery policy

4. [ops bridge] worker-pool incident runbook
   tuple: ⟨L2, ai+human, approved, 3⟩
   why included: describes failure mode and on-call handling

5. [structural neighbor] runIncrementalSync
   file: reference-impl/code-kg/service.go:278-314
   why included: recursion / re-entry context
```

### 4. Generator output

LLM 不再需要仅凭原始文件推断理据。它可以回答：

```text
`runSync` recovers panics so a single failed sync target does not crash
the whole worker loop. The implementation is in the `runSync` span in
`reference-impl/code-kg/service.go`, and the policy rationale is
recorded in `ADR-0014`, which states that worker-pool panics should be
contained and logged rather than allowed to terminate sibling work.
`TriggerSync` is one caller (`service.go:161-184`), and
`runIncrementalSync` participates in the same control path
(`service.go:278-314`).
```

这个回答比单纯的 top-k 检索更好，不是因为它看到了更多文本，而是因为它在一个显式的权威性过滤器下看到了**正确的局部子图**。

## Conclusion —— 小结

图引导上下文组装是把知识库从一个搜索系统变成仓库记忆的那一步。第 19 章给了我们种子；第 25 章给了我们权威性标签；本章把两者组合成一个 LLM 真正可以推理的有界证据包。

更广泛的研究脉络现在看起来是连贯的。Zhu 等人解释了为什么结构化证据有助于 LLM 推理。GraphRAG 和 HippoRAG 解释了为什么图形状的上下文在很多场景下击败了平坦的 top-k。RepoAgent 表明仓库知识必须被持续维护。CGM 表明同样的图信号可以移向更靠近模型本身的位置。本章的赌注是这条脉络的保守系统版本：保持图和元组的显式性，在查询时组装局部上下文包，让使用工具的 agent 和无 agent 的仓库阅读器消费同一套被持续维护的底座。

## 参考文献

```{bibliography}
:filter: keywords % "hybrid" or keywords % "graph-rag"
```
