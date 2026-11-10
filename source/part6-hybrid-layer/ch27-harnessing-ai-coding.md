---
title: "第 27 章 —— 用知识库驾驭 AI 编码"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - hybrid
  - agents
  - mcp
---

# 第 27 章 —— 用知识库驾驭 AI 编码

## Why —— 为什么

AI 编码助手 —— Cursor、GitHub Copilot、Windsurf、以及数不清的 IDE 插件 —— 已经从自动补全进化到了仓库级对话。它们能读文件、跑命令、改代码。它们**不能**做的事情同样重要：它们不知道团队为什么选了这个设计、三个月前的 ADR 做了什么取舍、那份 runbook 的第 4 节包含了凌晨 3 点的唯一正确操作步骤。

没有知识库的 AI 编码助手，是一个**没有记忆的操作员**。它对当前文件很擅长；对跨文件结构还行；对*为什么代码是这样的* 一无所知 —— 除非你在 prompt 里手动粘贴相关上下文。这就是每天都在发生的事：工程师花 5 分钟给 AI 粘 context，AI 花 5 秒钟回答。粘贴动作本身就是在做人工检索。

本章的命题是：**一个有结构的知识库可以取代那个人工检索步骤**。AI 编码助手不再需要工程师手动选择哪些文件和文档与当前任务相关 —— 知识库通过第 26 章的上下文组装自动提供那个上下文包。 CodePlan 在 FSE 2024 上展示了这种场景的完整框架：把仓库级变更建模为"影响分析 → 规划 → 迭代执行"的三阶段流水线 {cite}`bairi2024codeplan`。知识库已经维护着的代码依赖图（第 18 章）和跨层链接（第 24 章），恰好是 CodePlan 所需的影响分析输入 —— 不需要从零构建。代价是需要维护知识库（已有答案：第五部分的运维纪律）；收益是 AI 编码的每一次交互都建立在可验证的组织知识之上，而不是模型的参数记忆。

## What —— 是什么

本章不讨论如何构建一个 AI 编码助手 —— 那是产品问题。本章讨论的是：**一个已经存在的 AI 编码助手，如何通过知识库获得以下三种它原本不具备的能力。**

### 能力 1：设计理据感知

AI 助手能读函数签名，但不知道签名背后的设计决策。一个知识库暴露的 `kb.search(query, filter="layer in {L0, L1}")` 可以只返回代码和 ADR 层的内容 —— 恰好是设计理据住的地方。助手在回答 *"为什么用 mutex 而不是 channel？"* 时，可以引用 ADR 而不是猜测。

### 能力 2：引用可校验

没有知识库的助手回答中的代码引用 —— *"见 `auth.go:42`"* —— 可能是幻觉。有知识库的助手调用 `kb.cite()` 得到的每一条 `file:line` 锚点都在构建期经过了解析。第 22 章门禁 H4 保证了只有可解析的引用才能发布。助手拿到的引用不需要人去二次验证。

### 能力 3：权威性分层

助手从知识库获取的每个结果都带有第 25 章的元组 $\langle L, U, R, S \rangle$。助手可以只使用 `review_status == approved` 的页面来支撑回答，把 `pending` 的页面降级为"仅供参考"。这在安全敏感场景中尤其重要 —— 一个安全审计问题的回答不应该建立在 AI 起草的、尚未评审的页面之上。

## How —— 怎么做

### 交互模式：三种典型的 KB 增强编码场景

**场景 A：上下文注入式编码。** 工程师在 IDE 中编辑一个文件时，AI 助手在后台调用 `kb.search()` 获取与当前文件相关的文稿页面，注入到系统提示词（system prompt）中。工程师不需要手动选择上下文。

```text
工程师打开 internal/pipeline/csv_loader.go
AI 助手后台执行:
  seeds = kb.search("csv_loader.go pipeline ingest", k=3,
                     filter="review_status==approved")
  → 命中: ch06 (分类流水线), ingest-runbook (操作手册), ADR-0009 (CSV 解析策略)
  
  将三份文档的摘要注入 system prompt
  
工程师输入: "这个 loader 支持 TSV 吗？"
AI 助手回答时引用了 ADR-0009 §3 的 "仅支持 RFC 4180 格式" 决策。
```

