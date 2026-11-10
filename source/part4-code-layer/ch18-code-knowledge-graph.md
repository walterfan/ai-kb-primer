---
title: "第 18 章 —— 代码知识图谱"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - code-layer
  - graph
  - deepwiki
---

# 第 18 章 —— 代码知识图谱

本章一图览 —— 三条规则、一处理论锚点，以及四类把图从资产变成负担的典型错误：

```{mermaid}
mindmap
  root((Code Graph))
    Three rules
      Closed edge taxonomy
      Location-aware node ID
      Noise filter at write
    Edges
      CONTAINS
      IMPORTS
      CALLS
      IMPLEMENTS
    Theory anchor
      Graph locality
      Data-flow prior
      Beyond embeddings
    Common mistakes
      Unbounded edge types
      Graph on hot-write path
      Index every identifier
      Bodies inside nodes
```

晚上 11 点，一位 support 工程师来敲你们团队的群：客户报告 billing 作业在重试时会悄悄重复扣款。你需要在过去一年的 Go 代码里，找到 `chargeCard` 的每一个调用方 —— * 现在就要 * 。向量存储返回了五个名字看起来很像的函数；其中三个根本没有调用 `chargeCard`，只是和它共用一些词汇。按名字 grep 也能用 —— 只不过你还用 Python 写了客户端 SDK，那边的写法是 `charge_card`，拼写完全不同。**本章讲的就是：代码知识里那一半任何嵌入都捕捉不到的东西 —— 谁连接到谁 —— 以及一张小小的、不含内容的图如何把这一类问题的召回率从 0.3 变成 1。**

## Why —— 为什么

第 17 章的向量回答的是 * “什么东西和这个相似？” * 。一个代码知识库还需要同样好地回答第二个问题： * “什么东西和它相连，又是如何相连？” * 。向量空间不编码边，它编码相似度。两个天天互相调用的函数，如果签名差别大，可能在嵌入空间里位于对角线的两端；而两个毫不相关、docstring 类似的函数，可能即使分处不同子系统也会紧紧聚在一起。

图谱层正是用来填这个空缺。它是一个小巧、有类型、不含内容的结构 —— 节点代表实体，边代表关系，不冗余存正文 —— 它唯一的工作就是回答 * 谁连接到谁 * 这一类查询：谁调用了这个函数，这个包导入了什么，哪些类型实现了这个接口。图数据库的入门读物首推 Robinson 等人的书 {cite}`robinson2015graph`；我们的实现沿用 DeepWiki 方法论 {cite}`fanyamin2026deepwiki`（§6），遵循 **边的集合必须是有界的** 和 **每个仓库每次全量重建** 两条写入准则。

## What —— 是什么

三条规则塑造了图谱层。

**1 —— 关系类型是封闭集合。** 新增一种边需要走“设计评审”级别的流程。完整清单就在 `graph.go` 里：`CONTAINS`、`IMPORTS`、`CALLS`、`IMPLEMENTS`、`EMBEDS`、`DEPENDS_ON`、`RETURNS`、`ACCEPTS`。没有别的。新词汇很便宜，新图语义不便宜 —— GraphCodeBERT {cite}`guo2021graphcodebert` 已经表明：一套代码感知的表示，它的表达力其实主要是被少数几种边扛起来的。所以我们把自己的边限制在八种，并且让每一种都恰好只意味着一件事。

**2 —— 实体 ID 带位置信息。** 节点身份是 `(repoID, filePath, entityType, name, startLine)` 的 SHA-256 截取前 16 位十六进制。这正是增量同步（第 21 章）能工作的原因：当一个文件变了，里面的每个实体都是 * 先删后插 * ；只有当实体在文件中的位置发生变化时 ID 才会变化。第 21 章会解释：为什么这种设计让我们能把“向量增量写”和“图全量重建”搭配起来，同时不留下悬空边。

