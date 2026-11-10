---
title: "第 30 章 —— 成本、隐私、安全"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - governance
  - cost
  - privacy
---

# 第 30 章 —— 成本、隐私、安全

## Why —— 为什么

到目前为止，每一章都把成本、隐私、安全当作对 * 局部 * 决策的约束 —— 选一款定价让“重新嵌入”变便宜的嵌入模型（第 17 章）；把 L2 的上下文限制住（第 23 章）；拒绝把 `architecture/` 发给 LLM（第 23 章的派发器）。这些都是正确的局部选择。但局部正确不等于全局姿态健康。一个每一章的规则都能单独通过的知识库，仍然可能把团队的 LLM 预算烧在噪声上、在提示词日志中泄露私密内容，或发布一个由攻击者写就的页面。

本章把这三条约束整合为贯通各章的运维纪律。核心是 **token 预算纪律** —— 第 23 章三级更新策略所支撑的成本纪律。隐私和安全补全这张图景。

## What —— 是什么

三个相互关联、都要诚实回答的问题：

1.  **成本。** *运行这个知识库每次 commit、每次同步、每年要花多少钱？* 不是逐次 LLM 调用的问题（那很简单），而是汇总问题（那才是预算对话需要的）。
2.  **隐私。** *我们的哪些内容离开了我们的围墙，给了谁，在什么保留策略下？* 这不是修辞性问题；知识库可以给出并强制执行具体的答案。
3.  **安全。** *一个页面的作者可信吗？一个页面的内容可信吗？一个页面的引用可信吗？* 这三者是不同的；各有各的机制。

## How —— 怎么做

### Token 预算纪律

核心那一步是这样说的： **一个知识库的 LLM 成本，应该是其 commit 频率的可预测函数，而不是页面数的可预测函数。** 如果成本随页面数扩，那么知识库越有用、就越贵 —— 这是错误的激励。如果成本随 commit 数扩，那么知识库的成本就与项目本身的变更率成正比 —— 而那本来就是团队已经在做预算的那个量。

这是作者在一个私有的 project-knowledge-base 工具包 {cite}`fanyamin_pkb_skill` 中反复打磨出的运维主张之一：第 23 章的三级更新策略存在的唯一目的，就是让 commit 成本在知识库增长时仍然保持有界。做一道算术题：

**朴素基线 —— “每一个话题发生变化的页面都重新生成”** ：

$$
\text{cost}_\text{naive} \approx N_\text{candidates} \times C_\text{full-page}
$$

其中 $N_\text{candidates}$ 会同时随“知识库规模”和“commit 的爆炸半径”一起增长， $C_\text{full-page}$ 则是一次整页 LLM 调用的成本（上下文加重新生成的正文，约 8–20 k tokens）。

**三级纪律 —— “rule first, LLM second”** ：

$$
\text{cost}_\text{3L} \approx N_\text{L1} \times 0 + N_\text{L2} \times C_\text{bounded} + N_\text{L3} \times 0
$$

其中 L1 和 L3 花 0 个 LLM token（L1 是机械操作，L3 是人类操作）， $C_\text{bounded}$ 是被第 23 章上限在 ~5 k tokens 的有界调用。以第 23 章例子里那一周的同步为例（19 个候选，12 → L1、5 → L2、2 → L3）：五次 L2 调用，每次约 3 k tokens。按 2024 年 `text-embedding-3-small` / `gpt-4o-mini` 的定价 {cite}`openai_embeddings_v3` 来算，每个仓库每周就是几美分的事。朴素基线在同样 19 个候选上按每个 15 k tokens 计，花费高出十倍以上， * 而且 * 会产出一堆 diff 去烧评审者的时间 —— 正如上一章所论，那才是更贵的资源。

Token 预算纪律有三条规则：

1.  **按 commit 而非页面做预算。** 提前确定每次 commit 的成本上限（例如：小团队每 commit \$0.10；调优后的流水线只要几美分）。跟踪它。在回归时告警。
2.  **L1 比例是一个 KPI。** 由 L1 处理的候选占比是一个运维指标。L1 比例 50% 是健康的；90% 是优秀的（通常意味着你有很好的自动生成页面）。L1 比例低于 20% 意味着规则没有捕获常见场景，或者知识库 L2 的比重不正常地高。
3.  **L3 的瓶颈是人的注意力，不是 token 成本。** 不要试图用 L2 替代 L3 来省钱。钱从来都不是约束。

