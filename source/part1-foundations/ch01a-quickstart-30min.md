---
title: "第 1a 章 —— 30 分钟快速上手"
status: draft
authors:
  - AI
last_verified_commit: HEAD
zh_status: complete
keywords:
  - quickstart
  - tutorial
  - hands-on
---

# 第 1a 章 —— 30 分钟快速上手

本章是一份**独立可用**的动手教程。你不需要读过第 1 章，也不需要理解 IR、RAG 或 Diátaxis —— 这些概念会在后面的章节逐一展开。这里的目标只有一个：**在 30 分钟内，让你亲手跑通一个软件知识库的完整流水线**，从导入 Markdown 到看到带引用的 RAG 回答。

全貌一页看完：

```{mermaid}
flowchart LR
    subgraph "文本层 Prose Layer（~15 min）"
        A[init] --> B[import]
        B --> C[ingest]
        C --> D[verify]
        D --> E[build]
        E --> F[status]
    end
    subgraph "代码层 Code Layer（~10 min）"
        G[parse] --> H[embed]
        H --> I[graph]
        I --> J[retrieve]
        J --> K[generate]
    end
    F --> L[HTTP Server\n搜索 & 浏览]
    K --> M[RAG Answer\n带引用的回答]
```

## 为什么

很多技术书在第一章就抛出 50 页理论，读者直到第四部分才能"动手"。本章反其道行之：**先动手、后理论**。你花 30 分钟跑通一遍之后，后面每一章讲的概念——embedding、graph、provenance、document layering——都会有一个你亲手摸过的锚点。

## 是什么

一份分步指南，覆盖两条流水线：

1. **文本层（Prose Layer）**：把 3 份 Markdown 文件导入一个基于文件的 wiki，经过分类、校验、静态构建，最后在浏览器里看到搜索结果。
2. **代码层（Code Layer）**：对一个小型 Go 仓库做代码解析、向量嵌入、图构建、混合检索，最后向知识库提问并看到带引用的 RAG 回答。

## 怎么做

### 第 0 步 —— 环境准备（约 3 分钟）

**前置条件清单：**

| 工具 | 最低版本 | 检查命令 |
|------|----------|----------|
| Python | 3.10+ | `python3 --version` |
| Poetry | 1.7+ | `poetry --version` |
| Git | 2.30+ | `git --version` |
| Docker & Docker Compose | 24.0+ | `docker compose version` |
| Go（可选，仅代码层） | 1.21+ | `go version` |

```bash
# 确认版本
python3 --version   # >= 3.10
poetry --version    # >= 1.7
git --version       # >= 2.30
docker compose version  # >= 2.20
```

如果你缺少上面任一工具，请先安装。Poetry 的推荐安装方式：

```bash
curl -sSL https://install.python-poetry.org | python3 -
```

### 第 1 步 —— 克隆参考实现（约 2 分钟）

本书有两个参考实现仓库。文本层是一个 Go + Vue 的 wiki-cli，代码层是一套 Python 服务。

```bash
# 克隆文本层参考实现
git clone https://github.com/walterfan/lazy-kb-wiki.git
cd lazy-kb-wiki

# 克隆本书仓库（含构建工具链和示例文件）
cd ..
git clone https://github.com/walterfan/ai-kb-primer.git
```

安装书稿仓库的 Python 依赖（后续步骤会用到）：

```bash
cd ai-kb-primer
make setup   # 等价于 poetry install
cd ..
```

### 第 2 步 —— 准备 3 份示例 Markdown（约 2 分钟）

在 `lazy-kb-wiki` 目录下创建一个 `inbox/` 文件夹，放入 3 份示例文件。这模拟的是"有人把一堆乱糟糟的文档倾倒进来"的真实场景。

```bash
cd lazy-kb-wiki
mkdir -p inbox
```

**文件 1 — 一份 tutorial：**

```bash
cat > inbox/getting-started.md << 'EOF'
# Getting Started with the Wiki

This tutorial walks you through setting up the wiki
from scratch in under 5 minutes.

## Prerequisites
- Go 1.21+
- A working Git installation

## Steps
1. Clone the repository
2. Run `make build`
3. Start the server with `./wiki serve`
EOF
```

