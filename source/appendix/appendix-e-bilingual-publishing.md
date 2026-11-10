---
title: "附录 E —— 发布一个双语知识库"
status: review
authors:
  - Walter (Yamin) Fan
last_verified_commit: HEAD
zh_status: complete
keywords:
  - appendix
  - i18n
  - sphinx
---

# 附录 E —— 发布一个双语知识库

> 适用范围：这个附录刻意保持为**纯文档、自包含**。它描述的是作者在一个私有的
> project-knowledge-base 工具包里逐步打磨出来的双语发布纪律
> {cite}`fanyamin_pkb_skill`，并把它适配到本书正在使用的 `sphinx + myst`
> 工具链上。这里不会额外接入新的 `make` target；直接使用本书 Makefile
> 里现有的 `make book-i18n` / `make book-build` 即可。

## 为什么要做双语发布

一个只用作者第一语言写成的知识库，会把那些第二语言或第三语言很流利、但无法以
专业深度阅读第一语言的读者挡在外面。一个只用英文写成的 KB，则会把大量更习惯用
中文进行技术深读的华语工程团队成员排除在外（反之亦然）。真正的问题不是
*选哪种语言*，而是*用什么样的纪律，在不把写作成本翻倍的前提下，让两种语言保持同步*。

有三种失败模式，会让这件事比看上去难得多：

1.  **翻译漂移（translation drift）。** 英文页面更新了，中文翻译没更新。
    中文读者看到的是一页**看起来还很新**、其实已经过时的内容。
2.  **锚点漂移（anchor drift）。** 英文 H2 标题改写了，中文翻译还保留旧锚点。
    跨文档链接会静悄悄地失效。
3.  **真理源漂移（source-of-truth drift）。** 两份并行 prose 文件并存，
    各自独立编辑，最后两种语言在内容层面分叉，而不只是措辞不同。

Sphinx 工具链再加上一点点纪律，就足以同时处理这三类问题。

## 双语流水线长什么样

一种源语言，多个翻译目标，每种语言各构建一次：

```text
book/                            <- single source of truth (English)
  ch08-prose-operational-model.md
  ...
  locale/
    zh_CN/
      LC_MESSAGES/
        ch08-prose-operational-model.po  <- Chinese translation
        ...
```

`.po` 文件是 gettext 的消息目录。每一条目录项都把一段英文字符串
（`msgid`）与一段译文字符串（`msgstr`）配对起来。Sphinx 在构建时提取
`msgid`；翻译者（人或机器）补上 `msgstr`；随后 Sphinx 再从**同一份源文本**
为每种目标语言重新渲染 HTML。

英文 prose 仍然是唯一的 source of truth。中文页面是被 *rendered* 出来的，
而不是被独立维护的。这一步是关键：如果一个 KB 拥有两个同等地位的源头，
那它实际上也就拥有了两个漂移源头。

## 怎样把它做可靠

四条纪律，每一条都不贵：

### 1. 打开 `gettext_uuid`

如果没有 UUID，每一条翻译项都是以英文字符串本身作为 key。
把 "The retrieval layer consults the tuple" 改写成
"The retriever reads the tuple"，译文就会立刻变成孤儿：旧条目指向已经不存在的
英文，新条目又没有翻译。翻译者只能从头再来。

在 `conf.py` 中设置 `gettext_uuid = True` 之后，每个条目在第一次出现时都会获得
一个稳定 UUID。之后哪怕英文措辞改了，只要含义没变，UUID 就不变；中文翻译追随的
是 UUID，而不是字面字符串。`conf.py` 里应当这样写：

```python
gettext_uuid = True
gettext_compact = False   # one .pot per source file
gettext_additional_targets = ["literal-block", "image"]
```

结果是：**保留语义**的英文修改，也会**保留原有翻译**。

### 2. 把 `.po` 编辑保持为机械操作

手工编辑 `.po` 文件既枯燥又容易出错（一个 stray quote 就能把整个文件弄坏）。
应当用 `polib` 来做程序化修改：

```python
import polib

po = polib.pofile("locale/zh_CN/LC_MESSAGES/ch08-prose-operational-model.po")

for entry in po.untranslated_entries():
    if should_auto_translate(entry.msgid):
        entry.msgstr = call_llm_translator(entry.msgid)
        entry.flags.append("fuzzy")  # mark for review; never silently approve

po.save()
```

要让这件事安全，有两条规则：

- **所有机器翻译一律以 `fuzzy` 落地。** 这是 gettext 的惯例：
  fuzzy 条目在通过人工审阅并去掉该标记之前，要么以未翻译形式展示，
  要么回退到英文。它与第 5 章 frontmatter/footer 里
  “AI 永远只能写 pending” 的规则是同构的。
- **不要因为 `msgid` 变了就删除已有译文。** 让 `sphinx-intl update`
  去处理。更新器在英文发生漂移时会把相关条目标成 `fuzzy`，
  既保住已有工作量，也把“需要复核”的信号明确打出来。

### 3. 通过 hook 翻译 H1

Sphinx 的默认行为是从 frontmatter 的 `title:` 字段渲染文档标题，而这个字段通常是
英文。对于中文输出，读者期待看到的是中文标题。一个很小的 `conf.py` hook
就可以在不复制 prose 的前提下解决这个问题：

