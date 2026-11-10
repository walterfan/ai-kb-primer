---
title: "第 4 章 —— 一个基于文件的 wiki"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - prose-layer
  - file-based
  - competing-tools
---

# 第 4 章 —— 一个基于文件的 wiki

本章浓缩成一页 —— 后面所有文稿层章节都默认接受的那个**存储选择**，它所解锁的**把 Git 当基础设施**的那笔回报，那个小小的 Go 接口 —— 它把文件系统挡在整个代码库的其它部分之外；以及**基于文件的 wiki 在真实世界里真正出现过的失败形态**：

```{mermaid}
mindmap
  root((File-based wiki))
    Why
      Provenance (git blame)
      Review (PR-diffable)
      Automation (pipelines)
      Longevity (plain text)
    What
      content/ (pages)
      metadata/ (schema, config)
      raw/ (staging)
      _assets/ (binaries)
    How
      WikiRepository interface
      FileRepository (5 methods)
      In-memory rebuild on boot
      No DB, no cache invalidation
    Git gets you
      Revision history
      Blame
      Diff, review, rollback
      Branching, offline edit
    Failure modes
      Binary blobs in tree
      Merge-conflict storms
      Slug rename churn
      Large repo clone time
    Not the right fit
      Multi-thousand concurrent editors
      WYSIWYG-only authors
      Realtime collaboration (Figma-like)
```

周一早上，你发现周末发生了三件事。一位资深同事**上周五交了离职信**，而公司 wiki 上有两个页面，**只有她那个账号**能编辑。兄弟团队的 release notes，被 wiki 引擎的后台任务在凌晨 03:14 以“解决冲突”的名义**悄无声息地回滚了**。还有人给你发了一个 10:00 的日历邀请，问你能不能“**把整个 wiki 导出一下**”，好喂给一个 AI agent 去读 —— 然而**这个 wiki 没有导出功能**。

这三件事**都是同一个决定的症状** —— 那个决定就是：**把整个组织的文稿，放进一个你并不拥有的数据库里**。这一章要讲的是**另一个方向的决定**：一棵放在 Git 里的 Markdown 文件树，然后让知识库的其他所有部分都把**文件系统**当作**权威的数据面**。这样做的回报**不是美学上的** —— 而是：刚才那一个周一早上的每一个问题，都从一个需要 *向上升级* 的问题，变回了一个你可以 *自己修* 的问题。

## 为什么

第一部分论证过：一个软件知识库应该**像代码一样**行事 —— 有版本、可评审、可 diff、可自动化。第 3 章又把文稿层再收窄到 Diátaxis 的四种文档类型。这一章要回答的是下一个问题： *这些文稿，在物理上到底放在哪里？*

**基于数据库的 wiki** 是为“编辑”这件事优化的。它们给你实时渲染、多人并发会话、WYSIWYG 工具栏、以及版本历史 —— 代价是把每一份文档塞进一个你并不拥有其 schema 的不透明表里{cite}`notion_ai,obsidian` 。这么一搞，三件事就坏掉了：

1.  **溯源。** "这段话是谁写的？写的时候对应代码的哪个 commit？" —— 到了数据库型 wiki 里，这变成了对一个专有审计日志的查询，而不是对一份 Markdown 文件跑 `git blame`。
2.  **评审。** Pull request 能把代码变更和文档变更*并排*展示，*前提是*文档也是同一个仓库里的一份文件。否则，评审就分裂在两个系统里，然后渐行渐远 {cite}`gruver2016starting,docs_as_code`。
3.  **自动化。** 第二部分所讲的 `init → import → verify → build → serve` 流水线，需要一种它能读、能写、能 diff、能版本化的输入。文件全部满足；数据库行一条都不满足。

所以本书用的**文稿层参考实现**，就是 `content/` 底下一棵纯 Markdown 文件树，一页一文件，和“描述这份软件的代码”共享同一个 Git 仓库。这里的 wiki *就是* 这个仓库 {cite}`gitit,dokuwiki,moinmoin` 。

## 是什么

具体一点：一个**文稿层参考实现**的实例，就是一个满足四条约定的目录：

