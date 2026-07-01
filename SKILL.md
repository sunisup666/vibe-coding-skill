---
name: vibe-coding-pipeline
description: 引导用户把一个项目原型概念、全新想法或已有项目的改造想法,通过一条固定的工程化文档流水线(概念→业务→技术架构→数据模型→开发切片→AGENTS.md→逐切片实现),一步一步生成、审阅、迭代成可交给 AI agent 自动实现的完整工程资产;每一步生成后都让用户审阅、按反馈迭代到满意,把文档落进项目的 docs/,并在磁盘上维护进度以支持隔天新开会话续做;满意后自动复盘该阶段提示词、只提炼可泛化改进、更新提示词并准备回流 commit 给用户确认后推送。Use this skill whenever the user brings a software project idea, wants to do "vibe coding", wants to hand a project off to an agent to implement, wants to write business / technical / architecture / data-model / slice docs before coding, wants to scaffold a project, or mentions 工程化流水线 / 文档链 / 项目骨架 / 切片开发 / 提示词回流 / 续做上次的流水线. 即使用户没有明说"流水线"或"skill",只要他们抛出一个项目想法并希望系统化地推进到可开发状态,就使用本 skill;新开会话说"继续上次的项目"也要用它。
---

# Vibe Coding 工程化流水线

把"脑子里的项目想法"翻译成一条**文档链**,再交给 agent 逐切片实现。用户只负责三件事:**写/审文档、定边界、审关键点**,不碰实现代码。

本 skill 是方法论引擎的**唯一事实源**,全局安装一份(`~/.claude/skills/vibe-coding-pipeline/`),自带引擎和提示词。骨架不在这里——它跟技术栈绑定、按语言存在独立的 `project-skeletons` 仓库,运行时按项目语言拉取。每个项目是独立目录,只接收生成的 docs 和拉来的骨架。**支持隔天新开会话续做**——机制见下。

## 核心理念:三条铁律

1. **唯一事实源(SSOT)**:数据模型文档(`docs/03-data-model.md`)是唯一字段权威,别处只引用不重写。
2. **完成判定必须是能跑的命令**:切片验收撞 `make verify`,不是"能正常工作"这种空话。
3. **垂直切片,不是横切**:交给 agent 时一次只给一条"能独立跑通、独立判对错的完整功能链"。

呈现给用户前,任何违反这三条的文档必须自己先修掉。

## ★ 每个会话开始必做:从磁盘重建进度(支持隔天续做)

**绝不假设自己记得上次状态。新会话 context 为空,一切从磁盘重建。** 流水线的真实状态活在两处磁盘产物里:`docs/` 里实际存在的文档 + `docs/_progress.md` 进度文件。会话一开始,先做:

1. 在当前项目目录找 `docs/_progress.md`。
2. **有** → 读它,再扫 `docs/` 里实际存在的文件交叉核对,重建:到第几步、什么状态、有哪些待确认的提示词回流、有哪些未决问题、还缺哪些人工素材。用一两句跟用户对齐,例如:"上次到第 3 步数据模型、状态是审阅中,还有一个字段默认值没定,接着来吗?"然后从断点继续。
3. **进度文件与实际文件冲突**(如 progress 说第 2 步没完成但 `docs/02-*.md` 已存在)→ 以**文件系统为准**重新推断,并告诉用户你据实修正了进度。
4. **有 docs 但没 progress**(上次会话没写状态就断了)→ 按已存在的文档推断进度,补建 `docs/_progress.md`。
5. **两者都没有** → 这是新项目,从第 0 步开始,建 `docs/` 和 `docs/_progress.md`。

> 提示用户:Claude Code 原生的 `claude -c` 也能续会话,但它有时不恢复上下文、且大会话吃配额。本 skill 不依赖它——只要 `docs/` 和 `docs/_progress.md` 在,任何新会话都能重建,这才是可靠机制。

