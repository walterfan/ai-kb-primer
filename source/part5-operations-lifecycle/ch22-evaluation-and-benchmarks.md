---
title: "第 22 章 —— 评测、基准与发布门禁"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - operations
  - ir-rag
  - evaluation
---

# 第 22 章 —— 评测、基准与发布门禁

## Why —— 为什么

一个回答不了 * “我怎么知道这次 build 好不好？” * 的知识库，就不能自称被“维护着”。这个问题有两种诚实的拆解，本章把两种都讲一遍：

1.  **评测** —— *知识库所驱动的检索 + 生成系统到底好不好？* 这是 IR 文献的问题：precision at k、nDCG、回答忠实度。BEIR {cite}`thakur2021beir` 和 RAGAS {cite}`es2024ragas` 这样的基准之所以存在，正是因为不同团队在趋向同一个答案。
2.  **门禁** —— *这一次具体的 build 是否格式合规到可以发布？* 这是构建工程的问题：每一页的 footer 都在吗？引用都能解析吗？变更的页面有人类审过吗？这就是第 8 章 C4 角所问的那个问题 —— 也是 IR 基准不能直接帮上忙的那个。

评测告诉你 * 好到什么程度 * 。门禁告诉你 * 要不要发 * 。两者互补：一个没有门禁就放出去的知识库，最终会因为结构不合规的页面不断累积，在评测上失分；一个只做评测、不做门禁的知识库，拿出来的漂亮数字，对应的是一个根本无法真正部署的快照。

本章列出的门禁目录源自作者的 project-knowledge-base 工具包 {cite}`fanyamin_pkb_skill`，并新增了一列 —— * 由谁强制执行 * —— 把每一条门禁映射到一段具体的 CI 机制。原工具包的做法对“强制”是隐式处理的；把它显式化是本书的一份贡献。

## What —— 是什么

### 评测

对代码层检索系统（第 19 章）而言，有用的指标就是那些标准的 IR 指标：在一份留出的 `(问题, 引用)` 对测试集上跑 nDCG@10、recall@10、MRR。对文稿层检索，这些指标同样适用，但有一个注脚：ground truth 更难标注（同一个问题常常对应多个正确页面），并且“部分得分”更重要。两类面板都能从 RAGAS {cite}`es2024ragas` 及其同类框架里的“端到端 answer-faithfulness”指标中获益。

一个可以直接上手的初始指标集：

| Metric | Layer | What it measures | Benchmark corpus |
|:--|:--|:--|:--|
| nDCG@10 | code | Are the top-10 retrieved entities ranked correctly? | Held-out `(question, code_entities)` pairs |
| Recall@10 | code | Did we at least find the right entities? | Same |
| Answer faithfulness (RAGAS) | prose+code | Does the generated answer cite its own context? | RAGAS pipeline on sample Q&A |
| Citation resolve-rate | prose+code | Do the `file:line` anchors in generated answers resolve at HEAD? | Every generated answer |
| BEIR-style domain scores | prose | How does retrieval quality compare against public IR corpora? | BEIR {cite}`thakur2021beir` |

评测数字应当在一个固定的基准集上沿时间线追踪；单独一个 nDCG@10 = 0.74 不是什么主张，但一次同步之后从 0.74 变成 0.71 就是了。

值得注意的是，学界已经为分层评测提供了成熟的框架。RepoBench 将评测拆成三个独立的子任务 —— 检索层（R）、补全层（C）和端到端流水线（P），使得"检索质量差还是生成质量差"这个诊断问题可以被精确回答 {cite}`liu2023repobench`。这种 R/C/P 拆解与本章"评测 + 门禁"的二元结构高度一致：R 对应代码层检索指标，C 对应生成忠实度，P 对应端到端。Long Code Arena 则把评测推向了更接近真实工作负载的尺度：它覆盖了项目级补全、CI 修复、Bug 定位、模块摘要、仓库 QA 和 Commit 消息生成六类长上下文任务，平均上下文长度在 30K–80K tokens {cite}`bogomolov2024longcodearena`。知识库的评测集设计应当参考这类多任务框架，而不只是在 nDCG 上做单一维度的追踪。

### 门禁

