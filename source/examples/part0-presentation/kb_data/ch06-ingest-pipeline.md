---
title: "Ch 6 — Ingest Pipeline"
status: review
authors:
  - walterfan
last_verified_commit: a1b2c3d
---

# Ingest Pipeline

The ingest pipeline converts raw data sources (CSV, JSON, API responses) into
structured wiki pages with frontmatter and PKB-metadata footers.

## How it works

1. **File discovery** — the pipeline scans a configured directory for new or
   modified files.
2. **Parsing** — each file type has a dedicated loader. CSV files are handled
   by `pipeline/csv_loader.go:42` (the `parseCSV` function).
3. **Transformation** — the `Transformer` in `pipeline/transformer.go:27`
   applies normalisation, validation, and enrichment steps.
4. **Classification** — an LLM classifier assigns a Diátaxis category
   (tutorial / how-to / reference / explanation).
5. **Storage** — the resulting page is written to the prose-layer wiki with
   full frontmatter and a `PKB-metadata` footer.

## CSV-specific details

The CSV loader (`pipeline/csv_loader.go:42`) supports:

- Custom delimiters (comma, tab, pipe)
- Header-row detection
- Required-column validation via `Validate()`

After loading, records pass through a `Transformer` pipeline
(`pipeline/transformer.go:27`) that normalises whitespace and rejects
empty required fields.

<!-- PKB-metadata -->
<!-- layer: L2 -->
<!-- updated_by: human -->
<!-- review_status: approved -->
<!-- review_score: 4 -->
<!-- reviewed_by: alice -->
<!-- commit: a1b2c3d -->
