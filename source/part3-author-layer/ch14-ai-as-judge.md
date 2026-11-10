---
title: "第 14 章 —— AI 作为评判者：评估生成质量与引用合规"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - author-layer
  - evaluation
  - judge
---

# 第 14 章 —— AI 作为评判者：评估生成质量与引用合规

本章收束第三部分：当 AI 已经能起草、分类、抽取、摘要之后，怎么判断这些产物是否值得进入知识库？答案不是“再让 AI 自己说好不好”，而是把 **judge** 当作一个受校准、受约束、永远不能盖章的辅助评审者。它可以放大人工 review 的覆盖面，但不能替代人的最终批准。

```{mermaid}
flowchart LR
    A[AI-authored artifact] --> B[Rubric]
    A --> C[Sources / citations]
    B --> D[AI judge]
    C --> D
    D --> E[Score + reasons + failed checks]
    E --> F[Human reviewer]
    F -->|approve| G[approved]
    F -->|reject| H[pending / rewrite]
```

## Why —— 为什么

第三部分前四章把 AI 放进生产侧：写草稿、打标签、抽实体、做维护。每一步都扩大了 KB 的覆盖面，也扩大了错误进入 KB 的入口。只靠人工全量审查，成本会回到原点；完全不审，幻觉会复利。judge 的位置就在中间：它不做最终决定，但它把 review 从“读完整篇再凭感觉判断”变成“按 rubric 看失败项”。

LLM-as-judge 已经成为评测领域的常见做法。G-Eval 用大模型按 rubric 评估生成文本，并显示出比传统自动指标更贴近人工偏好的结果 {cite}`liu2023geval`；MT-Bench / Chatbot Arena 则系统讨论了“让强模型评弱模型”的可行性与偏差 {cite}`zheng2023judgingllm`。RAGAS 把 faithfulness、answer relevancy 等指标做成 RAG 流水线评测工具 {cite}`es2024ragas`。这些工作说明 judge 有用，但也共同提醒一件事：**judge 是测量仪器，不是真理来源**。

作者层 judge 的目标更窄：评估某个 AI-authored artifact 是否满足入库要求。它不回答“这个检索系统整体好不好”——那是第 22 章的任务；它回答“这份草稿有没有引用、有没有改事实、有没有违反术语表、有没有自相矛盾”。

## What —— 是什么

### 生成侧 vs 检索侧

**关键区分**：本章只讨论“AI 写出的某段内容好不好”。**检索侧**评估（用户问问题，知识库给出的答案在 recall / faithfulness / 是否引用 file:line 上表现如何）属于第五部分第 22 章。两者使用的指标家族不同，混淆会导致 false confidence。

| 维度 | 本章（生成侧 judge） | 第五部分第 22 章（检索侧 eval） |
|:--|:--|:--|
| 被评对象 | 一篇 AI 起草 / 重写的内容 | 一次 retrieval+generation 给出的回答 |
| 主要指标 | 事实正确性、引用完整性、与原始材料的一致性 | recall@k、faithfulness、引用解析率 |
| 典型时机 | 入库前的 review gate | 上线后的 nightly benchmark |

### 评估什么

作者层 judge 至少评五项：

1. **事实一致性**：草稿里的断言是否能在 `source_refs` 中找到证据；
2. **引用合规性**：每个关键断言是否有可解析锚点；
3. **范围保真**：是否保留时间、版本、数据集、适用范围等限定词；
4. **术语一致性**：是否遵守 `vocab/glossary.yaml`；
5. **安全与边界**：是否泄露敏感信息、是否引入不该公开的名称。

judge 输出不应该是一句“looks good”。它必须是结构化结果：

```json
{
  "score": 12,
  "max_score": 15,
  "verdict": "needs_human_changes",
  "failed_checks": [
    {
      "id": "citation.missing",
      "severity": "medium",
      "span": "Step 3 says clear cache",
      "reason": "No source_ref supports deleting or moving cache."
    }
  ]
}
```

这个 JSON 的唯一用途是**帮助人审**。它不能自动 promote 页面。

## How —— 怎么做

### 1. 先写 rubric，再调用模型

没有 rubric 的 judge 只是另一个聊天模型。最小 rubric：

| Check | Weight | Can rule handle it? |
|:--|:--:|:--|
| 所有 `source_refs` 都存在 | 3 | 是 |
| 每个命令 / 路径 / 数字有来源 | 3 | 部分 |
| 没有伪造引用 | 3 | 是 |
| 保留术语表 | 2 | 是 |
| 摘要没有丢限定词 | 2 | 否 |
| 读者路径清楚 | 2 | 否 |

