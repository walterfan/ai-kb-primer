---
title: "第 3 章 —— Diátaxis 与软件文档"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - foundations
  - diataxis
  - software-docs
  - competitor
---

# 第 3 章 —— Diátaxis 与软件文档

本章全貌，一页看完 —— 本章主张一个软件知识库需要的两根分类轴、Diátaxis 的四种文档类型、本书采用的那份有态度的十页实例，以及 *刻意不放进* 这份清单里的东西：

```{mermaid}
mindmap
  root((Diataxis for software))
    Reader-intent axis
      Tutorial (learning)
      How-to (task)
      Reference (information)
      Explanation (understanding)
    Author-and-trust axis
      verification_status
      source (original / git / doc / url)
      Layers L0-L4 (ch18)
    What it replaces
      Freeform tags
      "Miscellaneous"
      Filing problems
    Ten-page instantiation
      00 overview
      01 quick-start
      02 architecture (C4)
      03 repo-map
      04 data-and-api
      05 workflows
      06 conventions
      07 testing
      08 runbook
      09 observability
    Not in the cut
      ADRs (a collection, ch18)
      changelog (subsumed by git)
```

## 为什么

一个没有文档类型分类的知识库，是一个搜索问题。一个采用了 *错的* 分类的知识库，是一个归档问题。归档问题比搜索问题更贵 —— 因为归档问题是人主动“写”进去的，而被写进去的错误会不断累积利息。

Lethbridge 等人调研的那些一线工程师 {cite}`lethbridge2003software` 被问到的，本质上就是一句：*"你们为什么不看文档？"* 答案归结为：*因为我不知道哪份文档能回答我的问题，而等我好不容易找到一份，它又不是我想要的那种文档。* Aghajani 等人 {cite}`aghajani2019software` 二十年后以另一种口径报告了同一个症状：关于文档的 bug report 中，有很大一部分是"内容类型和它所在的章节不匹配"。

一套好的分类学能让这种症状消失。Daniele Procida 提出的 **Diátaxis** {cite}`procida_diataxis` —— 截至本书写作时 —— 是最能在真实软件项目里活下来的那套分类。它只有四种类型；它沿两根正交的轴展开；它小到能印在一张明信片上；而且 —— 对本书最重要的一点是 —— 它小到一条分类流水线（无论是人工的还是 LLM 驱动的）真的可以被训练起来，而不至于陷在二十种类型的平票里无从决断。

## 是什么

Diátaxis 沿两根轴把技术文档划分开，产生四个象限 {cite}`procida_diataxis`：

```{mermaid}
graph TB
  subgraph Theory
    Explanation["Explanation<br/>(understanding-oriented)"]
    Reference["Reference<br/>(information-oriented)"]
  end
  subgraph Practice
    Tutorial["Tutorial<br/>(learning-oriented)"]
    HowTo["How-to<br/>(task-oriented)"]
  end
  Tutorial <---> Explanation
  HowTo <---> Reference
```

|  | 实用（Practical） | 理论（Theoretical） |
|-------------------|-------------------------------|------------------------------|
| **学习中（Studying）** | **Tutorial（教程）** —— 面向学习 | **Explanation（解释）** —— 面向理解 |
| **工作中（Working）** | **How-to（操作指南）** —— 面向任务 | **Reference（参考）** —— 面向信息 |

这两根轴（*实用 ↔ 理论* 与 *学习中 ↔ 工作中*）讲的不是写作者的心情，讲的是阅读者的状态。一个读者要么在“学习”（正尝试构建一个心智模型），要么在“工作”（正想立刻把一件事做完）；与此正交地，他要么在找具体行动，要么在找概念。任何一篇号称同时如实回答 *两根* 轴的文字，大概率其实是两篇被粘在一起的文字。

对本书的目的来说，这套分类有一条很重要的性质：一份文档的类型会决定 *它理想开头第一句话的形状*：

- 一份 **tutorial（教程）** 从“读者将会造出什么”开始写。
- 一份 **how-to（操作指南）** 从“读者想到达的目标状态”开始写。
- 一份 **reference（参考）** 从“被描述之物的名字”开始写。
- 一份 **explanation（解释）** 从“正在被回答的问题”开始写。

这一点到第 6 章会变得重要 —— 届时我们会让一个 LLM 把原始导入的文档分类到这四种类型中去。

## 怎么做

文本层参考实现把 Diátaxis 直接搬到了它的 frontmatter schema 里。摘自 `wiki-template/metadata/SCHEMA.md`：

