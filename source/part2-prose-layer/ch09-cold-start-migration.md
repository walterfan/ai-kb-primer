---
title: "第 9 章 —— 冷启动与批量迁移"
status: draft
authors: [AI]
last_verified_commit: HEAD
zh_status: complete
keywords: [cold-start, migration, confluence, notion, jira, google-docs]
---

# 第 9 章 —— 冷启动与批量迁移

## 为什么

知识库最大的敌人不是技术，而是**空白页面**。

一个典型的中型团队在决定"建知识库"之前，往往已经积累了：

- 500+ 页 Confluence wiki（其中 60% 两年没人碰过）
- 200+ 篇 Notion 文档（一半是 database 视图）
- 1000+ 条 Jira ticket（epic description 里藏着架构决策）
- 几十份 Google Docs（评审会议纪要、设计讨论稿）

如果你的 KB 第一天只有空白模板，没有人会用它。**冷启动的核心问题不是"怎么导"，而是"怎么在导入过程中建立信任"** —— 导进来的东西必须有正确的 `verification_status`、合理的 `doc_type`、和真实的 `source_path`。

本章把这个过程拆成六个阶段，每个阶段都有明确的入口条件和出口检查。


## 是什么

```{mermaid}
flowchart LR
    A["Phase 1<br/>盘点与分流"] --> B["Phase 2<br/>源端导出"]
    B --> C["Phase 3<br/>转换为 Markdown"]
    C --> D["Phase 4<br/>批量分类"]
    D --> E["Phase 5<br/>去重与冲突"]
    E --> F["Phase 6<br/>增量追赶"]
```

| 阶段 | 输入 | 输出 | 关键决策 |
|:--|:--|:--|:--|
| Phase 1 盘点与分流 | 所有源端页面列表 | keep / archive / discard 三分表 | 什么值得迁移？ |
| Phase 2 源端导出 | keep 列表 | 原始文件（HTML / Markdown / JSON） | 导出粒度（全量 vs 增量） |
| Phase 3 格式转换 | 原始文件 | 带 frontmatter 的 `.md` 文件 | 元数据映射规则 |
| Phase 4 批量分类 | `.md` 文件 | 带 `doc_type` 的页面 | LLM vs 启发式回退 |
| Phase 5 去重与冲突 | 分类后的页面 | 去重后的唯一页面集 | 同源 vs 跨源冲突 |
| Phase 6 增量追赶 | KB + 源端变更流 | 持续同步状态 | webhook vs 轮询 |


## 怎么做

### Phase 1：盘点与分流

在碰任何导出按钮之前，先做一次**盘点**：

```bash
# Confluence: 用 REST API 拉页面清单
curl -u user:token "https://wiki.example.com/rest/api/content?limit=500&expand=version" \
  | jq '.results[] | {id, title, version: .version.number, lastUpdated: .version.when}' \
  > confluence_inventory.jsonl

# Notion: 用 Notion API 拉数据库
curl -X POST "https://api.notion.com/v1/search" \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: 2022-06-28" \
  -d '{"page_size": 100}' > notion_inventory.json
```

然后按三个维度打分，决定 keep / archive / discard：

| 维度 | 权重 | 评分标准 |
|:--|:--|:--|
| **新鲜度** | 40% | 最后更新距今 < 6 月 = 3, < 2 年 = 2, > 2 年 = 1 |
| **引用密度** | 30% | 被其他页面引用 ≥ 3 次 = 3, 1-2 次 = 2, 0 次 = 1 |
| **知识类型** | 30% | ADR/runbook = 3, how-to/tutorial = 2, meeting notes = 1 |

- 综合分 ≥ 7：**keep** —— 迁入 KB
- 综合分 4-6：**archive** —— 保留源端只读副本，不导入
- 综合分 ≤ 3：**discard** —— 标记为废弃

> **经验法则**：典型团队的 keep 率约 30-40%。如果你的 keep 率超过 70%，说明分流标准太松。


### Phase 2：源端导出

#### Confluence → 原始 HTML

```bash
# 方法一：REST API 逐页导出（推荐，可控性强）
for page_id in $(cat keep_list.txt); do
  curl -u user:token \
    "https://wiki.example.com/rest/api/content/${page_id}?expand=body.export_view,ancestors" \
    -o "confluence_raw/${page_id}.json"
  sleep 0.5  # 限速，避免触发 429
done

# 方法二：Confluence 空间导出（一次性，适合小空间）
# Admin → Space Settings → Export → HTML
```

#### Notion → Markdown

```bash
# Notion 原生导出：Settings → Export → Markdown & CSV
# 注意：database 页面会导出为 CSV，需要后处理

# 更好的方式：用 notion2md 工具
pip install notion2md
notion2md --token $NOTION_TOKEN --output notion_raw/
```