能用规则做的先用规则。LLM judge 只处理“是否丢限定词”“是否忠于来源”这类需要语义判断的项。

### 2. 用不同模型起草和评判

如果 drafter 和 judge 是同一个模型、同一个 prompt 家族，judge 很容易偏爱自己的写作风格。小团队未必总能负担两个顶级模型，但至少可以做到：

- drafter 用模型 A，judge 用模型 B；
- 或 drafter 用大模型，judge 用小模型 + 规则；
- 或每周抽样用不同模型复判。

这不是为了追求理论纯洁，而是降低共同模式错误。

### 3. 保留校准集

judge 本身也要被评估。维护一个很小的 `eval/judge-calibration/`：

```text
good-runbook.md
missing-citation.md
changed-fact-during-rewrite.md
bad-translation-term.md
overbroad-summary.md
```

每个样本都有人工标注的期望失败项。每次改 rubric、换模型、改 prompt，都先跑校准集。如果 judge 连“伪造引用”样本都放过，它不能进入主流水线。

### 4. Judge 只写评审记录，不改正文

judge 输出写到 `_audit/judge/*.json`，页面 footer 只引用结果：

```text
judge_run: _audit/judge/2026-05-18-003.json
judge_score: 12/15
review_status: pending
```

正文仍由 drafter 或 human edit 修改。不要让 judge 同时“发现问题并改掉问题”，否则评估记录和修复 diff 混在一起，reviewer 很难知道究竟发生了什么。

## Example —— 范例

林宇用第 11 章的流程生成了 `runbooks/data-preprocess-failure.md`。页面进入 `pending` 后，judge 跑 rubric：

```json
{
  "artifact": "runbooks/data-preprocess-failure.md",
  "score": 12,
  "max_score": 15,
  "verdict": "needs_human_changes",
  "failed_checks": [
    {
      "id": "scope.cache-deletion",
      "severity": "high",
      "span": "Delete the local cache directory",
      "reason": "Source audio says do not delete shared cache; policy says move local cache to .trash for 24h."
    },
    {
      "id": "citation.command",
      "severity": "medium",
      "span": "python tools/print_expected_schema.py --experiment <id>",
      "reason": "Command has no source_ref; reviewer must confirm."
    }
  ]
}
```

第一条确实是模型写错：它把“只清当前实验目录”扩写成了“删除缓存目录”。第二条是人后来补的命令，judge 不知道来源，所以要求确认。林宇修改正文，把删除改成移动到 `.trash/`，并把命令来源补到 `_audit/manual-note-2026-05-18.md`。第二次 judge：

```json
{
  "score": 14,
  "max_score": 15,
  "verdict": "ready_for_human_approval",
  "failed_checks": []
}
```

注意最后一步：页面仍然是 `pending`。林宇读 diff 后，手动设置 `review_status: approved` 和 `reviewed_by: lin-yu`。Judge 没有盖章。

## Common Mistakes —— 常见错误

1. **同一模型自写自评。** 这会放大风格偏好和共同盲点。至少用规则校准，最好换模型。
2. **没有 rubric。** “请评估这篇文档”得到的是模型的审美，不是你的质量标准。
3. **把分数当 ground truth。** 12/15 只是 review 队列排序信号，不是批准。
4. **让 judge 自动修正文。** 发现问题和修改问题要分开，否则审计记录会丢。
5. **只评语言，不评引用。** KB 质量的核心是可追溯；漂亮文字没有来源，分数应当很低。
6. **忘记 prompt injection。** 如果 judge 读取未经信任的正文，它也会受到间接 prompt 注入影响；正文必须作为 data block 传入，并按 schema 解析输出 {cite}`greshake2023injection`。

## Conclusion —— 小结

到这里，作者层的五个角色都成了一等公民：drafter 写，classifier 分，extractor 抽，summarizer 维护，judge 评估。它们共同点是：AI 都只能把内容推进到 `pending` 或生成辅助评审记录；人的角色是决定、删除、盖章。第 22 章会继续讲检索侧评测；下一部分则回到代码层，把同样的结构化纪律应用到代码这个更硬、更可解析的领域。

## 参考文献

```{bibliography}
:filter: keywords % "author-layer"
```

<!-- PKB-metadata
layer: author
updated_by: human
review_status: pending
review_score: 0
commit: HEAD
-->
