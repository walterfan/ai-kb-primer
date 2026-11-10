---
title: "第 7 章 —— 发布与搜索"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - prose-layer
  - search
  - publishing
---

# 第 7 章 —— 发布与搜索

本章一页速览 —— “**定制的内存索引** vs. **一套要运维的搜索集群**”之间那一次**决定性的取舍**；那个**十行**的分词器 —— 它的默认值却意外地承重；**前缀匹配搜索**在**标识符字面量查询**和 **CJK 查询**上的失败形态；以及一整条**台阶式**的扩容路径 —— 让 `Search(...)` 在**五个数量级**的语料规模上仍然是**同一个稳定的接口**：

```{mermaid}
mindmap
  root((Publish & search))
    Why
      Navigation is not search
      Search fails first at scale
      Static site must survive the server
    In-memory index
      Slug map
      Inverted postings (term -> slugs)
      Derived views (author, type, status)
      Whole-map rebuild on write
    Tokeniser
      Unicode letter/digit
      No stemming (identifier-friendly)
      Length >= 2, deduped
      Lowercase, locale-insensitive
    Query
      Prefix match (not exact)
      Score = term hits
      No BM25, no phrases
      Linear in vocabulary
    Static publishing
      Sphinx + MyST
      Client-side search (Porter stem)
      Cross-refs as hard errors
      Themeable output
    Failure modes
      CJK segmentation missing
      Identifier-literal paraphrase
      Typo intolerance
      Scale cliff at ~100k pages
    Staircase
      In-memory -> Bleve
      Bleve -> Meilisearch
      Meilisearch -> ES
      Interface unchanged
```

上周**新来了一位工程师**。周四，她往 wiki 的搜索框里输了 `recieve` —— **零条结果**。周五，她输的是 `配置文件` —— **零条结果**。周一，她输的是 `rate limit` —— **排第一的**是**一篇三年前的博客帖**，而真正的那份 runbook —— **标题就叫** *“Rate limit configuration”* —— **在第四页**。到了周二，她**开了一个 JIRA 工单**，标题是“**wiki 的搜索坏了**”，**指派给你**。

wiki 的搜索**其实并没有坏**。它是**一个十行的 tokenizer****加一段前缀匹配循环** —— **对着它那份规格**，**它工作得好极了**。**坏的是这件事**：**这个团队把“十行 tokenizer”****当成了知识库 *全生命周期* 里交付的东西**，**而没有把它当成一段阶梯的第一级** —— **一段后面几级都很容易上的阶梯**。这一章**两件事都讲**：**那个小小的、凭实力占住阶梯最底层的引擎**，以及**一张诚实的地图** —— **当什么类型的查询开始失败时，****应该往上爬到阶梯的哪一级**。

## 为什么

第 6 章走到尾声时，我们已经有了**一棵提交进 Git 的、格式规范的 Markdown 页面树**，带着诚实的 frontmatter 。在第二部分能够收尾之前，还剩两个问题：

1.  **读者怎么找到一页？** 导航是一个答案（Diátaxis 形状的 toctree 干的就是这个）。*搜索*是另一个，而且它是知识库在增长到几百页之后最先失败的地方。
2.  **读者怎么阅读页面？** 后端负责动态渲染，但同一批文件也必须能构建为静态站点 —— 这样知识库才能在服务器不在时存活、可以被镜像、可以被 vendor 进其他仓库。

这一章两个都讲，同时**顶住生态里两种被反复推销的诱惑**。第一种：“直接上 Elasticsearch”。一套要运维的搜索集群本身就是**一套独立的 on-call 排班**；一个几万页级别的文件型 wiki **还配不上这个代价**。第二种：“别自己发自家搜索”。**十行**分词器 + **一个前缀匹配循环**已经足够让 95% 的软件知识库**够用**，并且它**完全在你手里**、**可调试**、**能塞进单个 binary 里发布**。

## 是什么

两个系统合作完成这件事：

- **内存索引**（本章，`backend/internal/wiki/index.go`）：一个 Go struct，把 slug 映射到解析后的页面，外加一个辅助映射 `searchTerms: map[term]→[]slug`。它运行在 `kb-cli serve` 进程内部，持有全部数据以供实时查找，并在每个会写入的流水线阶段从磁盘重建。

