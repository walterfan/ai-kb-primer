---
title: "第 12 章 —— AI 处理元数据：分类、打标签、实体与关系抽取"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - author-layer
  - classification
  - extraction
---

# 第 12 章 —— AI 处理元数据：分类、打标签、实体与关系抽取

本章展开第 10 章五角色分类法里的 **classifier** 和 **extractor**。第 11 章把内容从 0 写到 1；本章把内容从“可读”推进到“可被机器稳定处理”：分类、标签、frontmatter、实体、关系。没有这一层，知识库只是有搜索框的文件夹；有了这一层，后面的维护、评估和 agent 才能按结构工作。

```{mermaid}
flowchart TD
    A[Markdown notes / PDFs / emails] --> B[classifier\nclosed vocabulary]
    A --> C[extractor\nentities + relations]
    B --> D[frontmatter diff]
    C --> E[entity table / graph edge]
    D --> F[pending review]
    E --> F
    F --> G[approved metadata]
```

## Why —— 为什么

在非代码场景里，知识库最常见的失败状态不是“没有内容”，而是“只有内容”。一千篇 Markdown 笔记、三百封导出的邮件、几十份 PDF，单看每一份都像知识，合在一起却不能回答更高一级的问题：哪些笔记属于同一个主题？哪几份 policy 引用了同一个术语？哪些实验结论依赖同一篇论文？哪几份 runbook 涉及同一个失败模式？

分类和抽取是从“文件堆”走向“知识库”的最便宜门票。分类把内容投到有限词汇表，让导航、统计、评审队列和 coverage audit 有了维度；抽取把散文里的实体与关系拉出来，让“同一件事写在三处”可以被发现。零样本文本分类在 NLP 里早已有成熟基线 {cite}`yin2019zeroshot`，大语言模型让这件事对普通团队可用 {cite}`brown2020gpt3`；但把它放进 KB，难点从“能不能打标签”变成“打出来的标签能不能被信任、重跑、回滚”。

> **与第 6 章的关系。** 第 6 章（第二部分）讲的是文稿层场景下分类流水线的**具体工程化**：六动词、heuristic-default 契约、LLM 失败模式与兜底。本章则把**为什么要做、做哪几件事、人审契约**抽出来，让它对任何源（不只是文稿型 wiki）成立。两章互为补充：先读本章拿到框架，再读第 6 章看到一种成熟实现。

## What —— 是什么

### 分类：把内容投到封闭词汇表

classifier 的输出必须来自封闭词汇表。这里的封闭不是建议，而是 schema 约束：

```yaml
topics:
  allowed:
    - hierarchical-clustering
    - single-cell
    - data-retention
    - preprocessing
doc_type:
  allowed: [tutorial, how-to, reference, explanation, runbook, decision]
status:
  allowed: [draft, review, approved, archived]
```

AI 可以选择、排序、解释为什么选择；但不能发明 `overview-guide`、`misc`、`interesting` 这种新标签。一个超出词汇表的标签不是“模型有创造力”，而是一次 schema 失败。第 6 章把这条原则落在文稿层 ingest pipeline；本章把它推广为所有 metadata 写入的硬约束。

分类输出至少包含四项：

| 字段 | 例子 | 谁能写 | 谁能批准 |
|:--|:--|:--|:--|
| `topics` | `[preprocessing, data-retention]` | AI 可建议 | 人或抽样闸 |
| `doc_type` | `runbook` | AI 可建议 | 人或规则 |
| `confidence` | `0.82` | AI / 校准器 | 不批准，只作路由信号 |
| `reason` | “contains ordered recovery steps” | AI 可写 | 人可读，不作事实来源 |

`confidence` 不是信任状态。它只决定路由：高置信度进入抽样人审，低置信度进入逐条人审，越界值直接拒绝。

### 实体与关系抽取

extractor 的任务不是给页面贴标签，而是从页面里抽出可连接的对象：

- 实体：人、论文、数据集、脚本、policy、runbook、实验、决策；
- 关系：`cites`、`depends_on`、`supersedes`、`contradicts`、`mentions_failure_mode`；
- 锚点：`file#section`、DOI、sha256、commit、行号。

一个抽取结果应该像这样：

```json
{
  "source": "notes/2026-02-09-reviewer-reply-draft.md#section-2",
  "entities": [
    {"id": "paper:wang2024-bootstrap", "type": "paper", "label": "Wang et al. 2024"},
    {"id": "method:bootstrap-threshold", "type": "method", "label": "bootstrap threshold"}
  ],
  "relations": [
    {
      "subject": "note:2026-02-09-reviewer-reply-draft",
      "predicate": "chooses",
      "object": "method:bootstrap-threshold",
      "evidence": "第 2 节：we committed to Wang's bootstrap..."
    }
  ]
}
```

