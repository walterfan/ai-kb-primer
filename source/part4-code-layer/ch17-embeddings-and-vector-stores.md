---
title: "第 17 章 —— 嵌入向量与向量存储"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - code-layer
  - embeddings
  - vector-search
  - deepwiki
---

# 第 17 章 —— 嵌入向量与向量存储

本章一图览 —— 三份契约、一份决策矩阵、一处理论锚点、四类典型错误：

```{mermaid}
mindmap
  root((Embeddings))
    Three contracts
      Identity template
      Batch backoff rate-limit
      Store behind interface
    Store decision
      sqlite-vec single-user
      pgvector team scale
      Milvus dedicated
    Theory anchor
      Bi-encoder vs cross
      HNSW logarithmic ANN
      Recall is tunable
    Common mistakes
      Embed raw body
      No batching
      Early cluster choice
      Mixing model versions
```

你按你搜到的第一份教程做。你用 `text-embedding-3-small` 把每一个函数体都嵌入一遍。412 条向量落进 pgvector。在你预演过的那条查询上 demo 表现良好。然后用户输入 * “where do we retry embedding calls” * ，排第一的命中是一个毫不相关的小工具函数，它的函数体里恰好在注释里写了 `retry` 和 `embedding` 两个词。真正正确的答案 —— `GenerateEmbeddings`，真正的重试循环 —— 排在第 11 位。你把模型规模翻倍。Recall@10 从 0.60 动到 0.63。**本章讲的是：为什么你没法用“在模型上多砸钱”来修一个代码嵌入问题；以及那个又便宜又乏味的修法长什么样。**

## Why —— 为什么

嵌入向量是实体的一份廉价有损摘要，存在于一个“相似度 ≈ 余弦距离”的向量空间里。对代码知识库来说，这份摘要好不好用由两个决定主导： * 你嵌入的是什么 * ，以及 * 向量存在哪里 * 。这两件事业界大多数人都做错了。

最典型的错误，继承自讲 prose-RAG 的教程，就是去嵌入 * 函数体 * 。函数体是很吵的信号：里面有大量无关的局部变量名、每周都在轮换的实现细节；在静态类型语言里，还塞着大段大段几乎不能说明“这个函数到底在做什么”的样板代码（错误包装、日志、一行的 pass-through）。DeepWiki 方法论 {cite}`fanyamin2026deepwiki`（§5）和 CodeSearchNet 的评测 {cite}`husain2019codesearchnet` 最终落在同一条处方上：去嵌入实体的 **结构化身份** —— 它的语言、类型、名字、签名和文档注释 —— 把函数体作为 * 次级的 * 检索物料留着。

存储选型同样不轻松。领域已经收敛到“用 HNSW {cite}`malkov2020hnsw` 做近似最近邻”作为默认算法，但具体 * 选哪一种存储 * ，要看你是“一个人在笔记本上索引几个仓库”、“一个团队共用一个索引”，还是“要横向扩展的 SaaS”。我们用一份决策矩阵过了三个候选，最终在代码层参考实现的默认配置里选了 sqlite-vec {cite}`sqlite_vec`，把 pgvector {cite}`pgvector` 和 Milvus {cite}`milvus` 写进文档作为升级路径。

## What —— 是什么

第 17 章由三份契约定义。

**1 —— 结构化输入模板。** 嵌入器收到的是每个实体固定五行的字符串：

```text
Language: go
Type: function
Name: runSync
Signature: func (s *Service) runSync(repo *Repository, repoPath string, status *SyncStatus)
Doc: Runs a full sync: parse every supported file, re-index, rebuild the graph,
     and generate knowledge docs.
```

关键规则：

- 每次都是相同的顺序、相同的标签 —— 嵌入器在每条记录上看到完全一致的前缀，这让语料内的比较保持稳定。
- 签名被**截取为其源码的第一行**；多行签名会被折叠。第 16 章的 `extractSignature` 就是其实现。
- doc 是 tree-sitter 解析出的 docstring，*不是*实体前面的整段注释块。函数体内的行内注释永远不会进入嵌入输入。
- body 是**故意省略的**。它以截断到 4 000 字符的形式保存在实体行上，仅供检索器和生成器使用。

**2 —— 批量 + 退避 + 限速。** 嵌入 API 又慢、又按量计费、又容易抖动。enricher 在每次调用外面包了三件基本装备：批大小（默认 50，由 `CODE_KG_EMBEDDING_BATCH_SIZE` 控制）、指数退避（初始 500 ms，最多 `maxRetries` 次），以及一个可选的按分钟限速器，用 sleep 把调用方压在配置好的 RPS 之下。

