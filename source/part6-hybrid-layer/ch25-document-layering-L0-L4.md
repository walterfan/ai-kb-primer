---
title: "第 25 章 —— 文档分层 L0–L4"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: partial
keywords:
  - hybrid
  - adr
  - governance
---

# 第 25 章 —— 文档分层 L0–L4

## Why —— 为什么

一个成熟的知识库里，有些页面即便属于同一种 Diátaxis * 类型 * （第 3 章），也不是同一 * 种类 * 。一份由资深 on-call 工程师写的 `runbook.md` 和一份昨晚由 LLM 重新生成的 `runbook.md` 都是 how-to，都住在 `content/` 下，frontmatter schema 也一样 —— 但它们的可信程度并不等同。一个把它们一视同仁的检索系统，迟早会在凌晨 3 点的事故问询中拿错页面。

本书所基于的那篇博客 {cite}`fanyamin2026deepwiki` 引入了一个五层 **document-layer** 模型（L0 – L4），按 * 作者出处 * —— 页面是谁写的、它附带多少权威 —— 来区分页面。本章把这个模型精确化，说明“单轴分层”为什么不够用，并把它升级为一个四元组，与第 5 章引入的 footer 字段整合起来。

## What —— 是什么

### 单轴的 L0 – L4 模型

博客原文里的 L0–L4，紧凑地重述一遍：

| Tier | Authority | Who writes | Examples |
|:--:|:--|:--|:--|
| **L0** | Strongest | Code and its comments | Function bodies, docstrings, OpenAPI specs |
| **L1** | Strong | Humans, committed | ADRs, architecture docs, hand-written runbooks |
| **L2** | Moderate | Humans, collaborative | Wiki pages, design docs, retros |
| **L3** | Weak | AI, reviewed | LLM-generated summaries a human approved |
| **L4** | Conditional | AI, unreviewed | LLM-generated drafts waiting for review |

作为第一步，单轴模型已经相当好。它正确地点名了最常见的那种混淆：“AI 起草的文稿”和“人写的 ADR”不应该以相同权重被检索出来。一个按层级给结果加权的检索层 —— L0 加权、L1 加权、L4 降权 —— 立刻就会比不分层的检索层表现更好。

### 为什么一根轴不够

单轴模型仍然把知识库必须分开处理的两件事混在了一起： **当前版本是谁写的** ，以及 **有没有人评审过它** 。两者相关但不等同。考虑四个都属于 Diátaxis `how-to` 类型的页面：

1.  人三个月前写的，之后没有人评审过。
2.  人昨天写的，另一个人今天评审过了。
3.  LLM 昨天起草，今天一个人评审并以分数 4 批准了它。
4.  LLM 昨天起草，没有人评审过。

单轴模型把 (1) 和 (2) 并入 L2，把 (3) 和 (4) 并入 L3/L4。但一个真正能用的检索层希望把 (3) 当作更像 (2)、而不是更像 (4) 的东西 —— 因为在 (3) 这一场景里，确实有一个人签过字，这才是真正重要的信号。文稿是 AI 起草的，但权威已经由人核实过。

这正是 footer 中的评审字段所填补的那道缝隙。第 5 章的 `PKB-metadata` footer 给每个页面提供了四个值，它们合起来可以比任何单根轴更精确地指认该页面的种类：

- `layer` ∈ {L0, L1, L2, L3, L4} — *where in the hierarchy does this
  page sit?*
- `updated_by` ∈ {`human`, `ai`, `ai+human`} — *who authored the
  current version?*
- `review_status` ∈ {`pending`, `approved`} — *has a human signed
  off?*
- `review_score` ∈ [0, 5] — *how good is it, in the reviewer's
  judgement?*

### 元组模型

升级后的文档层是这样一个元组：

$$
\text{layer} = \langle L, U, R, S \rangle
$$

其中 $L$ 是单轴层级， $U$ 是 `updated_by`，$R$ 是 `review_status`，$S$ 是 `review_score`。作者的私有 project-knowledge-base 工具包 {cite}`fanyamin_pkb_skill` 正是以这种“四值立场”编码。旧刻度上“同一层”的两个页面，现在可以被区分开来：

| Page | Old tier | Tuple | Retrieval weight |
|:--|:--:|:--|:--:|
| L1 ADR, human-written, approved, score 4 | L1 | ⟨L1, human, approved, 4⟩ | 1.0 |
| L1 ADR, LLM-updated, pending review | L1 | ⟨L1, ai, pending, 0⟩ | 0.3 |
| L3 summary, LLM-drafted, human-approved, score 4 | L3 | ⟨L3, ai+human, approved, 4⟩ | 0.8 |
| L3 summary, LLM-drafted, no review | L3 | ⟨L3, ai, pending, 0⟩ | 0.2 |

表中的“检索权重”只是一种合理的投影方式；第 26 章的图引导上下文组装用的是一个更丰富的函数，但原理相同。元组向检索代码暴露了四个 * 可以真正用得上的 * 数值，而旧单轴只暴露一个。

## How —— 怎么做

### 在入库阶段为页面指派元组