注意 `evidence`。抽取不是“把散文变成图”的魔法；它是把散文里的某个断言变成结构化记录，并保留能回到原文的证据。Zhu 等人对 LLM 构建知识图谱的综述强调，前沿模型更适合在结构化上下文上推理，而不是独自承担无约束抽取 {cite}`zhu2024llmsknowledgegraphconstruction`。所以 extractor 必须被 schema、证据锚点和抽样人审夹住。

## How —— 怎么做

### 1. 词汇表先行

不要从 AI 的输出里“长出”词汇表。先写 `vocab/topics.yaml`，哪怕只有 20 个主题；再让 AI 在里面选。新增标签必须走人的 PR，而不是模型在一次 batch 里顺手发明。弱监督系统很早就把“标签函数 + 小金标集”作为生产分类的基本纪律 {cite}`ratner2017snorkel`；小团队 KB 可以用更轻的版本：20 个固定标签 + 30 篇人工标注样本。

### 2. 输出只收结构化 JSON

classifier 和 extractor 都不应该返回一段自然语言解释再让脚本猜。输出必须按 JSON schema 解析：

```json
{
  "topics": ["preprocessing", "data-retention"],
  "doc_type": "runbook",
  "confidence": 0.82,
  "evidence": [
    {"field": "doc_type", "quote": "Step 1... Step 2..."}
  ]
}
```

解析失败、枚举越界、缺少 evidence，都是 `pending` 队列里的高优先级项；不是静默 fallback。

### 3. 分离“建议”和“写入”

AI 的原始输出先写到 `_audit/metadata-proposals/*.json`。一个 deterministic writer 再把通过校验的字段写入 frontmatter 或实体表。不要让模型直接编辑 Markdown 文件。这样做有三个好处：可以重跑、可以 diff、可以把失败样本收进金标集。

### 4. 抽样人审，而不是全量人审

对 classifier，合理策略是：

- 越界：自动拒绝；
- 低置信度（例如 `< 0.65`）：逐条人审；
- 中等置信度：抽样 20%；
- 高置信度且金标集稳定：抽样 5%。

对 extractor，抽样比例要更高，因为关系方向错的代价比标签错更大。第 10 章里林宇的三元组抽取 23 条错了 6 条，这种错误率下整批应降级为人审。

## Example —— 范例

林宇的笔记库现在有 120 篇研究笔记、5 份 runbook、3 份 policy。她先写一个很小的主题词汇表：

```yaml
topics:
  - hierarchical-clustering
  - single-cell
  - preprocessing
  - data-retention
  - reviewer-response
doc_type:
  - note
  - runbook
  - policy
  - decision
```

第一次 batch 跑 30 篇 `notes/2026-*.md`，AI 输出 88 个 topic 建议：

- 78 个在词汇表内；
- 5 个越界（`bioinformatics`, `methodology`, `misc` 等），自动拒绝；
- 5 个置信度低于 0.65，进入逐条人审。

她抽审 6 篇，发现 `reviewer-response` 被过度使用，于是把 prompt 改成：只有当页面明确写给审稿人或答复信时才使用该标签。第二次 batch 稳定下来。

接着跑 extractor，对 `runbooks/*.md` 抽 `(artifact, failure_mode, recovery_step)`：

```text
data-preprocess-failure.md
  artifact: preprocessing script
  failure_mode: schema mismatch on sample_id / batch_id
  recovery_step: run print_expected_schema.py
  evidence: runbooks/data-preprocess-failure.md#step-2
```

这些记录没有直接成为“事实真相”；它们先进入 `_audit/entity-proposals/2026-05-18.jsonl`。林宇确认后，writer 把它们写进 `entities.sqlite`。从此以后，第 13 章的维护脚本可以发现“同一个 failure_mode 在两份 runbook 里写了两套恢复步骤”。

## Common Mistakes —— 常见错误

1. **开放词汇表。** “让模型自由打标签”会在三个月内制造 200 个近义标签。先写小词汇表，再让模型在里面选。
2. **把 `confidence` 当信任。** 高置信度只说明模型觉得自己像；不说明标签对你的读者有用。
3. **自然语言输出。** 让模型输出解释段落，再用正则抓字段，是 schema 漂移的邀请函。
4. **实体没有 evidence。** 没有证据锚点的实体只是二手笔记，不能进图。
5. **把分类和抽取混成一步。** 分类是页面级投影；抽取是断言级结构化。两者错误模式不同，review 队列也应该分开。

## Conclusion —— 小结

classifier 和 extractor 把知识库从“文本集合”推进到“可维护结构”。关键不是模型有多聪明，而是输出被封闭词汇表、JSON schema、evidence 锚点和人审队列约束住。第 6 章展示了这件事在文稿层流水线里的一个成熟实现；本章给出更通用的作者层框架。下一章会利用这些 metadata 做维护：摘要、去重、过期和漂移检测。

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