门禁属于 * 构建工程 * 的问题。每一条门禁都是一个跑在 CI 上、只返回 pass / fail 的机械检查。目录把它们区分为 **硬门禁** （阻止发布，非零退出码）和 **软门禁** （发警告，允许发布但提醒评审者）。

#### 硬门禁 —— 阻止发布

| # | Name | What it checks | Enforced by |
|:-:|:--|:--|:--|
| H1 | Footer present | Every content page has a well-formed `PKB-metadata` footer | `check_frontmatter.py` extension |
| H2 | Footer fields valid | `review_status` ∈ {`pending`, `approved`}; `updated_by` ∈ {`human`, `ai`, `ai+human`}; `review_score` ∈ [0,5] | Same |
| H3 | AI-approval consistency | No page has `review_status: approved` with empty `reviewed_by` | Same |
| H4 | Citations resolve | Every `file:line` anchor in every page points at an existing line at HEAD | `verify` step (Chapter 7) |
| H5 | Links resolve | Every internal `](*.md#*)` link points at an existing anchor | `verify` step |
| H6 | Excerpts provenance | Every vendored excerpt's `content_sha256` matches the current file | `refresh_excerpts.py --check` |
| H7 | No `[TODO]` / `[FIXME]` in published pages | Pages with `status: published` contain no unresolved markers | grep-style check |

每条硬门禁都对应着知识库中“可能为假、且一旦为假就必须阻止发布”的一件具体事。H1–H3 强制 C2（footer 可观测且诚实）。H4–H6 强制“引用与 excerpt 持续可信”。H7 则负责抓最常见的那种“哎呀我们忘了”的失误。

#### 软门禁 —— 发警告，但允许发布

| # | Name | What it warns about | Enforced by |
|:-:|:--|:--|:--|
| S1 | Stale footers | Any page's footer `commit:` is more than N commits behind HEAD | `check-staleness.py` |
| S2 | Low review score | Any published page has `review_score` < 2 | Footer scan |
| S3 | Imbalanced Diátaxis | The published KB has no how-to pages (or no tutorial, etc.) | Histogram over `doc_type` |
| S4 | Answer-faithfulness regression | Nightly RAGAS-style scores fell > 5% from last build | CI job |
| S5 | Coverage gap | A 10-page set (C1) page is missing | `required_pages.txt` check |

软门禁是诚实的警告，不是发布阻塞项。一个带着 S3 警告的知识库仍然是知识库；它只不过多了一条评审者可见的提醒：“你正在偏离自己对页面集合做出的承诺”。

## How —— 怎么做

### 门禁 runner

整个门禁系统就是一个 CI 脚本：逐条跑每项检查并汇总结果。一个最小实现：

```text
function run_gates():
    hard_failures := []
    soft_failures := []

    for gate in HARD_GATES:
        result = gate.check()
        if not result.ok:
            hard_failures.append((gate.name, result.detail))

    for gate in SOFT_GATES:
        result = gate.check()
        if not result.ok:
            soft_failures.append((gate.name, result.detail))

    emit_report(hard_failures, soft_failures)

    if hard_failures:
        exit 1        # block publish
    exit 0            # allow publish; soft warnings in the report
```

每条门禁都暴露相同的接口：一个返回 `(ok, detail)` 的 `check()` 方法和一个稳定的 `name`。这让 CI 脚本本身很短 —— 领域知识被封装在门禁自身里 —— 并且让每条门禁都可以单独测试。

### 门禁报告

一次门禁跑完的输出是一份两段式报告：硬失败（如果有的话）在前，软警告在后。报告是 Markdown 格式的，作为 build artefact 落盘，并从触发本次构建的 PR 处链过来：

```markdown
## Hard gates

❌ H4 Citations resolve — 2 failure(s)
   content/architecture.md:47 cites internal/worker/pool.go:142
     → line 142 does not exist at commit 9f8e7d6c (file has 120 lines)
   content/data-and-api.md:23 cites cmd/sync/main.go:58
     → file does not exist at commit 9f8e7d6c

## Soft warnings

⚠ S1 Stale footers — 7 page(s)
   content/observability.md  (footer commit a1b2c3d is 42 commits behind HEAD)
   ...
⚠ S3 Imbalanced Diátaxis
   Published KB has 0 tutorials; the page-set commitment (C1) requires
   at least 1.
```

