---
title: "第三部分 —— 作者层"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - author-layer
---

# 第三部分 —— 作者层

```{toctree}
:maxdepth: 1

ch10-why-ai-as-author
ch11-ai-drafting-and-rewriting
ch12-ai-for-metadata
ch13-ai-for-maintenance
ch14-ai-as-judge
```

第三部分一图览 —— 五章把 AI 从“知识库的消费者”移到**知识库的合作作者**：起草、分类、抽取、总结、评判。这是个人和小团队真正能从 AI 拿到杠杆的地方，也是程序员把 第 6 章 那条分类流水线推广到非代码内容的地方：

```{mermaid}
mindmap
  root((Author Layer))
    Why
      第 10 章 Cost of pretending AI only consumes
      Trust boundary
      What humans keep
    Drafting
      第 11 章 AI as drafter
      Provenance recording
      Review gates
    Metadata
      第 12 章 AI as classifier + extractor
      Frontmatter inference
      Entity & relation extraction
    Maintenance
      第 13 章 AI as summarizer
      Dedup
      Expiration & drift signals
    Judging
      第 14 章 AI as judge
      Generation-side eval
      Citation compliance
```

## Why —— 为什么

第二部分讲了文稿层的承载和契约 —— 文件、frontmatter、流水线、闸。第四部分及之后讲检索器、混合层、agent，全部是**消费侧**：知识库已经在那里，AI 帮你从里面取出东西。但**入库前**的工作 —— 起草一份 runbook、给一篇新笔记打分类、从一封邮件里抽实体、检测两份说法在过去半年是否漂移 —— 在大多数现实知识库里仍然是 100% 人力。这是 AI 在 2026 年最被低估的杠杆：不是替你回答问题，而是替你**让知识库本身变得更值得被问**。

对个人 KB 用户（Obsidian 用户、研究笔记、policy 集合的小团队），这一部分就是核心 —— 你没有一支专职的内容运营团队，你**就是**那个运营团队，AI 是你唯一可负担的助手。

对程序员读者，本部分把 第 6 章 里只点了一下的 "AI 在分类流水线里" 的话题推广为一个完整的 AI-作为-生产者 的方法论，并且明确划清和 第 27 章（AI 作为编码助手）的边界 —— 一边是入库，一边是查询。

## What —— 是什么

五章覆盖 AI 作为**生产者/策展者**的五种角色：

- **第 10 章 —— 为什么让 AI 当作者。** 经济账、信任边界、五角色分类法（drafter / classifier / extractor / summarizer / judge）、以及人审契约。
- **第 11 章 —— AI 起草、重写、翻译。** 起草工作流、provenance 记录、review gate 的状态机、常见幻觉模式。
- **第 12 章 —— AI 处理元数据。** 分类、打标签、frontmatter 抽取、实体/关系抽取 —— 这一章把 第 6 章 的分类流水线推广到非代码源。
- **第 13 章 —— AI 做维护。** 摘要、去重、过期/漂移检测；为第五部分的检索侧维护提供生产侧对应。
- **第 14 章 —— AI 作为评判者。** 生成侧质量评估、引用合规检查；明确区分于第五部分的检索侧评估（第 22 章）。

## 优点、缺点与结构性局限

把第 10–14 章里散落的 trade-off 收在一张表上，让读者在动工**之前**就知道什么时候这条流水线赚、什么时候亏：

| 维度 | 优点（已兑现） | 缺点（用错时会发生） | 结构性局限（用对了也仍存在） |
|:--|:--|:--|:--|
| 起草 drafter | 把 0 页 → 1 页 pending 的边际成本从 35 分钟压到 3 分钟；让“没空写”的内容被写下来 | 没有 `source_refs` 就让模型写；改写时丢限定词；让 AI 写引用 | 模型有风格底色；翻译/改写依赖术语表完整度 |
| 分类 classifier | 让“文件堆”有维度；coverage audit、评审队列、统计都依赖它 | 开放词汇表；把 `confidence` 当信任；自然语言输出靠正则抓 | 词汇表本身是治理问题，AI 解决不了 |
| 抽取 extractor | 让“同一件事写在三处”可被发现；为 graph / 维护脚本提供 typed edges | 抽取没有 evidence；分类和抽取混成一步 | 关系方向错的代价比标签错大；高复杂度领域抽取错误率仍偏高 |
| 维护 summarizer | 把“全量重读 120 页”压成“每周看几条 proposal” | 摘要丢限定词；自动合并；把“旧”等同于“错” | AI 是 detector，不是 decider；判断仍由人做 |
| 评判 judge | 把人审从“凭感觉读完”变成“按 rubric 看失败项” | 同一模型自写自评；把分数当 ground truth；让 judge 改正文 | judge 是测量仪器，不是真理；rubric 必须由人写 |

**几条本部分没有把章节铺开、但读者必须自己评估的结构性局限**：