```
wiki-root/
├── content/              # Markdown files, one per page
│   ├── getting-started/
│   │   └── install.md
│   ├── reference/
│   │   └── api.md
│   └── _assets/          # images, downloaded binaries
├── metadata/
│   ├── SCHEMA.md         # frontmatter contract (see ch05)
│   ├── wiki.yaml         # instance config
│   ├── users.yaml        # auth
│   └── _sphinx/          # publishing toolchain
└── raw/                  # staging area for unprocessed imports
```

**每一页**是一份带 YAML frontmatter 块的 Markdown 文件。**每一个分类**是一个目录。下划线开头的目录（ `_assets/` 、 `_sphinx/` ）留给基础设施使用，对读者不可见。这个 wiki 没有数据库，没有对象存储，没有隐藏索引：所有**能观察到的东西都是一份文件**，你可以 `cat` 、 `grep` 、或对它跑 `git log` 。

强制这条约定的代码放在 `backend/internal/wiki/` 。真正扛事的就两个文件：

- `repository.go` —— 文件系统 I/O 的边界。
- `service.go` —— 读侧 API，完全由那个 repository 和一个内存索引驱动。

## 怎么做

### 仓库接口

`WikiRepository` 接口是整个后端**唯一**为页面内容碰文件系统的地方。它上面的一切 —— 服务、HTTP handler、CLI —— 都只对 `*Page` 值打交道，**永远**不直接调 `os.ReadFile` 。这是一个经典的 ports-and-adapters 分层，带来的直接好处就是：哪天想换成一个懂 Git 的实现、或者一个基于 S3 的实现，都可以**直接塞进来**，不用动任何业务逻辑。

```{literalinclude} ../examples/part2-prose-layer/ch04/repository.go
:language: go
:lines: 10-16
:caption: The full repository interface, five methods.
```

注意看这个接口上 *没有* 什么：没有“update”，没有“lock”，没有“transaction”。一个基于文件的 wiki 用 `git commit` 把这些全顶掉。在文件系统这一层，写入是 last-write-wins 的；**想要评审闸**，请去仓库托管平台上拿，**不要**找 wiki 引擎本身要。

### 读一页就是解析一份文件

`ReadPage` 是从“磁盘上的字节”到“一个 `*Page` 值”的规范路径：

```{literalinclude} ../examples/part2-prose-layer/ch04/repository.go
:language: go
:lines: 26-53
:caption: `FileRepository.ReadPage` — read, split, parse, derive.
```

四步，全本地、全确定性：

1.  **读。** `os.ReadFile` 把字节拿进来。
2.  **拆分。** `ParseFrontmatter` 把 YAML 头部和正文分开（第 5 章会详述格式）。
3.  **兜底。** 如果没有 frontmatter，`InferFrontmatter` 会从文件名和 mtime 推断出一个最小版本。一份没有元数据的页面仍然可以完整往返。
4.  **派生。** `DeriveCategory` 根据文件相对于 `ContentDir` 的路径计算分类。目录结构*本身就是*分类层级结构；不存在单独的"标签表"。

`ListFiles` 把另一条文件系统约定固化下来：

```{literalinclude} ../examples/part2-prose-layer/ch04/repository.go
:language: go
:lines: 79-103
:caption: `ListFiles` — walk the tree, skipping reserved names.
```

目录名以 `_` 开头的被跳过（留给基础设施），`_category.md` 文件对普通页面列表不可见 —— 因为它描述的是**一个目录**，而不是**一页独立的内容**。两条非常小的约定，扛起了“**什么算一页 wiki 页面**”全部的重量。

### service 是对 repository 的一层很薄的壳

`WikiService` 在 repository 之上叠了一个索引、一个渲染器、以及筛选 / 分页能力；但它的构造函数**是一份依赖清单**，不是一条数据库连接：

```{literalinclude} ../examples/part2-prose-layer/ch04/service.go
:language: go
:lines: 32-47
:caption: `WikiService` — three dependencies, no driver.
```

启动时，service 向 repository 要一份 `ContentDir` 下的全部 Markdown 文件，逐个读成 `*Page` ，把整张清单交给索引去做一次**全量重建**：