#### Jira → JSON

```bash
# JQL 查询知识密集型 ticket
curl -u user:token \
  "https://jira.example.com/rest/api/2/search?jql=project=PROJ+AND+type+in+(Epic,Story)+AND+resolution=Done&maxResults=200&fields=summary,description,comment,resolution,components" \
  -o jira_raw/tickets.json
```

#### Google Docs → DOCX

```bash
# 使用 Google Drive API 批量下载为 DOCX
# 然后用 pandoc 转为 Markdown
for doc_id in $(cat gdocs_keep.txt); do
  curl -H "Authorization: Bearer $GCLOUD_TOKEN" \
    "https://www.googleapis.com/drive/v3/files/${doc_id}/export?mimeType=application/vnd.openxmlformats-officedocument.wordprocessingml.document" \
    -o "gdocs_raw/${doc_id}.docx"
done
```


### Phase 3：格式转换

核心任务是把各种格式转为**带 frontmatter 的 Markdown**。

```bash
# Confluence HTML → Markdown
for f in confluence_raw/*.json; do
  page_id=$(basename "$f" .json)
  # 提取 HTML body + 元数据
  jq -r '.body.export_view.value' "$f" | pandoc -f html -t markdown-raw_html \
    --wrap=none -o "converted/${page_id}.md"
  # 注入 frontmatter
  python3 inject_frontmatter.py "converted/${page_id}.md" \
    --source "confluence" \
    --source-path "https://wiki.example.com/pages/${page_id}" \
    --created-by "$(jq -r '.version.by.displayName' "$f")" \
    --created-at "$(jq -r '.version.when' "$f")"
done

# Google Docs DOCX → Markdown
for f in gdocs_raw/*.docx; do
  pandoc "$f" -t markdown --extract-media=_static/gdocs/ \
    -o "converted/$(basename "$f" .docx).md"
done
```

`inject_frontmatter.py` 生成的 frontmatter 模板：

```yaml
---
title: "从 Confluence 迁移的页面标题"
status: draft
authors: ["原始作者"]
last_verified_commit: HEAD
zh_status: none
keywords: []
source: confluence
source_path: "https://wiki.example.com/pages/12345"
created_by: "John Doe"
created_at: "2024-03-15T10:30:00Z"
migrated_at: "2026-04-26T12:00:00Z"
---
```

### Phase 4：批量分类

复用第 6 章的分类流水线，但需要注意批量模式的特殊处理：

```bash
# 不要一次性跑完所有页面！分批处理
ls converted/*.md | split -l 50 - batch_

for batch in batch_*; do
  for f in $(cat "$batch"); do
    lazy-kb ingest "$f" --classifier llm --dry-run  # 先 dry-run 检查
  done
  echo "Batch $batch done. Sleeping 60s for rate limit..."
  sleep 60
done

# 确认分类结果
lazy-kb status --format table | head -20
```

> **成本预算**：假设 300 页 × 平均 2k tokens/页 × $0.15/1M input tokens (GPT-4o-mini) ≈ $0.09。批量分类的成本几乎可以忽略。


### Phase 5：去重与冲突

同一份文档可能在 Confluence 和 Notion 里各有一份。去重算法：

```python
import hashlib

def content_hash(body: str) -> str:
    """归一化后计算内容指纹"""
    normalized = body.strip().lower()
    normalized = re.sub(r'\s+', ' ', normalized)  # 压缩空白
    normalized = re.sub(r'!\[.*?\]\(.*?\)', '', normalized)  # 移除图片
    return hashlib.sha256(normalized.encode()).hexdigest()[:16]

def dedup(pages: list[Page]) -> list[Page]:
    """保留最新的版本"""
    seen: dict[str, Page] = {}
    for page in sorted(pages, key=lambda p: p.created_at, reverse=True):
        h = content_hash(page.body)
        if h not in seen:
            seen[h] = page
        else:
            # 记录冲突，保留较新的
            log.info(f"Duplicate: {page.source_path} ≈ {seen[h].source_path}")
    return list(seen.values())
```

**跨源冲突解决规则**：

| 冲突类型 | 策略 |
|:--|:--|
| 内容完全相同 | 保留 `created_at` 较新的 |
| 内容相似（hash 不同但标题相同） | 手动审核，标记为 L3 |
| 同一主题不同视角 | 两份都保留，在 frontmatter 中互相引用 |


### Phase 6：增量追赶

迁移不是一次性的。从你开始导出到完成分类，源端可能已经有了新变更。

