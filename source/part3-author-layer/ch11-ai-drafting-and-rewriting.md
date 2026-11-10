---
title: "第 11 章 —— AI 起草、重写、翻译"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - author-layer
  - drafting
---

# 第 11 章 —— AI 起草、重写、翻译

本章展开第 10 章五角色分类法里的 **drafter**：AI 负责把人的意图、原始材料和局部约束变成一份可评审的文字制品。这里的关键词不是“自动生成文档”，而是**把开始写的摩擦降下来**，并且让每一次 AI 写作都留下足够的 provenance，进入同一条 `pending → approved` 人审闸。

```{mermaid}
flowchart LR
    A[Intent\n人给目标和边界] --> B[Sources\n原始材料与引用]
    B --> C[Draft\nAI 起草/改写/翻译]
    C --> D[Provenance\n模型、prompt hash、来源]
    D --> E[Pending\n进入人审队列]
    E -->|human edits| F[AI+Human diff]
    F -->|human approves| G[Approved]
    E -->|reject| H[Back to draft]
```

## Why —— 为什么

个人和小团队的 KB 失败，很少是因为大家完全不知道应该写什么。更常见的是：大家知道要写，甚至知道大概怎么写，但**没有一个足够轻的动作把它写成第一页**。第 10 章的林宇 runbook 例子就是这个形状：她知道数据预处理脚本失败时要先看输入路径、再看 schema、再看缓存，但把这些判断写成一份新人能照着做的 runbook，仍然要四十分钟。

AI drafter 的价值首先不在于把 8 分内容写成 10 分内容，而在于把 **0 页** 变成 **1 页 pending 草稿**。这件事的边际收益很大：一个 pending 草稿至少能被搜索到、能被评论、能被改；一段只在脑子里的经验没有任何维护表面。RepoAgent 把仓库级文档建设拆成结构分析、文档生成和文档更新流水线 {cite}`luo2024repoagentllmpoweredopensourceframework`，说明“让 AI 起草”只有在进入流水线之后才是工程能力，而不是一次漂亮的 prompt demo。

第二个价值是**把改写和翻译变成正常维护动作**。知识库里的很多内容并不是从无到有，而是从“口语化会议纪要”改成“runbook”、从“中文草稿”改成“中英混排的内部说明”、从“旧版 API 名称”改成“新版术语”。这些动作需要风格一致，但不一定需要作者从空白页开始。Notion AI 这类工具把“写、找、问”放在同一个界面里 {cite}`notion_ai`，说明市场已经捕捉到这个需求；本章关心的是如何把同样能力放进一个可审计、可构建的文件式 KB。

第三个价值是**让人审变得具体**。没有 AI 时，评审常常是“这页有没有人愿意写”；有 AI 后，评审变成“这 37 行 diff 是否忠于这些来源”。后者更窄，也更容易分配给小团队里的真人。

## What —— 是什么

drafter 覆盖三种动作，它们共享人审契约，但 review 难度不同：

| 动作 | 输入 | 输出 | 人审重点 | 风险 |
|:--|:--|:--|:--|:--|
| **起草** | 意图 + 原始材料 | 新页面或新段落 | 事实是否有来源、步骤是否可执行 | 幻觉事实、遗漏前提 |
| **改写** | 旧文本 + 新目标读者 | 同义但更清楚的文本 | 是否改变语义、是否丢限定条件 | “润色”时悄悄改事实 |
| **翻译** | 源语言页面 | 目标语言页面 | 专有名词、引用、数字是否保留 | 术语漂移、句法自然但概念错 |

这三种动作都不应该直接产出 `approved` 内容。第 10 章的人审契约在这里落实为三条具体规则：

1. **AI 产物先进入 `review_status: pending`。** 如果作者还没准备好让别人审，可以先保存在工作分支；一旦进入主 KB，它就必须可见为 pending。
2. **AI 不写最终 provenance。** AI 可以建议标题、摘要和标签，但 `created_by`、`reviewed_by`、`last_approved_by` 这类字段只能由流水线或人写。
3. **AI 不能删除已批准内容。** 改写可以产生替代段落，但删除原段落必须是人的 diff。

一个页面被 AI 起草后，最小 footer 应该长这样：

```text
<!-- PKB-metadata
layer: author
updated_by: ai
ai_role: drafter
ai_model: claude-sonnet-2026-04
prompt_hash: sha256:7d9c...
source_refs:
  - notes/2026-05-12-data-pipeline-debug.md
  - _audit/audio/2026-05-17-runbook-briefing.wav
review_status: pending
review_score: 0
commit: HEAD
-->
```

这里最重要的是 `source_refs` 和 `prompt_hash`。Buneman 等人关于 data provenance 的基本问题是 “why and where” —— 为什么这个结果出现、它来自哪里 {cite}`buneman2001provenance`。AI 起草的 provenance 也一样：读者不需要知道模型每个 token 的 logprob，但必须能知道这页草稿由哪些材料推导而来。

## How —— 怎么做

### 1. 先写 task brief，不直接让模型写正文

对 drafter 最有用的输入不是“帮我写一篇关于 X 的文档”，而是一份短 task brief：

```yaml
task: draft_runbook
target_file: runbooks/data-preprocess-failure.md
reader: new team member
goal: debug preprocessing failures without waking the maintainer
must_include:
  - input path check
  - schema mismatch check
  - cache invalidation step
  - when to escalate
must_not_include:
  - credentials
  - private dataset names
sources:
  - notes/2026-05-12-data-pipeline-debug.md
  - _audit/audio/2026-05-17-runbook-briefing.wav
reviewer: lin-yu
```

