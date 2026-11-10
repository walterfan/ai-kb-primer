---
title: "第五部分 —— 运维与生命周期"
status: draft
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - operations
---

# 第五部分 —— 运维与生命周期

```{toctree}
:maxdepth: 1

ch21-incremental-sync
ch22-evaluation-and-benchmarks
ch23-maintenance-drift-and-link-rot
```

## Why —— 为什么

让一个代码知识库一直保持健康，比从零搭出一个更难。

## What —— 是什么

由 git diff 驱动的增量同步、RAG 评测、漂移与链接腐化。

## How —— 怎么做

三章内容，锚定在博客 §10 {cite}`fanyamin2026deepwiki` 以及 `SyncJob` 状态机上。

## Example —— 范例

一次完整走完的增量同步过程，复现博客中报告的 36×/180×/9× 提速。

## Conclusion —— 小结

运维纪律是“原型级知识库”与“一个团队真正敢依赖的知识库”之间的那条分界线。

## 参考文献

```{bibliography}
:filter: keywords % "operations"
```