**3 —— 噪声过滤在写入时就执行。** 没有它们，图里会塞满叫做 `string`、`int`、`len`、`make` 的节点，以及代码库里每一个闭包。过滤器就放在 builder 旁边，源码见下。

### 理论锚点：把图上的“邻近性”作为一等公民信号

这两个概念，把“代码图”从一个工程小玩意儿变成检索体系里真正承重的一块。

**图上的邻近性（graph locality）。** 在自然语言里， * “和 X 相似的是什么” * 能干净地映射到稠密向量空间的邻域：经验上，嵌入离 X 最近的那些文档，确实大多是同一主题。换到代码， * 这个推断就不成立了 * 。X 最常调用的那个函数，很少是 * 名字 * 在向量空间里离 X 最近的那个 —— 而是从 X 出发沿一条 CALLS 边就能到的那个。代码里的“邻近”是一种 * 图 * 的属性，不是一种 * 度量 * 的属性。任何只有度量投影的检索系统，都会悄悄丢掉 30%–40% 资深工程师真正会问的问题。

**数据流作为一种学习到的先验。** GraphCodeBERT {cite}`guo2021graphcodebert` 这篇论文给上面这个直觉上了数字：如果在预训练一份代码表示时明确加入数据流图（“变量 x 在这里被写、在那里被读、被传入这个函数”），相比原生 token 基线，下游任务 —— 代码搜索、克隆检测、类型推断 —— 会 * 可度量地 * 变好。本书第 18 章的图是同一条信号的一个更粗粒度、设计时版本：我们不 * 学 * 这张图，我们 * 抽 * 出来。由此带来的实际后果见第 19 章：任何忽略这一层显式图的检索器，忽略的就是一个已被研究社区独立证明“值得花力气去捕捉”的信号。

最近的 LLM-for-code 研究进一步强化了同样的分工。Zhu 等人关于 LLM 用于 KG 构建与推理的综述指出，大模型作为**结构化知识上的推理器**通常比作为从原始输入中 few-shot 提取知识的提取器更可靠 {cite}`zhu2024llmsknowledgegraphconstruction`。这就是为什么本章坚持用确定性的代码分析来抽取一个*小型的、显式的、有类型的*图，而不是在查询时让模型从一堆文件内容里推断关系。CGM 把这个主张推得更远：它把仓库图结构注入到模型的 attention 机制里，并在下游软件工程任务上展示了仓库级的增益 {cite}`tao2025codegraphmodelcgm`。我们的图层是同一思路的检索时对应物。我们把图放在模型外面以获得可移植性和可运维性，但我们把它当作一等公民对待，因为模型家族正在朝同一个方向演进，而非背离。 RepoGraph 在 ICLR 2025 上展示了同样的思路在检索端的工程落地：它不满足于语义相似度，而是用 BFS 在类型化的代码图（calls、imports、inherits、defines）上扩展种子命中，使得送入 LLM 的上下文不仅语义相关、而且结构完整 {cite}`ouyang2025repograph`。此外，LLM 与 KG 在软件仓库问答中的协同使用也已被独立验证：将仓库结构知识建模为专用本体（Repository、Module、File、Class、Function、Dependency、Issue、Commit、Developer），并在查询时路由结构性问题到 KG、语义性问题到稠密检索，覆盖面远超任一单系统 {cite}`xu2024llmkgsynergy`。

## How —— 怎么做

图谱在解析完的文件上单次 pass 构建，整个过程用到三张进程内的 map：按 ID 索引的实体、按名字索引的函数（用来解析调用），以及一个用于去重的“已见关系”集合。以下代码来自 `reference-impl/code-kg/graph.go`：

```{literalinclude} ../examples/part3-code-layer/ch11/graph.go
:language: go
:lines: 12-67
:caption: Noise filters and the closed relation-type enum.
```

关于这份代码的两个关键点：