**场景 B：代码审查辅助。** AI 助手对一个 merge request 做自动审查时，通过 `kb.search()` 查找变更涉及的代码实体是否有关联的 ADR 或架构文档。如果有，审查意见中可以引用相关决策；如果 ADR 已过期或被变更违反，审查意见中可以标记。

```text
MR 修改了 pkg/auth/token_validator.go 中的过期逻辑
AI 审查器执行:
  cite_result = kb.cite("token_validator expiry", k=3)
  → 命中: ADR-0021 (Token grace period for clock skew)
  
  读取 ADR-0021，发现 grace period 被设定为 30 秒
  MR 将 grace period 改为 0（即不接受过期 token）

AI 审查器标记:
  "⚠️ 此变更与 ADR-0021 的设计决策冲突。
   ADR-0021 规定 30 秒的 grace period 以容忍时钟偏差。
   如需更改此策略，请先更新 ADR。"
```

**场景 C：多文件重构导航。** 工程师要求 AI 助手重构一个跨多个文件的功能。助手通过 `kb.search()` 加上图扩展（第 26 章），找到所有受影响的文件和相关文档。

```text
工程师: "重构 ingest pipeline，把 CSV 和 JSON loader 合并为统一接口"
AI 助手执行:
  seeds = kb.search("ingest pipeline csv json loader", k=5)
  → 种子: csv_loader.go, json_loader.go, pipeline.go, ingest-runbook, ADR-0009

  graph_expand(seeds, relations=["CALLS", "IMPLEMENTS", "DOCUMENTS"])
  → 扩展: loader_interface.go, pipeline_test.go, config.yaml

AI 助手生成重构计划，列出所有受影响文件，
引用 ADR-0009 作为当前设计的约束条件。
```

### 集成架构

AI 编码助手与知识库的集成不需要修改助手的内部实现。集成发生在**工具层** —— 助手调用 MCP 工具，知识库作为 MCP 服务器响应。

```{mermaid}
flowchart LR
    subgraph ide ["IDE / CLI"]
        DEV["工程师"]
        AGENT["AI 编码助手"]
    end
    subgraph kb_server ["知识库 MCP 服务器"]
        SEARCH["kb.search"]
        READ["kb.read"]
        CITE["kb.cite"]
    end
    subgraph kb_stack ["知识库存储栈"]
        VEC["向量索引"]
        GRAPH["代码图 + 跨层边"]
        PROSE["文稿页面 + frontmatter"]
    end
    DEV -->|"提问 / 指令"| AGENT
    AGENT -->|"MCP 调用"| SEARCH
    AGENT -->|"MCP 调用"| READ
    AGENT -->|"MCP 调用"| CITE
    SEARCH --> VEC
    SEARCH --> GRAPH
    READ --> PROSE
    CITE --> VEC
    CITE --> GRAPH
```

这个架构的关键性质是**助手不需要知道知识库的内部结构**。它只需要知道三个工具的输入输出格式。这意味着同一个知识库可以同时服务 Cursor、Copilot、或任何支持 MCP（或其他函数调用协议）的助手。

### 上下文预算纪律

AI 编码助手的上下文窗口有限。即使是 100k+ token 的模型，有效利用也需要纪律。知识库提供的上下文包不应该"把能找到的全塞进去" —— 那只是把信息过载的问题从工程师转移到了模型。

来自第 26 章的上下文组装已经有预算约束。面对 AI 编码场景，补充三条规则：