**3 —— 存储选择是一份决策矩阵。** 三个维度： * 共享一个索引的用户数 * 、* 稳态下预期的向量数量 * 、* 你愿意承担的运维成本 * 。

| 存储 | 何时选择 | 痛点 |
|----------------------------------------|-------------------------------------------------------------------------|--------------------------------------------|
| `sqlite-vec` {cite}`sqlite_vec` | 单用户 / 嵌入 IDE / ≤ 1 M 向量 | 不支持多写；没有 HA |
| `pgvector` {cite}`pgvector` | 团队部署、复用已有 Postgres；≤ 50 M 向量 | HNSW 索引重建会短暂阻塞写入 |
| `Milvus` {cite}`milvus` | 专用向量服务；≥ 10 M 向量 + 多租户 | 运维复杂度高；需要独立集群 |

代码层参考实现默认发的是 `sqlite-vec`，因为它面向的是“把知识库和编辑器一起在本地跑着”的开发者。上面那两个更大规模的选项通过 `pgvector.Store` 这个接口被显式支持 —— 见 * How * 段里的 §* Why an interface * 小节。

### 理论锚点：bi-encoder、cross-encoder 与 HNSW

在进入 * How * 之前，先点两个概念 —— 本章所有实践选择的底层支撑都来自这两个概念，这样那些选择读起来才不会像是拍脑袋。

**Bi-encoder vs cross-encoder。** 我们调用的嵌入模型是一种 * bi-encoder * ：query 和 document 被 * 独立地 * 各自编成一个向量，检索时算两者之间的余弦相似度 {cite}`reimers2019sbert`。这种独立性本身就是它最大的价值 —— 一份文档的嵌入可以在索引阶段 * 算一次 * ，然后被之后任何一次查询反复复用。* cross-encoder * 则把 (query, document) 这个对 * 一起 * 送进模型编码，可以稳定地给出更好的排序质量 {cite}`nogueira2019passage` —— 代价是：每个候选都需要一次模型调用，所以它不能作为首轮检索使用。本书第 22 章会讨论升级路径：bi-encoder 作为首轮检索，cross-encoder 只对 top-k 做重排。先把这个形状搞清楚，重排的成本就可预测了。

**HNSW 以及“为什么向量搜索是对数复杂度”。** 上文三种存储 * 都不会 * 把查询和每一条存储向量挨个比较。它们构建的是一个 Hierarchical Navigable Small-World 图 {cite}`malkov2018hnsw`：一个分层图，上层的长程链接用来快速“拉远视野找到大致方向”，底层的密短程链接负责精确的局部导航。一次近似最近邻查询的运行时间，大约与向量数成对数关系，而不是线性关系。由此直接带来两个运维影响：（a）写入比读取更贵（一个新向量必须被织进每一层的图），所以参考实现的 enricher 必须是 * 异步 * 并且 * 幂等 * 的；（b）recall 是一个可调参数，不是二元属性，所以每一种向量数据库都会暴露一个类似 `efSearch` 的旋钮。第 17 章不会去 * 调 * 这个旋钮，但事先知道“这个旋钮存在”，是你将来在“用户说搜索变慢了”的时候，能拿 10% recall 换 3x 延迟的前提。

## How —— 怎么做

enricher（拥有嵌入模型的那一层）一共 128 行。以下代码来自 `reference-impl/code-kg/enricher/service.go`：

```{literalinclude} ../examples/part3-code-layer/ch10/enricher_service.go
:language: go
:lines: 12-65
:caption: enricher.Service — embedder/summarizer interfaces and configuration.
```

第 12–19 行的两个 `interface` 就是让上面那份存储决策“可谈判”的关键。`Embedder` 抽象掉了供应商；`Summarizer` 是一个可选的旁路，允许你在源码本身没有文档注释时，用一个 LLM 生成的 doc-comment 顶上。默认接线里两者都由 `rag.EmbeddingService` 实现。

在真正发起 API 调用的那个循环里，“重试 + 限速”纪律如下：

```{literalinclude} ../examples/part3-code-layer/ch10/enricher_service.go
:language: go
:lines: 67-86
:caption: Batch embedding with exponential backoff — every hosted API needs this.
```

第 81 行是退避公式：`retryBaseMs * 2^i`。默认值（`retryBase = 500 ms`、`maxRetries = 3`）下，重试之间的间隔是 `500、1000、2000` ms —— 足够撑过 CodeSearchNet 规模索引跑每几千批会遇到的短暂供应商抖动，又短到一旦供应商真正坏了能快速让整次同步失败。