**文件 2 — 一份 ADR：**

```bash
cat > inbox/adr-0001-use-sqlite.md << 'EOF'
# ADR-0001: Use SQLite for local storage

## Status
Accepted

## Context
We need a lightweight database for the wiki's local index.

## Decision
Use SQLite via `modernc.org/sqlite` (pure-Go driver).

## Consequences
- No CGO dependency
- Single-file database, easy to back up
- Limited concurrent write throughput (acceptable for our scale)

## Rejected alternatives
- PostgreSQL: too heavy for single-user local mode
- BoltDB: no SQL, harder to query ad-hoc
EOF
```

**文件 3 — 一份 runbook：**

```bash
cat > inbox/runbook-backup.md << 'EOF'
# Runbook: Wiki Backup Procedure

## When to use
- Before any major upgrade
- Weekly automated backup (cron)

## Steps
1. Stop the wiki server: `systemctl stop wiki`
2. Copy the data directory: `cp -r /var/wiki/data /backup/wiki-$(date +%F)`
3. Verify the backup: `sqlite3 /backup/wiki-*/index.db "SELECT count(*) FROM pages;"`
4. Restart: `systemctl start wiki`

## Escalation
If backup fails, page oncall via PagerDuty.
EOF
```

### 第 3 步 —— 运行文本层 6-verb 流水线（约 5 分钟）

这是本书第 6 章详述的那条流水线。六个动词分别是：`init`、`import`、`ingest`、`verify`、`build`、`status`。

```bash
# ❶ init — 初始化 wiki 的内容目录和配置
./wiki init --content-dir ./content

# ❷ import — 把 inbox/ 里的原始文件复制到暂存区
./wiki import --from ./inbox --to ./content/staging

# ❸ ingest — 分类、生成 frontmatter、归一化文件名
#    这一步会调用 LLM 对每份文件做 doc_type 分类
#    （如果没有 API key，可以加 --classifier=rule 用规则分类器）
./wiki ingest --staging ./content/staging \
              --target ./content/pages \
              --classifier=rule

# ❹ verify — 校验每份文件的 frontmatter schema
./wiki verify --content-dir ./content/pages

# ❺ build — 生成静态 HTML 和搜索索引
./wiki build --content-dir ./content/pages \
             --output-dir ./public

# ❻ status — 一行命令看全局状态
./wiki status --content-dir ./content/pages
```

`status` 的输出类似：

```text
Pages:    3
  tutorial:    1  (getting-started)
  adr:         1  (adr-0001-use-sqlite)
  runbook:     1  (runbook-backup)
Pending review: 3
Last build:     2026-04-26T16:30:00Z
Index entries:  3
```

### 第 4 步 —— 启动 HTTP 服务，看搜索结果（约 3 分钟）

```bash
# 启动 wiki 的 HTTP 服务
./wiki serve --port 8800 --content-dir ./content/pages
```

打开浏览器，访问 `http://localhost:8800`。你应该能看到三份页面。

在搜索框中输入 `SQLite`，你会看到 ADR-0001 出现在结果列表里 —— 搜索走的是 build 阶段生成的倒排索引（BM25）。输入 `backup`，runbook 会排到第一位。

```{tip}
如果你看到 "0 results"，检查一下 `wiki build` 是否成功生成了 `public/search-index.json`。
```

按 `Ctrl+C` 停止服务，继续下面的代码层流水线。

### 第 5 步 —— 运行代码层流水线（约 10 分钟）

代码层流水线对一个真实的 Go 仓库做五件事：parse → embed → graph → retrieve → generate。

先启动必要的基础设施（Memgraph + pgvector）：

```bash
cd ../ai-kb-primer
docker compose -f docker-compose.dev.yml up -d
```

等待服务就绪后，运行代码层的五步流水线。这里我们以 `lazy-kb-wiki` 自身的 Go 代码作为目标仓库：