1. **流水线本身有维护成本**。词汇表、rubric、prompt 模板、校准集这些设施每年要花 5–10 小时升级。第 10 章经济账里的“20 小时人审”**不含**这块；动工前要在心里加上。
2. **有破局规模**。粗估：每年 < 30 页的 KB 不要上这条流水线（隐性成本 > 收益）；30–500 页是甜区；> 500 页要考虑把每个角色拆成独立服务而不是脚本。第 10 章经济账针对的是 100 页/年。
3. **隐私和数据出境**。把笔记发给商用 LLM API，对持有 policy / 合规 / 客户材料的小团队是真实顾虑。本部分默认 LLM 调用可发生；若不可，要么用本地模型（成本结构改变）、要么把流水线限制在脱敏样本上（覆盖率改变）。
4. **模型升级会破坏校准**。换模型 / 模型版本变化时，第 14 章的校准集必须重跑；判分阈值、prompt 模板可能要重新调参。这是周期性返工，不是一次性投入。
5. **审计日志会变重**。`_audit/` 半年内会比 KB 正文体积大几倍。要么定期归档，要么接受这是“知识库的另一半”，纳入备份策略。

承认这五条局限，不是退回 “AI 只当读者”；是把第 11–14 章的方法放在它最可能赢的区间里使用。

## 人的角色 —— 不止是评审者

人审契约（第 10 章）把人的下游角色写得很清楚：审 `pending` 的 diff、按 approve 按钮、删 approved 内容。但这条流水线**之前**还有四个上游人类角色，目前各章只点了一下，集中在这里：

| 阶段 | 人的角色 | 干什么 | 在哪一章被涉及 |
|:--|:--|:--|:--|
| **入口 0** | 意图作者 | 写 task brief：目标、读者、必须包含、不可包含、来源 | 第 11 章 How §1 |
| **入口 1** | 词汇表 / schema 作者 | 维护 `vocab/topics.yaml`、frontmatter schema、实体类型 | 第 12 章 How §1 |
| **入口 2** | rubric 作者 | 定义“好”的 checklist；维护校准集 | 第 14 章 How §1, §3 |
| **入口 3** | provenance 设计者 | 决定 `_audit/` 结构、`prompt_hash` 怎么算、保留多久 | 第 10 章 § 人审契约 |
| **回路** | reviewer | 看 `pending` diff，改 `ai+human`，按 approve | 第 10/11/12/13/14 章 |
| **回路** | escalator | AI 越界、judge 拒绝、维护 proposal 冲突时的决策人 | 第 13/14 章 Example |
| **回路** | 升级判断者 | 模型 / prompt 升级时拍板是否回归测试通过 | 上一节“结构性局限” §4 |

```{mermaid}
flowchart LR
    H1[Intent author<br/>writes task brief] --> P1[drafter pipeline]
    H2[Vocab/schema author<br/>closes the label space] --> P2[classifier + extractor]
    H3[Rubric author<br/>defines what good means] --> P5[judge]
    H4[Provenance designer<br/>defines _audit/] --> ALL((All pipelines))
    P1 --> R[pending artifact]
    P2 --> R
    P5 --> R
    R --> HR[Reviewer<br/>approve / reject]
    R --> HE[Escalator<br/>resolves conflicts]
    HR --> A[approved]
    HU[Upgrade decider<br/>model/prompt change] -.regression test.-> P1
    HU -.regression test.-> P2
    HU -.regression test.-> P5
```

关键结论：**把人画成“最后那个按按钮的”是错读了这张图**。这条流水线之所以能省 47 小时，是因为入口四个角色提前花了 5–10 小时把约束（brief / vocab / rubric / provenance schema）写下来。把这件事说清楚，是为了避免读者读完第三部分后觉得“好像只要装个 prompt 就行了”—— 实际上**第三部分省下来的人审时间，被部分地转移到了流水线设计这一上游环节**。这种转移是值得的，但不能假装它不存在。

## How —— 怎么做

整部分使用同一个**贯穿案例**：一个个人 / 小团队的 Markdown 知识库（一两百页 note + 几份 runbook + 几份 policy）。Obsidian 是可选的 UI，但不是硬依赖 —— 任何编辑器、Git、加上一个 AI API 就能复现每一步。每章的 `## Example` 段把案例往前推一步，让读者看到 AI 在那一个角色上的完整生命周期。

## Example —— 范例

本部分各章的 Example 段共享一个虚构 fixture *林宇的笔记库*（结构在第 10 章定义），并构成一个最小的有向回路：drafter（第 11 章）写出 → metadata（第 12 章）打分类与抽取实体 → maintenance（第 13 章）几周后发现过期 → judge（第 14 章）评估重写质量。读完四章后，把 mindmap 上每一支在你自己的 KB 里跑一遍，就是端到端示例。

## Conclusion —— 小结

作者层是把 AI 从“查询界面的小工具”升级为“知识库的合作者”的那一层。它不是替代人 —— 五角色里的每一角色都附带一条人审契约 —— 而是把人审从“无差别校对每一页”降级为“只评审 AI 标了 `pending` 的那些 diff”。这个改变是杠杆的所在。

## 参考文献

```{bibliography}
:filter: keywords % "author-layer"
```
