---
title: "第 29 章 —— 在知识库上跑的 AI Agent"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: partial
keywords:
  - hybrid
  - agents
  - mcp
---

# 第 29 章 —— 在知识库上跑的 AI Agent

## Why —— 为什么

一个知识库有两种读者。人类读者打开浏览器、输入查询、扫一眼页面、关掉标签页。Agent 读者 —— IDE 里的 LLM 助手、聊天客户端、CI 任务 —— 不会打开浏览器。它调用一个函数、收下一份结构化响应、然后继续跑下去。本书的大部分篇幅都以人类读者为对象，本章讨论的是另一种。

这不是新思路。本书的种子博文 {cite}`fanyamin2026deepwiki` 早已主张：当 LLM 可以直接调用知识库、而不再依赖人去阅读它时，这个知识库的价值就会复利增长。那篇博文之后的新变化有两件：一是出现了这种调用的 * 协议 * —— Model Context Protocol {cite}`mcp_spec` ；二是出现了一种常见的 * 部署模式 * —— edge function，它让知识库变成一个 agent 可以从任何地方够得到的东西。“跑在知识库上的 agent”可以拆成两半，本章依次处理：协议半（agent 如何与知识库对话）和模式半（知识库如何离 agent 近到足以值得被对话）。

近期文献也让设计空间更容易命名。RepoAgent 展示的是仓库智能的**生产端**：一个分析全局仓库结构、生成文档并随时间持续更新文档的系统 {cite}`luo2024repoagentllmpoweredopensourceframework`。CGM 展示的是不同的**消费端**：由一个图集成模型解决仓库级任务，不需要显式的工具使用 agent 循环 {cite}`tao2025codegraphmodelcgm`。本章讨论的是那个三角形中的第三个位置：一个**使用工具的 agent**，把知识库当作外部记忆。Zhu 等人的综述正是这个位置不仅是产品选择、更是技术选择的原因：LLM 在对结构化上下文进行推理时，通常比按需从原始语料中提取结构更可靠 {cite}`zhu2024llmsknowledgegraphconstruction`。

## What —— 是什么

### Where this chapter sits in the design space

仓库级 AI 系统现在有三种形态，如果不显式命名就很容易混淆：

1.  **生产端自动化。** 像 RepoAgent 这样的系统生成和更新仓库文档。它们的重心是*维护循环*。
2.  **无 agent 的图感知阅读器。** 像 CGM 这样的系统在一个模型栈内消费代码图和仓库上下文。它们的重心是*阅读器模型*。
3.  **基于显式知识库的工具使用 agent。** 本章的模式把知识库放在模型外面，通过工具暴露它。它的重心是*检索接口*。

本书选择第三种形态作为架构默认，因为它让谱系、过滤策略和更新节奏都是可检查的。但它被设计为与前两种组合，而不是竞争。一个类 RepoAgent 的流水线可以保持文稿层的新鲜；一个类 CGM 的阅读器可以离线消费同样的图导出；MCP 接口是普通 IDE agent 使用的在线合约。

### 协议半：MCP + 元组

就本章而言，Model Context Protocol 是一个被精确化了的简单思路：一个 agent 可以枚举服务器对外暴露的 * 工具 * ，并用带类型的参数调用这些工具。知识库可以成为一个 MCP 服务器；一个定义得体的知识库会暴露三个工具：

| Tool | Input | Output |
|:--|:--|:--|
| `kb.search(query, k, filter)` | natural-language query, top-k, filter expression | list of $\{page\_id, score, \text{tuple}, snippet\}$ |
| `kb.read(page_id)` | stable page id | full body, frontmatter, footer |
| `kb.cite(query, k)` | natural-language query, top-k | list of $\{page\_id, \text{file:line} \text{ anchors}\}$ |

值得关注的细节是 **`kb.search` 的 filter 参数** ：它接受一个基于第 25 章文档层元组的谓词。只要经过批准内容的 agent 可以写 `filter: review_status == approved`；只要 L0–L1（代码和 ADR，不要 AI 起草的文稿）的 agent 可以写 `filter: layer in {L0, L1}`；要“已评审的文稿或代码”的 agent 写两者的合取即可。知识库 * 不必 * 信任 agent；它只要在检索阶段确定性地过滤就行。

这就是元组模型的回报。没有它，一个对 agent 开放的知识库要么把 AI 起草、未评审的内容也塞进每个回答里（危险），要么一刀切地全部排除（浪费）。有了它，决定权被推到调用方那一侧，知识库自身的责任收敛为 * 诚实地打标签 * —— 正如第 8 章所论证的，运维模型被建立起来，就是为了交付这件事。

### 模式半：在 edge 上部署的知识库

