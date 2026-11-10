---
title: Use AI to build a knowledge base for a software project
status: draft
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - preface
---

# Use AI to build a knowledge base for a software project

> A practitioner's field manual that blends IR / RAG theory with live,
> measured code from two reference implementations — a prose-layer
> reference implementation and a code-layer reference implementation —
> and expands the author's methodology blog post
> {cite}`fanyamin2026deepwiki` into a book-length treatment.

## How to read this book

Each Part follows the same six-section narrative arc:

1. **Why** — the problem, the cost of ignoring it, a short literature survey.
2. **What** — the precise definitions, the data model, the scope boundaries.
3. **How** — the methodology, the architecture, the code that actually runs.
4. **Example** — an end-to-end recipe with commands, inputs, and expected
   output.
5. **Conclusion** — what we learned, what's still open, when to apply it.
6. **References** — every citation used in that Part, rendered from the
   shared `references.bib`.

Every chapter repeats the arc in miniature. You can read the book linearly,
or you can dip straight into a single Part — each one stands on its own.

## Sources

This book draws on three primary sources:

- **The prose-layer reference implementation** — a file-based wiki
  engine that backs this book's prose-layer narrative.
- **The code-layer reference implementation** — the companion
  Code-RAG + Knowledge-Graph engine that backs Part III.
- **Blog post** *给代码仓库造一个 DeepWiki* {cite}`fanyamin2026deepwiki` — the
  author's pre-existing methodology article, used as the book's narrative
  spine. See `sources/blog-deepwiki-methodology/` for the vendored snapshot
  and the non-copy policy (every chapter that cites the blog adds at least
  one new citation, one code excerpt, and one measurement the blog did not
  report).

## Relationship to the blog post

The blog post is licensed CC-BY-NC-ND 4.0. The author holds the copyright,
but we preserve the ND (no-derivatives) signal for downstream readers by:

1. Vendoring the blog HTML as an **immutable** snapshot under
   `sources/blog-deepwiki-methodology/snapshot.html`.
2. Paraphrasing and expanding the arguments in the author's own words.
3. Fencing any short verbatim quote inside an `{epigraph}` block with a
   `{cite}` to `fanyamin2026deepwiki`.

A pre-build validator (`_tools/check_blog_quotes.py`) enforces rule (3)
at build time.

## How to cite this book

```bibtex
@book{fanyamin_ai_kb_software,
  author    = {Fan, Walter (Yamin)},
  title     = {Use AI to build a knowledge base for a software project},
  year      = {2026},
  publisher = {Self-published},
  note      = {Version 0.1-draft}
}
```

## Table of contents

```{toctree}
:maxdepth: 2
:caption: Part I — Foundations

part1-foundations/index
```

```{toctree}
:maxdepth: 2
:caption: Part II — The Prose Layer

part2-prose-layer/index
```

```{toctree}
:maxdepth: 2
:caption: Part III — The Author Layer

part3-author-layer/index
```

```{toctree}
:maxdepth: 2
:caption: Part IV — The Code Layer

part4-code-layer/index
```

```{toctree}
:maxdepth: 2
:caption: Part V — Operations & Lifecycle

part5-operations-lifecycle/index
```

```{toctree}
:maxdepth: 2
:caption: Part VI — The Hybrid Layer

part6-hybrid-layer/index
```

```{toctree}
:maxdepth: 2
:caption: Part VII — Governance & Outlook

part7-governance-and-outlook/index
```

```{toctree}
:maxdepth: 1
:caption: Appendices

appendix/appendix-a-example-wiki
appendix/appendix-b-example-code-kg
appendix/appendix-c-deepwiki-checklist
appendix/appendix-d-glossary
appendix/appendix-e-bilingual-publishing
```

```{toctree}
:maxdepth: 2
:caption: 参考文献

reference/index
```

## Colophon

Built with **Sphinx + MyST** using `sphinx-rtd-theme`, `sphinxcontrib-bibtex`,
and `sphinxcontrib-mermaid`.

Build locally:

```bash
make build        # HTML  → source/_build/html/
make serve        # HTTP  → http://localhost:8800
make pdf          # PDF   → source/_build/latex/ai-kb-for-software.pdf
```
