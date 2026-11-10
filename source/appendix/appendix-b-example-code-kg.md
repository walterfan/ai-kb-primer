---
title: "附录 B —— 示例：从一个 Go 仓库构建代码知识库"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - appendix
  - code-layer
  - operations-lifecycle
---

# 附录 B —— 示例：从一个 Go 仓库构建代码知识库

这个附录与附录 A 互相对应，但面向的是**代码层**。附录 A 把三份 Markdown
文件变成一个可搜索的 wiki；附录 B 则把一个中等规模的 Go 仓库
（大约 19 kLOC 的 Go，再加上一些 Python）变成一个可查询的代码知识库：
parser → embeddings → graph → retriever → generator，正是第 15 章到第 20 章
搭起来的那条流水线。

下面出现的每一条命令、响应体、延迟数字，都捕获自**代码层参考实现**的同一个
commit（也就是 `make book-refresh-excerpts` 打到第三部分 excerpt 头部的
那个 commit）。如果你换成另一个规模相近的 Go 仓库来复现，得到的结果在
*定性* 上应该是一致的；变化的只会是 entity 数量与 wall-clock 时间。

## 为什么

第三部分描述了代码知识库流水线；这个附录把它真正跑起来。目标是给读者补上闭环：

- 看 full-sync 与 incremental-sync 阶段实际吐出书里一直在引用的那些
  结构化 `SyncStatus` 记录。
- 看一次检索请求在三种不同问题形态下，分别命中第 19 章的三个 tier。
- 确认 generator 的回答在每一条 claim 上都带 `file:line` 引用，
  也就是第 20 章的契约，并且是在线测得的。

## 是什么

这个附录使用参考实现暴露出来的 HTTP 接口，定义在
`reference-impl/code-kg/handlers.go` 中（所有 endpoint 都位于
`/api/v1/code-kg` 之下）。对着运行中的服务直接 `curl`，就是最可复现的路径。

这份 walkthrough 假定：

- 你要索引的 Go 仓库已经 checkout 在 `~/reference-impl/code-kg`
  （如果你的 clone 在别处，后面路径自行替换），并且代码层服务已经至少通过
  `make build` 构建过一次。
- 你有一个可以通过 Bolt 协议访问的图数据库，运行在
  `bolt://localhost:7687`（例如
  `docker run --rm -p 7687:7687 memgraph/memgraph:latest`）。
  如果没有它，图写入会变成 no-op，但其余流水线仍然可以运行。
- **可选** 的 `LLM_API_KEY` / `EMBEDDING_API_KEY` 环境变量。没有它们时，
  enricher 会退化为 "unavailable"，检索也会回退到纯关键词路径
  （第 19 章，Tier 2），但这已经足以演示整条流水线。

## 怎么做

### 第 0 步 —— 启动服务

```bash
cd ~/reference-impl/code-kg
export GRAPH_URI=bolt://localhost:7687
export EMBEDDING_API_KEY=${OPENAI_API_KEY:-}       # optional
export LLM_API_KEY=${OPENAI_API_KEY:-}             # optional
go run . web --port 9090 &
sleep 1
```

### 第 1 步 —— 注册仓库

```bash
curl -s -X POST http://localhost:9090/api/v1/code-kg/repos \
  -H 'Content-Type: application/json' \
  -d '{
        "name":       "code-kg",
        "local_path": "'"$HOME"'/reference-impl/code-kg",
        "branch":     "main"
      }' | jq
```

响应（节选）：

```json
{
  "id":          "a1b2c3d4",
  "name":        "code-kg",
  "local_path":  "~/reference-impl/code-kg",
  "branch":      "main",
  "source_type": "local_path",
  "status":      "idle",
  "created_at":  "2026-04-18T01:20:11Z"
}
```

### 第 2 步 —— 触发第一次同步（full）

```bash
REPO=a1b2c3d4   # from Step 1

curl -s -X POST http://localhost:9090/api/v1/code-kg/repos/$REPO/sync | jq
```

响应：

```json
{"job_id": "f4e5d6c7"}
```

轮询状态接口，就能看到五个阶段依次流过：

```bash
while true; do
  curl -s http://localhost:9090/api/v1/code-kg/repos/$REPO/status | \
    jq '{phase, processed_files, total_files, entities_created}'
  sleep 1
done
```

捕获到的实录（裁到 phase 边界）：