```yaml
---
title: "Page Title"
doc_type: reference            # tutorial | how-to | reference | explanation
verification_status: supported # supported | unreviewed | uncertain | contradicted | superseded
created_by: alice
updated_by: alice
source:
  type: original               # original | git_repo | doc | url
  uri: ""
---
```

有两条观察值得停一下。

**第一**：`doc_type` 是一个一等公民的 *字段*，不是一个自由格式的 tag。它的合法取值只有四个，不是一百个。这个约束是承重的。下游的每一个组件 —— 搜索索引（第 7 章）、LLM 辅助分类器（第 6 章）、ADR 与 runbook 的分离（第 25 章） —— 都假设 `doc_type` 恰好落在这四个标签其中之一。一旦项目接受了第五个标签，分类器就得重训、索引就得重加权、分层模型里的每一条规则都得被重新审一遍。守在四个是便宜的，长到五个从来都不便宜。

**第二**：这份 schema 把 `doc_type` 和一个 *信任* 字段 （`verification_status`）、一个 *来源* 字段（`source`）捆在一起。这不是偶然。本书底稿的那篇博客 {cite}`fanyamin2026deepwiki` 在 §11 里论证道：*文档分层*（L0 – L4，第 25 章会完整展开）正是那个用来防止AI 生成页、人工 runbook、架构决策这三者被混为一谈的机制。Diátaxis 沿 *读者意图* 这根轴给文档分类；`verification_status` 和 `source` 沿 *作者与信任* 那根轴给它分类。对一个诚实的软件知识库来说，这两根轴都必须在场。

有一种具体的对立观点值得正面回应。**DITA** {cite}`oasis_dita` —— OASIS 维护的 “Darwin Information Typing Architecture” —— 是大型技术写作行业实际使用的那套正式分类。DITA 比 Diátaxis 更丰富，带 XML schema，支持任意多的内容类型（`task`、`concept`、`reference`、`troubleshooting`，以及它们各自的特化）。对那种要跨多条产品线生产印刷手册的技术写作团队来说，DITA 是对的工具。但对一个要在代码旁边同步生产 wiki 页的软件团队来说，DITA 强加的一整套工具链成本 —— XML 编辑器、DITA-OT 构建流水线、专业化的撰稿人 —— 已经压过了它的灵活性。在我们这个场景下，Diátaxis “愿意让自己 *很小* ” 这一点本身是优点，不是缺陷。Write-the-Docs 社区的风格指南语料 {cite}`wtd_write_the_docs` 得到的也是类似的实务结论。

## 示例

把 `wiki-template/content/` 下的页面导出一份来看 —— 这也是本仓库用来初始化一个新 wiki 的模板 —— 它们能干净利落地落进 Diátaxis 的四个类型里：

| 文件 | Diátaxis 类型 | 读者的状态 | 为什么 |
|------|---------------|-----------------|------|
| `content/devops/kubernetes.md` | **reference（参考）** | 工作中 | 概述 Kubernetes 的基本概念，附带签名和示例 |
| `content/devops/docker.md` | **reference（参考）** | 工作中 | 同样的模式，聚焦在 Docker CLI + Compose |
| `content/programming/go/concurrency.md` | **explanation（解释）** | 学习中 | “面向理解” —— 解释 goroutine、channel 和 `sync` 为什么长成现在这个样子 |
| `content/programming/go/interfaces.md` | **explanation（解释）** | 学习中 | 讨论的是 Go 里结构化类型的 *理念*，而不是一份步骤化的菜谱 |
| `content/programming/python/asyncio.md` | **explanation（解释）** | 学习中 | 和上面那份 Go 并发页的形状一样 |
| `content/programming/python/decorators.md` | **explanation（解释）** | 学习中 | 讨论的是 decorator *是什么*、以及什么时候该动用它 |

这个模板目录“出厂状态”主要是 explanation，加一个 reference 分区 —— tutorial 和 how-to 还都没有。这算是对当前模板的一条 *现状* 观察 —— 它是一个起点模板，而不是一个完成态的知识库 —— 不是缺陷。而“做一次分类”这个动作，让这个缺口立刻变得可见：一个基于本 wiki 长起来的成熟项目，随着时间推移应当长出 tutorial（上手路径）和 how-to（runbook）。第 6 章的分类流水线在报告 doc_type 直方图时，检测的就是这种失衡。

对 `wiki-template/metadata/SCHEMA.md` 这份文件本身做同样的练习，结果挺有意思：它是一份 **reference**（它在描述 schema），但其中又包含一小段 **explanation**（“Trust Model”一节，48–53 行）。这是一个有用的提醒：Diátaxis 是 *以文件为粒度* 的分类器，而不是以段落为粒度。一次文档分层复审（第 25 章）应当把正文跨了两个象限的文件标出来，并考虑要不要把它拆开。

