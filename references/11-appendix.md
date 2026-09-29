# Appendices（附录：环境集成 / 嵌入 Git / 命令速查）

本文件覆盖 Pro Git 附录 A、B、C，内容从简，按需展开。
This file covers Pro Git appendices A, B, and C in condensed form.

## A. Git in Other Environments（其他环境中的 Git）

中英对照段说明：Git 除了命令行，还内置在众多 GUI 与编辑器中，操作等价、界面不同。
English: Beyond the command line, Git ships inside many GUIs and editors — the operations are equivalent, only the interface differs.

### Graphical Interfaces（图形界面）

- 常见 GUI：GitHub Desktop、GitKraken、SourceTree、Git 自带的 `gitk` / `git gui`。
- 命令行仍是理解底层与排障的最佳方式；GUI 适合可视化历史与冲突。

> 出处 Graphical Interfaces：https://git-scm.com/book/en/v2/Git-in-Other-Environments-Graphical-Interfaces

### 编辑器集成（VS Code / Visual Studio / JetBrains / Sublime Text）

- VS Code：内置源码管理面板 + GitLens 扩展。
- JetBrains IDE：内置 VCS 菜单（commit / log / merge / rebase）。
- 关键点：编辑器里的操作最终都对应到命令行命令，出问题时回到命令行核对。

> 出处：https://git-scm.com/book/en/v2/Git-in-Other-Environments-Visual-Studio-Code 、https://git-scm.com/book/en/v2/Git-in-Other-Environments-Visual-Studio

### Shell 集成（Bash / Zsh / PowerShell）

- Bash：`git-completion.bash` 与 `git-prompt.sh` 提供补全与分支提示。
- Zsh：`vcs_info` / oh-my-zsh 的 git 插件。
- PowerShell：`posh-git` 提供提示与补全。

> 出处 Git in Bash：https://git-scm.com/book/en/v2/Git-in-Other-Environments-Git-in-Bash
> 出处 Git in Zsh：https://git-scm.com/book/en/v2/Git-in-Other-Environments-Git-in-Zsh
> 出处 Git in PowerShell：https://git-scm.com/book/en/v2/Git-in-Other-Environments-Git-in-PowerShell

## B. Embedding Git in your Applications（在应用中嵌入 Git）

中英对照段说明：若要在自己程序里调用 Git，可选择命令行、libgit2、JGit、go-git、Dulwich 等库。
English: To call Git from your own program, choose among the command line, libgit2, JGit, go-git, or Dulwich.

- **Command-line Git**：`git <cmd>` 子进程，最通用但需解析输出。
- **libgit2**：C 库，绑定到多种语言；GitHub 等大量工具基于它。
- **JGit**：纯 Java 实现，用于 JGit/Eclipse。
- **go-git**：纯 Go 实现。
- **Dulwich**：纯 Python 实现。

> 出处 Libgit2：https://git-scm.com/book/en/v2/Embedding-Git-in-your-Applications-Libgit2
> 出处 JGit：https://git-scm.com/book/en/v2/Embedding-Git-in-your-Applications-JGit

## C. Git Commands（命令速查 / Command quick reference）

按用途分组，每条为「命令 — 一句话用途」。日常答疑优先引用前四组。
Grouped by purpose; prefer the first four groups for daily Q&A.

### 设置与配置 / Setup & config
- `git config` — 读写配置
- `git help` — 查看帮助

### 获取与创建 / Getting & creating
- `git init` — 初始化仓库
- `git clone` — 克隆仓库

### 基础快照 / Basic snapshotting
- `git add` — 暂存改动
- `git status` — 查看状态
- `git diff` — 查看差异
- `git commit` — 提交
- `git reset` — 撤销/移动指针
- `git rm` / `git mv` — 删除/移动文件
- `git stash` — 贮藏改动

### 分支与合并 / Branching & merging
- `git branch` — 管理分支
- `git checkout` / `git switch` — 切换分支
- `git merge` — 合并
- `git mergetool` — 冲突解决工具
- `git log` — 历史
- `git tag` — 标签
- `git rebase` — 变基

### 共享与更新 / Sharing & updating
- `git fetch` — 拉取远程引用
- `git pull` — 拉取并合并
- `git push` — 推送
- `git remote` — 管理远程

### 检查与比较 / Inspection & comparison
- `git show` — 显示对象
- `git shortlog` / `git describe` — 汇总/描述

### 调试 / Debugging
- `git bisect` — 二分定位
- `git blame` — 逐行追溯
- `git grep` — 在仓库中搜索

### 补丁 / Patching
- `git cherry-pick` — 挑选提交
- `git format-patch` / `git apply` / `git am` — 生成/应用补丁
- `git revert` — 反向提交

### 管理 / Administration
- `git gc` — 垃圾回收
- `git fsck` — 校验对象
- `git reflog` — 引用日志（找回丢失提交）

> 出处 Git Commands：https://git-scm.com/book/en/v2/  （命令按章节汇总于全书各章末尾）