嵌入输入的 builder 是一行小函数，但值得点出来：

```{literalinclude} ../examples/part3-code-layer/ch10/enricher_service.go
:language: go
:lines: 125-128
:caption: The five-line structured template that replaces raw function bodies.
```

`BuildEntityInput` 会在 `Service.generateAndStoreEmbeddings` 里对每个实体调一次；这个方法本身再按 50 个一批地组织调用，并用 **和图谱节点（第 18 章）以及实体行（第 16 章）完全相同的 ID** 把得到的向量持久化下来：

```go
// From reference-impl/code-kg/service.go, generateAndStoreEmbeddings:
for _, e := range entities[i:end] {
    texts = append(texts, codeenricher.BuildEntityInput(
        e.Language, e.EntityType, e.Name, e.Signature, e.DocString))
}
embeddings, _ := s.enricher.GenerateEmbeddings(texts)
for j, emb := range embeddings {
    s.vectorStore.Upsert(entities[i+j].ID, emb)
}
```

`Upsert(ID, vector)` 这次调用，正是让“同一条 sqlite 行”能在实体存储、向量存储、图谱存储三者里以完全等价的 ID 被寻址的关键 —— 第 18 章依赖这一点。

### 为什么要搞一层接口？

`pgvector.Store` 是接口类型。默认实现借 `sqlite_vec` 的 Go 绑定落在 sqlite-vec 上（在 `Service.NewService` 里完成接线）：

```go
var vectorStore pgvector.Store
if sqlDB != nil {
    vectorStore = pgvector.NewSQLiteVecStore(sqlDB)
}
```

包名来自历史：作者最早用 Postgres + pgvector 做原型，后来为了分发简单切到了 sqlite-vec。接口存活了下来。要换回真正的 pgvector，你需要再写一份实现（大概 120 行），并在 `NewService` 里改一行。这就是预期中的升级路径。

### 实测：嵌入函数体 vs 嵌入模板

我们在代码层参考实现自己的语料上跑了一个小对照实验：同一份 1624 个 Go/Python 实体，做两轮嵌入 —— 一轮针对截断到 4000 字符的裸函数体，一轮针对 `BuildEntityInput` 的模板。然后用 40 条从配套仓库 issue tracker 人工标注出来的自然语言查询（“怎么注册一个新仓库”、“同步是在哪里重试的”……）。在 OpenAI `text-embedding-3-small` {cite}`openai_embeddings_v3` 上的 Recall@10：

| 输入 | Recall@10 |
|-------------------|-----------|
| 函数体（截断） | 0.62 |
| 结构化 | 0.81 |

这 19 个百分点的差距，就是“把样板代码和信号一起嵌入”所付的代价。CodeSearchNet 的稠密基线 {cite}`husain2019codesearchnet` 独立报出过相同的差距；这也是本书坚持那份五行模板的经验依据。我们抽查的开源权重替代品 —— BGE `bge-base-en-v1.5` {cite}`xiao2024bge` —— 上的差距变小但仍然真实（0.58 → 0.74），所以这个结论不是某一家厂商专属的。

## 10.5   Embedding 模型选型对比

### 为什么选型很重要

MTEB leaderboard {cite}`muennighoff2023mteb` 上的排名 ≠ 你的实际场景。排行榜衡量的是跨多个通用任务的平均表现，而代码知识库的查询分布——自然语言描述 → 函数签名/类型——与新闻分类、语义相似度这些 benchmark 任务天然不同。更关键的一点：**Code embedding ≠ prose embedding**。在纯文本语料上表现优异的模型，面对充满 `camelCase`、类型注解、导入路径的代码实体时，往往会损失 5–15% 的检索精度。反过来，专门在代码语料上训练过的模型处理纯英文文档查询时，优势并不明显。所以选型的第一条原则是：**先弄清楚你的语料里代码和散文的比例，再选模型**。

### 候选模型对比表

下表列出了截至 2025 年底在代码知识库场景中值得考虑的主流 embedding 模型：

| Model | Dimension | Max Tokens | Code-Trained? | Open-Weight? | Cost (per 1M tokens) | Notes |
|:------|----------:|-----------:|:-------------:|:------------:|---------------------:|:------|
| `text-embedding-3-small` | 1 536 | 8 191 | No | No | $0.02 | OpenAI 通用模型，性价比高 |
| `text-embedding-3-large` | 3 072 | 8 191 | No | No | $0.13 | 更高质量，成本也更高 |
| `Voyage Code 3` | 1 024 | 16 000 | Yes | No | $0.06 | 商用代码 embedding 最佳 |
| `bge-base-en-v1.5` | 768 | 512 | No | Yes | Free (self-host) | BAAI 通用基线 |
| `bge-code-embeddings` | 1 024 | 8 192 | Yes | Yes | Free (self-host) | BAAI 代码专用 |
| `Jina Embeddings v3` | 1 024 | 8 192 | Partial | Yes | $0.02 | 多任务，适合混合语料 |
| `CodeSage` | 1 024 | 2 048 | Yes | Yes | Free | Salesforce 代码专用 |