流水线（第 7 章）会在页面被创建或更新时指派 $\langle L, U, R, S \rangle$。三条规则几乎能覆盖所有情形：

1.  **创建时**，$L$ 从路径推断：`adr/` → L1；`architecture/` → L1；`content/` 且 `created_by: human` → L2；`content/` 且 `created_by: ai` → L4。$U$ 镜像 `created_by`。$R$ 在 `created_by: ai` 时为 `pending`，否则为 `approved`。$S$ 在 `created_by: ai` 时为 `0`，否则为作者设定的默认值。
2.  **L1 机械化更新时**（第 23 章）：$L$ 不变（重盖 footer 不会让一个页面在层之间移动）；$U$ 设为 `ai`（自动化执行者）；$R$ 设为 `pending`；$S$ 设为 `0`。
3.  **L2 有界 LLM 更新时**（第 23 章）：$L$ 不变；$U$ 设为 `ai`；$R$ 设为 `pending`；$S$ 设为 `0`。如果人类随后批准，$U$ 迁移为 `ai+human`，$R$ 迁移为 `approved`，$S$ 迁移为评审者的分数。

这些规则就是 **“AI 总是置 pending”** 规则（第 5 章）和 **L3 人类评审** 路径（第 23 章）在元组词汇下的重述。它们不是新政策 —— 只是其他地方规则的推论，被重新表达为“元组之间的状态迁移”。

### 在检索阶段使用元组

检索层（第 19 章、第 26 章）用一个大致如下的函数给候选页面打分：

```text
function score(page, query):
    base   = vector_score(page, query)       # cosine, etc.
    L_w    = tier_weight(page.layer)          # e.g. L0=1.0 … L4=0.4
    R_w    = 1.0 if page.review_status == approved else 0.4
    S_w    = max(0.5, page.review_score / 5)
    return base * L_w * R_w * S_w
```

具体权重是一种调参选择，不是本书的主张。关键的一步是 * 得分依赖 footer 的四个值，而不是一个 * 。一个 layer 为 L1、但经过一次 L2 改写后仍处于 unreviewed 状态的页面，会下沉到一个真正 approved 的 L2 页面下面。这就是知识库需要的行为；元组让这一点可以用几行代码表达出来。

### 在发布阶段使用元组

第 22 章的门禁其实已经直接读取 footer 字段，但元组把另外一类检查形式化： **权威一致性。** 一个 $U = \text{ai}$ 且 $R = \text{approved}$ 的页面必须有非空的 `reviewed_by`；一个 $L = \text{L0}$ 的页面必须 $U = \text{human}$ ，因为在这一层代码并不是 LLM 生成的。这类检查每条一行代码即可；它们能在发布前就把错误标签抓出来。

## Example —— 范例

同一个页面在两种模型下的“前”和“后”对比：

**单轴视角。** 一个页面 `runbook/ingest-pipeline-failure.md` 的 frontmatter 是 `doc_type: how-to, status: published, verification_status: unreviewed`。博客那套单轴模型会把它分类为 L3（AI 文稿，理论上可评审）。检索对它的加权约 0.6。问题是：这个页面其实是两周前由一个人写的，它之所以是 `unreviewed`，只是因为一个 cron 任务在“没人再次核验它”时把状态翻转了。单根轴看不出这种差别。

**元组视角。** 同一个页面完整的 footer 是这样的：

```markdown
<!-- PKB-metadata
last_updated: 2026-04-05
commit: a1b2c3d4
updated_by: human
review_status: pending
review_score: 0
reviewed_by:
-->
```

元组：⟨L2, human, pending, 0⟩。检索层读到的是：人作者，尚未被正式评审，分数未知。这与 ⟨L3, ai, pending, 0⟩（AI 起草，从未评审）并不相同，也与 ⟨L2, human, approved, 4⟩（人写、人批、高分）不同。在旧模型里这三个本来都会被归到 L2 或 L3；在元组模型里它们各自得到一个不同的分数。

一个软门禁（第 22 章 S2）会把第一个页面提交给评审者： * “人写的 runbook，从未被评审过；请至少看一次。花一分钟，防一次凌晨 3 点的问题。” *

## Conclusion —— 小结

单轴的 L0–L4 是一个好的第一步：它明确把“作者出处”提为文档的一等属性。四元组升级是对这件事的诚实延续：“作者出处”本身其实是两个独立问题 —— * 这是谁写的 * 和 * 谁签了字 * —— 一个想在 LLM 时代编辑浪潮中活下来的知识库必须把两者都回答。四元组放得下一个 footer；写便宜、读便宜、强制执行也便宜。

下一章会在检索阶段使用这个元组：第 26 章的图引导上下文组装用 $L$ 决定层级优先级，用 $R$ 和 $S$ 把未评审或低质量的页面从 LLM 上下文窗口里过滤掉，并用关系图把 top-k 页面所引用的其他内容一并拉进来。

## 参考文献

```{bibliography}
:filter: keywords % "hybrid" or keywords % "adr" or keywords % "governance" or keywords % "self-citation"
```