1. **代码优先。** 当问题涉及代码时，上下文包的 token 预算中至少 60% 分配给代码片段（L0），剩余分配给文稿（L1–L4）。原因：AI 编码助手的核心任务是写代码，文稿是辅助理据，不应该挤占代码的上下文空间。
2. **评审状态作为断路器。** 如果上下文窗口紧张，优先丢弃 `review_status: pending` 的页面。已评审的少量页面比未评审的大量页面更有用 —— AI 编码场景尤其如此，因为一条错误的文稿引用可能导致助手写出违反设计决策的代码。
3. **陈旧度惩罚。** `last_updated` 超过 90 天的文稿页面，在上下文预算竞争中降权。代码层内容（L0）不受此规则影响 —— 代码的陈旧度由 Git 本身管理。 此外，Repoformer 的选择性检索思路在此同样适用：并非每次 AI 编码交互都需要从知识库检索 —— 纯局部代码逻辑（如 `return a + b`）不需要额外上下文 {cite}`li2024repoformer`。一个"是否需要知识库上下文"的轻量判断器，可以进一步降低延迟和噪声。

### Cursor Rules 与知识库的互补

许多团队使用 Cursor Rules（`.cursor/rules/` 下的规则文件）来约束 AI 编码助手的行为。Rules 和知识库是互补的，不是竞争的：

| 维度 | Cursor Rules | 知识库 |
|:--|:--|:--|
| 内容类型 | 编码规范、代码风格、禁止的模式 | 设计理据、架构决策、操作手册 |
| 更新频率 | 低 —— 规范变化缓慢 | 高 —— 文档随代码同步更新 |
| 上下文注入方式 | 自动注入每次会话 | 按需检索 —— 只在相关时注入 |
| 权威性模型 | 二值 —— 规则要么生效要么不生效 | 元组 $\langle L, U, R, S \rangle$ —— 多层级 |

最佳实践是：**把"总是适用"的约束放在 Rules 里，把"有时相关"的知识放在知识库里。** Rules 是常量上下文，知识库是变量上下文。

## Example —— 范例

一个开发者在 Cursor 中的 30 分钟工作片段，展示知识库如何改变 AI 编码体验：

```text
09:00  开发者打开 pkg/auth/token_validator.go
       Cursor 后台: kb.search("token_validator auth", k=3)
       → 注入上下文: ADR-0021 (grace period), auth-runbook §2, ch20 (安全审计)

09:03  开发者: "@agent 这个函数的 grace period 是怎么确定的？"
       Cursor: "30 秒的 grace period 在 ADR-0021 中确定，
               理由是服务器间最大时钟偏差的经验值。
               见 ADR-0021 §3。"
       → 引用来自 kb.read("adr-0021")，不是模型记忆。

09:05  开发者: "@agent 修改这个值需要注意什么？"
       Cursor: "修改 grace period 需要：
               1. 更新 ADR-0021 或写一份新的变更提案
               2. 同步更新 ingest-runbook §4 中的超时配置
               3. 通知安全团队（ADR-0021 标记为安全相关）
               见 kb.cite 结果: token_validator.go:89,
               adr-0021.md:§3, ingest-runbook.md:§4。"
       → 三个引用全部来自知识库，每个都可解析。

09:10  开发者修改 grace period 为 15 秒，提交 MR
       AI 审查器: kb.search("token_validator grace period", k=3)
       → 检测到与 ADR-0021 的冲突
       → 审查意见: "建议先更新 ADR-0021 再合并此变更。"

09:15  开发者写了一份 OpenSpec 变更提案
       → 第 31 章流水线将其以 review_status: pending 摄入知识库
       → 评审通过后，ADR-0021 更新，知识库增量同步
       → 下次 AI 助手读到的 ADR 已包含新的 grace period 值
```

这个工作流与没有知识库的工作流的区别不是 AI 助手变"聪明"了。它是：**每一步的上下文都来自有出处、有评审状态的组织知识，而不是模型的参数权重或工程师的手动粘贴。**

## 20.5   Codex 集成：后台 Agent 模式

Codex 是 OpenAI 推出的 CLI 编码 Agent。与 Cursor 等编辑器内置助手不同，Codex 在一个**沙箱环境**中自主运行：它接收一条高层任务描述，然后独立完成代码阅读、修改、测试和解释，最终返回一组变更和自然语言总结。这种"后台 Agent"模式特别适合批量重构、迁移、和跨文件修改——工程师不需要在 IDE 中逐步引导。