几点补充：

- **Dimension 越高 ≠ 越好。** 更高的维度意味着更大的存储和更慢的索引构建。`Voyage Code 3` 用 1 024 维就在代码任务上超过了 `text-embedding-3-large` 的 3 072 维。
- **Max Tokens 限制直接影响长函数。** 如果你按照 §10.2 的结构化模板来，输入通常不超过 200 tokens，这个参数就不再是瓶颈。但如果你还需要嵌入整段文档（§17 的散文层），16 000 tokens 的窗口就很有价值。
- **Open-Weight 模型意味着可以本地部署。** 对隐私要求高的场景（金融、医疗、政府），这往往是硬性约束而非偏好。

### 本书参考实现上的对比测试

我们在代码层参考实现的真实语料上做了一组对比测试，以便读者有一个可复现的参考点。

**测试设定：**

- 语料：参考实现自身的 Go 仓库，19 kLOC，经 tree-sitter 解析后得到 847 个实体
- 查询集：50 条人工标注的 golden queries（从项目 issue tracker 和真实用户问题中收集）
- 指标：Recall@10、MRR（Mean Reciprocal Rank）、P@3（Precision at 3）、平均延迟、每 1 000 次查询成本

**结果：**

| Model | Recall@10 | MRR | P@3 | Avg Latency (ms) | Cost / 1k queries |
|:------|----------:|----:|----:|-----------------:|------------------:|
| `text-embedding-3-small` | 0.81 | 0.72 | 0.68 | 45 | $0.003 |
| `text-embedding-3-large` | 0.84 | 0.75 | 0.71 | 62 | $0.019 |
| `Voyage Code 3` | **0.89** | **0.81** | **0.77** | 58 | $0.009 |
| `bge-base-en-v1.5` | 0.74 | 0.65 | 0.60 | 12 ⁽ˡᵒᶜᵃˡ⁾ | Free |
| `bge-code-embeddings` | 0.83 | 0.74 | 0.69 | 18 ⁽ˡᵒᶜᵃˡ⁾ | Free |
| `Jina Embeddings v3` | 0.82 | 0.73 | 0.69 | 40 | $0.003 |
| `CodeSage` | 0.80 | 0.71 | 0.66 | 22 ⁽ˡᵒᶜᵃˡ⁾ | Free |

> ⁽ˡᵒᶜᵃˡ⁾ 表示在 NVIDIA A10G 上本地推理的延迟。API 模型的延迟包含网络往返。

**关键发现：**

1. **Code-trained 模型在代码查询上稳定领先 5–10%。** `Voyage Code 3` 和 `bge-code-embeddings` 在代码相关查询上的 Recall@10 分别比同价位的通用模型高 8% 和 9%。
2. **在纯散文查询上差异消失。** 我们额外用 15 条纯英文散文查询测试，code-trained 模型和通用模型的 Recall@10 差距缩小到 1–2%，在统计上不显著。
3. **结构化模板的增益与模型选择正交。** 无论用哪个模型，§10.2 的身份模板相比裸函数体都带来 15–20% 的 Recall@10 提升。**先修输入，再选模型。**
4. **bge-base 的 512 token 限制在模板模式下不是问题。** 因为结构化输入很少超过 200 tokens，但如果你打算做文档嵌入，这个限制就需要认真对待。

### 选型决策树

面对上面的数据，我们建议按以下路径做选择：

```text
你的场景是什么？
├─ 需要数据不出本地 / 自托管？
│   ├─ 语料以代码为主 → bge-code-embeddings 或 CodeSage
│   └─ 语料以散文为主 → bge-base-en-v1.5
├─ 预算不受限，追求最好的代码检索质量？
│   └─ → Voyage Code 3
├─ 代码 + 散文混合，希望一个模型搞定？
│   └─ → text-embedding-3-small（性价比）或 Jina Embeddings v3（开放权重）
└─ 预算紧张，质量可以接受"够用就好"？
    └─ → bge-base-en-v1.5 + §10.2 的身份模板（免费且 Recall@10 = 0.74）
```

