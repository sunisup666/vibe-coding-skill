# vibe-coding-pipeline

一个 Claude Code / Codex skill:把项目想法,通过一条固定的工程化文档流水线(概念→业务→技术架构→数据模型→开发切片→AGENTS.md→逐切片实现),一步步生成、审阅、迭代成可交给 agent 自动实现的工程资产。你只负责写/审文档、定边界、审关键点。

这是方法论引擎的**唯一事实源**:引擎(`SKILL.md`)+ 提示词(`references/prompts/`)。骨架跟技术栈绑定,单独放在 `project-skeletons` 仓库、按语言拉取,不在本仓库内。全局装一份 skill,每个项目独立、只接收生成的 docs 和拉来的骨架。

## 安装(用 git clone,别用只读打包)

**Claude Code:**

```bash
git clone https://github.com/sunisup666/vibe-coding-pipeline.git ~/.claude/skills/vibe-coding-pipeline
```

Windows PowerShell:

```powershell
git clone https://github.com/sunisup666/vibe-coding-pipeline.git "$HOME\.claude\skills\vibe-coding-pipeline"
```

**Codex:** 换成 `~/.codex/skills/`,SKILL.md 格式通用。

> 用 `git clone` 装,是因为提示词自优化(回流)要往这个目录 `commit && push`——它必须是你自己的可写 git 仓库。用只读分享/打包方式装的话回流写不动。装完重启 Claude Code。骨架不用预装,skill 运行时联网从 `project-skeletons` 拉。

## 用法

```bash
cd ~/projects/my-new-idea      # 任意空目录 = 一个新项目
claude                          # 跟它说你的想法
```

skill 会:一次推进一步 → 每步生成后先自检翻车点 → 呈现并指出重点审查项 → 你迭代到满意 → 文档写进 `./docs/`、第 2 步定语言后从 `project-skeletons` 拉对应骨架进项目根 → 满意后复盘、把可泛化改进写回 `references/prompts/`、给你看 diff 确认后由你 push。

## 隔天续做

新会话 context 为空,skill 不靠记忆:它读项目里的 `docs/` + `docs/_progress.md` 从磁盘重建进度,跟你对齐断点后继续。所以随时关窗,第二天 `cd` 回项目目录重新开聊即可。(Claude Code 原生 `claude -c` 也能续会话,但不稳定,本 skill 不依赖它。)

## 结构

```text
vibe-coding-pipeline/
├── SKILL.md                    # 引擎(手动改)
├── references/
│   ├── pitfalls.md             # 翻车点自检清单
│   ├── prompt-tuning.md        # 提示词复盘与回流方法
│   └── prompts/00-06*.md       # 7 步提示词(数据,自动优化)
└── README.md                   # skill简介
```

## 骨架仓库

多语言骨架在独立仓库 `https://github.com/sunisup666/project-skeletons.git`(python-fastapi / java-springboot / go-standard / c-cmake / rust-cargo),每种语言一套业内标准结构,自带 `make verify` 且开箱绿。skill 在第 2 步定语言后**联网浅拉**对应骨架进项目根。新增语言只动骨架仓库,不动本 skill。