### 集成模式：知识库作为 Codex 的上下文源

Codex 的核心挑战是**任务启动时的上下文质量**。一个没有额外上下文的 Codex 任务只能看到仓库中的代码——它不知道 ADR、runbook、或架构决策。知识库恰好补了这个缺口。

集成分三个阶段：

1. **任务前（Pre-task）：查询知识库，注入上下文。** 在提交 Codex 任务之前，脚本调用 `kb.search()` 获取与任务相关的文稿页面，将结果写入 `.codex/instructions.md` 或直接拼接到任务 prompt 中。这是最简单、最可靠的集成方式——不需要 Codex 本身支持外部工具调用。

2. **任务中（During-task）：通过 MCP 桥接实时访问。** 如果 Codex 配置了 MCP 支持，它可以在执行过程中调用 `kb.search`、`kb.read` 来获取额外上下文。这需要在沙箱中暴露一个 MCP 端点——目前需要额外的桥接配置。

3. **任务后（Post-task）：输出回流知识库。** Codex 完成任务后产生的变更说明和设计解释，可以作为 L3（AI 起草层）候选内容自动摄入知识库。第 31 章的 AI-authored 摄入管道可以直接对接这个输出。

### `.codex/` 目录约定

Codex 使用 `.codex/instructions.md` 作为项目级的系统提示词。这个文件相当于 Codex 的"项目记忆"——每次任务启动时自动加载。

```markdown
# .codex/instructions.md — 项目级 Codex 指令

## 项目概览
本项目是一个支付网关服务，核心模块包括 payment、settlement、reconciliation。

## 知识库集成
- 在进行架构变更前，请查阅 ADR 目录 (docs/adr/) 中的相关决策
- 设计理据查询: 使用 kb.search(query, filter="layer in {L0, L1}")
- 所有变更必须符合 docs/adr/ADR-0015 中的错误处理策略

## 编码规范
- 错误处理: 使用 pkg/errors 包，不使用裸 error
- 日志: 结构化日志，必须包含 trace_id
- 测试: 每个公开函数必须有表驱动测试

## 知识库上下文（由 CI 脚本自动注入）
<!-- KB_CONTEXT_START -->
<!-- 此区域由 pre-task 脚本自动填充，每次任务前更新 -->
<!-- KB_CONTEXT_END -->
```

### Codex 优势与局限

**优势：自主多文件变更 + KB 背景支撑。** Codex 擅长的是"给我一个任务，我自己搞定"的场景。当 `instructions.md` 中注入了知识库上下文（ADR、架构文档、代码规范），Codex 的自主决策质量显著提升——它不再需要猜测设计意图。

**局限：沙箱隔离意味着默认无实时 KB 访问。** Codex 的沙箱设计是安全优势，但也意味着它不能像 Cursor 那样在每次对话轮次中实时调用 `kb.search`。除非配置了 MCP 桥接，否则知识库上下文只能在任务前一次性注入。对于长时间运行的复杂任务，这可能导致上下文不够新鲜。

### 范例：带 KB 支撑的重构任务

```text
任务: "重构 payment 模块，将同步的支付确认流程改为异步事件驱动"

Pre-task KB 查询:
  kb.search("payment confirmation async event", k=5,
            filter="review_status==approved")
  → 命中: ADR-0012 (事件驱动架构决策), ADR-0015 (错误处理策略),
          payment-runbook §3 (支付确认超时处理),
          ch16 (消息队列集成模式), settlement-design.md

注入到 .codex/instructions.md 的 KB_CONTEXT 区域

Codex 执行:
  1. 读取 payment/ 目录下所有文件
  2. 参考 ADR-0012 确定事件架构（选择 CloudEvents 格式）
  3. 参考 ADR-0015 确定错误处理（异步失败走死信队列）
  4. 修改 12 个文件，新增 3 个文件，删除 2 个废弃文件
  5. 生成变更说明 + 设计解释

Post-task:
  Codex 的变更说明作为 L3 候选页面摄入知识库
  → 标记 review_status: pending，等待人工评审
```