- **静态发布器**（`wiki-template/metadata/_sphinx/`）：一个 Sphinx {cite}`sphinx` + MyST-Parser {cite}`myst_parser` 构建目标，输入就是同一棵 `content/` 树。运行 `make sphinx-build` 会在 `metadata/_build/html/` 下产出一个完全静态的 HTML 站点，可以部署在任何地方（GitHub Pages、S3、一台笔记本）。

**两套系统都不拥有内容；****两者都是对文件系统的 *视图* 。**把服务器删掉，静态构建照常渲染。把静态构建删掉，服务器照常搜索。**文件才是权威制品**，第 4 章已经这么承诺过。

## 怎么做

### 内存索引

核心数据结构**一屏就能装下**：

```{literalinclude} ../examples/part2-prose-layer/ch07/index.go
:language: go
:lines: 66-79
:caption: `IndexEngine` — slug map, plus a few specialised views.
```

有八张辅助视图 —— 按标题首字母、按 category、按词、按作者、按 `doc_type` 、按 source type、按 `verification_status` —— **全部**从 `pages` 派生，**一轮遍历全部重建**。就**全文检索**而言，**真正重要的字段只有 `searchTerms`** 。

#### 分词

分词器是**十行**的 Go：

```{literalinclude} ../examples/part2-prose-layer/ch07/index.go
:language: go
:lines: 285-299
:caption: `tokenize` — lowercase, letter-or-digit split, length ≥ 2, dedupe.
```

这里编码进去的**三个选择**值得单独辩护一下：

1.  **Unicode 字母/数字，而非 ASCII 空白。** `unicode.IsLetter` 涵盖 CJK 字符、西里尔文、阿拉伯文 —— 任何 Unicode 字母类别都包括 —— 因此任何语言的 wiki 都能正确分词。
2.  **不做词干提取。** Porter 1980 年的算法 {cite}`porter1980stemming` 能提升英文的召回率（"install" / "installing" / "installed" 全部折叠为 "instal"），但它是语言相关的，而且会让期望字面匹配的工程师觉得搜索结果不对劲。在软件知识库里，这个取舍朝另一个方向倒。
3.  **长度 ≥ 2，去重。** 杀掉单字母噪声；让内存消耗对文档长度线性增长，而非二次增长。

#### 构建 postings list

**构建索引就是一层嵌套循环，不是一棵 B-tree：**

```{literalinclude} ../examples/part2-prose-layer/ch07/index.go
:language: go
:lines: 275-283
:caption: `buildSearchIndex` — tokenise every page, invert into a map.
```

对每一页，把 `title + body + tags` 一起分词，**每一个 token 都把这一页的 slug 追加进去**。结果就是经典 IR 意义下的**一张 postings list** {cite}`manning2008introduction` —— 一张从 term 到“包含该 term 的文档列表”的 map 。**没有**per-token 的 tf-idf 权重，**没有**位置信息，**所以**查询那一侧也很短。

#### 查询

查询也用**同一套分词**，每一个 term 都对索引的 key 做**前缀匹配**，然后 slug 按“**有多少个查询 term 命中了这页**”打分：

```{literalinclude} ../examples/part2-prose-layer/ch07/index.go
:language: go
:lines: 354-394
:caption: `Search` — prefix match, term-count score, sort, return summaries.
```

这份实现有**四条属性**：

- **前缀匹配**，不是精确匹配。查询 `insta` 能匹配到 `install`、`installation` 和 `installer`。这是一个务实的折中，覆盖了词干提取能给你的大部分东西。
- **没有 BM25，没有 tf-idf** {cite}`manning2008introduction,robertson2009probabilistic`。得分是匹配到的查询词项的数量。这比真正的 IR 排序器弱，但在这个文档数量级下，用户很少察觉。
- **没有布尔运算符**，没有短语查询，没有字段查询。软件知识库的典型查询是 1–3 个关键词；放弃 Lucene 风格的语法，换来的是简单性。
- **每次查询对 `|searchTerms|` 线性。** 对本书面向的万页级规模，"对索引线性"在单线程上就够快了。超过这个规模，正确的做法是引入 Bleve 或 Meilisearch，而不是在 CLI binary 里重新发明它们。

整套搜索子系统，端到端，**不到 100 行 Go** 。它在它瞄准的规模上 *是正确的* ，**任何一个 Go 程序员都能调它**，而且它就**跟着那个 binary 一起发布**。规模变了，**接口也不用变** —— `SearchService` **已经**接受一个查询字符串、返回一个 `SearchResult` ；哪天换成一个 Bleve 后端，可以直接**塞在接口后面**，**HTTP 和 CLI 这两层都不用动**。

