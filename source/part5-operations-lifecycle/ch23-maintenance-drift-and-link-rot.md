---
title: "第 23 章 —— 维护、漂移与链接腐化"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - operations
  - drift
---

# 第 23 章 —— 维护、漂移与链接腐化

## Why —— 为什么

第 21 章告诉你哪些页面变了。第 22 章告诉你其中哪些按现状无法发布。本章讲的是：对其中每一个页面，你到底要 * 干什么 * 。用第 8 章的话说，这就是第 C3 角 —— 三级更新策略 —— 的运维版展开。

LLM 时代改变了“维护”的经济学，但改的方向和很多人想的不一样。重新嵌入一个仓库几乎是免费的 {cite}`openai_embeddings_v3`。用现代 LLM 把整页重新生成一遍也不过是几分钱。这些都不再是瓶颈。瓶颈是 * 一个人去理解“到底变了什么” * 。如果流水线一周重新生成 400 个页面，那评审者要么变成橡皮图章（违背评审的目的），要么干脆不评审了（违背知识库的目的）。

三级策略就是作者对此给出的答案 —— 在一个私有的 project-knowledge-base 工具包 {cite}`fanyamin_pkb_skill` 中反复打磨而来，在本书里被作为正式的纪律呈现出来。它有一条比各级本身更重要的规则： **rule first, LLM second。** 如果一条确定性规则能搞定，就不要调 LLM。每一次调用，都对应一个人可能必须要去读的 diff。

链接腐化 {cite}`kiesling2017linkrot` 是“维护债”最常见的症状，也是演示这三级最干净的例子。本章把它作为贯穿全章的范例。

## What —— 是什么

### 三个级别

第 21 章同步产出的每个候选页面，都会被一个规则确定性的派发器精确地路由到三个级别中的某一个。各级别的差异在于“谁来买单”以及“买的是什么”：

| Level | Who acts | Token cost | When it applies | What the reviewer sees |
|:--|:--|:--|:--|:--|
| **L1 Mechanical** | A script | **0** | Change is resolvable by rule: footer-only drift, broken link to a moved file, auto-generated page (`repo-map`), stamp refresh | Nothing, usually; the pending marker on the footer |
| **L2 Bounded-LLM** | An LLM + small context | 1–5 k tokens | Body references something that genuinely changed; the fix is localised to a paragraph or two | A small diff; a `pending` footer asking for approval |
| **L3 Human-led** | A human; LLM copy-edits | Variable | Architectural change, new ADR, security-sensitive content, or L2 repeatedly fails for the same page | A real review; sometimes a new ADR |

这三级不是一份可以点的菜单。页面是被 * 指派 * 到某一级的，不能自选。下面的规则决定指派。

### 派发器规则

```text
function dispatch(candidate):
    tags = candidate.layers_that_caught_it     # from Chapter 14
    path = candidate.path

    # Policy overrides go first.
    if path is in {architecture/, adr/, security/, runbook/}:
        return L3
    if path is in {repo-map.md, api-reference.md}:
        return L1_REGENERATE

    # L1 cases — rules that mechanically finish the job.
    if tags == {L-git} and only_footer_drift(candidate):
        return L1_STAMP
    if tags & {L-link} and link_target_moved_but_still_exists(candidate):
        return L1_FIX_LINK
    if auto_generated_marker in candidate.frontmatter:
        return L1_REGENERATE

    # L2 cases — body actually changed semantics.
    if tags & {L-entity} or tags & {L-link}:
        return L2

    # Fall-through — be conservative.
    return L3
```

有四点值得停下来说：

1.  **策略覆盖优先。** 架构页面永远不走自动改写，即使同步认为变更很小。自动生成的页面永远不过 LLM，即使变更很大（它*本来就该*是机械式的）。
2.  **L1 是第一个正向匹配。** 派发器会主动尝试用规则解决候选，然后才考虑 LLM。这就是"先走规则，后用 LLM"变成真实代码的地方。
3.  **L2 只在有证据时才触发。** 标签就是证据 —— 要么是一个被移动的实体（L-entity），要么是一个存活过 L1 规则的断链（L-link）。一个只有 L-git-only 标签且 body 没有漂移的候选，无法触及 L2。
4.  **Fall-through 是保守的。** 当规则含糊时，路由给人类。错误的人工评审的代价是一次烦恼；错误的 LLM 改写的代价是一次无声的信任回退。