一个躲在数据中心 VPN 后面的搜索工具，不是 agent 的工具；它只是一个套着 agent 形壳子的人类工具。要对 agent 有用，知识库必须能在 agent 运行的任何地方被够得到 —— 通常是笔记本上的 IDE 进程、云上的 CI runner、另一个区域的容器化 worker。常见形态是 **edge function** ：一个小巧、无状态、可水平扩展的单元，从多个地理接入点提供 HTTP 服务。

这一模式的公共示例，上述三个 MCP 工具在其中任一平台上都可以托管：

- **Cloudflare Workers** {cite}`cloudflare_workers` —— V8 隔离，全球分发，KV 和 D1 存状态，个位数毫秒冷启动。
- **Deno Deploy** {cite}`deno_deploy` —— TypeScript 优先，全球分发，`Deno.kv` 存状态。
- **Vercel Edge Functions** {cite}`vercel_edge` —— 按区域部署，`@vercel/edge-config` 存状态。

三者支持的运维形态是一样的：来自 `book/_build/html/` 的 * 构建好的 HTML * 放进一个 asset bundle 里；一个小服务函数读取这个 bundle，用匹配的 HTML 正文来回答 `kb.read(page_id)`，并通过查询一个同样保存在 bundle 里的索引（知识库较小时）或一个远端向量存储（知识库较大时）来回答 `kb.search(...)`。知识库发布；函数重部署；agent 下一次调用时就看到新鲜答案。

这层抽象是有意做成通用的。作者的私有 skill {cite}`fanyamin_pkb_skill` 对某一具体的运行时有一份完整的部署配方，包含构建 bundle 的脚本和函数注册 manifest。这里之所以省略那些配方，是因为它们与雇主的基础设施强绑定；上面列出的三个公开同类可以作为直接替换。

## How —— 怎么做

### 一份最小的 MCP 服务器骨架

服务端伪代码 —— 同一套骨架可以套到上面三个运行时的任意一个上：

```text
function handle(request):
    tool, args = parse(request)
    switch tool:
        case "kb.search":
            seeds      = hybrid_search(args.query, k=args.k * 3)   # ch19
            expanded   = graph_expand(seeds, max_hops=1)           # ch19/ch26
            filtered   = apply_tuple_filter(expanded, args.filter)
            top        = rerank(filtered)[:args.k]
            return [
                {
                  page_id: p.id,
                  score:   p.score,
                  tuple:   p.tuple,        # ⟨L, U, R, S⟩ from Chapter 18
                  snippet: p.snippet
                }
                for p in top
            ]
        case "kb.read":
            page = lookup(args.page_id)
            return {
                body:        page.body,
                frontmatter: page.frontmatter,
                footer:      page.footer     # full PKB-metadata block
            }
        case "kb.cite":
            candidates = vector_search(args.query, k=args.k)
            anchors    = [
                {page_id: p.id, file_line: extract_anchors(p.body)}
                for p in candidates
            ]
            return anchors
```

有三点值得注意：

1.  `kb.search` 已经是**混合检索**，不是原始的向量查找。这很重要，因为一个仓库 agent 的第一次查询通常是语义性的（*"我们在哪里校验 token？"*），但它的第二次查询是结构性的（*"谁调用了这个 validator？"*）。把第 19 章的混合搜索和图扩展折叠在一个工具后面，既保持了 agent 接口的精简，又保留了仓库级的覆盖范围。
2.  `kb.read` 返回 body 的同时也返回 **footer**。一个想要负责任地引用某页的 agent 会在引用之前查看 `review_status` 和 `review_score`。知识库不决定 agent 是否应该信任这个页面；它给 agent 标签，让它自己决定。
3.  `kb.cite` 返回 `file:line` 锚点，不是文稿。这就是第 19 章的硬规则做成工具的样子。一个调了 `kb.cite` 然后捏造行号的 agent，是在违背服务器的声明 —— 而第 22 章门禁 H4 会在这个页面被 commit 回知识库时抓住它。
4.  `kb.search` 接受一个 `filter`。这个 filter 就是第 25 章 §3.2 的元组谓词。运行时来求值；agent 来指定。

### Agentless readers and tool-using agents can share one KB

CGM 在这里有用，不是因为本章应该变成一篇模型架构论文，而是因为它清晰地标记了边界。一个无 agent 的仓库求解器想要的是一张仓库图加上代码表示，以及足够的结构化上下文来在一次推理中解决一个任务 {cite}`tao2025codegraphmodelcgm`。一个使用工具的 IDE agent 想要的是同样的信息，但是增量式的：搜索、检视、引用，然后再问。一个显式的知识库同时支持两者。

这意味着知识库应该被当作一个**控制面**，而不仅仅是一个工具端点：

- 类 RepoAgent 的系统可以通过保持仓库文档的及时更新来向它写入 {cite}`luo2024repoagentllmpoweredopensourceframework`；
- 第 19 章的混合检索器可以从中导出图引导的证据；
- MCP agent 可以在线查询它；
- 图集成阅读器可以离线消费它的快照或图导出。