### 实时更新

每一条写入路径（第 5 章的 `CreatePage` 、第 6 章的 ingest 循环、更新路径、删除路径）最后都会调一下 `index.AddPage` 、 `index.UpdatePage` 、 或 `index.DeletePage` ，**每一个**都会拿锁、重建派生视图：

```{literalinclude} ../examples/part2-prose-layer/ch07/index.go
:language: go
:lines: 101-107
:caption: `AddPage` — single-writer, whole-map rebuild, simple enough to trust.
```

“**全部重建**”听起来很浪费，但它在页面数上是**线性**的，并且每一次写入**只跑一次**，**全程锁内**。**它干掉了增量索引器那种典型失败形态：****那些没人发现的过期条目**——直到用户抱怨才冒出来。**代价以毫秒计；****收益是每一次操作之后，**索引与 `pages` 之间是**可证一致的**。

### 用 Sphinx 做静态发布

在发布这一块，文稿层参考实现**不去发明构建器** —— **直接借力** Sphinx + MyST-Parser，和**这本书自己**用的是同一套栈。一个**最小版**的 `metadata/_sphinx/conf.py` 长这样：

```python
extensions = ["myst_parser", "sphinxcontrib.mermaid"]
source_suffix = {".md": "markdown"}
html_theme    = "sphinx_rtd_theme"
master_doc    = "index"
```

根目录的 `index.md` 用的就是**那份 `INDEX.md`** —— `pipeline.Build()` 在**每一次 ingest 之后都会重新生成**它。**那份文件**就是**实时服务器**和**静态站**之间的桥：**两种视图**都消费它。站点构建**一条 shell 命令**就够了：

```bash
$ make sphinx-build WIKI_DIR=./demo
$ ls ./demo/metadata/_build/html/
index.html  search.html  _static/  _sources/  ...
```

**从 Sphinx 里白捡的三件东西：**

- **页内搜索。** Sphinx 在构建时用 Porter stemmer 生成一份客户端 JavaScript 索引，离线也能用。静态站的读者拥有一份不输给服务器的搜索体验。
- **交叉引用。** `{doc}`、`{ref}` 和 citation 角色在整棵树上解析，解析不到就构建失败。这比实时服务器的前缀匹配严格得多，能捕获 `verify` 没抓到的断裂引用。
- **可主题化的输出。** Read the Docs 主题、Furo、book 主题、通过 xelatex 出 PDF —— 同一棵 Markdown 源文件树可以渲染出所有这些。

**你放弃的是 *交互性* ：**静态站**不能**创建页面、**不能**根据点击率把搜索结果重新排序、**也不能**显示“最近一小时的变更”。对一个**故意用 Git 做版本控制**的软件知识库来说，**这正是对的取舍**。**变更在仓库里、在服务器里；****发布是一份派生的、不可变的制品。**

## 示例

**对同一批页面走三次查询**，**手工**演算一遍。

假设 `content/` 下有以下三页：

- `how-to/enable-tls.md` —— `title: How to enable TLS` ，tags `[tls, security]` 。
- `reference/api.md` —— `title: API Reference`，正文提到 "install" 和 "tokenization"
- `explanation/auth-model.md` —— `title: Authentication Model`，正文讨论 "TLS" 和 "tokens"

**查询 `tls`：**

- tokens：`["tls"]`
- 对 `tls` 做前缀匹配 → 命中第 1 页和第 3 页中的 `tls` 。
- 得分：`{how-to/enable-tls: 1, explanation/auth-model: 1}`
- 按得分倒序，两页都返回。

**查询 `token`：**

- tokens：`["token"]`
- 对 `token` 做前缀匹配 → 命中第 2 页里的 `tokenization` 和第 3 页里的 `tokens` 。
- 得分：`{reference/api: 1, explanation/auth-model: 1}`
- 两页都返回。

**查询 `tls token`：**

- tokens：`["tls", "token"]`
- 第 3 页同时命中两个前缀 → 得分 2
- 第 1、2 页各命中一个前缀 → 得分 1
- 排序：`explanation/auth-model` 排第一，然后是另外两页。

**没有一个查询比用户预期的更聪明 —— 这正是一个 feature 。**

## 这台小小的搜索引擎在哪里会失败 —— 以及为什么那仍然没关系