```bash
# ❶ parse — 用 tree-sitter 解析 Go 源码，提取函数、类型、调用关系
poetry run python -m codebase_kb.parse \
    --repo ../lazy-kb-wiki \
    --lang go \
    --output parsed.json

# ❷ embed — 对每个代码块生成 embedding 并存入 pgvector
poetry run python -m codebase_kb.embed \
    --input parsed.json \
    --model text-embedding-3-small \
    --db postgresql://localhost:5432/kb

# ❸ graph — 把调用图和类型关系写入 Memgraph
poetry run python -m codebase_kb.graph \
    --input parsed.json \
    --memgraph bolt://localhost:7687

# ❹ retrieve — 混合检索（向量 + 图 + 关键词）
poetry run python -m codebase_kb.retrieve \
    --query "How does the wiki handle page creation?" \
    --db postgresql://localhost:5432/kb \
    --memgraph bolt://localhost:7687 \
    --top-k 5

# ❺ generate — 用检索到的上下文生成带引用的回答
poetry run python -m codebase_kb.generate \
    --query "How does the wiki handle page creation?" \
    --db postgresql://localhost:5432/kb \
    --memgraph bolt://localhost:7687
```

### 第 6 步 —— 提问并看到 RAG 回答（约 3 分钟）

上一步的 `generate` 命令会输出一段类似这样的回答：

```text
Page creation is handled by the CreatePage function in
internal/service/service_write.go:47.

The flow is:
  1. Validate frontmatter via ValidateFrontmatter()
     (internal/model/frontmatter.go:112)
  2. Generate slug from title
     (internal/model/slug.go:23)
  3. Write file to content directory
     (internal/service/service_write.go:78)
  4. Update search index
     (internal/search/indexer.go:55)

The frontmatter merge logic (MergeFrontmatterUpdate) ensures
that created_by is never overwritten after initial creation,
as discussed in the codebase documentation.

Confidence: medium (4 citations; no ADR found for this flow;
call graph current as of HEAD).
```

注意这里每一行引用都包含 `file:line` —— 这些是 tree-sitter 解析出来的真实位置，不是 LLM 从训练数据中"回忆"出来的。

### 第 7 步 —— 清理（约 2 分钟）

```bash
# 停止基础设施
docker compose -f docker-compose.dev.yml down

# （可选）删除测试数据
rm -rf ../lazy-kb-wiki/inbox ../lazy-kb-wiki/content ../lazy-kb-wiki/public
```

## 示例：你刚刚搭了什么

下面这张图是你在 30 分钟里走过的完整路径：

```{mermaid}
graph TB
    subgraph "输入 Inputs"
        MD["3 × Markdown<br/>(tutorial, ADR, runbook)"]
        GO["Go 源码仓库<br/>(lazy-kb-wiki)"]
    end

    subgraph "文本层 Prose Pipeline"
        P1[init] --> P2[import]
        P2 --> P3["ingest<br/>(分类 + frontmatter)"]
        P3 --> P4["verify<br/>(schema 校验)"]
        P4 --> P5["build<br/>(HTML + 索引)"]
        P5 --> P6["status<br/>(全局快照)"]
    end

    subgraph "代码层 Code Pipeline"
        C1["parse<br/>(tree-sitter)"] --> C2["embed<br/>(pgvector)"]
        C2 --> C3["graph<br/>(Memgraph)"]
        C3 --> C4["retrieve<br/>(混合检索)"]
        C4 --> C5["generate<br/>(RAG 回答)"]
    end

    subgraph "输出 Outputs"
        WEB["Wiki 站点<br/>localhost:8800"]
        ANS["RAG 回答<br/>带 file:line 引用"]
    end

    MD --> P1
    GO --> C1
    P6 --> WEB
    C5 --> ANS

    style WEB fill:#e1f5fe
    style ANS fill:#e8f5e9
```

这就是本书在第 1 章所说的"软件知识库"的最小可工作版本：**文本层**承载 tutorial、ADR 和 runbook，**代码层**承载函数、调用图和类型签名。两层合在一起，覆盖了第 1 章定义的四类材料中的全部四类。

## 常见错误

1. **Python 版本太低。** `codebase_kb` 依赖 Python 3.10+ 的 `match` 语法和 `typing` 特性。如果你在 `poetry install` 时看到 `SyntaxError`，几乎可以确定是版本问题。运行 `python3 --version` 确认，或用 `pyenv` 切换。