这份 brief 是人写的，也可以由一个 UI 帮忙生成，但它承担的是**人的意图**。AI 起草只能在这条边界内活动。没有 brief，模型会用“常见 runbook”填空；有 brief，模型被迫围绕具体来源写。

### 2. 将起草和改写分成两次调用

一次调用同时“理解来源 + 规划结构 + 写正文 + 自查引用”，很容易让错误混在一起。更稳的做法是两步：

1. **plan pass**：让模型只输出大纲、引用点和开放问题。
2. **draft pass**：把通过人或规则检查的大纲作为输入，再写正文。

这一步看似慢，实际省 review 时间。评审人可以先看 12 行大纲是否对，再看 80 行正文是否执行了大纲；不用在 80 行正文里猜模型到底从哪里开始走偏。

### 3. 改写时锁定“不可改字段”

改写比起草更容易被低估。模型会把一句“不建议在默认路径下开启缓存”改成“默认情况下不应开启缓存”，语气更顺，语义却变了。改写 prompt 必须显式列出不可改字段：

- 数字、日期、版本号；
- 文件路径、命令、API 名；
- 引用锚点；
- `must` / `should` / `may` 这类规范词；
- 否定词和范围限定词。

如果改写触碰这些字段，输出必须带 `requires_human_review: true`。不要让“只是润色”绕过评审。

### 4. 翻译保留术语表

翻译不是自然语言任务，而是术语一致性任务。小团队至少需要一个 `vocab/glossary.yaml`：

```yaml
review_status: review_status
provenance: 溯源
runbook: runbook
source_refs: source_refs
drift: 漂移
```

翻译 prompt 应该把这张表作为硬约束，而不是“参考风格”。术语表越短越好，但必须被测试：随机抽 10 个术语，确认译文没有把 `review_status` 翻成“审查状态”之类看似自然但破坏 schema 的词。

### 5. Review gate 状态机

```
[AI drafts]
    ↓
review_status: pending  ← 出现在评审队列
    ↓ (human reviews diff, decides)
updated_by: ai+human     ← 人修改过草稿
    ↓ (human approves)
review_status: approved
```

这里故意没有 `review_status: draft`。本书当前 footer contract 只把 `pending` 和 `approved` 当作可发布态；“draft”属于 frontmatter 的 `status`，不是 footer 的信任状态。AI 一旦修改了进入仓库的页面，就把 footer 打回 `pending`。人审通过后，才允许 `approved`。

## Example —— 范例

林宇要写 `runbooks/data-preprocess-failure.md`。她没有从空白页开始，而是在手机里录了 37 秒语音：

> 预处理失败先看输入路径，不要先重跑全量。第二步看 schema drift，尤其是 `sample_id` 和 `batch_id`。第三步清缓存，只清当前实验目录。还失败再看最近一次 policy 里对数据保留路径的改动。别让新人直接删 shared cache。

流水线把语音转成 `_audit/audio/2026-05-17-runbook-briefing.wav` 和对应 transcript，然后生成上面的 task brief。plan pass 输出：

```text
Outline:
1. Confirm input path and file count.
2. Compare incoming schema with expected `sample_id`, `batch_id`.
3. Clear only experiment-local cache.
4. Check data-retention policy change.
5. Escalate when shared cache appears involved.

Open question:
- Which command prints expected schema?
```

林宇补上一条命令：`python tools/print_expected_schema.py --experiment <id>`。draft pass 生成 640 字 runbook。她改了三处：

1. 把“删除缓存”改成“移动到 `.trash/` 并保留 24 小时”；
2. 给 schema 检查加上真实命令；
3. 删除模型添加的一句“如果仍失败，联系平台团队”（这个小团队没有平台团队）。

最终 diff 的 footer：

```text
updated_by: ai+human
ai_role: drafter
ai_model: claude-sonnet-2026-04
source_refs:
  - _audit/audio/2026-05-17-runbook-briefing.wav
  - policies/data-retention.md
review_status: approved
review_score: 4
reviewed_by: lin-yu
```

这页不是 AI 自动写进知识库的；它是 AI 把口头知识变成 pending 草稿，再由人把它变成 approved 页面。差别在这里。

## Common Mistakes —— 常见错误

1. **没有来源就让模型写。** 这会得到一篇像文档的文章，但它不是 KB 条目。KB 条目的最低要求是能回答“这句话从哪来”。
2. **把改写当低风险。** 改写最容易丢掉限定词。对规范、policy、runbook，改写必须和起草一样进 `pending`。
3. **让 AI 写引用。** AI 可以把来源材料映射到引用点，但引用 ID、文件路径、sha256 应由工具层填充。伪造引用比没有引用更危险。
4. **翻译时忘记术语表。** 术语漂移会让中英文版本看似等价、实际无法互相维护。
5. **让 drafter 自己评审。** 第 14 章会讲 judge，但 judge 只是辅助信号；drafter 的输出永远不能因为“模型自评通过”而 approved。

## Conclusion —— 小结

AI drafter 最适合做的不是替人做最终判断，而是把“我知道该写但还没写”的内容变成一份可评审 diff。起草、改写、翻译三种动作都要进入同一条人审闸：AI 写 `pending`，人改成 `ai+human`，人最后决定是否 `approved`。一旦把 provenance 做足，AI 起草就不再是一次不可追溯的写作外包，而是知识库流水线里的一个普通阶段。

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
