---
title: "附录 D —— 术语表"
status: draft
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - appendix
  - glossary
---

# 附录 D —— 术语表

**译者按：关于 *prose* 与 *document* 。** 英文里有两个**意思完全不一样**的词，
英语世界的软件工程师通常不会把它们混为一谈：*prose* （**人写的叙述性内容本身**）
与 *document* （**承载内容的文件对象**，里面可以是 prose、可以是代码、
也可以两者皆有）。中文软件工程语境里往往把这两个词一并压扁成「文档」，
这会直接抹平本书第二、三部分一直依赖的关键区分。因此，中文版采用如下约定：

- **prose → 文稿** （叙述性内容本身；**文稿层** = the prose layer）
- **document → 文档** （文件对象、Diátaxis 分类、或被分级的制品，
  例如 document-layering 中的 document）
- **documentation → 文档工作 / 文档体系** （作为一项**工程活动**）

下面每一条术语与交叉引用都**遵循这套区分**；尤其请留意 *Prose* 、
*Document* 与 *L0–L4* 这三条。

```{glossary}
ADR
  **架构决策记录**（Architecture Decision Record）。一份**简短、只追加**
  的文档，用来记录**单一架构决策**的背景、决策本身以及其后果。
  MADR 是一个具体模板。 {cite}`nygard_adr` {cite}`madr`

BM25
  一族**从 tf-idf 演化而来**的排序函数，在全文检索中被广泛使用。
  {cite}`robertson2009probabilistic`

Diátaxis
  一种把文档分为四类的分类法：教程（Tutorial）、操作指南（How-to）、
  参考（Reference）、解释（Explanation）；第二部分通篇使用它。
  {cite}`procida_diataxis`

Entity
  在**代码知识图谱**（Code Knowledge Graph）中，一个可哈希的单元，
  例如文件、函数、类或包，拥有稳定的**内容寻址** ID。

HNSW
  **Hierarchical Navigable Small World**（分层可导航小世界）。一种
  **近似最近邻**（ANN）索引，是当下多数向量存储（包括 `pgvector`）
  的底层算法。 {cite}`malkov2020hnsw`

KB
  **知识库**（Knowledge Base）。在本书中，它指一个团队为了完成日常工作而
  查询的**文稿、代码、向量、以及图**的并集。

L0–L4
  第 25 章引入的**五层文档模型**：L0 代码与注释；L1 已提交的人写文稿
  （如 ADR）；L2 协作式人写文稿（如 wiki 页面）；L3 经人复核的 AI 草稿；
  L4 未经复核的 AI 草稿。第 25 章进一步把这个**单轴模型**升级为四元组
  ⟨L, U, R, S⟩。

Layer tuple
  四元分级 ⟨L, U, R, S⟩，分别表示文档层（Layer）、由谁更新
  （Updated-by）、复核状态（Review-status）、复核评分（Review-score）。
  第 25 章将它引入为**单轴文档分层**在工程上的替代方案。

L1 / L2 / L3 (update strategy)
  第 23 章提出的**三级更新策略**。**L1 mechanical**：只跑规则，零 LLM
  token。**L2 bounded-LLM**：只把一页正文与一个 diff 喂给 LLM，总量控制在
  几 KB 之内。**L3 human-led**：人来写，LLM 只做润色。不要把它与上面的
  文档层 L0–L4 混淆；这两条数轴不是一回事。

MCP
  **Model Context Protocol**。本书采用的标准协议，用来让 IDE 里的 agent
  查询知识库。 {cite}`mcp_spec`

MyST
  **Markedly Structured Text**。一种 Markdown 方言，Sphinx 可通过
  `myst-parser` 原生理解。

PKB
  **Project Knowledge Base**。范围只覆盖单个软件项目的知识库，与团队级或
  组织级知识库相对。本书推荐的十页骨架（从 `00-overview` 到 `09-runbook`）
  就是一个典型的 PKB 形态。

PKB 元数据页脚
  每一页内容页底部的一段 HTML 注释块，携带六个字段：`last_updated`、
  `commit`、`updated_by`、`review_status`、`review_score`、
  `reviewed_by`。第 5 章将它引入为 YAML frontmatter 的运维侧配对结构：
  frontmatter 记录来源（谁在什么 commit 创建了该页），页脚记录复核状态
  （谁签字、给了几分、在什么时候）。

Prose / 文稿
  **人写的叙述性文本**，例如教程、how-to、参考页、解释、ADR 与运维手册。
  在本书中，*prose* 是第二部分**文稿层**处理的对象，并被刻意拿来与 *code*
  对照，后者由第三部分的**代码层**通过解析与向量化来处理。这个区分之所以
  重要，是因为文稿以段落为自然切块单位，会相对于它所描述的代码发生漂移，
  并在“人判断它写错了”时更新；代码以函数为自然单位，不会漂移
  （它本身就是真理来源），而且每一次 commit 都在更新。第 15 章正是从这一分野
  开场。与 *Document* 对照阅读。

Document / 文档
  内容的**容器**，与里面装的是 prose 还是 code 无关。一个带 YAML
  frontmatter 的 Markdown 文件就是一个 document；它的正文可以是文稿、
  一段代码、或二者兼有。*Document-layering* （第 25 章，L0–L4）按来源与
  权威度给 document 分级，而不是按其内部内容类型分级。经验法则是：
  当我们指**可读内容本身**时，用 *prose* （文稿）；当我们指**承载它的文件对象**
  时，用 *document* （文档）。

RAG
  **检索增强生成**（Retrieval-Augmented Generation）。一种把 LLM 输出
  锚定在检索到的上下文之上的模式。 {cite}`lewis2020rag`
  {cite}`gao2023retrieval`

RRF
  **倒数排名融合**（Reciprocal Rank Fusion）。一种在混合检索中常用的
  多路排序融合方法。 {cite}`cormack2009rrf`

tree-sitter
  一个**增量式、多语言**解析器，支撑代码知识库的解析阶段。
  {cite}`brunsfeld2018treesitter`
```

## 参考文献

```{bibliography}
:filter: keywords % "foundations" or keywords % "ir-rag" or keywords % "adr"
```