2. **忘记设置 `OPENAI_API_KEY`（或等价的环境变量）。** `embed` 和 `generate` 步骤需要调用 embedding / LLM API。如果你不想用 OpenAI，可以在 `.env` 中配置兼容的本地模型端点（如 Ollama）：
   ```bash
   export OPENAI_API_KEY="sk-..."
   # 或者用本地模型
   export EMBEDDING_BASE_URL="http://localhost:11434/v1"
   ```

3. **Docker 服务没起来就跑代码层。** `embed` 需要 pgvector，`graph` 需要 Memgraph。如果你看到 `ConnectionRefusedError`，先检查 `docker compose ps` 确认两个容器都在 `running` 状态。

4. **`wiki ingest` 用 LLM 分类器但没有 API key。** 默认的 `--classifier=llm` 需要网络和 API key。本教程用 `--classifier=rule` 来绕开这个依赖。规则分类器根据文件名和正文关键词做分类，准确率低于 LLM 分类器，但足够完成本教程。

5. **`wiki build` 报 "no pages found"。** 通常是因为 `ingest` 那一步的 `--target` 路径和 `build` 的 `--content-dir` 路径不一致。六个动词之间的目录约定是：`staging/` → `pages/`。

6. **在 M1/M2 Mac 上 Docker 镜像拉不下来。** Memgraph 的官方镜像目前只有 `amd64`。解决办法是加 `--platform linux/amd64`，或者用 `docker-compose.dev.yml` 中已经配置好的 platform 字段。

7. **`parse` 输出为空。** tree-sitter 的 Go 语法需要 `.go` 文件。如果你传入的 `--repo` 路径下没有 Go 源码（比如不小心指向了 `ai-kb-primer`），`parsed.json` 会是一个空数组。确认 `--repo` 指向的是 `lazy-kb-wiki`。

## 时间预算总览

| 步骤 | 预计时间 | 备注 |
|------|----------|------|
| 第 0 步：环境准备 | 3 min | 已有工具则跳过 |
| 第 1 步：克隆仓库 | 2 min | 取决于网速 |
| 第 2 步：准备示例文件 | 2 min | 复制粘贴即可 |
| 第 3 步：6-verb 流水线 | 5 min | `ingest` 最慢 |
| 第 4 步：HTTP 服务 + 搜索 | 3 min | 含浏览器操作 |
| 第 5 步：代码层流水线 | 10 min | Docker 冷启动约 3 min |
| 第 6 步：RAG 回答 | 3 min | 含阅读输出 |
| 第 7 步：清理 | 2 min | 可选 |
| **合计** | **~30 min** | |

## 小结

你刚才在 30 分钟里跑通了两条完整的流水线：

- **文本层**：`init → import → ingest → verify → build → status`，把 3 份裸 Markdown 变成了一个带 frontmatter、schema 校验和搜索索引的静态 wiki。
- **代码层**：`parse → embed → graph → retrieve → generate`，把一个 Go 仓库变成了一个可查询的知识图谱，并用 RAG 生成了带 `file:line` 引用的回答。

这两层合在一起，就是本书所说的"软件知识库"的最小形态。后面的章节会逐层深入：

- **第 2 章** 会解释 "retrieve" 那一步背后的 IR 与 RAG 理论。
- **第 3 章** 会解释 "ingest" 那一步为什么按 Diátaxis 分类。
- **第 5 章** 会解释 frontmatter 和 footer 为什么长那个样子。
- **第 6 章** 会把文本层流水线的每一个 verb 拆开来看。
- **第 8–13 章** 会把代码层流水线的每一步都展开成一整章。

现在你手上有了一个可以回来反复对照的实物。往后读的时候，如果某一段理论让你犯困，回到这一章重跑一遍对应的那个步骤——它永远比抽象描述更好懂。

## 参考文献

```{bibliography}
:filter: keywords % "quickstart"
```

<!-- PKB-metadata
last_updated: 2026-04-26
commit: HEAD
updated_by: ai
review_status: pending
review_score: 0
reviewed_by:
-->
