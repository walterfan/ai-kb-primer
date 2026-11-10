---
title: "第 13 章 —— AI 做维护：摘要、去重、过期与漂移检测"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - author-layer
  - maintenance
  - drift
---

# 第 13 章 —— AI 做维护：摘要、去重、过期与漂移检测

本章展开第 10 章五角色分类法里的 **summarizer**，但范围比“写摘要”更宽：归并重复内容、发现过期候选、标出漂移信号。第 11、12 章把内容写进来并结构化；本章回答几个月后的问题：**这些内容还能不能继续待在知识库里，以及是否应该被合并、降级或重写**。

```{mermaid}
flowchart TD
    A[Existing KB pages] --> B[Summaries\npage/topic level]
    A --> C[Duplicate candidates]
    A --> D[Expiration signals]
    B --> E[pending maintenance diff]
    C --> E
    D --> E
    E --> F[Human review]
    F -->|approve| G[merged / refreshed / expired]
    F -->|reject| H[kept with note]
```

## Why —— 为什么

一个知识库的失败通常不是突然发生，而是慢慢变成“数字坟场”：每一页单独看都像有用，合在一起却互相重复、互相矛盾、引用外部工具的旧名字、保留已经不再成立的前提。Aghajani 等人把软件文档问题归纳为过期、不完整、面向错误读者等类型 {cite}`aghajani2019software`；个人和小团队 KB 也一样，只是没有专职文档团队替你清扫。

AI 在维护侧的价值，不是自动删除旧内容，而是**让维护队列可见**。一个人很难每周重读 120 篇笔记；AI 可以每周扫一遍，输出“这 6 页可能重复”“这 3 页引用的工具已经停更”“这 2 份 runbook 对同一 failure mode 给了不同步骤”。这些都是候选，不是判决。真正的删除、合并和批准仍然是人的动作。

这也是本章与第五部分第 23 章的区别。第 23 章讲 link rot、`file:line` 锚点失效、外部引用腐化 —— 那是检索侧和发布侧的漂移。本章讲内容本身的生产侧维护：摘要是否还代表原文、两页是否应该合并、某个结论是否该被标为 `expired-candidate`。

## What —— 是什么

### 摘要：跨页归并

summarizer 的第一个职责是给页面或主题生成短摘要。摘要不是“短一点的正文”，而是一个维护索引：

- 页面摘要：帮助 reviewer 快速判断这页是否还值得读；
- topic 摘要：把同一主题下的多页结论合并成一段；
- change 摘要：把一次 AI 或人类修改压缩成评审说明；
- stale 摘要：解释为什么某页被标为过期候选。

摘要的危险在于它会丢限定条件。一篇笔记说“在 2025 年 11 月的这个数据集上，Wang 的 bootstrap 阈值更稳健”，摘要写成“Wang 的 bootstrap 阈值更稳健”，就把时间和数据集都删掉了。维护摘要必须保留范围限定词。

### 去重：同一件事写了三遍

重复有三种：

1. **字面重复**：两页内容大段相同，通常来自复制粘贴或迁移；
2. **语义重复**：两页讲同一件事，但标题、措辞不同；
3. **冲突重复**：两页讲同一件事，却给出不同结论。

AI 最适合发现第 2、3 类，因为它能跨措辞做匹配；但它不适合直接合并。合并会删除上下文、改变引用位置、影响历史读者，所以只能输出 merge proposal：

```yaml
candidate_type: duplicate
pages:
  - notes/2026-01-10-cache-policy.md
  - policies/data-retention.md
reason: both define retention behavior for experiment-local cache
suggested_action: keep policy page as source of truth; move note content into history section
review_status: pending
```

这份 proposal 是维护制品，不是内容本身。人接受后，再产生真实 Markdown diff。

### 过期检测：内容还成立吗？

过期检测关注“这页是否还应该被读者当作当前建议”。信号包括：

- 页面超过 N 天未更新；
- 引用的工具、API、论文版本被新材料替代；
- 同主题下有更近的 approved 决策；
- 页面中的 `last_verified_commit` 或外部 URL 已失效；
- 第 12 章抽取出的实体关系显示同一 failure mode 已有新 runbook。

注意本章是 detector，第 14 章是 grader。detector 问“这页是否值得进入维护队列”；grader 问“一段生成内容质量是否达标”。一个页面可以被 detector 标成 `expired-candidate`，但它的语言质量依然很好；也可以被 judge 打低分，但并不过期。