```yaml
# cron_catch_up.yaml — 每日增量追赶
schedule: "0 2 * * *"  # 每天凌晨 2 点
tasks:
  - name: confluence-catchup
    type: webhook  # Confluence 支持 webhook
    endpoint: /api/webhooks/confluence
    filter: "space = PROJ AND lastModified > ${LAST_SYNC}"

  - name: jira-catchup
    type: polling  # Jira 用轮询
    jql: "project = PROJ AND updated > -1d"
    interval: 24h

  - name: notion-catchup
    type: polling  # Notion API 用 last_edited_time 过滤
    filter: "last_edited_time > ${LAST_SYNC}"
    interval: 12h
```

追赶阶段的页面与首次导入走同样的 Phase 3-5 流水线，但 `migrated_at` 字段更新为当前时间。


## 来源信任默认值

不同来源的文档，信任起点不同：

| 来源 | 默认 layer | review_status | verification_status | 理由 |
|:--|:--|:--|:--|:--|
| **OpenSpec / ADR** | L1 | approved | supported | 已经过正式评审流程 |
| **Confluence（人写）** | L1 | pending | unreviewed | 未经迁移后验证 |
| **Notion（混合）** | L2 | pending | unreviewed | 可能含 AI 生成内容 |
| **Jira ticket** | L1 | pending | unreviewed | 上下文可能已过时 |
| **Google Docs** | L2 | pending | uncertain | 多人协作，出处不清 |
| **AI 生成计划** | L3 | pending | unreviewed | 始终需要人类审核 |

> **铁律**：任何自动导入的页面，`review_status` **一律设为 `pending`**。绝对不能因为 Confluence 上"看起来像是被审核过的"就设为 `approved`。


## 示例

假设 Acme 团队决定迁移到 KB。他们的存量：

- Confluence：50 页（PROJ 空间）
- Notion：30 篇文档（工程部 workspace）

```
Phase 1: 盘点
  Confluence 50 页 → keep 18, archive 20, discard 12
  Notion 30 页 → keep 12, archive 10, discard 8
  合计 keep: 30 页

Phase 2: 导出
  Confluence: REST API 逐页导出 → 18 个 JSON 文件
  Notion: notion2md 导出 → 12 个 .md 文件
  耗时: ~15 分钟

Phase 3: 转换
  pandoc 转换 + frontmatter 注入
  18 + 12 = 30 个标准 .md 文件
  耗时: ~10 分钟（脚本自动化）

Phase 4: 分类
  30 页 × LLM 分类 → 8 tutorial, 10 how-to, 7 reference, 5 explanation
  成本: 30 × 2k tokens × $0.15/1M ≈ $0.01
  耗时: ~5 分钟

Phase 5: 去重
  发现 3 对跨源重复 → 保留较新版本
  最终: 27 个唯一页面

Phase 6: 追赶
  设置 Confluence webhook + Notion 轮询
  第一周捕获 2 个更新页面
```

**从零到一个 27 页的可搜索 KB，总耗时约 1 小时，成本 < $0.10。**


## 常见错误

1. **什么都迁**。不做 Phase 1 分流，把 500 页全导进来。结果：KB 充满过期内容，搜索质量暴跌，用户失去信任。

2. **丢失页面层级**。Confluence 的 parent-child 关系在导出时被拍平。应该把 `ancestors` 字段映射为目录结构。

3. **信任默认值设太高**。导入的页面全部标记为 `approved`。后果：stale 内容被当作权威来源。**一律设为 `pending`**。

4. **忽略附件和图片**。Confluence 页面嵌的图（附件存储）在 HTML 导出里是相对路径。必须下载附件并映射到 `_static/` 目录。

5. **一次性跑完所有分类**。300 页同时发给 LLM API，触发 429 限速。应该分批处理（每批 50 页，间隔 60 秒）。

6. **不处理 Confluence 宏**。`{code}`, `{panel}`, `{expand}`, `{toc}` 等 Confluence 宏在 HTML 导出中会变成复杂的 `div` 结构。pandoc 默认丢弃它们。需要预处理脚本把宏转为等价 Markdown。

7. **忘记增量追赶**。迁移耗时一周，这一周源端新增了 10 页修改。如果不做 Phase 6，KB 从第一天就是过时的。


## 小结

冷启动不是技术问题，是**信任建设问题**。六阶段框架确保：

1. **只迁有价值的**（Phase 1 分流）
2. **保留出处链**（Phase 3 frontmatter 注入）
3. **诚实标记信任级别**（Phase 4-5 默认 pending）
4. **不掉队**（Phase 6 增量追赶）

迁移完成后的 KB，每一页都能回答三个问题："这是从哪来的？""谁审核过？""最后一次验证是什么时候？"


## 参考文献

```{bibliography}
:filter: keywords % "migration"
```


<!-- PKB-metadata
layer: L2
updated_by: ai
review_status: pending
review_score: 0
commit: HEAD
-->