```{literalinclude} ../examples/part2-prose-layer/ch04/service.go
:language: go
:lines: 57-73
:caption: `LoadAndIndex` — walk the tree, parse each file, rebuild the index from scratch.
```

**这是故意简单的。** 没有增量索引、没有 change-data-capture、没有缓存失效策略。启动时全量扫一次，对几万级页面来说已经够快 —— 这个遍历是 I/O-bound，不是 CPU-bound —— 同时它干掉了“**索引和磁盘对不上**”这一整类 bug，而这类 bug 是基于数据库的 wiki 永远会反复踩到的那种。你用 `vim` 在服务器外面改了一份文件，下次重启时 `LoadAndIndex` 就会看到。索引引擎在“写入”这个事件里把页面推进来时，它也一直是同步的 —— 因为唯一的**写入者**只有一个：就是 service 自己。**文件系统是真相来源；其余的一切都是派生缓存。**

### Git 白送给你什么

因为内容目录**本身就是一个 Git 工作树**，一个 wiki 平时需要自己实现的很多东西，这里都由外围平台直接白送给你：

| 功能 | wiki 引擎通常自带的实现 | 基于文件的模型把它委托给 |
|---------|------------------------------|-----------------------------------|
| 版本历史 | 自建的 page_revisions 表 | `git log` |
| Blame | 每段一个自建的作者字段 | `git blame` |
| Diff / 评审 | 应用内置的 diff 查看器 | GitHub / GitLab 的 PR UI |
| 回滚 | 自建的 restore 接口 | `git revert` |
| 分支 | 通常没有 | 原生支持 |
| 离线编辑 | 专有的同步客户端 | `git pull` / `git push` |

这就是把 docs-as-code 的那笔交易**具象化**出来 {cite}`gruver2016starting,docs_as_code` ：wiki 放弃“在浏览器里的那种顺滑编辑”，换来**整个 Git 生态**。第 17 章会讲为什么这笔交易对**图谱层**也是决定性的 —— 一个能指向某个 commit SHA 的代码知识库，和一个只能指向“某个数据库当前状态”的代码知识库，是两种完全不同的东西。

### docs-as-code 为什么真的有效：**评审 locality**

“docs-as-code”**被推销**的时候，卖的大多是它的**工具链好处**—— Git、PR、CI。这些是 *表面上看得见的* 胜利。而 *真正承重* 的那个胜利，是**结构性的**；要看清它，得借助两个**更老的想法**。

**康威定律** {cite}`conway1968committees` 说：**一个系统的结构**，会**镜像**构建它的那个团队的**沟通结构**。这个定律**套到文档上**，通常被**随口一说**成“**文档会镜像组织架构图**” —— 但其实它**比这更犀利**。 *评审* 本身就是一种沟通结构；在**任何一个**没有显式文档评审流水线的组织里，**代码评审**的沟通线和**文档评审**的沟通线**不是同一条**。代码是**那些往仓库里提交的工程师**来评审的；文档 —— 如果有人评审的话 —— 是**碰巧打开那个 wiki 页面的人**。这是**不同的人**，在**不同的节奏**上，对着**不同的基线**做出的判断。代码和它的文档之间那种**漂移** —— **不是纪律问题**，而是**康威定律问题**：**两条沟通结构**在产出**两份制品**，而这两份制品**本应描述的是同一件事**。

**Parnas 在 1972 年关于信息隐藏的经典论文** {cite}`parnas1972decomposition` 提供了第二个视角。一个系统应该被**这样分解**：**每个模块**都隐藏一个**很可能要变**的设计决策，并且**每个模块的接口**，即使在实现本身**不稳定**时，**也要稳定**。而 docs-as-code 这一步，**恰好就是**一次 Parnas 式的分解 —— **跨越** *文稿/代码* 的边界：它把文稿和**它所描述的代码**，强行放进**同一个模块** —— **同一个仓库、同一个 PR、同一个评审人** —— 这样 *任何一次* 影响到文稿所描述的那个接口的变更，都会**强制产出一个评审人能看到的 diff**。**Notion 页面**和**它对应的 Go 文件**—— 从 Parnas 的意义上讲 —— 是**两个模块**；而**同一个仓库里**的`docs/auth.md` 和 `internal/auth/handler.go` —— **不是**。

