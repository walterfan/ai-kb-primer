---
title: "第 15 章 —— 为什么代码不是文稿"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: none
keywords:
  - code-layer
  - deepwiki
  - parsing
  - ir
---

# 第 15 章 —— 为什么代码不是文稿

本章一图览 —— 三条结构性主张、一个实验、四个反模式，共同说明为什么第二部分那条 wiki 流水线承载不了代码：

```{mermaid}
mindmap
  root((Code != Prose))
    Structural claims
      Parse before index
      Embed identity not body
      Relations are first-class
    Experiment
      Vector-only vs hybrid
      Recall@10 on code queries
      Results in Appendix B
    Anti-patterns
      Chunked file embeddings
      Dense-only retrieval
      Graph-free design
      Boiling the ocean
    Hand-off to Part V
      ch9 Parser
      ch17 Embeddings
      ch18 Graph
      ch19 Hybrid retrieval
      ch20 Prompt and generation
```

周二下午两点。团队里新人入职第六周。她在 pull-request 群里问了一句：* “`runSync` 是从哪儿被调用的？” *。资深工程师甩了一张 `grep` 截图，37 条命中 —— 注释、测试、日志输出里的一个字符串字面量、两个跟目标类型毫无关系的方法同名。她依然看不懂这个仓库到底在做什么。wiki 里有三段讲“同步”，但没有一段提到任何函数名字。聊天 LLM 又一脸笃定地编了两个根本不存在的调用方函数。六周过去，她还在猜。**本章讲的就是：为什么你在第二部分刚搭好的那套 wiki 救不了她 —— 以及代码层必须长成什么样才能救回这个下午。**

## Why —— 为什么

第二部分做出的 wiki 把每一页都当作一份 * 文档 * 来对待：每个主题一个 Markdown 文件，按纯文本切词，按前缀匹配排序。这个模型撑到一定程度就够了 —— 它把文稿层问题解决得很彻底。但一旦问题落到 **代码** 上，这个模型立刻失灵。

考虑配套博客 {cite}`fanyamin2026deepwiki` 开篇就用的新人场景：有人问 * “`runSync` 是从哪儿被调用的？” *。在一个七年老的 Go 仓库里，这句话有三个糟糕的回答和一个正确的回答。糟糕的回答是：

1.  `grep -r runSync` —— 会返回每一条注释、每一行日志、每一个测试、每一个字符串字面量，以及恰好同名但位于不相关类型上的每一个方法。
2.  *"自己读代码。"* —— 高级工程师的回答，也是一种自白：代码库已经超越了个人的阅读能力。
3.  把整个仓库贴进一个对话 LLM —— 撑爆上下文窗口，返回一个自信的错误答案，且不提供任何可追溯的引用。

正确的回答是 * “总共有 5 个调用方，都在 `reference-impl/code-kg/service.go` 的 154、161、278、364、376 行” * —— 要在规模上可复现地给出这个回答，知识库就必须 **把代码当代码建模**、而不是当成文稿。也就是：解析成语法树、抽取实体、以稳定标识符存储，并记录谁调用了谁。

本章会解释为什么这件事不是可选项，顺手梳理一下学界过去十年一直在提醒我们的那套文献 {cite}`allamanis2018survey`，并为后续五章搭好舞台：解析（第 16 章）、嵌入（第 17 章）、图谱（第 18 章）、检索（第 19 章）与生成（第 20 章）。

## What —— 是什么

* “代码的自然性” * 这一脉文献 {cite}`allamanis2018survey` 中的三条主张，塑造了第四部分的所有设计。

**主张 1 —— 代码带有文稿检索器会丢弃的结构。** 一个函数不是词袋。它有名字、签名、docstring、函数体、起始行、结束行、import，而且 —— 一旦有了 import —— 还会有一张 * 调用图 * 。你给 Markdown 配的 `RecursiveCharacterTextSplitter` 会兴高采烈地把一个 Java 方法切成三段，让签名与实现分离，让实现与它的结束花括号分离。切出来的嵌入会把这三段摆在向量空间里三个不同的区域。查询这个 * 方法 * 时，你只能捞到若干碎片、顺序错乱，还没法拼回去。