这些规则让 dashboard 上的成本曲线成为知识库健康的 * 先行指标 * ，而不只是一个会计数字。

### 外部文档摄入的成本

上面的预算纪律针对的是**日常维护** —— commit 触发的增量更新。但知识库不只有内生内容。真实的知识库需要**摄入外部产出物**：设计规格书（OpenSpec specs、ADRs、RFCs）、AI agent 生成的技术方案（TDS、Superpowers plan）、以及从已有文档站点导出的页面（Confluence、Notion、内部知识平台、旧 Sphinx 站点）。摄入是一次性成本高峰，不是持续性开支 —— 分类一次，此后增量维护。但如果不做预算，高峰本身就能把一个季度的 LLM 预算烧掉。

摄入成本的结构可以这样拆：

$$
\text{cost}_\text{ingest} \approx N_\text{pages} \times C_\text{classify} + N_\text{reclassify} \times C_\text{bounded}
$$

其中 $C_\text{classify}$ 是把一份外部文档跑一遍第 6 章分类流水线（`ingestFileWithOptions`）的成本（启发式分类约 0 token；LLM 分类约 1–3 k tokens），$N_\text{reclassify}$ 是分类后在后续同步中被三级更新策略重新触发的页面数。

不同来源的成本差异极大：

| 来源类型 | 典型规模 | 分类成本 | 后续维护成本 | 隐私风险 |
|:--|:--|:--|:--|:--|
| OpenSpec / ADR / RFC | 小（10–50 页），结构化 Markdown，有 frontmatter | 低 —— 启发式即可归类为 `reference` 或 `explanation` | 低 —— 更新频率与 spec 自身同步 | 低 —— 已在仓库内 |
| AI 生成的技术方案 | 中（每份 1–5 k tokens），结构化但需校验 | 中 —— 多数可启发式归类，引用需 L2 校验 | 低 —— 一旦评审通过就不再变化 | 中 —— AI 可能在 prompt 中泄露上下文 |
| 文档站点导出 | 大（100–10 000 页），HTML/富文本，无 frontmatter | 高 —— 每页都需 LLM 分类 | 高 —— 源站更新不可控，陈旧风险大 | 高 —— 来源可能包含未标记的敏感内容 |

实操建议：**先算再导。** 在执行大规模文档站点导入前，先跑一轮**干跑**（dry run）：只做分类和 frontmatter 分配，不写入 `content/`，输出一份统计报告（预估 token 数、Diataxis 分布、重复检测命中数）。这份报告本身就是一个成本 gate：如果预估成本超过团队的单次预算上限，就分批导入，而不是一口气跑完。

第 31 章会进一步讨论这三类来源的信任层级和出处链。


### 隐私：把“有界上下文”视作一种隐私属性

第 23 章那条 L2 有界 LLM 策略，还有一个成本论证未能穷尽的性质： **有界的上下文，等于有界的披露。** 每一次调用 LLM， * 总有一些 * 内容离开宿主。视供应方不同，这些内容可能被保留用于模型训练、滥用监控，或只是进了请求日志。即使签了“无数据保留”合同，提示词仍然会在某处短暂驻留。

L2 纪律给了我们一个直接的隐私把手：

| Strategy | Sent per call | Annual disclosure surface |
|:--|:--|:--|
| Naive full-regeneration | The whole page ~8–20 k tokens | ≈ $N_\text{pages} \times$ weekly-regen-rate |
| L2 bounded | One page's body + one diff ~2–5 k tokens | ≈ $N_\text{L2-candidates}$ only |
| L1 only | Nothing | None |
| L3 human-led | Whatever a human pastes | Human-controlled |

L1 的“零披露”性质是“rule first, LLM second”最有力的隐私论据。每一条在 L1 消化掉变更的规则，对应着一段“从未离开宿主”的内容。对那些含有敏感内容的知识库 —— 安全 runbook、事故复盘、内部架构 —— 而言，rule-first 纪律不只是成本论证，也是披露论证。

由此可以落地三个具体动作：

1.  **标记敏感路径。** `security/`、`incidents/` 和 `adr/`（如果 ADR 讨论了敏感的权衡）等路径在派发器配置中被标记为 `privacy: restricted`，仅路由到 L3。派发器规则在第 23 章就已经有了；隐私框架是那条规则*存在的原因*。
2.  **在敏感路径上优先使用自托管模型做 L2。** 一个开源权重的 embedding 和一个小型开源 LLM 跑在本地 GPU 上（例如 BGE 家族 {cite}`xiao2024bge`），可以在零第三方披露的情况下处理除最棘手的 L3 升级之外的所有 L2。
3.  **记录每一次 L2 调用。** 第 22 章的门禁报告可以包含一份按 commit 的账本，列出哪些页面走了 L2。这给团队一个可审计的答案："我们这周给模型提供商发了什么？" —— 这既是隐私控制，也是调试辅助。

