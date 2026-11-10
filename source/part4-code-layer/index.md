---
title: "第四部分 —— 代码层"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - code-layer
---

# 第四部分 —— 代码层

第四部分一图览 —— 六章把第二部分“wiki 好用、grep 不行”的痛点，升级成一个双栈（向量 + 图谱）检索器，能为结构化、标识符精确匹配以及模糊自然语言三类问题都给出带引用的答案：

```{mermaid}
mindmap
  root((Code Layer))
    Starting point
      ch15 Why code is not prose
      Parse Embed Relate
    Indexing pipeline
      ch16 Parser and entities
      ch17 Embeddings and stores
      ch18 Graph of relations
    Retrieval pipeline
      ch19 Three-tier hybrid
      ch20 Prompt and generation
    Properties shipped
      Structural queries
      Identifier-exact queries
      Fuzzy queries
      Citations in every answer
    Hand-off to Part V
      Drift and maintenance
      Evaluation
      Incremental sync
```

```{toctree}
:maxdepth: 1

ch15-why-code-is-not-prose
ch16-parsing-and-entity-extraction
ch17-embeddings-and-vector-stores
ch18-code-knowledge-graph
ch19-hybrid-retrieval-and-rrf
ch20-prompt-and-generation
```

## Why —— 为什么

第二部分的文稿层解决了一类问题 —— * “我们对 X 了解多少？” * —— 但对另一类 —— * “代码库里哪里用到了 X？” * —— 束手无策。第 15 章会把这个失败讲得更精确；第四部分其余各章则围绕“代码层参考实现”（路径位于 `reference-impl/code-kg/` 下）构建起一条能回答第二类问题的双栈（向量 + 图谱）流水线。

这个失败并不是 wiki 的遗漏，而是范畴错误。人会写一篇 * “同步是怎么工作的” * 的 wiki 页，却不会去写 * “谁调用了 `runSync`” * —— 因为前一句值得动笔写，后一句可以从代码推导。代码层知识库就是那台 * 持续做推导 * 的机器，让每个工程师都能在两秒内拿到答案，而不是 grep 二十分钟再滚动翻看。

## What —— 是什么

覆盖一切：在代码库上回答结构化、标识符精确匹配以及模糊自然语言三类问题，并 * 为每条结论给出引用 * 。按博客 §§2–8 {cite}`fanyamin2026deepwiki` 所划分的流水线阶段，每章对应一个阶段：

- **第 15 章 —— 为什么代码不是文稿。** 三个结构性论断（先解析、嵌入身份、图承载关系）；一个实测实验（纯向量 vs. 混合检索在真实代码查询上的表现）；四个让初次构建代码知识库翻车的反模式。
- **第 16 章 —— 解析与实体提取。** 一个真正的 parser（tree-sitter {cite}`brunsfeld_treesitter`）；一套封闭的实体 schema（`package`、`type`、`method`、`function`）；由 `sha256(repoID + filePath + entityType + name + startLine)[:16]` 生成的稳定 ID；资源泄漏、闭包噪声和多行签名陷阱。
- **第 17 章 —— Embedding 与向量存储。** 一个五行的身份模板（`Language / Type / Name / Signature / Doc`），在 recall@10 上大约以 2 倍优势胜过直接嵌入函数体；batch + backoff + rate-limit 纪律；`sqlite-vec` {cite}`sqlite_vec` / `pgvector` {cite}`pgvector` / `Milvus` {cite}`milvus` 的决策矩阵；基于 bi-encoder {cite}`reimers2019sbert`、cross-encoder {cite}`nogueira2019passage` 和 HNSW {cite}`malkov2018hnsw` 的理论锚点。
- **第 18 章 —— 代码知识图谱。** 在图数据库（如 Memgraph {cite}`memgraph`）上的八种封闭边分类法；位置感知的节点 ID；按仓库全量重建的写模式；基于图局部性和 GraphCodeBERT 数据流先验 {cite}`guo2021graphcodebert` 的理论锚点。
- **第 19 章 —— 混合检索与 RRF。** 三层策略（向量优先 → 关键词兜底 → 图扩展）；为什么分数级融合没意义，而排名级融合（RRF {cite}`cormack2009rrf`）才是升级路径；节省延迟的查询路由启发式。
- **第 20 章 —— Prompt 与生成。** 两条硬规则（必须引用 `file:line`；上下文不足时拒绝回答）；结构化上下文格式；可度量的 faithfulness {cite}`es2024ragas`；间接 prompt 注入威胁模型 {cite}`greshake2023injection`。

## How —— 怎么做

每章都至少附带一段来自 vendored 代码节选的 `{literalinclude}`（见 `examples/SOURCE.md`）、至少三条引用，以及至少一项博客中未出现的测量数据 —— 这是本书的“不照搬”原则。节选通过 `make book-refresh-excerpts` 刷新，并用 `make book-check-excerpts` 做字节级稳定性校验。

每一章也沿用第二部分立下的五段式骨架： * 周一早晨的钩子 * 把本章和你真实踩过的坑连起来；* Why / What / How * 段读起来像工程而非综述；* Theory anchor * 段（第 17–20 章）明确点出设计背后的已发表结果；* Common mistakes * 段列出反模式；* Example * 段给出可在附录 B 的代码层参考实现上复现的操作步骤。

## 运维模型

代码层继承了第二部分“先验证后相信”的思路，并做了针对代码的特化：

- **Build** —— `code-kg sync --repo-id <id>` 执行解析、嵌入并重建图。幂等；可安全重跑。
- **Verify** —— 在保留查询集（标识符、结构化、模糊）上的 recall@5 是首要检索健康指标。Prompt 指标（引用合规率、拒绝率）是次要的，且*仅在* recall 不再是瓶颈时才值得调。
- **Operate** —— 检索器优雅降级：没有 embedding API key → 纯关键词模式；图连接不可用 → 向量 + 关键词模式；没有向量 → 纯关键词模式。每种模式都仍然有用；没有哪种模式是硬性失败。

## Example —— 范例

附录 B 端到端跑一遍完整流水线：注册仓库、全量同步、按三个检索层各问一个问题，再对一行改动做增量同步，实测增量相对全量的加速比。

## 要点清单

若需要一屏内看完“从第四部分可以带走的六个关键决策”，直接跳到第 20 章末尾的 **“第四部分指针清单”** 段。每一条都是“写代码前要先拍板”的决策，而非事后总结。

## Conclusion —— 小结

有了第二部分和第四部分，这个知识库就能用带引用、可复现的方式同时回答文稿问题和代码问题。第五 —— 第七部分会让结果 * 在运维上持续可靠 * （漂移、评估、治理），并把它扩展到混合工作流（AI 编码助手、智能体）。

## 参考文献

```{bibliography}
:filter: keywords % "code-layer"
```