这两个视角 —— **康威定律意义下的评审 locality**，和 **Parnas 意义下的共处一地**（co-location）—— 就是 *为什么* “文件胜于数据库”这套**工程成本的论证**，在**实践中反复赢**。数据库型的 wiki **不是因为文件更优雅而错** —— 它**错在**：它**保证了**文稿会被**在一条和代码评审不同的沟通线上**评审，然后**康威定律**会接着保证**两者必然漂移**。**基于文件的 wiki 并不能消除漂移** —— **没有任何东西能消除** —— 但它能做一件事：**把两条沟通结构重新对齐成同一条**，并让工具链（PR UI、CI、git blame）**去吃掉这次对齐带来的红利**。

第 5 章**依赖这次对齐**把元数据**显式化**（frontmatter 作为契约）；第 6 章**依赖这次对齐**把分类**推进到评审人可审计的流水线**里去；第 21–23 章（第五部分）则**利用这次对齐**做**自动化漂移检测**。这些后续章节，**全部预设**了本节所点明的**那次康威式再对齐**。没有它，**下游一切都站不住**。

## 示例

下面是**最小可行的端到端闭环**，足够证明这个模型是成立的。（完整的分步走在附录 A。）

```bash
$ make wiki-init WIKI_DIR=./demo-wiki
$ ls demo-wiki/
content/   metadata/   raw/

$ cat > demo-wiki/content/reference/hello.md <<'MD'
---
title: Hello
slug: hello
doc_type: reference
status: published
---

# Hello

This is a wiki page. It is also a file in a Git repository.
MD

$ cd demo-wiki && git add -A && git commit -m "add: hello"
$ cd .. && make serve WIKI_DIR=./demo-wiki
# → page appears at http://localhost:7500/pages/hello
```

三个观察，每一条都是“基于数据库的 wiki”提供的反面：

1.  `content/reference/hello.md` 这份文件就是权威来源。删掉它、重启服务，页面就消失了。不存在"孤儿行"状态。
2.  页面的分类（`reference`）就是父目录。把文件移到 `content/tutorials/`，分类就变了。不需要管理后台，不需要迁移脚本。
3.  引入这个页面的那次 commit 就是它的溯源。`git blame content/reference/hello.md` 能回答所有"谁写的、什么时候写的"问题，wiki 不需要自带审计子系统。

## 其他做法

| 系统 | 存储 | 版本化方式 | 作为软件知识库的契合度 |
|--------|---------|------------|-----------------------|
| DokuWiki {cite}`dokuwiki` | 一页一文件，放在磁盘上 | 仓库内的 `attic/` 目录 | 所有权归属清晰；但在**可编程写作**上偏弱。 |
| MoinMoin {cite}`moinmoin` | 每一页一个目录，内带版本历史 | 内建 | 和 DokuWiki 折中取舍类似；schema 更重一些。 |
| Gitit {cite}`gitit` | 放在 Git 仓库里的 Markdown 文件 | Git 自己 | 本章这个设计的**直系祖先**；项目已停更。 |
| Obsidian {cite}`obsidian` | Markdown files in a "vault" | 是否用 Git 由用户自己决定 | **单人用**极好的工具；没有服务端流水线。 |
| Notion / Confluence {cite}`notion_ai` | 专有数据库 | 内部审计日志 | UI 很好用；**与代码仓库的对齐度**很差。 |

本书的参考实现落在 **Gitit + docs-as-code** 这一象限 {cite}`docs_as_code` ：纯文件、Git 原生，服务器**在文件系统之上**提供流水线、搜索和 RBAC，而**不是**去取代文件系统。

## 常见错误，以及 *什么时候不要* 用基于文件的 wiki

“Docs-as-code”**不是**万金油。一个团队度过蜜月期之后，基于文件的模型也会**暴露出它自己的失败形态**。有四种失败模式值得专门点名，因为每一种都配有一个**标准的、错误的修法** —— 那种修法只会让情况更糟：

