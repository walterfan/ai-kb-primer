---
title: "第 32 章 —— 展望与尚未解决的问题"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - governance
---

# 第 32 章 —— 展望与尚未解决的问题

## Why —— 为什么

当读者翻到本书最后一章时，工程问题已经基本回答了：一个团队今天就可以用解析器、向量索引、一张小型代码图、一个有纪律的文稿层和一个维护循环来构建软件知识库。开放的问题不再是 *"LLM 能不能帮上仓库知识？"*，而是 *"结构应该住在哪里，谁来维护它，以及一旦结构存在，模型的职责是什么？"*

最能框定这个问题的三篇论文在短时间内接连出现。Zhu 等人综述了 LLM 用于 KG 构建与推理的工作，发现了一个应该让系统构建者警醒的模式：前沿模型作为**结构化上下文上的推理器**往往比作为从原始输入中 few-shot 提取结构的提取器更令人信服 {cite}`zhu2024llmsknowledgegraphconstruction`。RepoAgent 把仓库文档变成了一个包含全局结构分析、文档生成和更新的一等系统问题 {cite}`luo2024repoagentllmpoweredopensourceframework`。CGM 则更进一步，问仓库图结构是否应该进入模型本身；它的答案是肯定的 {cite}`tao2025codegraphmodelcgm`。

合在一起，这些论文把"AI 用于代码库知识"从一堆战术变成了一个研究计划。本章命名的是那个计划中仍然令人不安的部分。

## What —— 是什么

本章对**展望**的使用是狭义的：不是市场预测，而是即使接受了本书的论点，仍然保持开放的那些设计选择。六个问题最为重要。

1.  **提取 vs. 推理。** 哪些事实仍应由确定性解析器来导出，哪些可以留给 LLM 在查询时处理？
2.  **生成 vs. 维护。** 仓库文档主要是一个综合问题，还是一个重心在增量更新的生命周期问题？
3.  **外部图 vs. 集成图。** 代码图应该作为检索端底座留在模型外面，还是进入模型的 attention 和隐藏状态？
4.  **长上下文 vs. 选择性上下文。** 当上下文窗口越来越大时，检索变得不那么重要了，还是仅仅变得更有选择性了？
5.  **基准得分 vs. 运维信任。** 团队应该优化什么：SWE-bench 风格的解决率、文档覆盖率、引用精度，还是人工评审负载？
6.  **单源知识库 vs. 多源联邦。** 知识库应该把所有来源（OpenSpec、AI 方案、Confluence、Notion）的内容都摄入同一个存储，还是联邦式地跨来源搜索、在展示层合并结果？

本章余下部分的论点是：这不是六个不相关的问题。它们是对同一个系统边界的六种视角。

## How —— 怎么做

### 1. The field is converging on a split architecture

Zhu 等人为一个本书当作工程常识来处理的设计原则提供了最干净的实证依据：**不要让 LLM 做你唯一的提取器**。他们的综述发现，大模型在对 KG 上下文做下游推理方面，往往比在 few-shot 提取实体、关系和事件方面更强 {cite}`zhu2024llmsknowledgegraphconstruction`。对于软件知识库，这几乎可以直接转化为一种分工：

- 解析器提取实体、行范围和稳定标识符；
- 校验器检查链接、锚点和谱系；
- 检索器组装最小可用的证据集；
- LLM 负责解释、总结、拒答和路由。

即使模型不断变强，这种拆分依然有吸引力。模型越强，把它的预算花在高层综合而非解析器或编译器已经能确定性完成的重复结构性工作上，就越有价值。

### 2. Repository documentation is a maintenance problem

RepoAgent 之所以重要，不是因为它证明了 LLM 能写 Markdown，而是因为它把**仓库级文档当作一条有显式阶段（结构分析、生成、更新）的流水线**来对待 {cite}`luo2024repoagentllmpoweredopensourceframework`。这与本书第二部分到第五部分的契合度远超一个通用的"AI 文档"工具。开放问题不仅仅是页面质量；而是如何在仓库不断变动的情况下保持文档同步，又不产生一个比文档本身的收益还大的评审负担。

这就是本书为什么在增量同步、元组元数据和发布门禁上花了这么多篇幅。在实践中，在软件知识库上失败的团队很少是因为初稿写得差。它们失败是因为更新纪律缺失、评审队列停滞，或者引用悄悄腐烂。

### 3. Graphs are moving from retrievers into models

CGM 是迄今为止最清晰的信号：仓库图不仅仅是外部工具的一种实现技巧。它把仓库图结构集成进模型的 attention 机制，并在与图感知检索搭配时报告了仓库级的增益 {cite}`tao2025codegraphmodelcgm`。这个结果不会推翻本书的外部图设计；它阐明了其升级路径。

今天，把图放在模型外面有三个务实的优势：

- 图是可检查的，普通工具就能查询它；
- 更新可以在每次 commit 时发生，无需重新训练模型；
- 多个 agent 和检索器可以共享同一底座。

但 CGM 暗示，一个成熟的软件知识库栈最终可能变成**双层的**：一个显式的外部图用于运维和谱系，加上一个图感知模型，以比纯文本 prompting 更高效的方式消费同一份结构。

### 4. Long context does not remove the need for retrieval