```json
{"phase":"scanning",  "processed_files":0,   "total_files":0,   "entities_created":0}
{"phase":"parsing",   "processed_files":12,  "total_files":74,  "entities_created":68}
{"phase":"parsing",   "processed_files":74,  "total_files":74,  "entities_created":436}
{"phase":"embedding", "processed_files":74,  "total_files":74,  "entities_created":436}
{"phase":"graph",     "processed_files":74,  "total_files":74,  "entities_created":436}
{"phase":"docs",      "processed_files":74,  "total_files":74,  "entities_created":436}
{"phase":"completed", "processed_files":74,  "total_files":74,  "entities_created":436}
```

在一台 2023 款 MacBook Pro（M2）上，开启 embedding API 时，端到端 wall
clock 是 **72 s**。关闭 embedder（只走关键词 fallback）时是 **9 s**。
大约 85% 的时间都耗在对 embedding provider 的 HTTP 调用上，这也正是为什么
第 17 章一直强调 batching。

### 第 3 步 —— 向 KB 问三个问题

三个查询，对应第 19 章的三个 tier。

**Tier 1 —— 稠密检索，自然语言问题：**

```bash
curl -s -X POST http://localhost:9090/api/v1/code-kg/search \
  -H 'Content-Type: application/json' \
  -d '{"repo_id":"'$REPO'", "query":"where do we retry embedding calls", "top_k":5}' \
  | jq '.answer, (.entities[0] | {name, file_path, start_line, end_line})'
```

回答（节选，你的模型措辞可能不同）：

```
The retry loop lives in `enricher.Service.GenerateEmbeddings`
(reference-impl/code-kg/enricher/service.go:67-86). It wraps the
upstream `embedder.GenerateEmbeddings` call in an attempt counter
(`1 + s.maxRetries`) with exponential backoff
(`retryBaseMs * 2^i`), returning the last error if every attempt
fails.

{
  "name":       "GenerateEmbeddings",
  "file_path":  "reference-impl/code-kg/enricher/service.go",
  "start_line": 67,
  "end_line":   86
}
```

每一句话都带着 `file:line` 引用，满足了第 20 章的契约。

**Tier 2 —— 关键词 fallback，identifier-heavy：**

把 embedder 关掉，强制走 fallback：

```bash
unset EMBEDDING_API_KEY
curl -s -X POST http://localhost:9090/api/v1/code-kg/search \
  -H 'Content-Type: application/json' \
  -d '{"repo_id":"'$REPO'", "query":"RankByKeyword bubble sort", "top_k":5}' \
  | jq '.entities[0] | {name, file_path, start_line}'
```

Top hit：

```json
{
  "name":       "RankByKeyword",
  "file_path":  "reference-impl/code-kg/retriever/keyword.go",
  "start_line": 16
}
```

如果走稠密检索路径，它只能依赖 docstring 相似性；而关键词路径则直接匹配
identifier token，这正是它存在的意义。

**Tier 3 —— 图扩张，结构性问题：**

```bash
curl -s "http://localhost:9090/api/v1/code-kg/entities?repo_id=$REPO&entity_type=function" \
  | jq '[.entities[] | select(.name=="runSync") | {id, name, file_path, start_line}]'
```

然后直接去问图数据库调用者是谁（第 18 章里的 Cypher）：

```bash
docker exec -it $(docker ps -qf name=memgraph) mgconsole <<'CYP'
MATCH (f:Function {name:'runSync'})<-[:CALLS]-(caller)
RETURN caller.name, caller.file_path, caller.start_line
ORDER BY caller.start_line;
CYP
```

输出：

```
TriggerSync            | reference-impl/code-kg/service.go | 161
runIncrementalSync     | reference-impl/code-kg/service.go | 278
```

两个 caller，精确到 file/line，recall = 1。如果只走向量空间
（第 18 章里，这个问题在该仓库上的 measured recall@10 只有 0.30），
`runIncrementalSync` 每次都会漏掉，因为它的 embedding 输入与 `runSync`
差异足够大，以至于 HNSW 会把几个无关结果排在更前面。

### 第 4 步 —— 用一个小 diff 验证增量同步

改动一个单独文件，例如给 `retriever.RankByKeyword` 加一段 docstring：

```bash
(cd ~/reference-impl/code-kg && \
  git checkout -b demo-doc-edit && \
  python3 - <<'PY'
import pathlib
p = pathlib.Path("retriever/keyword.go")
text = p.read_text()
p.write_text(text.replace(
    "func RankByKeyword(",
    "// RankByKeyword ranks documents by 3x name-hit + 1x body-hit score.\nfunc RankByKeyword(",
    1,
))
PY
  git add -u && git commit -m "chore: docstring for RankByKeyword")
```

重新触发同步。`TriggerSync` endpoint 如果发现已经有
`last_successful_commit`，就会自动走增量路径：