- 第 12–27 行枚举了每一个 Go 内建类型和内建函数 —— `len`、`make`、`panic`、`append`、`string`、`int`、`error` 等。`isNoiseEntity` 和 `isNoiseFunction` 把它们过滤掉。结果是：一个对 `len()` 有 5 000 次调用的 Go 仓库不会创建任何 `len` 节点。没有这层过滤器，图的 `CALLS` 边数量会随 LOC 超线性增长。
- 第 56–67 行是封闭的 RelationType 枚举。每一条边都是八个常量之一（而不是自由字符串），这一事实由 Go 类型系统和评审人共同保障 —— 每个试图添加第九种边的 PR 都会触发一次全图级讨论。

下面是真正的构建 pass，它用一次遍历同时产出实体列表和关系列表：

```{literalinclude} ../examples/part3-code-layer/ch11/graph.go
:language: go
:lines: 99-194
:caption: buildParseResult — files CONTAIN entities, files IMPORT packages, functions CALL functions.
```

这个函数里有三处结构性选择值得点名：

- **`CONTAINS` 边：从文件到实体。** 每个函数、类、结构体和接口都作为其所在文件节点的子节点被发射。仅凭这一种边类型，第 19 章的检索器就能回答 *"给我看这个文件里定义的所有东西"*，而不需要额外的关系查询。

- **`IMPORTS` 边：文件 → 包。** 目标 ID 不是知识库里的一个真实实体；它是一个合成字符串 `"package:" + importValue`。图允许没有对应代码实体的包节点存在，这是正确的选择：外部依赖是真实的，尽管我们从未解析它们。

- **`CALLS` 边通过名字解析。** 第 170 行调用 `detectFunctionCalls(entity.Body, entity.Name, functionByName)`，它用正则把每个已知函数名和调用方的 body 做匹配。这就是那个**基于正则的调用检测器** —— DeepWiki 博客的"穷人版"（§6）。它是有损的：会漏掉动态分派，也会在跨包同名方法上过度报告。升级到真正的 language-server-protocol 解析器的路径在设计文档里是开放的；当前的务实选择是：正则对于开发者辅助查询"够好了"（在参考实现自身语料上精确率 ≥ 90%），且运维成本只是每次同步中每个函数名编译一次正则。

### 存储接口

builder 发射出 `[]Entity` 和 `[]CodeRelation`；再由一个存储适配器把它们翻译成后端的写调用。今天唯一的后端是 Memgraph {cite}`memgraph`，通过 Bolt 协议通信。以下代码来自 `memgraph_store.go`：

```{literalinclude} ../examples/part3-code-layer/ch11/memgraph_store.go
:language: go
:lines: 42-93
:caption: MemgraphStore — single UpsertRepositoryGraph entrypoint.
```

`UpsertRepositoryGraph` 这个调用是 * 故意 * 单体化的。它一次性接收整个仓库的节点和边，底层会先发 `MATCH (n {repo_id: $repo}) DETACH DELETE n`，再用 `UNWIND` + `MERGE` 写入新的载荷。这就是博客 §6 所倡导的 **每个仓库做一次全量重建** 模式 {cite}`fanyamin2026deepwiki`。它的代价是每次同步要把整个仓库的子图重写一遍；它的收益是：我们永远不必推理“部分图变更”，而后者正是图谱流水线里大多数运维 bug 的来源。

第 21 章会讨论例外情形：如何对小改动 * 增量 * 重建图，同时在概念上仍然保持“每仓库全量重建”；以及为什么对大 diff 允许 * 向量增量 + 图全量 * 的拆分而不破坏一致性。

### 实测：图能回答向量搜索回答不了的问题

在参考实现上跑两条查询（N = 412 个函数，噪声过滤后 782 条边）：

| 查询 | 纯向量 recall@10 | 图优先 recall@10 |
|------------------------------------------|-----------------------|-----------------------|
| * “谁调用了 `runSync`” * | 0.30 | 1.00 |
| * “返回 `*SyncStatus` 的函数” * | 0.10 | 1.00 |
| * “`service.go` 中定义的所有内容” * | 0.40 | 1.00 |

