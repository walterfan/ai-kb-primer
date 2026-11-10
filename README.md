# async-pkb-book — Knowledge-base engineering for software projects

> **Knowledge building for a software project** means turning prose (wikis,
> ADRs, runbooks) and code (repos, graphs, embeddings) into a **single,
> trustworthy layer** that teams and coding agents can query, cite, and keep
> in sync as the system evolves.
>
> This repository is the **Sphinx + MyST source and toolchain** for the book
> *Use AI to build a knowledge base for a software project*: a six-part field
> guide to **designing, populating, maintaining, and governing** that layer —
> from IR/RAG basics through hybrid retrieval, operations, and responsible use
> of AI on top of your project's knowledge.

| Source | `source/`（MyST Markdown，中文为主，保留英文术语） |
|---|---|
| Build | `make build` |
| Preview | `make serve` → http://localhost:8800 |

---

## What's inside

```
async-pkb-book/
├── Makefile              # setup / check / build / serve / pdf
├── pyproject.toml        # Poetry: Sphinx, MyST, bibtex, mermaid
├── poetry.lock
├── poetry.toml           # virtualenvs.in-project = true
└── source/
    ├── conf.py                       # Sphinx config (self-contained)
    ├── references.bib                # BibTeX, filtered per chapter
    ├── index.md                      # Book root
    ├── part0-presentation/           # 60-minute talk (index/outline/slides)
    ├── part1-foundations/            # What is a software KB; IR/RAG primer
    ├── part2-prose-layer/            # Markdown + Git wiki, frontmatter/footer
    ├── part3-author-layer/           # AI as author: draft, classify, extract, maintain, judge
    ├── part4-code-layer/             # parse → embed → graph → retrieve
    ├── part5-operations-lifecycle/   # Incremental sync, drift, evaluation
    ├── part6-hybrid-layer/           # L0–L4 tuple, graph-guided context, MCP agents
    ├── part7-governance-and-outlook/ # Cost, privacy, security, open problems
    ├── appendix/                     # Appendix A-E standalone pages
    ├── _tools/                       # check_frontmatter / check_blog_quotes
    ├── _static/ _templates/          # CSS
    └── examples/ sources/            # Runnable snippets, vendored code excerpts
```

The book source is written directly in Chinese (with English technical terms
retained). Sphinx builds a single Chinese HTML edition from the MyST Markdown,
mermaid diagrams, and BibTeX bibliography.

---

## Prerequisites

- **Python** 3.10+
- **Poetry** 1.8+ — the whole toolchain (Sphinx, MyST, bibtex, mermaid)
  is declared in `pyproject.toml`.
- **ripgrep (`rg`)** — used by `make check-redaction` to scan for internal
  URL / product / ticket-key leaks before every HTML build.
- **(Optional) TeX Live / MacTeX** with `xelatex` + `xeCJK` — only needed for
  `make pdf`.

That's it. No Node, no Go, no databases. The book project is deliberately
kept to a minimal Python + Sphinx stack so it builds from a clean venv.

---

## Quickstart

```bash
# 1. 一次性安装 Sphinx 工具链
make setup                  # → poetry install --no-root

# 2. 构建 HTML
make build                  # 产出：source/_build/html/

# 3. 本地预览
make serve                  # http://localhost:8800
```

### 常用 target

| Target | 作用 |
|:--|:--|
| `make check` | 校验 frontmatter、博客引用政策、PKB-skill 非复制政策 |
| `make check-redaction` | 扫描 `source/` 里**内部 URL / 产品名 / Jira ticket key / 作者私有项目名**是否泄漏。**每次提交前都应跑一遍** |
| `make refresh-excerpts` | 重新抓取 vendored 代码片段，更新 commit SHA |
| `make check-excerpts` | 任何 excerpt 过期就失败（CI 用） |
| `make pdf` | xelatex 出 PDF，需要本地 TeX |
| `make clean` | 清理 `source/_build/` |

前一个 repo 用的 `book-*` 前缀的 target（`book-build`、`book-check`、
`book-serve`…）依然可用，作为兼容别名。

### 写作节奏

1.  在 `source/partN-*/` 下改 Markdown。每页都有 frontmatter
    （title / status / authors / `last_verified_commit` / `zh_status` /
    `keywords`）和 `<!-- PKB-metadata -->` footer
    （layer / `updated_by` / `review_status` / `review_score` / `commit`）。
    完整 schema 见 `source/appendix/appendix-e-bilingual-publishing.md`。
2.  `make check` —— 本地快速 lint。
3.  `make build` —— Sphinx 构建 HTML；任何交叉引用问题都会报错。
4.  `make check-redaction` —— 防止把内部 URL / 产品名 / 工单号推出去。

---

## Design invariants（本书对自己的硬约束）

这五条每一条都是 `Makefile` 里的机械 gate，不是君子协定：

1.  **Frontmatter schema** —— 每一页都必须有 `status / authors /
    last_verified_commit / zh_status / keywords`。`make check` 挡住缺字段
    的页面。
2.  **No long verbatim blog copy** —— 作者博客内容可以引用一两句，但不允许
    整段复制粘贴。`check_blog_quotes.py` 负责扫描。
3.  **No PKB-skill copy** —— 作者的 skill 库内容不得搬运到书里。
    `check_pkb_quotes.py` 负责扫描。
4.  **No redaction leaks** —— 内部 URL、产品名、Jira 工单号、作者私有
    project 名字，一律不许出现在 `source/` 里。`check-redaction` 在每次
    `html` build 前跑。
5.  **Per-chapter bibliography** —— 每章的 References 都用
    `:filter: keywords % "xxx"` 按关键字过滤 `references.bib`，不允许
    手写脚注。


## License

Content: **[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)**.
Code snippets (Makefile, `_tools/*.py`): **MIT**.