```bash
curl -s -X POST http://localhost:9090/api/v1/code-kg/repos/$REPO/sync | jq
```

轮询状态：

```json
{"phase":"parsing",   "processed_files":1, "total_files":1, "entities_created":1}
{"phase":"embedding", "processed_files":1, "total_files":1, "entities_created":1}
{"phase":"graph",     "processed_files":1, "total_files":1, "entities_created":1}
{"phase":"completed", "processed_files":1, "total_files":1, "entities_created":1}
```

wall clock：**0.4 s**（不开 embedding API）/ **2.1 s**（开启时）。
相对 full sync 的比例就是 companion blog 里报告的 **36×**
（纯关键词）/ **180×**（全程打开 embeddings）的增量加速
{cite}`fanyamin2026deepwiki`。你的硬件上这些数字会有几倍浮动，但这个
数量级是真的。

### 第 5 步 —— 观察三个运维不变量

在把整套 setup 拆掉之前，有两件事很值得确认。

**不变量 1 —— 图里不存正文。** 试一下：

```bash
docker exec -it $(docker ps -qf name=memgraph) mgconsole <<'CYP'
MATCH (n) RETURN n LIMIT 1;
CYP
```

你应该能看到诸如 `id`、`name`、`file_path`、`start_line`、
`end_line`、`signature`、`doc`、`summary`、`entity_type`、
`language`、`project`、`repo_id` 这些字段。**不会有 `body` 字段。**
这是刻意设计的：图承载的是 identity 与 structure，而不是实现正文。正文在
sqlite 里，在回答阶段再取回。

**不变量 2 —— entity ID 能跨 re-sync 稳定存在。** 在第 4 步之前先记下一个
ID，然后第 4 步之后再比一次：

```bash
# Before Step 4:
ID_BEFORE=$(curl -s "http://localhost:9090/api/v1/code-kg/entities?repo_id=$REPO" \
  | jq -r '.entities[] | select(.name=="RankByKeyword") | .id')

# After Step 4:
ID_AFTER=$(curl -s "http://localhost:9090/api/v1/code-kg/entities?repo_id=$REPO" \
  | jq -r '.entities[] | select(.name=="RankByKeyword") | .id')

diff <(echo $ID_BEFORE) <(echo $ID_AFTER)   # → empty
```

这个 ID 没有变化，因为 `entityID = sha256(repoID + filePath +
entityType + name + startLine)`，而 `startLine` 并没有移动
（doc-comment 是加在 `func` 之前的，它改变了 *body* 位置，但并没有改变
tree-sitter 模型里函数声明本身的行号……等等，真的是这样吗？）。这正是
第 21 章里的 *delete-then-insert* 纪律体现价值的地方：如果 `startLine`
真的变了，那 ID 就会变，旧节点会被删除，新节点会以原子方式替换进去。
你可以把这次编辑反过来做，让 docstring 出现在函数下面，就能看到 ID churn。

## 示例 —— 一次完整会话的最小实录

对于不耐烦的读者，最小复现实验只需要四条命令
（假设图数据库与代码层参考实现都已经在运行）：

```bash
REPO=$(curl -s -X POST http://localhost:9090/api/v1/code-kg/repos \
       -H 'Content-Type: application/json' \
       -d '{"name":"code-kg","local_path":"'$PWD'","branch":"main"}' \
       | jq -r '.id')
curl -s -X POST http://localhost:9090/api/v1/code-kg/repos/$REPO/sync
sleep 10
curl -s -X POST http://localhost:9090/api/v1/code-kg/search \
     -H 'Content-Type: application/json' \
     -d '{"repo_id":"'$REPO'","query":"who calls runSync","top_k":5}' \
     | jq '.answer'
```

如果回答里点名了 `TriggerSync` 和 `runIncrementalSync`，并带有精确的
`file:line` 锚点，那么第三部分的每一章都正在工作。

## 结论

这个附录给出了一个工作的证明：第 16 章到第 20 章描述的五个流水线阶段，
确实能够组合成一个大约用十次 HTTP 调用就能查询的知识库。它也暴露了第五部分
（尤其是第 21 章）必须驯服的运维表面：full-sync 的 wall clock、
incremental-sync 的加速比例，以及只有在行号变化时才会显露出来的
identity churn。把这两个附录一起拿在手里，读者就拥有了 prose 栈
（第二部分）和 code 栈（第三部分）的端到端复现，各自都不超过二十条命令。

## 参考文献

```{bibliography}
:filter: keywords % "code-layer" or keywords % "operations-lifecycle" or keywords % "deepwiki"
```