## 20.6   Claude Code 集成：Agent 工具模式

Claude Code 是 Anthropic 推出的 CLI 编码 Agent，核心特点是**原生工具调用能力**。与 Codex 的沙箱自主模式不同，Claude Code 在终端中以交互方式运行，并且可以直接调用外部工具——包括 MCP 服务器提供的知识库工具。这意味着 Claude Code 不需要"预注入"上下文：它可以在编码过程中**实时查询知识库**。

### 集成模式：知识库作为 MCP 工具服务器

Claude Code 的 MCP 支持是原生的：在项目配置中声明 MCP 服务器，Claude Code 就可以像调用文件系统工具一样调用 `kb.search`、`kb.read`、`kb.cite`。这是三种编码助手中最直接的知识库集成方式。

集成步骤：

1. **配置 MCP 服务器。** 在项目根目录的 `.mcp.json`（或全局的 `claude_desktop_config.json`）中注册知识库 MCP 服务器。Claude Code 启动时自动加载这些配置。

2. **原生工具调用。** Claude Code 在编码过程中可以随时调用 `kb.search`、`kb.read`、`kb.cite`——不需要工程师手动触发。当 Claude Code 遇到一个它不确定的架构问题时，它会自动查询知识库。

3. **CLAUDE.md 引用 KB 约定。** `.claude/CLAUDE.md` 是 Claude Code 的项目上下文文件，相当于 Cursor 的 `.cursorrules`。在其中写明知识库的使用约定，Claude Code 会遵循。

### `.claude/` 目录约定

```markdown
# .claude/CLAUDE.md — 项目级 Claude Code 上下文

## 项目概览
支付网关服务。核心模块: payment, settlement, reconciliation。

## 知识库使用规则
1. **架构变更前必须查询 KB。** 任何涉及模块边界、接口定义、错误处理策略的
   变更，必须先调用 kb.search 查找相关 ADR。如果存在冲突的 ADR，
   停止修改并报告冲突。
2. **引用必须可验证。** 回答中引用的文档必须来自 kb.cite 的返回结果。
   不要凭记忆引用文件路径或行号。
3. **优先使用已评审内容。** 调用 kb.search 时始终加上
   filter="review_status==approved"。pending 状态的内容仅供参考。

## 编码规范
- 错误处理: pkg/errors, 不使用裸 error
- 日志: 结构化, 必须包含 trace_id
- 测试: 表驱动测试, 覆盖率 > 80%

## 自定义命令
- /kb-check: 查询 KB 中与当前文件相关的所有 ADR 和 runbook
- /kb-conflict: 检查当前修改是否与任何 ADR 冲突
```

### MCP 配置范例

```json
// .mcp.json — 项目级 MCP 服务器配置
{
  "mcpServers": {
    "project-kb": {
      "command": "npx",
      "args": ["-y", "@anthropic/kb-mcp-server"],
      "env": {
        "KB_ENDPOINT": "http://localhost:8741",
        "KB_INDEX_PATH": "./kb-index",
        "KB_DEFAULT_FILTER": "review_status==approved"
      }
    }
  }
}
```

### Claude Code 优势与局限

**优势：原生 MCP 支持 = 实时 KB 访问。** Claude Code 不需要预注入或桥接——它在每一步决策中都可以查询知识库。这对架构感知的编码任务尤其重要：当 Claude Code 修改一个接口时，它可以即时查询相关 ADR，确保变更不违反设计决策。

**局限：上下文窗口限制要求 KB 结果预过滤。** 虽然 Claude Code 可以实时查询，但每次查询返回的结果仍然受上下文窗口约束。知识库的元组 $\langle L, U, R, S \rangle$ 在此发挥关键作用——Claude Code 应该只请求高权威性、已评审的结果，而不是"把所有相关内容都拿来"。CLAUDE.md 中的使用规则正是为此设计的。

### 范例：带实时 KB 查询的 API 开发

