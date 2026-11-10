---
title: "Runbook — Ingest Pipeline Operations"
status: review
authors:
  - walterfan
last_verified_commit: a1b2c3d
---

# Ingest Pipeline Runbook

## When to use this runbook

Use this when the ingest pipeline fails or produces unexpected output.
Typical triggers: CSV parsing errors, missing columns, transformation
step failures.

## Step 1 — Check the CSV loader logs

Look for errors from `pipeline/csv_loader.go:58` — this is where
`parse()` reports per-line read failures.

## Step 2 — Validate required columns

Run the `Validate()` function at `pipeline/csv_loader.go:86` against
the failing input. It checks that every record has the configured
required columns.

## Step 3 — Inspect transformer steps

Each `TransformStep` logs its name. Check `pipeline/transformer.go:38`
for the `Transform()` method — it collects per-record errors and skips
broken rows.

## Step 4 — Re-run with debug logging

Set `LOG_LEVEL=debug` and re-trigger the ingest. The pipeline
(`service.go:32`) prints detailed timing for each phase.

<!-- PKB-metadata -->
<!-- layer: L1 -->
<!-- updated_by: human -->
<!-- review_status: approved -->
<!-- review_score: 5 -->
<!-- reviewed_by: bob -->
<!-- commit: a1b2c3d -->
