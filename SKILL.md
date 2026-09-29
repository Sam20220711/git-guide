---
name: git-guide
description: 基于 Pro Git 官方文档（https://git-scm.com/book/en/v2），针对用户描述的 Git 使用场景，给出具体操作命令、分步说明、示例与风险提示（中英对照）。Use this skill when the user asks how to do something with Git — identify the scenario, then provide concrete commands, step-by-step explanations, examples, and risk notes sourced from the Pro Git book, in bilingual Chinese/English form.
---

# Git Guide（Git 操作场景指南）

根据用户描述的 Git 使用场景，给出具体、可执行、带风险提示的操作方法与说明。
Give concrete, runnable, risk-annotated Git commands and explanations for whatever Git scenario the user describes.

## 何时使用 / When to use

- 用户问「怎么做某件 Git 操作」（提交、分支、合并、撤销、回退、变基、stash、打标签、推送、冲突解决、改写历史、找回丢失提交……）。
- 用户描述了一个目标或一个出问题的现状，需要对应的 Git 命令与解释。
- 需要权威出处时（指向 Pro Git 官方章节）。

## 工作流程 / Workflow

按以下顺序处理，一次只做一步：

1. **识别场景 / Identify the scenario**
   理解用户想达成什么（目标）以及当前状态（分支、是否已提交、是否已推送、单人还是多人协作）。

2. **澄清 / Clarify first**
   信息不足或含糊时，先简短确认关键点再作答，不要凭空假设。典型确认点：
   - 当前分支状态：`git status` 是否干净？改动是否已提交？
   - 是否已推送：改动是只在本地，还是已经 push 到远程？
   - 协作情况：是否已有人基于该分支/提交工作？
   - 目标分支：要合并/变基到哪个分支？

3. **匹配知识 / Match knowledge**
   用 `references/scenarios-index.md` 定位到对应章节，再读对应的 `references/NN-*.md` 取准确命令与参数。找不到现成条目时，依据 Pro Git 章节判断最贴近的章节。

4. **按模板回答 / Answer with the template**
   严格套用下方「回答模板」。

5. **危险操作防护 / Guard destructive operations**
   涉及不可逆/危险操作时，必须先给出安全替代或回退手段，再给命令。

## 回答模板 / Answer template

每个场景的回答按以下 6 段组织（段落正文用中英对照段：中文一段 + English paragraph）：

### 🎯 目标 / Goal
用一句话说明本场景要达成的结果，并点明前置条件。
State the intended outcome and its preconditions in one line.

### ✅ 前置检查 / Pre-flight checks
给出动手前应执行的检查命令（如 `git status`、`git log --oneline`、`git branch -vv`），并说明要确认什么。
List the read-only checks to run first and what to verify in their output.

### 🔧 操作步骤 / Steps
按顺序给出命令（代码块，命令本身不翻译），每条命令下用一句中文 + 一句英文说明它做什么。
Ordered commands, each with a one-line Chinese and one-line English explanation.

### 📝 示例 / Example
给出贴合用户场景的完整示例（假设仓库/分支名），展示输入与预期结果。
A concrete worked example matching the user's scenario.

### ⚠️ 注意事项 / Risks & notes
风险点、不可逆操作的后果、以及回退/安全替代方案（如 `git reflog`、`git stash`、`--force-with-lease`）。涉及 `reset --hard`、`rebase`、`force push`、`clean -f`、删除分支/标签时必须重点标注。
Risks, consequences of irreversible commands, and safer alternatives or recovery paths.

### 📚 出处 / Source
附 Pro Git 章节名与官方链接：
`> 出处 <Section Title>：https://git-scm.com/book/en/v2/<slug>`
Link the Pro Git section this answer draws from.

## 危险命令清单与安全替代 / Destructive commands and safer alternatives

| 危险命令 | 风险 | 安全替代 / 回退 |
|---|---|---|
| `git reset --hard` | 丢弃工作区与暂存区改动，不可恢复 | 先 `git stash` 或 `git reset --soft`；用 `git reflog` 找回 |
| `git push --force` | 覆盖远程历史，坑害协作者 | `git push --force-with-lease` |
| `git rebase`（已共享分支） | 改写已推送历史 | 只对未推送的本地分支使用；推送用 `--force-with-lease` |
| `git clean -fd` | 永久删除未跟踪文件 | 先 `git clean -nd` 预览 |
| `git branch -D` / `git tag -d` | 删除分支/标签 | 先确认该提交有其它引用；用 reflog 找回 |
| `git checkout -- .` | 丢弃工作区改动 | 先 `git stash` 或 `git diff` 查看 |
| `git filter-branch` / `filter-repo` | 重写大量历史 | 先克隆备份；优先用 `git filter-repo` |

原则：任何会丢失数据或改写已共享历史前，先做一次只读检查，并给出回退手段。
Rule: before anything that can lose data or rewrite shared history, run a read-only check first and state the recovery path.

## 章节索引 / Reference index

| 文件 | 内容 |
|---|---|
| `references/01-getting-started.md` | 版本控制概念、安装、`git config` 首次配置、`git help` |
| `references/02-git-basics.md` | `init`/`clone`、`add`/`commit`、`status`/`diff`、`log`、撤销、远程、标签、别名 |
| `references/03-branching.md` | 分支、合并、冲突、分支管理、工作流、远程分支、rebase |
| `references/04-server.md` | 协议、裸仓库、SSH 密钥、搭建服务器、GitLab |
| `references/05-distributed.md` | 分布式工作流、贡献者/维护者命令流（format-patch、apply、am、cherry-pick） |
| `references/06-github.md` | fork、Pull Request、gh CLI、组织管理、脚本化 |
| `references/07-git-tools-a.md` | 选择修订、交互式暂存、stash/clean、签名、搜索 |
| `references/07-git-tools-b.md` | 改写历史、reset 三棵树、高级合并、rerere、bisect、子模块、凭据存储 |
| `references/08-customizing.md` | 三级配置、.gitattributes、hooks、策略 |
| `references/09-other-systems.md` | git svn、迁移到 Git |
| `references/10-internals.md` | 对象模型、引用、packfile、refspec、数据恢复、gc |
| `references/11-appendix.md` | GUI/编辑器集成、嵌入 Git（libgit2 等）、命令速查 |

## 通用规则 / General rules

- 命令与参数保留英文原样，不翻译；说明性文字用中英对照段。
- Keep commands and flags in English; write explanatory prose as paired Chinese/English paragraphs.
- 先只读命令（`status`/`diff`/`log`/`reflog`），后写命令。
- Prefer read-only commands before any write command.
- 引用命令时尽量给出常用选项，并解释选项含义。
- When citing a command, include its common options and explain what each does.
- 遇到不确定的冷门内容，可联网抓取官方文档对应章节核对后再答。
- For rare or uncertain topics, fetch the official section first and verify before answering.