硬失败退出码会阻塞 CI 的发布步骤；软警告作为评论追加到 PR 里，这样 CI 不至于红掉，评审者仍然能看到它们。

### 一次具体的失败

假设有一个 PR 往 `internal/worker/pool.go` 里加了一个新函数 —— 诚意新增，无重命名、无破坏性改动。作者顺手编辑了 `architecture.md` 来提到这个新函数，并在那儿加了个 `<!-- TODO: add diagram -->` 标记，打算以后回来补。

跑一遍门禁：

- **H1 Footer 存在。** 通过。`architecture.md` 有 footer。
- **H2 Footer 字段合法。** 通过。字段都格式良好。
- **H3 AI 审批一致性。** 通过。作者设置了 `updated_by: human`、`review_status: pending`、`review_score: 0`、`reviewed_by:`。一切一致。
- **H4 引用可解析。** 通过。新函数的 `file:line` 可以解析。
- **H5 链接可解析。** 通过。
- **H6 excerpt 出处可验证。** 通过。
- **H7 已发布页面中无 `[TODO]`。** **失败。** `<!-- TODO -->` 标记出现在一个 `status: published` 的页面中。

门禁阻止了发布。评审者看到的是一行失败信息 + 一个指向 diff 第 47 行的指针。本次检查的成本：对 `content/**/*.md` 的一次 grep，在 30 ms 内完成。跳过这次检查的成本：线上知识库发布了一个图示写着“TODO”的页面 —— 有用户看到了，对它的信任感轻微折损。这种折损不会自己免费恢复。

## Example —— 范例

一次真实周五下午 PR 上的完整门禁跑动（数据已匿名化）：

```text
$ python build_kb_gates.py
Running 7 hard gates and 5 soft gates across 137 pages...

Hard gates:
  H1 Footer present              137/137 pass
  H2 Footer fields valid         137/137 pass
  H3 AI-approval consistency     136/137 pass — 1 FAIL
       content/runbook.md: review_status=approved but reviewed_by=''
  H4 Citations resolve           423/425 pass — 2 FAIL
       content/architecture.md:47  internal/worker/pool.go:142  (line missing)
       content/data-and-api.md:23  cmd/sync/main.go:58          (file missing)
  H5 Links resolve               612/612 pass
  H6 Excerpts provenance         18/18  pass
  H7 No [TODO] in published      137/137 pass

Soft gates:
  S1 Stale footers               7 page(s) > 30 commits behind HEAD
  S2 Low review score            0 page(s)
  S3 Imbalanced Diátaxis         WARN — published KB has 0 tutorials
  S4 Answer-faithfulness         pass — 0.81 (prev 0.82; delta −0.01)
  S5 Coverage gap                pass — all 10 required pages present

Hard failures: 3   Soft warnings: 2
FAIL — publish blocked. See report above.
```

作者看到的是三件具体要修的事，而不是一份 400 个文件的评审队列。修完这三件大约花 10 分钟：一个页面的 `reviewed_by` 字段被补上，一条引用被更新到新的行号，一条坏掉的 `file:line` 被重新指向替代 `cmd/sync/main.go` 的那个文件。重跑门禁：全部通过。发布继续。

## 15.4   检索质量调优 Cookbook

上面的评测指标和门禁机制回答了"好不好"和"能不能发"两个问题，但当数字开始下滑时，团队最需要的往往是第三个问题的答案：**到底哪里出了问题、怎么修？** 本节提供一套可直接落地的诊断 → 定位 → 修复 → 回归保护流程。

### 15.4.1  诊断流程图

当用户报告"搜出来的结果不相关"时，首先对查询本身做分类，再沿对应分支排查：