第 2 章把词汇表搭起来是有理由的 —— TF-IDF、BM25、混合检索、重排序 —— *那一层* 才是一条**严肃的 RAG 流水线**递给 LLM 的**检索面**。本章这台搜索引擎是**它非常谦卑的表亲**：几千字节的 Go 代码，直接给坐在搜索框前的**人**用的。**值得明确说一句**：这台引擎**做不了什么** —— 因为**“那就给它加个 reranker”**这种诱惑，**恰恰是一个团队应该顶住的诱惑**，**直到问题真的冒出来之前都不要动**。

有**四类查询**会以**可预见的方式**把前缀匹配索引干翻。每一类都**配一个便宜解和一个昂贵解**；**几乎所有团队一上来都先去抓昂贵解**。

1.  **带有意译的标识符查询。** 用户搜 `"rate limit ingest"`，但代码里的符号是 `ingestRateLimit`。分词器在 postings list 里按大小写边界拆分了标识符，所以 `rate`、`limit`、`ingest` 各自成为独立词项 —— 用户查询的 token 恰好也是这三个，因此这种情况实际上能匹配上。真正*会*失败的是反向的情况：用户搜的是意译 `"throttle incoming traffic"`，和标识符以及描述它的文字没有任何共同 token。这正是第 2 章点名的经典词汇不匹配失败；便宜的修法是给页面本身打一套激进的标签（frontmatter 的 `tags:` 字段已经在 postings list 里了），昂贵的修法是稠密检索（第 15 章）。小团队几乎应该总是先试便宜的修法。

2.  **在没有分词的树上做 CJK 查询。** 分词器用 `unicode.IsLetter` 在非字母/非数字处切分 —— 而对中文或日文来说，*每一个字符都是字母，因此什么都不会被切开*。一个包含 `配置文件路径` 的页面和一条查询 `配置文件`，只有当页面的*完整序列*以查询开头时才能匹配上，而通常不是这样。诚实的修法是加一个分词器：中文用 `go-jieba`，日文用 `kagome`，都是 `go get` 就能拿到。不诚实的修法是宣称 CJK 支持不在范围内；这样做的团队在第一个非英语用户到来时就会后悔。钩子已经在那里了 —— `tokenize` 函数是唯一把正文转成词项的地方，换成一个语言感知的分词器只是一次局部改动。

3.  **拼写容错。** 搜索 `recieve` 什么都返回不了，因为 postings list 里只有 `receive`。词干提取修不了这个问题（两个词的词干不同）；编辑距离可以。库方案是查询时做 `fuzzy` 匹配，典型配置是对超过某个最小长度的词项用 Damerau-Levenshtein 距离 1 或 2。内存实现在万页规模下可以用十几行 Go 加上这个功能，但它的成本是二次方的（每个查询词项要和 postings list 里的每个词项比较），在超过约 10 万个词项时就不可忽视了。Meilisearch 的拼写容错非常好，运维成本就是一个 binary；到了那个规模，它就是正确的选择。

4.  **按相关性排序，而非按匹配数。** `Search` 函数按"有多少个查询词项命中了这份文档"打分。一页足够长的页面，只要*提到了*每一个词项，就能打赢一页*就是关于*该词项的聚焦页面。这正是 BM25 通过其长度归一化项 $b$（第 2 章）来修复的那种病症：长文档不应该仅仅因为长就占据主导。内存实现中便宜的修法是字段加权 —— 标题匹配比正文匹配加分更高，第 2 章也点过这个名 —— `searchTerms` map 可以通过记录每个词项的来源（"这个 token 是在标题里还是正文里？"）并在查询时加一个小系数来实现。仓库里的实现今天没做这件事；但接口支持它，不需要迁移。第 2 章的 *Advanced RAG* 一节解释了为什么字段加权比它下面的大多数调优旋钮都重要。

**这几类的共同线索是：****抬手去抓第 2 章那套更重的机器的时机，***是当一类具体的查询开始失败时* ，**不是在此之前**。**一个**实时搜索是前缀匹配索引、**而**面向 agent 的检索是 BM25+稠密+重排序流水线的软件知识库，**不是自相矛盾** —— **那是健康的设计**。**不同消费者有不同预算、有不同期望；****用同一层存储（磁盘上的文件）把两边都伺候了，****这本身就是那个让“意见分歧”变便宜的架构层面胜利。**

## 在把搜索接起来时的常见错误

**除了“按查询类别”的失败形态，****还有四个**在扩容和接线环节里**频繁出现的错误**，值得单独列一下：