## 进度文件 `docs/_progress.md`(硬性维护)

它是隔天续做的命根子。**只有它被持续更新,新会话才续得上。把"更新进度文件"当成每一步、每一次状态变化的收尾动作。** 结构:

```markdown
# 流水线进度(本文件由 skill 维护,你也可手动修正)
- 项目: <名称>
- 当前步骤 / 状态: 3 数据模型 / in-review

## 各步骤(todo / drafting / in-review / approved)
- [x] 0 概念一页纸 — approved (docs/00-concept.md)
- [x] 1 业务文档 — approved (docs/01-business.md)
- [~] 3 数据模型 — in-review (docs/03-data-model.md)  ← 从这里继续
- [ ] 4 开发切片 …

## 第 6 步切片进度(到第 6 步才填)
- [x] slice-1 xxx — verified green
- [~] slice-2 xxx — in-progress

## 待确认的提示词回流(用户没确认不算数,可跨会话确认)
- (无)

## 未决问题 / 下次需先问用户的
- 数据模型 xxx 字段默认值未定

## 需人工准备的素材
- samples/ 缺 视频feed 真实样例
```

**必须更新它的时机**:生成草稿后(→in-review)、用户批准后(→approved 并记文件名)、开始某切片(→in-progress)、切片绿灯(→verified)、产生待确认回流、出现未决问题或缺素材时。

## 新建项目时的落盘

第一次在一个空目录启动时:
1. 建 `./docs/` 和 `docs/_progress.md`。
2. 第 0 步概念一页纸定稿后写入 `docs/00-concept.md`(否则它只活在会话里,断了就丢)。
3. 所有文档写进 `./docs/`,**不**写进 skill。

**骨架不在本 skill 里,存放在独立的骨架仓库,按语言拉取。** 等第 2 步技术架构文档定了语言后再拉(不是一开始就拉,要先知道用哪种语言)。流程见下节。

## 按语言拉取骨架(第 2 步定了语言之后)

骨架是跟技术栈绑的、基本不变的**只读素材**,存在独立仓库,含 `python-fastapi/`、`java-springboot/`、`go-standard/`、`c-cmake/`、`rust-cargo/` 等标准结构,每套自带 `make verify` 且开箱绿。

**骨架仓库地址(固定,联网拉取):** `https://github.com/sunisup666/project-skeletons.git`

技术架构文档确定语言后,把对应骨架拷进当前项目根:

1. 浅拉:`git clone --depth 1 https://github.com/sunisup666/project-skeletons.git /tmp/_skels`
2. 拷 `/tmp/_skels/<语言目录>/` 内容进项目根,再删 `/tmp/_skels`。
3. **拉不到(断网/仓库不可达)**:停下来告诉用户拉取失败,让他检查网络或确认仓库地址,不要硬编一个空壳糊弄。
4. 拷入后跑一次该骨架的验证命令(`make verify` 或等价),确认绿基线成立,再继续。

拷入后把"已拉取 <语言> 骨架"记进进度文件。骨架是只读素材,**不回流**——它基本不变,没有从项目往回同步的需要。

## 流水线总览

| 步骤 | 产物 | 提示词 | 门禁 |
| --- | --- | --- | --- |
| 0 | `docs/00-concept.md` | `references/prompts/00-concept-helper.md` | 确认概念 |
| 1 | `docs/01-business.md` | `references/prompts/01-business-doc.md` | 审阅 |
| 2 | `docs/02-architecture.md` | `references/prompts/02-tech-architecture.md` | 审阅 |
| 3 | `docs/03-data-model.md`(SSOT) | `references/prompts/03-data-model.md` | **重点审** |
| 4 | `docs/04-slices.md` | `references/prompts/04-dev-slices.md` | **审完成判定** |
| 5 | `AGENTS.md` | `references/prompts/05-agents-md.md` | 审阅 |
| 6 | 各切片代码 | `references/prompts/06-slice-implement.md` | **每切片审绿灯** |