**主张 2 —— 代码搜索需要精确匹配的逃生舱，稠密检索独自做不到。** 当用户输入 `entityID`，他期待的是名为 `entityID` 的那个函数 —— 不是 * “* entity * 和 * ID * 的语义邻居” * 。CodeSearchNet 的评测 {cite}`husain2019codesearchnet` 把这件事钉死了：在以标识符为主的查询上，早期纯稠密代码搜索完败于 BM25 {cite}`robertson2009bm25`。补救办法是混合检索（第 19 章），但前提依然是主张 1 —— 你首先得有命名实体可排。

**主张 3 —— 代码中存在再多嵌入也捕捉不到的关系。** 向量空间编码相似性，不编码 * 调用 * 、* 实现 * 、* 导入 * 、* 返回 * 。两个签名和注释都一样的函数，即使一个位于 HTTP handler 层、一个位于后台 worker，它们在向量空间里也会紧紧聚在一起。GraphCodeBERT {cite}`guo2021graphcodebert` 证明：把数据流边注入到训练里能显著提升代码理解任务；我们的工程版简化表述是：如果你要回答 * 谁调用了谁 * ，就只能自己建图（第 18 章） —— DeepWiki 方法论 {cite}`fanyamin2026deepwiki` 在 §§2 和 §§6 反复敲这个黑板。

## How —— 怎么做

对应的架构结论就是 **双栈** ：一个向量存储负责相似性，一个图存储负责结构。两者由同一个解析器喂数据，而解析器正是第一个会暴露设计决策的地方。

第四部分的组织方式，就照着代码层参考实现在 `Service.runSync` 和 `runIncrementalSync` 里真实跑的那条流水线展开：

```{mermaid}
flowchart LR
    A["Git working tree"] --> B["Parser (ch16)"]
    B --> C["Entity with<br/>stable ID"]
    C --> D["Embeddings (ch17)<br/>sqlite-vec / pgvector"]
    C --> E["Graph (ch18)<br/>Memgraph"]
    D --> F["Hybrid retriever (ch19)"]
    E --> F
    F --> G["Prompt &amp; generator (ch20)"]
```

这套栈的三项性质稍后各自有专章来展开：

- **稳定身份。** 实体 ID 是一个不含内容但感知位置的 hash —— `sha256(repoID + filePath + entityType + name + startLine)[:16]`。正是这个选择让第 18 章的图在重建索引时存活，让第 21 章的增量同步在对 `M` 执行"删除再插入"时不留悬空边。

- **结构化嵌入输入。** 我们不嵌入函数体。我们嵌入一个简短的模板化头部 —— *Language / Type / Name / Signature / Doc* —— 因为这才是承载用户真正查询的那份*意图*的东西。第 17 章度量了这二者在 recall 上的差距。

- **有界的关系分类法。** 我们只出厂八种边类型（`CONTAINS`、`IMPORTS`、`CALLS`、`IMPLEMENTS`、`EMBEDS`、`DEPENDS_ON`、`RETURNS`、`ACCEPTS`），仅此而已。加第九种是设计评审级别的变更 —— 这是一条让图在运维上保持合理规模的护栏。

本章之后的每一章都会基于代码层参考实现的 vendored 代码节选（路径位于 `reference-impl/code-kg/` 下）来展开，节选开头的来源元数据由 `make book-refresh-excerpts` 生成（见 `examples/SOURCE.md`）。

## Example —— 范例

一分钟的演示就能把三条主张落到实处。输入是两份文件，每层一份。

**文稿层**（Markdown） —— `runbook/onboarding.md`：

```markdown
## Syncing a new repository

To index a repo, register it with `code-kg register`, then call
`code-kg sync --repo-id <id>`. The sync pipeline parses the files,
generates embeddings, and updates the graph.
```

**代码层**（Go） —— 一个最小的 `service_sync.go`：