### 每一级到底做什么

**L1 机械化** 跑纯 Python 或 shell：

- `L1_STAMP` 更新 footer 中的 `commit:` 和 `last_updated:`；其他一切原封不动。
- `L1_FIX_LINK` 改写一个 markdown 链接或 `file:line` 锚点，指向目标的新位置。如果目标已完全不存在，升级到 L2。
- `L1_REGENERATE` 重跑页面的生成器（例如 `repo-map.md` 从 `git ls-files` 生成）并覆盖正文。Footer 按"AI 永远设回 pending"规则重置 —— 没错，即使没有调用 LLM；任何自动化的触碰都算。

所有 L1 动作都会重置 footer：`review_status: pending`、`review_score: 0`、`reviewed_by:`、`updated_by: ai`（如果是人类跑的脚本则是 `human`）。`ai` 这个名字是一个约定，并不主张“调了模型”。

**L2 有界 LLM** 用一个小心受限的上下文调一次 LLM：

- 该候选页面 * 当前 * 的正文（约 2 kB）。
- 触发本次变更的 * 那一份 diff * （约 300 B – 2 kB）。
- 该页面的 footer，作为 C2 词汇表的提示。
- 一个*系统提示词*，说明：你只能编辑引用了 diff 中内容的那些段落；你不可以编造引用；你必须留下一个显式的 `pending` footer。

这把每次调用的 token 成本钉在大约 5 k —— 更重要的是，它钉住了变更的 * 作用域 * 。L2 不能重写整页，因为提示词和上下文里本来就没有“整页的意图”。它只能修它被指向的那几段。限制在一小片局部区域的漂移，正是 L2 存在的那个场景。

**L3 人工主导** 是人亲自来写或来改，LLM 至多被用来做“人类明确发起”的文字润色。每一次 L3 变更都是一次正常 git-and-PR 意义上的真正评审。这里没有什么特别的机制 —— 真正的纪律是：其他级别 * 不要踏进 * L3 的地盘。

## How —— 怎么做

### 以链接腐化为正宗范例

链接腐化 {cite}`kiesling2017linkrot` 值得被单独走一遍，一是因为大多数读者都亲眼见过这种失败，二是因为它在一个场景里就能把三个级别全都练到。

**背景设定。** 一次仓库级重构把 `cmd/sync/main.go` 重命名为 `cmd/sync-tool/main.go`，同时把旧的 `runSync` 函数拆成 `runSync` 和 `runSyncBatch`。三个页面引用了旧路径：

- `repo-map.md` —— 自动生成；只需要重新生成一次。
- `runbook.md` —— 其中一段写了"运行 `cmd/sync/main.go`"；页面其余部分没问题。
- `architecture.md` —— 有两段在解释 `runSync` *为什么*被设计成那个样子；这次拆分是结构性变更。

**同步（第 21 章）。** L-git 和 L-link 都把这三页全都标了出来。L-entity 标了 `runbook.md` 和 `architecture.md`（两者都引用了“因文件被移动和拆分而实体 ID 已变”的那个函数）。

**派发器。**

- `repo-map.md`：路径在 `{repo-map.md, api-reference.md}` 集合中 → **L1_REGENERATE。**
- `runbook.md`：路径不在任何策略覆盖集合中。标签含 L-link；链接目标仍然是一个存在的文件吗？是 —— `cmd/sync-tool/main.go` 存在。**L1_FIX_LINK。**
- `architecture.md`：路径在 `{architecture/}` 集合中 → 立即 **L3**。派发器甚至不检查标签。

**执行。**

- L1 从 `git ls-files` 重新生成 `repo-map.md`（零 token，确定性）。
- L1 把 `runbook.md` 中的那一个链接从 `cmd/sync/main.go` 改写为 `cmd/sync-tool/main.go`（零 token，局部编辑）。
- L3 打开一个人工可评审的任务：*"`architecture.md` 引用了 `runSync`，而 `runSync` 已被拆分。§3 中的架构解释需要重写，还是只需加一句关于新 `runSyncBatch` 的说明？请评审。"* 人类决定；可能随之写一份 ADR；LLM 只在人类主动要求时才介入。

**门禁跑动（第 22 章）。** L1 和 L3 做完之后，发布门禁会对这三页都跑一遍。H4（引用可解析）和 H5（链接可解析）现在都通过。H1/H2/H3（footer 形状）对 L1 的 footer 都通过，因为脚本本来就写对了；对 `architecture.md` 也通过，因为那位人类记得在改正文的同时把 footer 一起更新。