```{mermaid}
flowchart TD
    A["用户报告：搜索结果不相关"] --> B{"查询类型判断"}
    B -->|"关键词 / 标识符<br/>e.g. getWorkerPool"| C["检查 BM25 索引"]
    B -->|"自然语言<br/>e.g. '怎么配置线程池'"| D["检查 Embedding 质量"]
    B -->|"结构性查询<br/>e.g. '谁调用了 X'"| E["检查 Graph 边"]
    B -->|"以上都不是"| F["检查 Query Routing 逻辑"]

    C --> C1["tokenizer 是否正确切分<br/>camelCase / snake_case？"]
    C --> C2["field weights 是否合理？<br/>函数名 vs. 注释权重"]
    C1 -->|"不合理"| C3["修复 tokenizer 或<br/>添加自定义 splitter"]
    C2 -->|"不合理"| C4["调整 BM25 field boost"]

    D --> D1["embedding model 是否<br/>针对代码做过微调？"]
    D --> D2["query template 是否<br/>包含必要上下文？"]
    D1 -->|"太通用"| D3["切换到<br/>code-specific model"]
    D2 -->|"缺失"| D4["补充 query prompt<br/>template"]

    E --> E1["CALLS / IMPORTS / INHERITS<br/>边是否完整？"]
    E1 -->|"缺失"| E2["检查 AST parser visitor<br/>补齐边类型"]

    F --> F1["routing classifier 是否<br/>覆盖该查询模式？"]
    F1 -->|"未覆盖"| F2["扩充 routing 规则<br/>或重新训练 classifier"]
```

### 15.4.2  常见 Bad Query → 调优策略

下表汇总了实际运维中高频出现的检索质量问题及对应修复策略：

| # | 症状 (Symptom) | 根因 (Root Cause) | 修复策略 (Fix) |
|:-:|:--|:--|:--|
| 1 | `getWorkerPool` 搜不到，但函数确实存在 | tokenizer 将 camelCase 视为单个 token，BM25 无法匹配子串 | 添加 camelCase / snake_case splitter，使 `getWorkerPool` 同时索引为 `get`、`worker`、`pool` |
| 2 | 中文注释内容搜不到 | 默认 tokenizer 不支持 CJK 分词 | 添加 jieba / mecab 作为 CJK tokenizer；索引时对中文字段单独分词 |
| 3 | 搜索返回大量相似函数，目标函数排名靠后 | embedding model 太通用，无法区分语义相近的代码片段 | 切换到 code-specific embedding model（如 CodeBERT、StarEncoder）；或在 query 中追加文件路径作为上下文 |
| 4 | `谁调用了 parseConfig` 返回 `parseConfig` 本身 | knowledge graph 中 CALLS 边缺失或不完整 | 检查 AST parser visitor 是否遍历了所有调用点；补齐 CALLS edge 类型 |
| 5 | 搜索结果的 top-1 是过期页面 | freshness weight 过低，旧文档因历史积累的引用权重高于新文档 | 在 ranking 公式中调高 `freshness_weight`；或为文档添加 `last_modified` 并纳入排序 |
| 6 | 搜 `HTTP client timeout` 返回 `HTTP server timeout` 的页面 | embedding 空间中 client/server 距离过近 | 为 query 添加 negative example 或使用 re-ranker 做二次排序 |
| 7 | 搜索 `pool.go` 返回同名但不同目录的文件 | 索引未区分文件路径，仅按文件名匹配 | 将完整相对路径纳入索引 field；对路径字段提高 BM25 boost |
| 8 | 自然语言问题 `如何部署到 K8s` 返回空 | query router 将其误分类为 keyword 查询，走了 BM25 通道 | 检查 routing classifier；添加自然语言模式的训练样本；或默认 fallback 到 hybrid 检索 |
| 9 | 搜索带泛型的类型 `Map<String, List<Int>>` 失败 | 特殊字符 `<>` 被 tokenizer 丢弃 | 在 tokenizer pipeline 中保留泛型括号；或额外索引一份"类型签名归一化"后的文本 |
| 10 | 同一查询在不同时间返回不同结果 | embedding index 在增量同步中出现 stale vector（旧版本未删除） | 在 incremental sync 时对已删除 / 已修改的 chunk 执行 hard delete；检查 vector store 的 upsert 逻辑 |
| 11 | 搜索返回大量测试文件，淹没了实际源码 | 测试代码与生产代码权重相同 | 为 `*_test.go` / `*_spec.ts` 等测试文件降低 boost；或允许用户按 `is_test` facet 过滤 |
| 12 | 复合查询 `parseConfig 的超时参数` 仅匹配到 `parseConfig` | 混合检索的融合策略只取了 BM25 结果，忽略了语义部分 | 检查 hybrid fusion weights（如 RRF 的 k 参数）；确保 BM25 和 vector 通道均参与 |

### 15.4.3  检索质量回归测试