## 每一步的标准循环(0~5 步)

1. **读提示词 + 收集输入**:读该步 `references/prompts/` 文件,以及它需要的前置文档。前置缺失就先补,不跳步。
2. **生成草稿**:按提示词要求生成,不写代码。
3. **呈现前自检**:对照 `references/pitfalls.md` 相关项自查,发现问题当场修。
4. **呈现 + 指出重点审查项**:主动告诉用户这一步该重点看哪(见"门禁"),别只说"你看看"。
5. **审阅迭代**:满意 → 落盘 `docs/` 并**更新 `docs/_progress.md`**;不满意 → 记录反馈,修改再呈现,顺着反馈一点点改到满意。不要一次问一堆。
6. **提示词复盘与回流**:定稿后按 `references/prompt-tuning.md` 复盘,只把**可泛化缺陷**改进进本步提示词,给用户看 diff,确认后写回本 skill 的 `references/prompts/`。项目特有内容绝不进提示词。拿不准就问,或先记进进度文件的"待确认回流",留到之后确认。
7. **收尾**:更新进度文件,简述"这步完成了什么、下一步做什么、需要用户先准备什么",再继续。

## 三处人工门禁(必须停下提醒用户重点审)

- **第 3 步后(数据模型)**:全项目地基,字段错了后面全错。引导逐字段确认命名、类型、唯一键、缺失处理。
- **第 4 步后(开发切片)**:逐条确认每个切片的完成判定是能跑的命令、都是垂直切片。
- **第 6 步每个切片后**:只看绿灯是否通过 + 难自动测试的边缘。

## 开工前硬前提(依赖外部真实数据的项目)

若项目依赖外部真实数据(爬取、第三方接口、外部文件格式),进第 6 步前**必须让用户亲手准备"真实输入 + 期望输出"样例**放进 `samples/`,并记进进度文件。样例不齐不进第 6 步,明确告诉用户。

## 第 6 步:逐切片实现

按 `docs/04-slices.md` 顺序,**一次只做一个切片**:读 `references/prompts/06-slice-implement.md` + 全部文档 + `AGENTS.md`;只做当前切片范围,严格对齐数据模型,禁止自创字段;先写测试再写实现;跑该切片验证命令必须全绿并贴结果;遇未定义结构/缺样例/歧义/新依赖就停下问用户。每个切片状态记进进度文件的切片区,绿灯后再下一个。

## 提示词自优化 = 写回本 skill 自己

提示词的唯一副本在本 skill 的 `references/prompts/`。优化直接写这里(全局安装是 `git clone` 的可写仓库,不是只读)。**分清:提示词是数据,可自动优化;`SKILL.md` 是引擎,只手动改——一次糟糕的引擎自动改写会污染之后所有项目。**

## 回流到 GitHub

一个项目走完(或用户要求)时:
1. 汇总本次对 `references/prompts/` 的优化。
2. **准备好 commit,给用户看 diff,用户确认后**才在 skill 目录执行 `git commit && git push`。不静默自动推送。
3. 给一份简短复盘:哪步提示词最容易生成低质量文档、这次改了什么。

跑的项目越多,这一份提示词越准——这是本 skill 沉淀的长期资产。

## 沟通风格

一次只推进一步,清楚交代"现在在哪、接下来干什么、你要审什么"。用户是技术人,直接、不绕弯。呈现文档时主动指出重点审查项。

## 参考文件

- `references/prompts/00-06*.md` — 每步详细提示词,执行到某步时读对应文件。
- `references/pitfalls.md` — 翻车点检查清单,每步"呈现前自检"对照。
- `references/prompt-tuning.md` — 提示词复盘与回流方法,含"可泛化 vs 项目特有"判别。
- 骨架不在本 skill 内,见独立仓库 `project-skeletons`,第 2 步定语言后按语言拉取。
