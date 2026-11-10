---
title: "第六部分 —— 混合层"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - hybrid
---

# 第六部分 —— 混合层

第六部分一图览 —— 五章把第二部分的文稿层和第四部分的代码层融合成一个统一的知识库栈：跨层链接把文稿和代码连成一张可遍历的图；文档分层元组让检索器按权威性过滤；图引导上下文组装把 top-k 种子扩展为 LLM 可安全推理的有界证据包；AI 编码助手和 agent 通过 MCP 工具消费这个知识库：

```{mermaid}
mindmap
  root((Hybrid Layer))
    Bridge
      ch24 Cross-layer links
      CITES EXPLAINS DOCUMENTS
      Prose ↔ Code traversal
    Layering
      ch25 Document tuple L U R S
      Single-axis L0–L4 → four-tuple
      Retrieval weight = f(tuple)
    Assembly
      ch26 Graph-guided context
      Seed → Expand → Filter → Render
      Budget-aware evidence packets
    AI coding
      ch27 KB-backed coding assistants
      Design rationale awareness
      Verifiable citations
    Agents
      ch29 MCP tools on the KB
      kb.search / kb.read / kb.cite
      Edge deployment pattern
```

```{toctree}
:maxdepth: 1

ch24-prose-meets-code
ch25-document-layering-L0-L4
ch26-graph-guided-context-assembly
ch27-harnessing-ai-coding
ch28-team-sdlc-integration
ch29-ai-agents-on-the-kb
```

## Why —— 为什么

第二部分和第四部分各自能回答一类问题 —— 文稿层回答 *"我们对 X 了解多少？"*，代码层回答 *"代码库里哪里用了 X？"* —— 但真正有价值的仓库问题几乎都是跨层的：*"为什么 `runSync` 恢复 panic 而不是让 worker 崩溃？"* 答案一半在代码（`recover()` 的位置），一半在文稿（ADR-0014 的 panic 策略论述）。混合层就是那座桥：让知识库成为一个**不分文稿和代码、按权威性分层、按图结构组装**的统一记忆体。

## What —— 是什么

六章内容覆盖从桥梁到消费者的完整链路：

- **第 24 章 —— 文稿与代码相遇。** 三种跨层链接类型（`CITES`、`EXPLAINS`、`DOCUMENTS`），从现有锚点和命名约定机械提取，物化在第 18 章的仓库图中。这是混合层的结构基础。
- **第 25 章 —— 文档分层 L0–L4。** 从博客的单轴权威性模型升级为四元组 $\langle L, U, R, S \rangle$（层级、更新者、评审状态、评分），与第 5 章 footer 字段整合，在检索和发布两个阶段使用。
- **第 26 章 —— 图引导的上下文组装。** 把检索种子、异构图、元组标签和 token 预算组合成一个有界的、有类型的上下文包 —— 生成器或 agent 真正消费的东西，而不是 top-k 文档。
- **第 27 章 —— 用知识库驾驭 AI 编码。** 知识库如何为 AI 编码助手提供三种它原本不具备的能力：设计理据感知、引用可校验、权威性分层。三种典型场景（上下文注入、代码审查辅助、重构导航）和上下文预算纪律。
- **第 29 章 —— 在知识库上跑的 AI Agent。** 知识库作为 MCP 服务器暴露三个工具（`kb.search`、`kb.read`、`kb.cite`），以及 edge function 部署模式。协议半和模式半的完整论述。

## How —— 怎么做

六章从结构到消费依次递进。第 24 章建桥（跨层链接），第 25 章给每个节点贴标签（元组），第 26 章用桥和标签组装上下文包，第 27 章让 AI 编码助手消费这个包，第 28 章把流程融进团队 SDLC，第 29 章把知识库变成任何 agent 可调用的一等工具。每一章都独立可读，但合在一起构成了从"两个分离的索引"到"一个统一的仓库记忆"的完整升级路径。

## Example —— 范例

一次完整往返：开发者在 IDE 中问 *"`runSync` 为什么恢复 panic？"*。AI 助手调用 `kb.search` 得到种子（第 19 章），沿跨层链接（第 24 章）扩展到 ADR（第 26 章），用元组过滤掉未评审页面（第 25 章），回答时引用代码位置和 ADR 段落（第 27 章），每个引用都来自 `kb.cite` 且可解析（第 29 章）。

## Conclusion —— 小结

混合层是让知识库从"文档搜索引擎"进化为**团队记忆**的那一层。它不是文稿层和代码层的简单合并 —— 它加入了跨层结构、权威性分层和预算感知的组装纪律。之后的第七部分讨论的治理问题（成本、信任、展望），全部建立在本部分交付的基础设施之上。

## 参考文献

```{bibliography}
:filter: keywords % "hybrid"
```