指标面板只能告诉你"当前好不好"，不能告诉你"什么时候变差了"。要捕捉回归，需要一套 **golden query set** 和配套的 CI 门禁：

1. **构建 golden query set**：从真实用户查询日志中挑选 20–50 条代表性查询，人工标注每条查询的 expected top-3 结果（文档 ID 或代码实体 ID）。覆盖关键词、自然语言、结构性三种类型。

2. **作为 CI 软门禁 (S6) 运行**：在每次 build 或定期 nightly job 中，用 golden query set 跑一遍检索，计算 nDCG@10 和 Recall@10。

3. **沿时间线追踪**：将每次跑分结果写入持久化存储（SQLite / CSV / Prometheus），绘制趋势图。

4. **设定告警阈值**：当 nDCG@10 相对上一次 build **下降 > 5%** 时触发告警（对应软门禁 S6）。告警内容应包含具体退化的查询及其新旧排名对比，以便快速定位。

```text
# golden_query_set.jsonl 示例
{"query": "getWorkerPool",           "type": "keyword",    "expected_top3": ["internal/worker/pool.go#L42", "internal/worker/pool.go#L110", "docs/architecture.md#worker-pool"]}
{"query": "怎么配置线程池超时",       "type": "nl",         "expected_top3": ["docs/how-to/configure-pool.md", "internal/worker/config.go#L15", "docs/reference/pool-options.md"]}
{"query": "谁调用了 parseConfig",     "type": "structural", "expected_top3": ["cmd/server/main.go#L23", "cmd/worker/main.go#L18", "internal/bootstrap/init.go#L55"]}
```

### 15.4.4  A/B 测试脚手架

当你想比较两套检索配置（例如切换 embedding model、调整 BM25 权重）时，最简单的做法是：**在同一份 golden query set 上跑两套配置，比较 Recall@10**。

以下是一个 10 行左右的 Python 骨架脚本：

```python
# ab_retrieval_test.py — 检索配置 A/B 对比脚手架
import json
from retrieval_client import RetrievalClient  # 你的检索客户端

def recall_at_k(results: list[str], expected: list[str], k: int = 10) -> float:
    top_k = set(results[:k])
    return len(top_k & set(expected)) / len(expected) if expected else 0.0

def run_ab_test(golden_path: str, config_a: dict, config_b: dict) -> None:
    queries = [json.loads(line) for line in open(golden_path)]
    client_a, client_b = RetrievalClient(config_a), RetrievalClient(config_b)
    scores = {"A": [], "B": []}
    for q in queries:
        scores["A"].append(recall_at_k(client_a.search(q["query"]), q["expected_top3"]))
        scores["B"].append(recall_at_k(client_b.search(q["query"]), q["expected_top3"]))
    avg_a = sum(scores["A"]) / len(scores["A"])
    avg_b = sum(scores["B"]) / len(scores["B"])
    print(f"Config A  avg Recall@10: {avg_a:.3f}")
    print(f"Config B  avg Recall@10: {avg_b:.3f}")
    print(f"Delta (B − A): {avg_b - avg_a:+.3f}")
```

使用方式：将 `config_a` 设为当前线上配置，`config_b` 设为候选配置。若 `Delta > 0` 且差值具有统计显著性（在 golden set 足够大时可用 paired t-test），则可以推进切换。对于小规模 golden set（< 50 条），建议同时人工审查排名变化最大的 5 条查询，避免被平均值掩盖的局部退化。

## Conclusion —— 小结

评测（IR 指标、faithfulness 分数）告诉你：知识库一旦发布出去是否有用。门禁（硬 / 软检查）告诉你：这一次具体构建是否根本就“可以发”。两者都是机械的，都很便宜。但今天大多数知识库两者都拒绝上 —— 因为要加它们，就得对第 8 章那四个角落做出承诺：footer 不在，H1 就跑不起来；运维模型不解析引用，H4 就跑不起来。

下一章把整条回路闭合：当硬门禁失败、或软门禁告警时， * 修复到底长什么样？ * 答案就是三级更新策略（C3）—— 它的真正机制在本章已经先铺了一下，但正经讲述属于第 23 章。

## 参考文献

```{bibliography}
:filter: keywords % "operations" or keywords % "ir-rag" or keywords % "evaluation" or keywords % "self-citation"
```