**成本审计。** LLM 调用次数：L1 为零；L3 为零，除非人类主动调了一次。人工评审范围：一个页面（`architecture.md`），而那确实是一次结构性变更， * 本来就该 * 交给人。与朴素基线（“对每一个提到 `runSync` 的页面都重新生成”）对比一下：那种做法会产出三份 LLM 生成的 diff 给评审者去蹚，其中两份本来就 * 不该被生成 * 。

### Why "rule first" beats "LLM first"

在 LLM 变便宜以后，很容易想完全跳过 L1、把每一个候选都扔给 L2。不要这么干。按重要性递减排三个理由：

1.  **人工注意力。** 每一次 LLM 调用都会产生一份人类要审的 diff。一周 10 次 L1 胜出就是省了 10 次评审。100 次就是 100 次。
2.  **确定性。** L1 规则每次都给出同一答案。L2 不是 —— 同一 prompt、同一模型、temperature 0 的情况下，随着模型升级，在不同日期也会产出微妙不同的文稿。对于任何规则能处理的事情，让规则做 source of truth 更稳定。
3.  **隐私。** L1 完全在本地文件系统和 git 上运行。L2 会把页面内容、diff 和 prompt 发给模型提供商。第 30 章会展开隐私影响；在这里只需要说：规则优先的纪律减少了发送给第三方的东西的暴露面，因为它在 LLM 被咨询之前就处理了更多的场景。

### 什么时候 L2 才是正确选择

L2 对应那种确确实实落在 L1（太机械）和 L3（太结构性）之间的情形：一次局部的内容变更 —— LLM 用几千 token 就能做的事，人要花十分钟、而且每分钟成本更高。函数重命名、类型签名变更、加一个参数、为一个明显新增的 helper 函数补文档 —— 这些都能干净地落在 L2，因为人类评审者可以很快核对一份一段长度的 diff。

L2 * 不 * 适合做跨页一致性（LLM 看不到别的页面），不适合做架构性重写（上下文不够），也不适合做“从零的新内容”（根本没有“被打补丁的页面”）。把这些情形塞进 L2，只会产出看起来像样、但评审者反正都得改一遍的输出，等于什么都没赚到。

## Example —— 范例

真实运营一周之后，汇总为一份批量报告 —— 周五下午推送到团队频道的那种摘要：

```text
Incremental sync summary (week of 2026-04-13 → 2026-04-20)
==========================================================

Commits scanned:   47
Candidate pages:   19
Routed to L1:      12 pages   (0 LLM calls, 0 tokens)
Routed to L2:       5 pages   (5 LLM calls, ~18k total tokens, ~$0.02)
Routed to L3:       2 pages   (human review queued)

Gate results (post-update):
  Hard gates:   137/137 pass
  Soft gates:   S1 (stale footers): 0; S3 (Diátaxis balance): pass

Human review queue:
  - architecture.md  — runSync split; needs explanation update
  - adr/0024-rate-limiting.md  — NEW (proposed; reviewer: alice)
```

有两点值得留意。其一，L1 以零成本处理掉了这一周 63% 的候选。其二，评审者的队列里只有两项 —— 两项都是真的需要人去想的 —— 而不是十九项。这个比例，而不是 token 成本，才是运维模型真正在优化的指标。

## Conclusion —— 小结

维护不是一个问题；它是三个“伪装成一个”的问题。L1 是一个文件系统与 git 的问题。L2 是一个上下文受限的 LLM 问题。L3 是一个“机器应当退开”的人类问题。在这三者之间做正确的路由 —— rule first、LLM second、人来管结构性变更 —— 就是让一个知识库在它脚下的代码加速时仍然可持续的关键。

至此，第五部分的三章已经铺出了整条回路：检测候选（第 21 章）、用硬软门禁检验它们（第 22 章）、把它们经由三级派发出去（第 23 章）。接下来的两部分（VI 和 VII）会让这条回路真正运转起来：第 25 章会把 footer 的评审状态作为文档分层的第四根轴；第 29 章会问：当“消费输出”的不是人、而是一个 agent 时，会发生什么？

## 参考文献

```{bibliography}
:filter: keywords % "operations" or keywords % "drift" or keywords % "self-citation"
```