纯向量一列走的是“嵌入函数体”式检索（第 17 章），查询按自然语言问题的形式发起。图一列只是从目标节点走一步 Cypher。混合检索（第 19 章）把两者组合起来，让那些“看起来模糊其实结构化”的查询 —— 比如 * “HEAD commit 是在哪里缓存的” * —— 也能赢。

## Example —— 范例

当一次同步完成后（见第 16 章的范例），这张活的图就可以用任何 Bolt 客户端查询。以下是一条用内置 `code-kg query` 子命令发出的单行查询：

```bash
export GRAPH_URI=bolt://localhost:7687
go run ./cmd/code-kg graph-query \
  --repo-id demo \
  --cypher "MATCH (f:Function {name:'runSync'})<-[:CALLS]-(caller)
            RETURN caller.name, caller.file_path, caller.start_line"
```

参考 commit 上的输出：

```
TriggerSync    | reference-impl/code-kg/service.go | 161
runIncrementalSync (recursion guard) | reference-impl/code-kg/service.go | 278
```

这种结构上的精确，单靠嵌入是达不到的。

## 11.5   图数据库选型对比

图数据库是基础设施级别的赌注。选错了，迁移成本会远超你一开始省下的部署时间。本节把我们评估过的六个候选者放在同一张表里，给出本书代码知识图谱场景下的推荐路径。

### 候选者一览

| Graph Store | License | Protocol | Clustering | Max Edges (tested) | Write Pattern | Query Latency (1-hop) | Notes |
|---|---|---|---|---|---|---|---|
| Memgraph | BSL + Community | Bolt / Cypher | No built-in | ~10M | Full rebuild fast | <5 ms | In-memory, simple ops |
| Neo4j Community | GPL v3 | Bolt / Cypher | No | ~100M+ | Transaction-based | <10 ms | Mature ecosystem |
| Neo4j Enterprise | Commercial | Bolt / Cypher | Causal cluster | Billions | Transaction-based | <10 ms | Expensive license |
| FalkorDB | MIT | Bolt / Cypher-ish | No | ~50M | Transaction-based | <3 ms | Redis-based, very fast reads |
| SQLite + recursive CTE | Public domain | SQL | N/A | ~1M | File-based | ~20 ms | Zero dependency |
| NetworkX (in-memory) | BSD | Python API | N/A | ~500K | In-memory dict | <1 ms | No persistence, testing/prototype only |

几点说明：

- **Memgraph** 的 Community Edition 采用 BSL 许可证，生产使用需注意条款。它的最大优势是纯内存架构——对于本章的"先 `DETACH DELETE`、再 `UNWIND` + `MERGE`"全量重建模式，写入速度极快，冷启动也只是把 snapshot 加载进 RAM。
- **Neo4j Community** 是生态最成熟的选择：驱动覆盖 Go / Python / Java / JS，社区文档丰富。GPL v3 许可证意味着如果你要分发嵌入了 Neo4j 的产品，需要做合规审查；但如果只是作为内部服务运行，通常不受影响。
- **FalkorDB**（前身为 RedisGraph）基于 Redis，读延迟极低（<3 ms），MIT 许可证也最宽松。缺点是 Cypher 方言与标准 openCypher 有细微差异，部分高级语法不支持。
- **SQLite + recursive CTE** 是"零依赖"方案：不需要任何外部进程，一个 `.db` 文件即可。代价是没有原生图语义——你需要自己用 `WITH RECURSIVE` 手写遍历，且在多跳查询上性能会急剧下降。适合 CI 环境里的集成测试或极小型项目。
- **NetworkX** 只适用于单元测试和原型验证。它没有持久化、没有并发安全、没有查询语言——但在 pytest 里造一个 fixture 图快得无与伦比。

### 本书场景的决策矩阵

代码知识图谱的典型工作负载有四个特征：