1.  **往内容树里提交二进制文件。** 一张截图放进了 `_assets/`，不到一年就有人提交了一个 40 MB 的 PDF，然后又来一个 2 GB 的训练数据集 —— "只是临时放一下"。clone 时间崩溃，`FileRepository.ReadPage` 循环因为目录列表变成 I/O-bound，bisect 也变得痛苦。**错误的修法**是拆出第二个"资产仓库"；**正确的修法**是 CI 里放一个*软门禁*（超过大小阈值的文件除非打了特定标签否则拒绝），外加对少数确实需要留在仓库里的二进制文件启用 Git LFS。让这个问题保持可见，而不是拖延。

2.  **把 slug 重命名当成廉价操作。** 因为分类就是目录路径，移动文件就等于改变了 URL。在一个经常被知识库代码侧（第五部分）或外部链接以 `file:line` 方式引用的内容集合里，slug 重命名会悄无声息地打断那些引用。**错误的修法**是"永远不改名"；**正确的修法**是一份存在仓库里的重定向映射表，服务器启动时读取 —— 以及一条 CI 检查（第 22 章），拒绝发布净效果是打断一个仍然被 `content/` 或代码图引用的 URL 的 PR。基于文件的模型让这种检查可以机械化执行；内部给页面编号的专有 wiki 通常做不到。

3.  **协作页面上的合并冲突风暴。** 两个作者并行重写同一份 `runbook.md`，Git 基于行的合并产出一个段落级别的冲突，评审 UI 根本没法正常展示。一个带实时光标的 wiki UI 能躲开这个问题；基于文件的模型不能。诚实的缓解方案是把高频变更的页面视为*有主的* —— 单一作者、短小的 PR，以及第二个贡献者做 rebase 而不是 merge 的规则。对于那些确实需要在真实时间里多人同时编辑的页面（比如一场正在进行的故障中的事件时间线），基于文件的 wiki 不是对的工具；在事件持续期间使用共享文档，事后再导入结果。

4.  **过度迷恋"一切皆文件"的美感。** 版本历史、访问控制、搜索 —— 这些*能*放在文件系统里，但不是全部*都应该*放。每个用户读过哪些页面的历史、临时的草稿自动保存、已发布页面上的评论线程 —— 这些是每分钟都在变、而且没有溯源价值的运营数据。把它们放进一个旁路数据库里，抵住"文件系统也能干"的诱惑。文件系统在内容上赢了；在可变的运营状态上，它输了。

**基于文件的 wiki 对另外三类内容也根本不是对的承载。** **上万编辑者并发写的公共 wiki**（Wikipedia 这种形态）需要的 merge 策略 PR UI 提供不了；**只能用 WYSIWYG 的团队** —— 那种**就是没法、也不会用 Markdown** 的作者 —— 需要另一种写作界面、外加一条同步到文件的管线；**实时协作编辑**（Figma、Google Docs）则是基于文件的 wiki 从来没服务好过的**产品类别**。在采用这个模型前，你真正要问的问题**不是** *“我们喜不喜欢 Markdown？”* ，而是 *“pull request 能不能成为我们组织里**文档评审的基本单位**？”* 如果答案是“能”，第二部分的四章直接适用。如果答案是“不能”，本章的论证**仍然**成立 —— 只不过它应用的是另一层（比如从一个专有 wiki 导入到一个基于文件的镜像，再让代码图谱可以信任那个镜像的那条管线）。

## 结论

基于文件的 wiki 之所以是软件知识库的正确承载层，是因为它**继承**了软件本身早已需要的那些保障：纯文本 diff、版本控制、评审、CI。`WikiRepository` 接口和它的 `FileRepository` 实现，说明“wiki 就是文件”这件事的工程代价很小 —— 一份 Go 文件里几个函数 —— **回报却是**：知识库的 *每一层其他部分* （第 5 章的 frontmatter、第 6 章的流水线、第 7 章的发布与搜索、第四和第五部分的代码图谱）都可以假设自己拿到的输入是**稳定的、可观察的、可 diff 的**。

下一章要讲的是这些文件 *里面* 装的是什么：把 YAML frontmatter 当作一份显式的**来源记录**，以及为什么一个软件知识库的**信任**是从这条边界开始建立的。

## 参考文献

```{bibliography}
:filter: keywords % "prose-layer" or keywords % "file-based" or keywords % "docs-as-code"
```