```text
任务: "添加一个新的 /api/v2/refund 端点"

Claude Code 在终端中交互式执行:

Step 1: 工程师输入任务
  > claude "添加 /api/v2/refund 端点，支持部分退款"

Step 2: Claude Code 自动查询 KB
  → kb.search("api endpoint refund convention", k=3,
              filter="review_status==approved")
  → 命中: ADR-0018 (API 版本策略: v2 使用 OpenAPI 3.1),
          api-style-guide.md (响应格式、错误码规范),
          ADR-0023 (退款需要幂等键)

Step 3: Claude Code 读取 ADR 详情
  → kb.read("adr-0018") → 得到 OpenAPI 3.1 的具体要求
  → kb.read("adr-0023") → 得到幂等键的实现规范

Step 4: Claude Code 生成代码
  - 创建 pkg/api/v2/refund_handler.go（符合 API style guide）
  - 创建 pkg/api/v2/refund_handler_test.go（表驱动测试）
  - 更新 pkg/api/router.go（注册新路由）
  - 创建 api/openapi/v2/refund.yaml（OpenAPI 3.1 spec）

Step 5: Claude Code 验证引用
  → kb.cite("refund handler idempotency", k=2)
  → 确认实现与 ADR-0023 一致

输出: 4 个新文件 + 1 个修改文件，全部引用可追溯。
```

## 20.7   三工具对比

Cursor、Codex、Claude Code 代表了三种不同的 AI 编码助手与知识库的集成范式。选择哪种工具（或组合使用）取决于任务类型和团队工作流。

| 维度 | Cursor | Codex | Claude Code |
|:--|:--|:--|:--|
| **KB 访问方式** | MCP 工具调用 | 预注入或 MCP 桥接 | 原生 MCP 工具调用 |
| **执行环境** | 编辑器内 | 沙箱（自主） | 终端（交互式） |
| **多文件支持** | 是 | 是（强） | 是 |
| **上下文配置** | `.cursorrules` + MCP | `.codex/instructions.md` + MCP | `CLAUDE.md` + MCP |
| **最佳场景** | 交互式开发 | 批量重构 | 架构感知编码 |
| **KB 反馈闭环** | 手动 | 任务后注入 | 工具实时回流 |
| **启动成本** | 低（IDE 内置） | 中（需配置沙箱） | 中（需配置 MCP） |
| **实时 KB 查询** | 是 | 仅通过桥接 | 是（原生） |

**选择建议：**

- **日常编码**（写新功能、修 bug、局部重构）→ Cursor。编辑器内的即时交互和 MCP 集成已经足够。
- **大规模重构**（跨模块迁移、批量 API 升级、代码风格统一）→ Codex。自主沙箱模式适合"设定目标、自动执行"的任务。提前把 KB 上下文注入 `instructions.md` 即可。
- **架构敏感的开发**（新增核心接口、修改安全相关逻辑、跨团队 API 约定）→ Claude Code。原生 MCP 支持确保每一步决策都有 KB 背书。

三种工具不是互斥的。一个典型的工作流可能是：用 Claude Code 做架构设计和接口定义（实时查询 KB），用 Codex 做批量实现和迁移（预注入 KB 上下文），用 Cursor 做日常的代码编辑和调试。知识库是三者共享的上下文基础设施。

## Conclusion —— 小结

AI 编码助手的瓶颈从来不是模型能力 —— 模型一年比一年强。瓶颈是**上下文质量**。一个没有知识库的助手只能看到当前打开的文件和有限的仓库搜索。一个有知识库的助手能看到设计理据、操作手册、架构决策，并且知道哪些是经过评审的、哪些是 AI 起草的、哪些已经过期。

知识库不是给 AI 编码助手加了一个功能。它改变的是 AI 编码的**信任基础**：从"我希望 AI 猜对了"到"我可以验证 AI 引用了什么"。第 29 章将把这个思路推向更完整的形态 —— 知识库不再只是 AI 编码的被动上下文源，而是 agent 的一等工具。

## 参考文献

```{bibliography}
:filter: keywords % "hybrid" or keywords % "agents" or keywords % "mcp"
```