1. **规模有限。** 1k–50k 实体，5k–200k 边。即使是百万行级别的 monorepo，经过噪声过滤后的实体数也很少超过 50k。
2. **读多写少，写是批量。** 日常使用全是只读查询（第 19 章的检索器）；写入只发生在同步时，且是全量重建。
3. **需要 Cypher 或 Cypher-like 查询语言。** 第 19 章的混合检索器需要在运行时动态拼 Cypher，包括 `MATCH`、`OPTIONAL MATCH`、`WITH`、`RETURN`。
4. **必须支持 `DETACH DELETE` + `MERGE` 模式。** 这是本章"每仓库全量重建"策略的核心原语。

综合以上约束，我们的推荐是：

- **<50k 实体（绝大多数单仓项目）：Memgraph。** 部署最简单（一个 Docker 容器），全量重建最快，Cypher 兼容性最好。本书参考实现的默认后端。
- **>50k 实体（大型 monorepo 或多仓聚合）：Neo4j Community。** 磁盘存储不受 RAM 限制，事务语义更完整，社区插件丰富。
- **零依赖回退方案：SQLite + recursive CTE。** 当你无法运行任何外部数据库（CI pipeline、serverless function、嵌入式场景）时，这是唯一的选择。需要自行实现一个 `SQLiteGraphStore` 适配器，把 Cypher 翻译成 SQL——工作量约 200–300 行代码。

### 迁移路径

除 SQLite 外，上述所有候选者都通过 Bolt 协议通信。这意味着本章定义的 `GraphStore` 接口（`UpsertRepositoryGraph`、`QueryGraph`）对上层完全透明——切换后端只需替换连接字符串和适配器实现，检索器代码无需改动。

```
应用层 (Retriever / CLI)
      │
      ▼
GraphStore interface  ← 本章定义的抽象
      │
      ├─ MemgraphStore   (bolt://localhost:7687)
      ├─ Neo4jStore       (bolt://localhost:7687)
      ├─ FalkorDBStore    (bolt://localhost:6379)
      └─ SQLiteStore      (file:///path/to/graph.db)  ← 需自行翻译 Cypher → SQL
```

从 Memgraph 迁移到 Neo4j Community，实测只需要改三处：连接 URI、认证方式、以及一个 `MERGE` 语法的微小差异（Neo4j 要求 `ON CREATE SET` 而 Memgraph 允许省略）。总迁移时间不应超过半天。

### 为什么不用通用分布式图数据库

Amazon Neptune、JanusGraph、TigerGraph 等分布式图数据库在社交网络、欺诈检测等十亿边级别场景下有不可替代的优势。但对于代码知识图谱：

- **规模不匹配。** 一个 100 万行的仓库，过滤后的图也就几万个节点、十几万条边。这甚至不到 Memgraph 内存模型的 1% 容量。
- **增加云依赖。** Neptune 绑定 AWS VPC，JanusGraph 需要 Cassandra/HBase + Elasticsearch 后端。每多一个外部组件，运维半径就扩大一圈。
- **查询模型过重。** Gremlin（Neptune / JanusGraph 的原生语言）比 Cypher 更通用但也更冗长。本书的查询模式——1-hop 邻居、2-hop 路径、`MATCH ... WHERE ... RETURN`——在 Cypher 里是一行，在 Gremlin 里是五行。
- **成本不对称。** Neptune 的最低实例费用（db.r5.large）约 $0.58/h，一年 $5,000+。Memgraph Community 在一台 2 GB RAM 的 VM 上就能跑。

**底线是：如果你的图装得进一台机器的内存，就不要引入分布式图数据库。** 代码知识图谱几乎总是满足这个条件。

## 11.6   图谱可视化与探索

一张你看不见的图，就是一张你无法信任的图。上面几节讨论了如何构建和存储代码知识图谱，但如果你从来不把它渲染出来，三类问题会悄悄潜伏：缺失的边（两个应该有 `CALLS` 关系的函数之间什么都没有）、孤立节点（过滤器遗漏导致噪声实体残留）、以及意料之外的聚类（本该独立的两个子系统被一条错误的 `IMPORTS` 边粘在了一起）。可视化不是图谱的"附加功能" —— 它是你验证图谱质量的第一道防线。