1.  **试图维持一个增量索引的正确性。** 文稿层在写入时做全量重建是有原因的：增量更新是几乎所有基于数据库的 wiki 里"搜索过期"bug 的来源。在本章面向的规模下，全量重建是毫秒级的；为了追逐常数因子的性能而放弃它，是经典的过度工程失败。

2.  **"给生产站点"硬接第二套搜索系统。** 一个团队加了 Elasticsearch，理由是"内存那套不够正经"，最终得到了两层有时候意见不一致的搜索 —— 实时服务器上的读者看到一套结果，已发布静态站上的读者看到另一套。"其他做法"里的那条台阶是一个*序列*，不是一个选择：挑当前失败的那个最低台阶，把整个知识库迁过去，并保持一个唯一的 `Search(...)` 接口。

3.  **索引了信任层不认可的内容。** 一个朴素的 `LoadAndIndex` 索引了 `content/` 下的所有文件，包括 `verification_status` 为 `contradicted` 或 `superseded` 的页面。这些页面随后出现在搜索结果里，没有任何过期的可见信号。正确的做法是索引它们，但在默认查询中过滤掉（通过一个显式的"包含已废弃"开关给考古者使用）。第 5 章的信任字段是检索元数据，不只是展示元数据。

4.  **把 Sphinx 的页内搜索当"生产"搜索。** Sphinx 的客户端搜索对静态镜像来说很好；如果一个组织完全依赖它就是灾难了。它看不到 frontmatter，无法按 `verification_status` 过滤，它的排序是由 Porter 词干提取驱动的 {cite}`porter1980stemming` —— 这对英文散文是好信号，但对标识符字面查询就是*噪声*信号。让实时服务器的搜索对已认证读者保持权威；让静态构建的搜索为公开或离线场景存在，也仅为那些读者存在。

## 其他做法

| 系统 | 1 万页的索引体积 | 搭建成本 | 与文件型知识库的契合度 |
|--------|---------------------------|------------|--------------------------|
| 内存前缀索引（本章） | ~20 MB | 0（内置） | 1 万页以下很好用。 |
| Bleve {cite}`bleve` | 磁盘上 ~100 MB | `go get` + 几行代码 | **自然的扩容路径**；仍然**内嵌**，**无需服务端进程**。 |
| Meilisearch {cite}`meilisearch` | 独立进程 | Docker + 一个 API key | 排序好，容错拼写好；**有运维成本**。 |
| Algolia DocSearch {cite}`algolia_docsearch` | **爬、托管** | 注册 + 爬虫配置 | 开源项目免费；**私有知识库会被锁定**。 |
| Elasticsearch / OpenSearch | JVM 集群 | **高** | **对本章规模的知识库是 over-engineer 的**。 |
| **只用** Sphinx 页内搜索 | 构建时 | 0 | 对**静态文档**来说够用；**没有实时更新**。 |

**本书的建议是这条台阶：**

1.  **从内存索引开始。** **对任何一个团队知识库来说都够用。**
2.  **并行**给已发布的站点加上 Sphinx 的静态搜索。
3.  当内存版本变成 CPU-bound 时，把 Bleve 塞进 `Search` 接口后面。
4.  只有当你有明确的产品理由时（拼写容错、字段加权、多租户排序），才升级到 Meilisearch 或 Elasticsearch。

**每一次上台阶都保留 `Search(query, type) → SearchResult` 这份契约 —— 客户端代码不用改。**

## 结论

**一个基于文件的 wiki 要拿到“尽可能简单、但别更简单”那枚徽章**，**前提是搜索层也简单到能被审计**。**一张内存 postings 映射表 + 一个十行的分词器，****对一个中小规模的软件知识库来说，****就是一台完整、诚实的搜索引擎**。**Sphinx 负责静态视图那一边**，**带着它自己那套基于词干提取的搜索**。**每一页之所以都被一致地索引，****是因为每一条写入路径都在同一个 mutex 里调同一个 `AddPage` 结束**。

**第二部分到此收尾**。文稿层**所有会动的零件**都齐了：**文件**（第 4 章）、**作为来源的 frontmatter**（第 5 章）、**流水线**（第 6 章）、以及**发布 + 搜索**（本章）。**第四部分**要**从文稿转向代码** —— **embedding、代码图谱、****以及让一个软件知识库****不同于一座文档仓库的那种混合检索**。

## 参考文献

```{bibliography}
:filter: keywords % "prose-layer" or keywords % "search" or keywords % "publishing"
```