同样的显式元组标签、稳定 ID 和引用锚点，让这四种模式都比让每次模型调用从原始文件中推断自己的世界状态更不脆弱。

### 把构建产物暴露出去

对小型知识库而言，放置构建好的站点最简单的方式就是把它直接打进函数 bundle 自身。构建步骤是：

1.  跑 `make book-build` 生成 `book/_build/html/`。
2.  跑一个小打包脚本，读取 HTML 树并把它写到一个 TypeScript/JavaScript 模块中，以 base64 编码的字节存储（或在支持的运行时上，以原生 asset 文件存储）。
3.  带着打包好的模块部署函数。

对更大型的知识库 —— 比方说构建输出超过几 MB —— asset bundle 会超出 edge 运行时的单函数体积上限。可选方案有：(a) 把 asset 放到一个函数可读的对象存储里（Cloudflare 上的 R2，其他平台上的 S3 兼容存储）；(b) 把知识库拆成多个函数（每个 Part 一个）；(c) 把 HTML 放到一个独立的 CDN，让函数只负责提供搜索索引和 MCP shim。(c) 通常是正确答案：HTML 本来就已经是一个静态站点，CDN 原生就能伺候它。

### 把元组 filter 当作发布门禁

第 25 章里一个微妙的点在此变得可操作。一个习惯性传 `filter: review_status == approved` 的 agent * 永远看不到 * `pending` 状态的页面。这正是我们要的效果。但是一个一直停留在 `pending` 的页面 —— 因为始终没人评审 —— 对 agent 来说就等于不存在，哪怕它其实很有用。第 22 章的软门禁 S2（footer 陈旧）已经在向人类发出这种页面的警告；第 23 章的 L3 纪律会在 L2 更新搞不定时把它们交到人手上。agent 这个面并没有引入新问题，它只是让老问题的成本变得显性。Agent 看不见的知识库，是它自己挣来的“看不见”。

## Example —— 范例

一个 agent、一个 IDE、一个知识库的具体“一日作息”：

```text
08:15  developer types /ask "how does the ingest pipeline handle CSVs?"
         IDE agent calls  kb.search(q=..., k=5, filter="review_status==approved")
         KB responds      [
                            {page_id: "ch06", score: 0.83, tuple: ⟨L2, human, approved, 4⟩, ...},
                            {page_id: "ch07", score: 0.79, tuple: ⟨L2, human, approved, 4⟩, ...},
                            {page_id: "ingest-runbook", score: 0.76, tuple: ⟨L2, ai+human, approved, 3⟩, ...},
                            ...
                          ]
         IDE agent calls  kb.read("ch06")
         KB responds      { body: ..., frontmatter: ..., footer: {review_status: approved, ...} }
         IDE agent answers, quoting ch06 ¶3 and linking ingest-runbook §4.

08:17  developer asks follow-up "which file actually parses the CSV?"
         IDE agent calls  kb.cite(q="CSV parsing", k=3)
         KB responds      [
                            {page_id: "ch06",  file_line: "internal/pipeline/csv_loader.go:42"},
                            ...
                          ]
         IDE agent opens that file:line in the editor. No hallucination;
         every anchor was resolved by the KB, not guessed.
```

这一连串动作中，没有一处需要 agent * 信任 * 知识库。每一个 filter 都是声明式的；每一次 cite 都靠构建期索引来解析；每一个 agent 本来可能凭空捏造的决定，都被服务器这一端给出的机械答案约束住了。

同一个知识库也可以服务于非交互式的仓库任务。一个问 *"这次接口变更影响了哪些模块？"* 的批处理任务不需要 IDE 循环；它可以调同一个 `kb.search` 入口，或者直接消费导出的图邻域。重要的不是消费者是否自称"agent"。重要的是所有消费者共享同一套被持续维护的底座，而不是各自从零开始重建仓库上下文。

## Conclusion —— 小结

“跑在知识库上的 agent” 不是叠加在文稿层和代码层之上的又一新层；它是运维模型本身早已蕴含的那种自然 API 形态。一个拥有“诚实层级元组、稳定页面标识、机械解析的引用”的知识库，恰好具备 agent 需要的那三个工具。协议（MCP）薄；模式（edge function）是现成的；信任模型就是第 25 章和第 22 章已经建好的那个。

剩下的 —— 第 30 章的 token 预算、第 31 章的信任与红队演练、第 32 章的展望 —— 都是“一个 agent 可调用的知识库”新近变得紧迫的治理问题。只被人类阅读的知识库可以承受一个马虎的 footer。一个被 agent 大规模调用的知识库承受不起。

## 参考文献

```{bibliography}
:filter: keywords % "hybrid" or keywords % "agents" or keywords % "mcp" or keywords % "self-citation"
```