### 工具对比

下表汇总了六种适合代码知识图谱场景的可视化工具。"可用节点上限"是指在交互操作仍然流畅的前提下的经验值，而非硬性限制。

| 工具 | 协议 | 交互式？ | 可用节点上限 | 可自托管？ | 备注 |
|---|---|---|---|---|---|
| Memgraph Lab | Bolt | 是 | ~5,000 | 是（Docker） | Memgraph 用户的首选 |
| Neo4j Browser | Bolt | 是 | ~3,000 | 是 | 经典方案，社区广泛使用 |
| Gephi | GEXF 文件 | 是 | ~100,000 | 桌面应用 | 大规模分析的最佳选择 |
| D3.js (force layout) | JSON API | 是 | ~2,000 | 自定义代码 | 可嵌入 KB Web UI |
| Cytoscape.js | JSON | 是 | ~5,000 | 自定义代码 | 适合生物信息/代码图谱 |
| yEd | GraphML | 是 | ~10,000 | 桌面应用 | 自动布局算法丰富 |

### 本书的推荐方案

针对代码知识图谱的三种典型使用场景，我们推荐不同的工具链：

- **日常开发与调试：Memgraph Lab。** 免费、Docker 一键部署、通过 Bolt 协议直连。你在上一节选好的 Memgraph 实例上什么都不用额外配置，打开浏览器访问 `http://localhost:3000` 即可开始探索。输入一条 Cypher，节点和边立刻以力导向布局渲染在画布上。
- **团队汇报与演示：导出子图为 JSON，用 D3.js force layout 渲染。** 当你需要把一个子系统的调用关系展示给不使用 Bolt 客户端的同事时，把 Cypher 查询结果序列化为 `{nodes: [...], links: [...]}` 格式的 JSON 文件，然后用一个不到 100 行的 D3.js 页面渲染。这种方式可以嵌入内部文档站或 Confluence 页面。
- **深度分析：导出完整图为 GEXF 格式，在 Gephi 中打开。** 当你需要做社区检测（哪些模块形成了紧密耦合的集群？）、度分布分析（是否存在超级 hub 节点？）、或者生成出版级别的图谱插图时，Gephi 的可视化能力和统计插件是其他工具无法替代的。导出命令只需要一行：`MATCH (n)-[r]->(m) RETURN n, r, m` 然后用脚本转成 GEXF。

### 用于可视化的实用 Cypher 查询

以下三条查询覆盖了可视化场景中最常见的需求。它们可以直接粘贴到 Memgraph Lab 或 Neo4j Browser 中执行：

```cypher
// 围绕特定实体的子图 —— 查看某个函数的 1~2 跳邻居
MATCH path = (e:Entity {name: 'HandleRequest'})-[*1..2]-()
RETURN path

// 孤立节点检测 —— 没有任何边的实体，通常是噪声过滤遗漏
MATCH (e:Entity) WHERE NOT (e)-[]-() RETURN e

// 最高连接度实体（hub 检测）—— 度数最高的 20 个节点
MATCH (e:Entity)-[r]-()
RETURN e.name, count(r) AS degree
ORDER BY degree DESC LIMIT 20
```

**第一条**查询是日常使用频率最高的：输入你正在调查的函数名，立刻看到它的调用者、被调用者、所在文件、实现的接口。在 Memgraph Lab 中，结果会自动渲染为可交互的力导向图。

**第二条**查询用于质量审计。如果你的图里有大量孤立节点，说明 builder 的关系抽取存在遗漏，或者噪声过滤器的停用表需要更新。一个健康的代码知识图谱，孤立节点比例应低于 5%。

**第三条**查询帮你发现图中的 hub 节点。在代码图谱中，hub 通常是工具函数（如 `log`、`handleError`）或核心入口（如 `main`、`router`）。如果一个不起眼的辅助函数出现在 top-20，可能意味着边的抽取存在误报。

