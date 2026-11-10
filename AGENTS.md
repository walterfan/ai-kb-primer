# Build Knowledge Base by AI

Sphinx + MyST source and toolchain for the book *Use AI to build a
knowledge base for a software project* — a seven-part field guide to
designing, populating, maintaining, and governing a knowledge layer
for software teams and coding agents. The book is written in Chinese
with English technical terms retained.

## Knowledge base index

The book content lives entirely under `source/`. Start here:

| Entry point | What it covers |
|:--|:--|
| `source/index.md` | Book root table of contents |
| `source/part1-foundations/` | What is a software KB; IR/RAG primer; Diataxis |
| `source/part2-prose-layer/` | Markdown+Git wiki, frontmatter/footer, LLM classification, publish/search, operational model |
| `source/part3-author-layer/` | AI as author: drafting, classifying/extracting, maintaining, judging; human-review contracts |
| `source/part4-code-layer/` | Parse, embed, graph, hybrid retrieval, prompt and generation |
| `source/part5-operations-lifecycle/` | Incremental sync, evaluation/benchmarks, maintenance/drift |
| `source/part6-hybrid-layer/` | Document layering L0-L4, graph-guided context, AI agents on the KB |
| `source/part7-governance-and-outlook/` | Cost/privacy/security, trust/provenance, open problems |
| `source/reference/` | 10 annotated research paper summaries |
| `source/appendix/appendix-*.md` | Glossary, bilingual publishing guide, DeepWiki checklist |
| `source/references.bib` | BibTeX bibliography, filtered per chapter via `:filter: keywords` |
| `README.md` | Project overview, quickstart, design invariants |

Reading order for AI: `README.md` first, then `source/index.md`, then
chapters in part order. Each chapter follows a `Why / What / How /
Example / Common mistakes / Conclusion` skeleton.

## Repository layout

```
ai-kb-primer/
├── Makefile                  # All build/check/serve targets
├── pyproject.toml            # Poetry: Sphinx, MyST, bibtex, mermaid
├── poetry.lock
├── poetry.toml               # virtualenvs.in-project = true
├── README.md
├── openspec/                 # Design specs and change proposals
│   ├── specs/
│   └── changes/
└── source/                   # *** All book content lives here ***
    ├── conf.py               # Sphinx config (self-contained)
    ├── index.md              # Book root
    ├── references.bib        # BibTeX (shared, filtered per chapter)
    ├── part0-presentation/   # Talk slides
    ├── part1-foundations/     # ch01–ch03
    ├── part2-prose-layer/    # ch04–ch09
    ├── part3-author-layer/   # ch10–ch14
    ├── part4-code-layer/     # ch15–ch20
    ├── part5-operations-lifecycle/  # ch21–ch23
    ├── part6-hybrid-layer/   # ch24–ch29
    ├── part7-governance-and-outlook/  # ch30–ch32
    ├── reference/            # Research paper summaries
    ├── appendix/              # Appendix A-E standalone pages
    ├── examples/             # Runnable code excerpts (vendored)
    ├── sources/              # Raw source materials
    ├── _tools/               # Validation scripts
    ├── _static/              # CSS, images
    └── _templates/           # Sphinx layout overrides
```

## Commands

All commands use `make`. Poetry manages the Python/Sphinx toolchain.

### Setup

```bash
make setup          # Install Sphinx toolchain via Poetry
```

### Build

```bash
make build          # Build Chinese HTML (source/_build/html/)
make pdf            # PDF via xelatex (requires TeX distribution)
```

### Validate

```bash
make check              # Frontmatter schema, blog-quote policy, PKB-skill policy
make check-redaction    # Scan for internal URL / product / ticket-key leaks
make check-excerpts     # Fail if vendored excerpts are out of date
```

Run `make check` and `make check-redaction` before every commit.

### Serve

```bash
make serve          # http://localhost:8800
```

### Maintenance

```bash
make refresh-excerpts   # Re-capture vendored code excerpts with fresh SHAs
make clean              # Remove source/_build/
```

## Conventions

- **Language**: Python 3.10+. No Node, Go, or databases.
- **Build tool**: Poetry for dependency management; Make for task running.
- **Markup**: MyST Markdown (`.md`). No reStructuredText except legacy.
- **Content language**: Chinese with English technical terms retained.
  Sphinx `language` is set to `zh_CN` directly in `conf.py`.
- **Frontmatter required** on every chapter page: `title`, `status`,
  `authors`, `last_verified_commit`, `zh_status`, `keywords`. A
  `<!-- PKB-metadata -->` footer block carries `layer`, `updated_by`,
  `review_status`, `review_score`, `commit`.
- **Per-chapter bibliography**: each chapter's References section uses
  `{bibliography}` with `:filter: keywords % "xxx"` against
  `references.bib`. Never hand-write footnotes for citations.
- **No long verbatim copies** from the author's blog or PKB skill.
  `check_blog_quotes.py` and `check_pkb_quotes.py` enforce this.
- **Redaction gates**: internal URLs, product names, Jira ticket keys,
  employer emails, and private project names must not appear in
  `source/`. `make check-redaction` enforces this before every HTML build.
- **Vendored excerpts**: code snippets from the reference implementation
  are vendored under `source/examples/` with provenance headers.
  Refresh with `make refresh-excerpts`; validate with `make check-excerpts`.

## Danger zones

| Path | Risk | Rule |
|:--|:--|:--|
| `source/conf.py` | Sphinx config; breaking it breaks all builds | Test with `make build` after any edit |
| `source/references.bib` | Shared bibliography; malformed entries break all chapters | Validate BibTeX syntax before committing |
| `source/_tools/check_*.py` | Gate scripts; a false positive blocks the pipeline | Run `make check` after editing |
| `source/_tools/refresh_excerpts.py` | Reads from sibling `lazy-kb-wiki` repo | Needs `LAZY_KB_WIKI_PATH` if repo is not at `../lazy-kb-wiki` |
| `Makefile` | Back-compat aliases at bottom; removing them breaks external CI | Keep `book-*` aliases intact |

## Change workflow

Design proposals and specs live in `openspec/`. For content changes:

1. Edit Markdown under `source/partN-*/`.
2. `make check` -- fast local lint.
3. `make build` -- Sphinx builds HTML; broken cross-references fail loudly.
4. `make check-redaction` -- catch leaks before pushing.

## Agent-client wiring

| Directory | Agent |
|:--|:--|
| `.cursor/skills/`, `.cursor/commands/` | Cursor |
| `.claude/skills/`, `.claude/commands/` | Claude Code |
| `.codex/skills/` | Codex |

## When things go wrong

| Symptom | Fix |
|:--|:--|
| `make build` fails with bibtex errors | Check `references.bib` for malformed entries; verify `:filter:` keywords match |
| `make check-redaction` flags a false positive | Add the pattern to `REDACTION_EXCLUDE` in the Makefile, or reword the content |
| Vendored excerpt is stale | Run `make refresh-excerpts`; ensure sibling repo is at expected path |
| Poetry install fails | Ensure Python 3.10+; run `poetry env remove --all` then `make setup` |
| PDF build fails | Ensure TeX Live / MacTeX with `xelatex` + `xeCJK` is installed |

<!-- last_updated: 2026-04-24 -->