## How —— 怎么做

### 1. 周期性扫描，而不是按需想起

维护要成为 cron，而不是作者心血来潮。一个小团队可以每周跑一次：

```text
kb-maintain summarize --changed-since 30d
kb-maintain dedupe --topics preprocessing,data-retention
kb-maintain stale --older-than 180d --with-newer-decisions
```

每个命令只产出 proposal，不直接改 approved 页面。proposal 进入 `maintenance/queue/*.md`，按 `review_status: pending` 处理。

### 2. 摘要必须引用原文锚点

每条摘要后面都要能回到原文：

```text
summary: Local cache should be moved to `.trash/` for 24h, not deleted.
evidence:
  - runbooks/data-preprocess-failure.md#step-3
  - policies/data-retention.md#local-cache
```

没有 evidence 的摘要不进 KB。它可以作为临时阅读辅助，但不能被后续 agent 引用。

### 3. 去重用“候选对”而不是“全局聚类”

全局聚类很漂亮，也很难审。更实用的是候选对：

```yaml
left: notes/2026-01-10-cache-policy.md
right: policies/data-retention.md
similarity: 0.87
conflict: true
conflict_field: retention duration
```

reviewer 一次只看两页，决定 merge、link、keep-both 或 mark-conflict。小团队不需要一个完美的知识图谱，先需要一个能消化的维护队列。

### 4. 过期标记先降权，不删除

AI 发现过期候选时，第一步不是删除，而是降权：

```yaml
review_status: pending
maintenance_status: expired-candidate
reason: newer approved policy changes cache retention from 7d to 24h
source_refs:
  - policies/data-retention.md#2026-05-change
```

消费侧 agent 可以选择降低这页权重或在答案里提示“可能过期”。人审通过后，才把它归档或重写。

> **与第 23 章的关系。** 第 23 章（第五部分）讲的是当 KB 之外的引用（链接、`file:line` 锚点、外链）腐化时的检测与修复。本章讲的是 KB 之**内**的内容陈旧时如何让 AI 协助识别。两章一起构成完整的“漂移”故事。链接腐化有成熟研究脉络 {cite}`kiesling2017linkrot`；内容语义漂移则需要本章这种作者层维护队列。

## Example —— 范例

林宇每周日晚上跑一次维护扫描。某次扫描产生两条 proposal。

第一条是去重：

```yaml
candidate_type: duplicate-or-conflict
pages:
  - notes/2026-01-10-cache-policy.md
  - policies/data-retention.md
reason: both describe experiment-local cache retention
conflict: true
conflict_detail:
  left: keep local cache for 7 days
  right: move to .trash and keep for 24 hours
suggested_action: make policies/data-retention.md source of truth; update note with historical note
review_status: pending
```

林宇确认 policy 是最新决策，于是把旧 note 的正文改成“历史记录”，并链接到 policy。

第二条是过期：

```yaml
candidate_type: expired
page: runbooks/old-preprocess-v1.md
signals:
  - not touched for 420 days
  - mentions deprecated script `preprocess_v1.py`
  - newer approved runbook exists: runbooks/data-preprocess-failure.md
suggested_action: archive page; keep redirect note
review_status: pending
```

这一次她不同意“archive”，因为旧脚本仍用于一个历史数据集。她把 suggested action 改成“加 warning banner”，并给页面加：

```yaml
maintenance_status: legacy-only
valid_for:
  - dataset: legacy-2024-cohort
```

AI 没有决定删除内容；它只是把“这页需要人看”从 120 页里挑出来。

## Common Mistakes —— 常见错误

1. **把旧当错。** 长时间未更新是信号，不是证据。历史页面可能仍然对历史数据集有效。
2. **摘要丢限定词。** 维护摘要必须保留时间、数据集、版本、适用范围。
3. **自动合并。** 合并会改历史和引用，必须人审。
4. **只看相似度。** 两页高度相似不一定重复；它们可能是同一流程在两个环境下的不同版本。
5. **过期即删除。** 先降权、标 warning、归档；删除 approved 内容永远是人的决定。

## Conclusion —— 小结

AI maintenance 的正确形状是一条候选队列：摘要候选、重复候选、过期候选、漂移候选。它把“维护知识库”从一次耗时的全量重读，变成每周处理几个 pending proposal。第 23 章会从检索侧和引用侧继续讲漂移；本章先把作者侧的问题解决掉：内容是否还代表当前知识。

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