```go
func (s *Service) runSync(repo *Repository, path string) error {
    files, _ := s.parser.CollectSupportedFiles(path)
    for _, f := range files {
        s.parseFileForSync(repo.ID, path, f)
    }
    return s.graphStore.UpsertRepositoryGraph(repo.ID, repo.Name, ents, rels)
}
```

向第二部分的文稿层索引问 * “怎么同步一个仓库？” * —— 你会拿到那份 runbook。这是对的，因为文稿层就是干这个用的。

现在换个问题再问文稿层索引：* “`runSync` 是从哪儿被调用的？” *。你会什么有用的都拿不到，原因是：

1.  没有任何页面包含字面字符串 `runSync` —— runbook 用的是 CLI 名 `code-kg sync`，不是 Go 方法名。
2.  就算有，你拿到的也只是描述 sync 的文稿，而不是一份调用点列表。

最后，把同一个问题丢给代码层。回答是一张行级列表 —— 5 条 —— 每条都以 `file:line` 锚点引用了 `reference-impl/code-kg/service.go`。这就是第四部分要教你造的成品。

这个实验的可复现版本（跑在代码层参考实现上）见附录 B。

## 常见错误

下面是我们在“首次搭建代码知识库”的团队里最常见到的四种设计失败。前三个分别是上文三条主张的 * 反面 * ；第四个是 scope 失败。

**1 —— 把代码当长文稿对待。** 症状是每个文件或者每个固定字符长度的“chunk”只生成一个嵌入。`RecursiveCharacterTextSplitter(chunk_size=1500)` 会把一个 Java 方法切成三段，扔到向量空间的三个不同区域。解决办法就是 * What * 那一节的主张 1：先解析、再按一个 * 实体 * 一个嵌入的方式生成，并给每个实体稳定身份。解析器（第 16 章）是让这件事成为可能的关键；按 chunk 切出来再嵌入做不到。

**2 —— 仅使用稠密检索。** 症状是：输入一个精确标识符（`entityID`、`runSync`）时，真正的定义会排在三个毫无关系的“邻居”之下。嵌入优化的是语义邻域，而精确标识符很少正好落在那个邻域的 * 最近点 * 。解决办法是第 19 章的混合检索 —— 把 BM25 风格的打分作为兜底层，或者和稠密检索做融合排名。

**3 —— 没有图谱的设计。** 症状是 * “谁调用了 X” * 、* “什么实现了 Y” * 这类问题根本 * 问不出来 * 。向量空间不编码边。业界用更大上下文窗口去遮盖这个问题（* “把整个仓库贴进去不就行了” * ）的做法，在代码规模上只能线性扩展，过了大约 20 kLOC 就会塌掉。解决办法是第 18 章：一个承载身份 + 关系的有界图谱，和向量存储并列存在。

**4 —— 想一口气烧开整片海洋。** 症状是：团队还没交付过哪怕一条可引用的答案，架构文档就提出要造一个带自定义嵌入、自定义索引、自定义微调的“统一存储”。第四部分的构造刻意保持乏味：tree-sitter + sqlite-vec + Memgraph + 一个单 prompt 的生成器。就从这里开始。等你量化了哪些指标真正能把检索数据往上抬之后，再按第 22 章的升级矩阵有针对性地升级某个阶段。

## Conclusion —— 小结

文稿检索器和代码检索器在 API 层面长得很像 —— 都是接收一个查询串、返回一串排序后的文档 —— 但它们会在不同的查询上失败，而且失败的原因也不同。第四部分后面的内容，是一次构造性证明：一个基于文件的 wiki（第二部分）加上一条四阶段的代码流水线（解析 / 嵌入 / 图 / 检索），可以 * 同时 * 覆盖两类问题，而无需把它们硬塞进一个统一存储。

第 16 章从第一个阶段开始讲：解析。

## 参考文献

```{bibliography}
:filter: keywords % "code-layer" or keywords % "deepwiki" or keywords % "parsing" or keywords % "ir"
```