```python
def translate_h1(app, docname, source):
    """Replace the frontmatter title with the zh_CN translation when
    building for zh_CN, so the navigation sidebar shows Chinese titles.
    """
    if app.config.language != "zh_CN":
        return
    translations = load_h1_map(docname)   # tiny YAML: path -> zh_CN title
    if docname in translations:
        source[0] = re.sub(
            r"^title:\s*.+$",
            f"title: '{translations[docname]}'",
            source[0],
            count=1,
            flags=re.MULTILINE,
        )

def setup(app):
    app.connect("source-read", translate_h1)
```

`h1_map.zh_CN.yml` 是作者与 prose 并排维护的一份很小的平面文件：

```yaml
ch08-prose-operational-model: 第 8 章 — 文稿层的运维模型
ch21-incremental-sync: 第 21 章 — 增量同步
ch22-evaluation-and-benchmarks: 第 22 章 — 评估、基准与发布闸门
```

这比直接把标题塞进 `.po` 目录要多做一点工作，但它能产出一个在每种语言里都
**像母语界面一样自然**的导航侧栏，而这恰恰是读者最先注意到的地方。

### 4. 在关键页面未翻译时让构建失败

并不是每一章都必须当天就有中文。Runbook 需要；概览章节晚一周也许可以。
把这种策略编码进 frontmatter：

```yaml
---
title: 'Chapter 23 — Maintenance, Drift, and Link Rot'
zh_status: required   # required | optional | none
---
```

构建时加上一条检查，就能把这个策略真正执行起来：

```python
# book/_tools/check_zh_status.py  (sketch)
for doc in all_docs:
    if doc.frontmatter.get("zh_status") == "required":
        po_path = f"locale/zh_CN/LC_MESSAGES/{doc.path}.po"
        if not po_path.exists() or po_untranslated_fraction(po_path) > 0.05:
            fail(f"{doc.path}: zh_CN translation required but missing or <95% complete")
```

三个值：`required`、`optional`、`none`，就给每一页定义了一份明确的翻译 SLA。
从运维视角看，这和第 22 章的 hard/soft gate 分裂是对应的：
`required` 是会阻塞发布的硬闸，`optional` 是软警告，`none` 则保持沉默。

## 示例 —— 一页内容穿过整条流水线

`ch08-prose-operational-model.md` 在一次编辑轮中的生命周期：

1.  作者修改英文源文本，例如重写一个小节标题。
2.  `make book-i18n` 运行 `sphinx-build -b gettext` + `sphinx-intl update`。
    被改写标题的 UUID 被保留下来；它对应的中文翻译现在会被标记为 `fuzzy`。
3.  `make book-build` 构建英文版本。构建成功；因为该章节的 `zh_status`
    是 `optional`，所以 fuzzy 条目不会让构建失败。
4.  一次受控的 LLM 翻译 pass（与第 23 章的 L2 类似）为改写后的标题生成新的
    `msgstr`，并把结果写回去，同时保留 `fuzzy` 标记。
5.  人类审阅者在一个支持 PO 的编辑器中（Poedit、Lokalize）打开 `.po`
    文件，对照英文阅读新中文文本，然后移除 `fuzzy` 标记。
6.  下一次 `make book-build -D language=zh_CN` 就会产出带有已审批译文的
    中文 HTML。

六步里有三步是机械操作（`make` target），两步是受控的 LLM 工作
（每个变更条目一次翻译调用），一步是人类在一份小 diff 上做审阅。
所以，每次英文编辑带来的总成本，不过是几分钱的 LLM token 再加上
每章大约一分钟的人类时间。

## 运维失败模式与修复

| 故障 | 症状 | 修复 |
|:--|:--|:--|
| 事后才打开 UUID | 一次随手改写之后触发大规模重翻 | 在第一次翻译 pass **之前**就打开 `gettext_uuid`；一旦打开，就一直保持开启。 |
| 手改 `.po`，语法弄坏 | Sphinx 报 "malformed entry near line N" | 始终通过 `polib` 编辑；并在 pre-commit hook 里对每个 `.po` 跑 `msgfmt --check`。 |
| 翻译长期落后于英文 | 中文读者看到过期 prose，却没有任何警告 | 对关键页设置 `zh_status: required`；当覆盖率跌破阈值时，CI 必须 hard-fail。 |
| 翻译者翻了 prose 却没翻标题 | 导航侧栏一半中文一半英文 | 维护 `h1_map.zh_CN.yml`，并在任何 `zh_status: required` 的章节上把它也纳入 CI gate。 |
| 机器翻译被静默放行 | 中文读者直接看到胡言乱语 | 每一条机器翻译 `msgstr` 都必须带着 `fuzzy` 落地；清除 `fuzzy` 的动作必须由人完成。 |

上面每一条都不过是一条一行规则；这个附录的核心主张是：如果你想让双语发布
保持可信，它们就都不是可选项。

## 保持简单

上面的整套纪律，完全可以舒适地装进几百行配置、每种语言一个小小的 `h1_map`
文件、再加一条构建检查里。它不需要翻译管理系统、不需要单独的内容仓库、
也不需要一棵并行 prose 树。它需要的，是从双语发布的第一天起，就把前述四条
纪律全部摆上桌面并强制执行，因为这些失败模式一旦事后修复，代价都会很高。

剩下的部分都只是常规的 Sphinx 工作流：用 `make book-i18n` 刷新，
用 `make book-build -D language=zh_CN` 渲染，用 CI 执行 `zh_status`
硬闸。这个附录不需要新工具；它需要的是把这些纪律**命名出来，并真正执行**。

## 参考文献

```{bibliography}
:filter: keywords % "appendix" or keywords % "i18n" or keywords % "sphinx" or keywords % "self-citation"
```