## 一份有态度的具体实例

Diátaxis 告诉你页面的 *类型*，但它并不告诉你一个软件项目 *具体要交付哪几页*。两个团队完全可以都“符合 Diátaxis”，最终却得到截然不同的目录：一个写了五篇 tutorial、没有 runbook；另一个写了一份 runbook、一份 reference，此外什么都没有。两种都能各自自圆其说。但两种都帮不到一个新来的工程师 —— 他要在一个从没见过的项目里找到 *runbook 放在哪儿*。

作者私人维护的那套 project-knowledge-base 工具链 {cite}`fanyamin_pkb_skill` 对这个问题的回答是：承诺一组小而稳定的页面 —— 对“一个软件知识库具体该有哪几页？”给出一个具体答案 —— 并且用编号把阅读顺序也钉死。本书取其中的一个十页版本，作为自己的标准实例。各团队完全可以另有选择；这里的主张不是“只有这十页是对的”，而是“早早挑定 *某一组* 稳定页面”比“反复争论该挑哪一组”更值钱：

| # | 页面 | Diátaxis 类型 | 读者的状态 | 它回答的是什么 |
|---:|------|---------------|-----------------|-----------------|
| 00 | `overview` | explanation（解释） | 学习中 | *“这个项目是什么？五分钟讲清楚。”* |
| 01 | `quick-start` | tutorial（教程） | 学习中 | *“我怎么在十分钟内跑通 hello world？”* |
| 02 | `architecture` | explanation（解释） | 学习中 | *“有哪些容器、哪些组件，以及为什么？”*（C4 模型；见第 4 章） |
| 03 | `repo-map` | reference（参考） | 工作中 | *“在当前 commit 下，什么放在哪儿？”* |
| 04 | `data-and-api` | reference（参考） | 工作中 | *“对外公开的契约是什么？”* |
| 05 | `workflows` | reference（参考） | 工作中 | *“代码实际上一步一步做了什么？”* |
| 06 | `conventions` | reference（参考） | 工作中 | *“命名、布局、风格规则。”* |
| 07 | `testing` | how-to + reference | 工作中 | *“我怎么跑测试，它们覆盖了什么？”* |
| 08 | `runbook` | how-to（操作指南） | 工作中 | *“它挂了的时候，我该怎么办？”* |
| 09 | `observability` | reference（参考） | 工作中 | *“我该看哪些指标，该去哪儿看？”* |

对于 *没* 出现在清单里的东西，有两点要说明。没有 `adr/` 页，因为 ADR 是一个 *集合*，不是一页；它们住在自己专用的目录里，并在第 25 章里单独分层处理。没有 `changelog` 页，因为对大多数团队来说，一份保养得当的 Git 历史加上 release note 已经把它覆盖掉了。

这套编号的作用，远比看上去大。一个完全冷启动落到项目文档上的读者，应该可以按顺序读 `00` 到 `04`，然后就能动手改代码；`05` 到 `09` 则是他在“工作中”回头查的那几页。一个扫描知识库的工具 —— 第 6 章里的分类流水线、第 25 章里的文档分层四元组 —— 也就因此拿到了一份稳定的目录形状可以对齐。而一个在审核是否接受新页面的维护者，手上也多了一个简单的判据：*这一页能塞进那十个槽里的某一个吗？还是它想成为第十一个？* 一旦答案是“第十一个”，对话就从“这东西我该放哪儿？”（一个归档问题）变成了“这一页是否值得我们重新调整整套分类？”—— 后者是一个有建设性的问题。

这就是一个具体答案。第 8 章（“文本层运营模型”）会把它定为本书四角运营模型中的 C1 角。

## 小结

一旦 `doc_type` 是一个一等公民、只有四个值的字段，下游的问题全都会收缩。搜索加权在碰到“我该怎么……”的查询时可以偏向 how-to。LLM 分类器拿到的是一份封闭词表。文档分层模型也终于有东西可以挂规则了。而 ADR、愿景文档、runbook —— 这几类都不是纯粹的 Diátaxis 类型 —— 会在第 25 章里拿到属于自己的一等公民层，而不是被硬塞进一个谁都不会去搜的 “Miscellaneous” 类别。把分类保持得小，不是为了“极简”本身 —— 而是为了让本书下游每一章都仍然有可能被真正实现出来。

## 参考文献

```{bibliography}
:filter: keywords % "diataxis" or keywords % "software-docs" or keywords % "competitor" or keywords % "deepwiki"
```