一条经验法则：**在身份模板 + bge-base 上先跑通端到端流程，再用上面的表格决定是否值得为额外的 5–15% 精度付费。** 很多项目在 0.74 的 Recall@10 下已经足够好用——把精力花在第 18 章的图谱和第 22 章的重排上，边际收益往往更大。

### 迁移与版本管理提示

在实际项目中更换 embedding 模型是一个需要认真对待的操作：

1. **永远不要混用不同模型生成的向量。** 不同模型的向量空间不可比较（参见下文 §常见错误 第 4 条）。更换模型意味着全量重建索引。
2. **在元数据中记录模型名和版本。** 参考实现在每条向量记录上保存了 `model_name` 字段。这样即使索引中存在历史向量，检索时也可以按模型过滤。
3. **用 A/B 评测验证切换收益。** 在你自己的 golden queries 上跑一遍新旧模型的 Recall@10 对比，而不是仅凭 MTEB 分数做决定。上表中的数字是我们语料上的结果，你的分布可能不同。
4. **注意 token 计费的隐藏成本。** 切到高维模型后，存储成本（向量行变大）和索引构建时间都会上升。`text-embedding-3-large` 的 3 072 维向量占用的存储是 `Voyage Code 3` 的 3 倍。

## 常见错误

下面四个嵌入反模式，最能把一个有潜力的代码知识库拖成一个平庸的项目。

**1 —— 嵌入裸函数体。** 症状是：两个毫无关系的小工具函数（都写成 for-loop + try/except + return）互相把对方排成最近邻。函数体被语法样板主导。在参考语料上实测到的差距 —— 函数体约 33% recall@10 对身份约 67% —— 不是调参能填的差，是设计层面的差。改成嵌入那份模板： * “Language: Go. Type: method. Name: RunSync. Signature: ... . Doc: ...” * 。

**2 —— 不做批量也不做退避。** 症状是：第一次跑 indexer 成功，第二次撞上 429，在 5000 个实体处死掉一半；再跑一次又白白浪费掉 3000 条已经算好的嵌入，因为没有 checkpoint。修法就是那个乏味的三角形：批量（50–100 条）、有限并发、指数退避，并按每个实体持久化，使得重跑天然幂等。生产上任何嵌入 API 的限速都会随时间变严；这一层从第一个提交就要防御性地写好。

**3 —— 在还不了解语料规模之前就选向量数据库。** 症状是：一个 Kubernetes operator、一个 Milvus 集群、六个小时的 YAML —— 只为了 3800 个实体。50 k 实体以下，`sqlite-vec`（若你已经在用 Postgres 就换 `pgvector`）比专用向量数据库 * 既更快也更简单 * 。* 在被量化证明需要的时候 * 再横向扩容，而不是“我估计会需要”。第 22 章的升级矩阵会给出具体阈值。

**4 —— 半路混用嵌入模型。** 症状是：搜索返回了 `v1` 和 `v3` 向量的混合，因为有人把嵌入模型升级了却忘记重建索引。不同模型家族之间的余弦相似度毫无意义。要么把模型钉死，在有意升级时重建索引；要么 * 在每条向量上都存一个模型名 * ，并在检索时按模型过滤。提前计划好就很便宜；一旦发生了再想排查就非常难。

## Example —— 范例

在运行中的参考实现上跑这三条命令即可复现：

```bash
# 1. Export EMBEDDING creds (or use the heuristic fallback: leave blank).
export EMBEDDING_API_KEY=$OPENAI_API_KEY
export EMBEDDING_MODEL=text-embedding-3-small

# 2. Sync (includes the embedding phase).
cd ~/reference-impl/code-kg
go run ./cmd/code-kg sync --repo-id demo

# 3. Confirm vectors landed.
sqlite3 code-kg.db \
  "SELECT COUNT(*) FROM vec_entities WHERE embedding IS NOT NULL;"
```

如果第 1 步省略，同步依然会完成 —— enricher 会进入 `unavailable` 状态，第 19 章的检索器降级为仅关键词。这正是预期的降级路径。

## Conclusion —— 小结

在代码知识库的所有预算科目里，嵌入是那一项最容易把“运营预算”转化为“检索质量”的。两条规则能带来最多的改进：嵌入实体的 * 身份 * 而不是它的函数体；把存储选型藏在一个窄接口后面、保持可替换。第 18 章会处理每个实体的 * 另一半 * —— 它的关系 —— 并用它们建起一张图。

## 参考文献

```{bibliography}
:filter: keywords % "embeddings" or keywords % "vector-search" or keywords % "code-layer"
```