### 在 KB Web UI 中嵌入可视化

如果你的知识库有一个 Web 前端（即使只是一个内部工具），添加一个 `/graph` 端点可以极大提升可发现性。推荐的架构是：

1. 后端提供 `/api/graph?filter=package:service` 接口，接收过滤参数（按 package、module、file），执行对应的 Cypher 查询，返回 `{nodes, links}` JSON。
2. 前端用 D3.js force layout（或 Cytoscape.js）渲染。节点颜色按 `entityType` 区分（函数 = 蓝色，接口 = 绿色，文件 = 灰色），边的粗细按关系类型区分。
3. 支持点击节点跳转到对应的源码位置（利用节点上的 `filePath` + `startLine` 属性）。

这个端点不需要很复杂 —— 一个 50 行的 handler + 一个 100 行的 D3.js 模板就够了。它的价值在于：让每一位团队成员都能在浏览器里直接"看见"代码之间的结构关系，而不必安装 Bolt 客户端或学习 Cypher 语法。

## 常见错误

代码图设计里有四个反模式贡献了我们见过的大多数失败。每一个都源自：把做 * 文档 * 图时的先验直接搬到一个先验并不成立的领域。

**1 —— 边的类型不收敛。** 症状是：图 schema 里有 40 种关系 —— `USES`、`NEEDS`、`REFERS_TO`、`DEPENDS_ON`、`TOUCHES`、`READS`、`WRITES`…… —— 彼此语义互相重叠。写查询的人会逐渐不再信任这张图，因为现实中的同一条边会出现在三个不同标签之下。一开始就从 * What * 段里的四种核心边做起（`CONTAINS`、`IMPORTS`、`CALLS`、`IMPLEMENTS`）。只有当“有一种查询不加这条边根本答不了”时，才引入新边。

**2 —— 把图放在热写路径上。** 症状是：提交一行代码改动会触发一次 200 ms 的图数据库往返。诱惑在于通过逐步 mutation 让图“随时保持一致”。正确做法是第 18 章的“全量重建”模式：用 30–60 秒的过时换一次“一行清空 + 一次性插入”，永远不会让图停留在半写入状态。当仓库规模超过约 500 kLOC，升级路径是“按文件失效” —— 但这是 * 升级 * ，不是起点。

**3 —— 把每一个标识符都当节点索引。** 症状是：一个 4 kLOC 的仓库在图里有 20 万个节点，大部分是 `len`、`append`、`make`、`fmt`、`errors` 这样的内建。噪声把信号淹没了。修法是在解析时（第 16 章）就套上一个按语言定制的噪声过滤器 —— 把标准库名当作“指向空的边”，而不是当作节点。参考实现里每种语言大约带 200 个名字的停用表；它在典型 Go 代码上能减少 60%–80% 的节点数，而查询能力毫无损失。

**4 —— 在图节点里存函数体。** 症状是：一个源码 40 MB 的项目，图数据库变成 4 GB。诱惑是“让所有东西都在一个地方”。图的职责是 * 身份 + 关系 * ；向量存储的职责是 * 内容 * 。让节点保持小（名字、文件、起止行、kind、签名）。函数体属于向量存储；如果你真的需要全文检索，就放到第三列。那些“一不小心扫了 4 GB 正文”的图查询，比只走 40 MB 身份索引的查询慢 100 倍。

## Conclusion —— 小结

图谱承载的是代码知识里那一半在任何有损投影下都无法保留的东西：谁连接到谁。有边的封闭集合、不含内容的节点身份、以及“每仓库全量重建”的写入模式，即使仓库变大，这份存储在概念上也一直简单。第 19 章会把两栈 —— 向量和图 —— 统一在同一个检索器接口之下。

## 参考文献

```{bibliography}
:filter: keywords % "graph" or keywords % "code-layer"
```