更大的窗口改变的是检索压力*在哪里*，而不是它*是否存在*。长上下文模型可以一次吃进更多文件，但仓库中仍然有很多信号不应该被扁平化为原始文本：调用边、实体身份、评审状态、谱系、陈旧度和策略。真正的竞争不是"RAG 对决长上下文"；而是**有类型的上下文组装对决无差别的填充**。

对软件仓库来说尤其如此，因为很多有用的查询不是"总结这些文件"式的查询。它们是"哪个符号真正拥有这个行为？"、"自从那份 ADR 之后什么变了？"、以及"这个回答安全可信吗？"这类问题。这些问题需要选择、排序和策略感知的过滤，即使一个模型原则上可以读更多字节。

### 5. Benchmarks still underspecify operational quality

CGM 的 SWE-bench Lite 结果很有价值，因为它给这个领域一个共享的仓库级目标 {cite}`tao2025codegraphmodelcgm`。RepoAgent 的文档评估很重要，因为它们迫使系统构建者去度量页面质量，而不仅仅是代码补丁 {cite}`luo2024repoagentllmpoweredopensourceframework`。然而这两类基准都没有捕获本书视为不可协商的若干性质：

- 每个回答是否都带有可解析的 `file:line` 引用；
- 陈旧页面是否被降权或被阻止发布；
- 更新成本是否随 commit 频率而非知识库规模扩展；
- 人工评审是否在仓库增长时保持有界。

下一代软件知识库基准应该度量以上全部四项。一个在补丁解决率上得分很高、但无法告诉你其答案来自哪个 commit 的系统，是令人印象深刻的，但仍然没准备好成为一个组织的记忆。

### 6. Single-source ingestion vs. multi-source federation

第 31 章建立了摄入的信任模型，但它隐含了一个假设：所有内容最终都被**导入**知识库。实际上还有另一种选择 —— **联邦**（federation）：知识库不复制外部内容，而是在搜索时同时查询 Confluence、Notion、内部知识平台 等来源，在展示层合并结果。

两种策略的取舍是清晰的：

| 维度 | 摄入（Ingest） | 联邦（Federate） |
|:--|:--|:--|
| 出处控制 | 强 —— 每页都有 frontmatter 和出处元组 | 弱 —— 依赖来源端的元数据 |
| 陈旧检测 | 知识库内闭环 —— 第 23 章漂移检查适用 | 依赖来源端 —— 来源不报告陈旧，知识库也不知道 |
| 维护负担 | 高 —— 导入后每页都是自己的维护对象 | 低 —— 来源端自己维护 |
| 搜索质量 | 高 —— 统一索引、统一排序 | 中 —— 跨源排序困难，元数据不一致 |
| 隐私 | 可控 —— 内容在知识库围墙内 | 依赖来源端 —— 联邦查询可能穿越安全边界 |

经验法则：**高质量、稳定的来源可以联邦；低质量或快速变化的来源必须摄入并治理。** 一个由专职技术写作团队维护的 Confluence 空间，可能联邦就够了 —— 它自己有评审流程，内容更新频率可控。一个由已离职工程师三年前写的、此后无人维护的 Notion 页面，必须摄入、分类、标记陈旧度，否则它就是一个沉默的误导源。

这个问题目前没有好的通用答案。本书的赌注是偏向摄入（因为出处控制和门禁是不可协商的），但承认联邦在特定场景下成本更低。未来的知识库框架可能需要同时支持两种模式，并提供一个决策规则来帮助团队选择。

## Example —— 范例

一个想跟随研究前沿但不想押上公司身家的团队的现实三步路线图：

| Stage | Keep fixed | Add experimentally | Success criterion |
|:--|:--|:--|:--|
| **1. Durable KB** | parser, stable IDs, hybrid retrieval, citations | none | engineers get grounded answers with low review overhead |
| **1.5 Multi-source ingest** | same retrieval stack + provenance model | OpenSpec / AI spec / doc-site import with 第 31 章 trust tiers | all ingested pages carry provenance; gate failures < 5% |
| **2. Continuous docs** | same retrieval stack | RepoAgent-style doc-update loop over selected page classes | candidate pages per commit stay small and reviewable |
| **3. Graph-aware models** | external graph remains source of truth | CGM-style graph-conditioned reader or reranker | better repo-level task accuracy *without* losing provenance |

顺序很重要。第 3 阶段很诱人，但它应该坐在第 1 和第 2 阶段之上，而不是替代它们。一个没有稳定谱系纪律的图感知模型是一个 demo。一个有增量维护和可信引用的朴素检索器已经是一个系统了。

## Conclusion —— 小结

近期文献中最持久的教训不是某一种架构赢了。而是这个空间变得可读了。Zhu 等人告诉我们要把提取和推理分开。RepoAgent 告诉我们仓库知识的生死取决于更新循环。CGM 告诉我们代码图重要到值得移向更靠近模型的位置。

因此本书的赌注是保守的：先构建显式的知识库，因为显式结构是可检查的、可维护的、可治理的。然后让更好的模型以更丰富的方式消费那些结构。如果这个领域的方向是对的，未来的系统会更少像"把仓库贴进一个更大的窗口"，更多像**仓库记忆栈**：解析器驱动、图形状、增量维护，且只在 LLM 确实是最佳工具的地方才由 LLM 介入。

## 参考文献

```{bibliography}
:filter: keywords % "governance"
```