### 安全：三个互相独立的信任问题

知识库的安全性不是单一问题。它是三个各有其机制的独立问题：

1.  **作者可信吗？** 机制：`created_by` + `updated_by` 字段（第 5 章），由 `MergeFrontmatterUpdate` 的不可变规则强制执行。一个页面永远不会丢失其原始作者身份。Git 历史提供第二份独立记录。
2.  **内容可信吗？** 机制：footer 的 `review_status` + `review_score`（第 5 章）、硬闸（第 22 章 H3）、以及"AI 永远设回 pending"规则。内容被碰过但没人评审的页面，其 `review_status: pending` —— 是显式的。
3.  **引用可信吗？** 机制：第 22 章 H4 —— 每个 `file:line` 锚点在 HEAD 上都能解析。对抗性的或幻觉出来的引用会导致门禁失败并阻止发布。

针对知识库的攻击也倾向于沿同样的三条轴分裂。作者攻击发生在 Git 层面（账号被盗、恶意 PR）；内容漂移攻击是“陈旧页面”攻击（从前为真的某件事如今不再为真，攻击者利用“人们仍相信它为真”这一点）；引用攻击则是其中最恶毒的一类 —— 页面声称自己引用了代码、实则没有 —— 而它们恰恰是最容易用机械手段防御的，因为“引用能不能解析”是可判定的。

### 三层威胁模型

把它们放在一起看：一个运作良好的知识库中的页面，背后有三份互相独立的证明：

```text
Authorship:  committed by alice, Git-signed commit a1b2c3d (L0 — git)
Content:     reviewed by alice, review_score: 4       (L1 — footer)
Citations:   every file:line resolves at HEAD          (L2 — verify)
```

攻击者必须同时攻破这三层，才能种下一个“看起来合法”的页面。如果团队使用签名 commit，攻破 Git 就很难；攻破评审要伪造评审者在 footer 中的签名；攻破引用则几乎不可能，因为引用在发布时是被机械式校验的。每一层单独代价都很低；三层叠加起来，比任何单层都要强得多。

## Example —— 范例

一份针对某中型知识库的具体“预算与披露表”（数据已匿名化，但运维上真实合理）：

```text
Cost
  Commits this month:            318
  Total L1 actions:              587   (0 LLM tokens)
  Total L2 LLM calls:             42   (avg 3.1k tokens each)
  Total L2 tokens:            ~130k
  LLM provider spend:         ~$0.65
  Embedding re-compute spend: ~$0.08
  Total monthly cost:         ~$0.73
  Per-commit cost:            ~$0.002

Privacy
  L1 actions sent to 3rd party:    0   (all local)
  L2 calls sent to 3rd party:     42
  L2 bytes disclosed:          ~520k   (bounded per Chapter 16)
  Sensitive paths touched:         3   (all routed to L3 - human review,
                                         not to LLM)

Security
  Hard-gate failures blocked:      7   (6 citation-resolve, 1 footer-integrity)
  Signed-commit ratio:          99.7%
  Pages with mismatched reviewer   0
```

每一行都对应之前某一章讲过的某条规则或某个机制。本章的贡献是把它们集中到同一张纸上，让“治理对话”在桌面上有个实实在在可看的东西。

## Conclusion —— 小结

成本、隐私、安全通常被当作三件独立的关切。但把它们放到第 7 章 a 节的运维模型里走一遍，就会发现它们其实共享同一条主线：三级更新策略把 token 界住（成本）、把披露界住（隐私）、并把最棘手的情形留给人（安全）。本章提到的机制没有一条是新的；新的是这个主张 —— * 这三个性质其实是同一个性质 * ：不让 LLM 去碰那些“规则或人能够更好、更便宜、更私密地处理”的东西，这条纪律本身。

下一章讨论信任与红队演练：一旦运维模型就位，一个知识库到底如何在对抗性使用下被测试？

## 参考文献

```{bibliography}
:filter: keywords % "governance" or keywords % "cost" or keywords % "privacy" or keywords % "self-citation" or keywords % "foundations"
```
