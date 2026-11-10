---
title: "Ch 14 — Sync Lifecycle and Three-Layer Filtering"
status: review
authors:
  - walterfan
last_verified_commit: a1b2c3d
---

# Sync Lifecycle

The sync pipeline uses a three-layer filter to minimise work on each commit:

| Layer      | What it checks           | Cost         |
|:-----------|:-------------------------|:-------------|
| **L-git**    | `git diff --name-status` | near-zero    |
| **L-entity** | stable entity IDs        | proportional to changed files |
| **L-link**   | broken edges in graph    | proportional to affected subgraph |

## Key insight

> "Detection is cheap and deterministic; update is expensive and sometimes
> uses an LLM. Merging the two gives you the worst of both."

The `Runner` at `internal/sync/runner.go:17` implements this pipeline.
It calls `DetectChanges()` (L-git), then `reconcileEntities()` (L-entity),
then `checkBrokenLinks()` (L-link).

## Stable entity IDs

Entity identity is computed as:

```
sha256(repoID + ":" + filePath + ":" + entityType + ":" + name + ":" + startLine)[:16]
```

This is the formula in `service.go:106`. It survives re-indexing, lets
the graph do `delete-then-insert` without leaving dangling edges, and
makes incremental sync safe.

<!-- PKB-metadata -->
<!-- layer: L2 -->
<!-- updated_by: ai+human -->
<!-- review_status: approved -->
<!-- review_score: 3 -->
<!-- reviewed_by: carol -->
<!-- commit: a1b2c3d -->
