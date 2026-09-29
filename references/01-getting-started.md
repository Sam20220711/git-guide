# Getting Started

## About Version Control

版本控制（version control）是一种记录文件随时间变化、以便日后可以回溯特定版本的系统。它能让你把文件恢复到之前的状态、比较不同版本之间的差异、查看是谁在何时改动了哪一行，以及定位是谁引入了问题。对于几乎任何类型的项目，使用版本控制都是明智的选择。

Version control is a system that records changes to a file or set of files over time so that you can recall specific versions later. It lets you revert files to a previous state, compare changes over time, see who last modified something, and see who introduced an issue. Using a version control system is generally a wise choice for almost any project.

版本控制经历了三代演进：本地版本控制（Local VCS）用数据库记录文件的差异；集中式版本控制（Centralized VCS，如 CVS、Subversion、Perforce）在单台服务器上保存所有版本，便于多人协作，但存在单点故障；分布式版本控制（Distributed VCS，如 Git、Mercurial）让每个客户端都完整镜像整个仓库，任何一处副本丢失都可以用任意客户端克隆恢复，天然支持多种协作工作流。

Version control evolved through three generations: local version control records file deltas in a local database; centralized version control (CVCS, such as CVS, Subversion, Perforce) keeps all versions on a single server for easier collaboration but has a single point of failure; distributed version control (DVCS, such as Git, Mercurial) gives every client a full mirror of the repository, so any lost copy can be restored by cloning from any client, naturally supporting multiple collaboration workflows.

本节以概念为主，没有直接可执行的命令；下面用一个典型场景说明“为什么需要版本控制”。

This section is conceptual and has no direct commands; the following scenario illustrates why version control matters.

示例：你在写一份文档，改坏了却无法撤销、也记不清改了什么。引入版本控制后，每一次有意义的改动都会被记录，你可以随时回到任意历史版本，或查看每次改动的原因。

Example: You are editing a document, break it, and cannot undo or recall what changed. With version control, each meaningful change is recorded, so you can return to any historical version at any time or see the reason behind each change.

⚠️ 注意：集中式版本控制存在单点故障——服务器一旦损坏且无备份，历史可能全部丢失；分布式版本控制（如 Git）通过让每个客户端持有完整副本规避了该风险。选择工具时优先考虑仓库的冗余与恢复能力。

⚠️ Note: centralized VCS has a single point of failure — if the server is damaged without backups, history may be lost entirely; distributed VCS (such as Git) avoids this by keeping a full copy on every client. Prefer tools that offer redundancy and recoverability.

> 出处 About Version Control：https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control

## A Short History of Git

Git 诞生于 Linux 内核开发社区。1991–2002 年间内核补丁通过补丁包与归档文件传递；2002 年起改用专有的分布式版本控制系统 BitKeeper；2005 年 BitKeeper 与内核社区的合作破裂，BitKeeper 的免费使用权被收回。于是 Linus Torvalds 决定开发自己的系统，Git 由此诞生。

Git was born in the Linux kernel development community. From 1991 to 2002 kernel patches were shared as patch files and archives; in 2002 the project moved to the proprietary distributed system BitKeeper; in 2005 the relationship between BitKeeper and the kernel community broke down and the free usage was revoked. Linus Torvalds then decided to build his own system, and Git was born.

Git 的设计目标包括：速度（speed）、简单的设计（simple design）、对非线性开发即分支的强支持（strong support for non-linear development）、完全分布式（fully distributed），以及能高效处理像 Linux 内核这样的大项目。自 2005 年诞生以来，Git 迅速成熟并成为最流行的版本控制系统之一。

Git's design goals include: speed, simple design, strong support for non-linear development (branching), being fully distributed, and handling large projects like the Linux kernel efficiently. Since 2005, Git quickly matured and became one of the most popular version control systems.

本节为背景介绍，没有可执行的命令。理解这段历史有助于记住 Git 的核心设计取向：快、分支友好、分布式。

This section is background and has no commands to run. Knowing this history helps you remember Git's core design priorities: fast, branch-friendly, and distributed.

示例：Git 从一开始就把“分支”当作一等公民，这使得隔离开发、功能分支、合并等工作流都非常高效，这也是它区别于许多早期版本控制系统的关键。

Example: Git treated "branching" as a first-class feature from the start, making workflows like isolated development, feature branches, and merging highly efficient — a key difference from many earlier VCS.

⚠️ 注意：Git 术语里常出现“snapshot”“commit”“branch”等词，初学者容易与 SVN 等工具的概念混淆。后续章节会逐一澄清，建议先建立“Git 以快照而非差异为核心”的直觉。

⚠️ Note: Git terminology frequently uses "snapshot", "commit", "branch", which beginners often confuse with concepts from tools like SVN. Later sections clarify them; it helps to first build the intuition that Git is centered on snapshots, not diffs.

> 出处 A Short History of Git：https://git-scm.com/book/en/v2/Getting-Started-A-Short-History-of-Git

## What is Git?

理解 Git 的关键在于三点：其一，Git 把数据视为“快照流”（stream of snapshots）而非“差异”（differences）。每次提交（commit）时，Git 为所有文件拍下快照，并保存一个指向该快照的引用；未变化的文件不重复存储，而是链接到之前已存的相同文件。

The key to understanding Git lies in three points. First, Git treats data as a "stream of snapshots" rather than "differences". On each commit, Git takes a snapshot of all files and stores a reference to it; unchanged files are not stored again but linked to the previously stored identical file.

其二，几乎所有的操作都在本地完成。因为整个仓库历史都在本地磁盘上，查看历史、比较差异、提交等操作几乎无需联网，速度极快，也支持离线工作。

Second, nearly every operation is local. Because the whole repository history lives on local disk, viewing history, comparing changes, and committing require almost no network, are very fast, and work offline.

其三，Git 用 SHA-1 哈希保证完整性。所有内容在存入前都会计算校验和（checksum），并以哈希值作为引用。这意味着任何改动都能被检测到，数据不会被悄无声息地破坏。

Third, Git ensures integrity with SHA-1 hashes. Everything stored is checksummed before it is saved and referenced by that hash, so any change is detectable and data cannot be silently corrupted.

Git 有三种状态与三个区域：已提交（committed，数据已安全存入本地数据库）、已修改（modified，文件已改动但尚未提交）、已暂存（staged，改动已标记将进入下一次提交）。对应区域为：工作区（working tree）、暂存区（staging area）、Git 仓库目录（.git directory）。

Git has three states and three areas: committed (data safely stored in the local database), modified (files changed but not yet committed), and staged (changes marked to go into the next commit). The corresponding areas are the working tree, the staging area, and the .git directory.

```
# 查看文件当前处于哪种状态
git status
# 把工作区的改动放入暂存区（modified -> staged）
git add <file>
# 把暂存区内容提交为一次快照（staged -> committed）
git commit -m "describe the change"
```

- `git status`：显示工作区与暂存区的状态，是理解三状态的入口。Shows the state of the working tree and staging area — the entry point to the three states.
- `git add <file>`：把改动加入暂存区，只做“标记”，不改历史。Stages changes without altering history.
- `git commit -m "..."`：把暂存区内容写入本地数据库，形成一次提交。Writes the staged content into the local database as a commit.

示例：修改文件后，先用 `git status` 看到它处于 modified 状态；执行 `git add` 后再次 `git status` 可见其变为 staged；最后 `git commit` 后变为 committed。

Example: after modifying a file, `git status` shows it as modified; after `git add`, `git status` shows it staged; after `git commit`, it becomes committed.

⚠️ 注意：`git add` 并不会把内容写入历史，只有 `git commit` 才会。若暂存后想撤销某文件的暂存，可用 `git restore --staged <file>`（较新版本）或 `git reset HEAD <file>`，这两个命令只影响暂存区，不会丢失工作区的改动。真正的历史数据都保存在 `.git` 目录中，切勿随意手动删除该目录。

⚠️ Note: `git add` does not write content into history; only `git commit` does. To unstage a file after staging, use `git restore --staged <file>` (newer Git) or `git reset HEAD <file>` — both affect only the staging area and never lose your working-tree changes. All real history lives in the `.git` directory; never delete it casually by hand.

> 出处 What is Git?：https://git-scm.com/book/en/v2/Getting-Started-What-is-Git

## The Command Line

Git 可以通过图形界面（GUI）工具或命令行使用。图形工具通常只暴露部分功能，而命令行能访问 Git 的全部能力，因此本书以命令行为主线；学会命令行后，再使用任何图形工具都会更容易。

Git can be used through GUI tools or the command line. GUIs usually expose only a subset of features, while the command line gives access to all of Git's capabilities, so this book uses the command line as the primary interface; once you learn the command line, using any GUI becomes easier.

```
# 确认 Git 已安装并查看版本
git --version
# 查看所有顶层命令及简要说明
git --help
```

- `git --version`：输出已安装的 Git 版本，用于确认环境可用。Prints the installed Git version to confirm the environment works.
- `git --help`：列出所有顶层命令并附简短说明。Lists all top-level commands with short descriptions.

示例：在终端中运行 `git --version`，若看到类似 `git version 2.x.x` 的输出，说明命令行可用；随后运行 `git --help` 可快速浏览命令列表。

Example: run `git --version` in a terminal; output like `git version 2.x.x` means the command line works; then `git --help` gives a quick overview of commands.

⚠️ 注意：不同的 shell（如 Bash、PowerShell、zsh）对引号与特殊字符的处理略有不同，示例命令中的引号在 Windows 的 cmd 或 PowerShell 下可能需要调整；建议在一致的环境中执行示例，遇到报错先核对 shell 差异。

⚠️ Note: different shells (Bash, PowerShell, zsh) treat quotes and special characters slightly differently; the quoting in examples may need adjustment under Windows cmd or PowerShell. Run examples in a consistent environment and check shell differences first when you see errors.

> 出处 The Command Line：https://git-scm.com/book/en/v2/Getting-Started-The-Command-Line

## Installing Git

安装前先确认是否已安装：若 `git --version` 能正常输出版本，则可跳过安装步骤。各平台安装方式如下。

Before installing, check whether Git is already present: if `git --version` prints a version, you can skip installation. Installation methods by platform follow.

Linux：推荐使用发行版自带的包管理器安装，例如 Debian/Ubuntu 使用 `apt`，Fedora 使用 `dnf`，Arch 使用 `pacman`。macOS：可通过 Xcode Command Line Tools、Git 官方安装器，或 Homebrew（`brew install git`）安装。Windows：推荐从 git-scm.com 下载官方“Git for Windows”，也可使用 `winget install Git.Git`。

Linux: prefer your distribution's package manager, e.g. `apt` on Debian/Ubuntu, `dnf` on Fedora, `pacman` on Arch. macOS: install via Xcode Command Line Tools, the official Git installer, or Homebrew (`brew install git`). Windows: download the official "Git for Windows" from git-scm.com, or use `winget install Git.Git`.

```
# Debian / Ubuntu
sudo apt update && sudo apt install git
# Fedora
sudo dnf install git
# macOS（Homebrew）
brew install git
# Windows（winget）
winget install Git.Git
```

- `sudo apt update && sudo apt install git`：先刷新软件源索引，再安装 git。Refresh package indexes, then install git.
- `sudo dnf install git`：在 Fedora 上安装 git。Install git on Fedora.
- `brew install git`：在 macOS 上通过 Homebrew 安装 git。Install git via Homebrew on macOS.
- `winget install Git.Git`：在 Windows 上通过 winget 安装 Git for Windows。Install Git for Windows via winget on Windows.

示例：在 Debian/Ubuntu 上执行 `sudo apt install git` 后，运行 `git --version` 验证安装成功；在 Windows 上运行安装程序时，可保留默认选项，完成后再用 `git --version` 确认。

Example: after `sudo apt install git` on Debian/Ubuntu, run `git --version` to verify; on Windows, keep default options in the installer, then confirm with `git --version`.

⚠️ 注意：从源码编译安装虽然能获得最新版本，但步骤繁琐且易出错，新手建议使用官方包或安装器。使用 `sudo` 或管理员权限安装时，请确认命令来源可信，不要盲目复制网上的安装脚本。安装后可运行 `git --version` 与 `git --help` 双重复核。

⚠️ Note: compiling from source gives the newest version but is tedious and error-prone; beginners should prefer official packages or installers. When using `sudo` or admin privileges, verify the command source and avoid blindly pasting install scripts from the internet. Double-check with `git --version` and `git --help` after installing.

> 出处 Installing Git：https://git-scm.com/book/en/v2/Getting-Started-Installing-Git

## First-Time Git Setup

首次使用 Git 前需要配置用户身份等信息。`git config` 可以读取和写入配置，配置作用于三个层级：`--system`（系统级，对所有用户生效）、`--global`（用户级，对该用户的全部仓库生效）、`--local`（仓库级，仅当前仓库，也是默认层级）。配置优先级从高到低为：local → global → system。

Before using Git for the first time, configure your identity and other settings. `git config` reads and writes configuration at three levels: `--system` (system-wide, all users), `--global` (user-wide, all repositories of that user), and `--local` (repository-level, current repo only, also the default). Priority from high to low is local → global → system.

最重要的两项配置是用户名和邮箱，它们会写入每一次提交。此外还常配置文本编辑器（用于输入提交说明）与默认分支名。

The two most important settings are your user name and email, which are recorded in every commit. Also commonly configured are the text editor (for typing commit messages) and the default branch name.

```
# 设置用户名与邮箱（对所有仓库生效）
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
# 设置默认文本编辑器
git config --global core.editor "code --wait"
# 查看全部配置
git config --list
# 查看某一项配置
git config user.name
```

- `git config --global user.name "Your Name"`：设置全局用户名，写入每次提交。Set the global user name recorded in every commit.
- `git config --global user.email "you@example.com"`：设置全局邮箱。Set the global email address.
- `git config --global core.editor "code --wait"`：把编辑器设为 VS Code（`--wait` 让 Git 等待编辑器关闭）。Set the editor to VS Code (`--wait` makes Git wait for the editor to close).
- `git config --list`：列出当前生效的所有配置项。List all effective configuration entries.
- `git config user.name`：查看某项配置，可看到不同层级合并后的最终取值。Show one config value as resolved across levels.

示例：新机器上先执行两条 `--global` 配置设置身份，再运行 `git config --list` 确认 `user.name` 与 `user.email` 已写入；提交一次后，可用 `git log` 看到作者信息来自这两项配置。

Example: on a new machine, run the two `--global` identity commands first, then `git config --list` to confirm `user.name` and `user.email` are set; after a commit, `git log` shows the author taken from these settings.

⚠️ 注意：`--global` 会作用于该用户的所有仓库，`--system` 会作用于整台机器的所有用户且通常需要管理员权限，请谨慎使用，不要随意改动系统级配置。若配置写错了，可用 `git config --global --unset <key>` 移除某一项，或直接编辑 `~/.gitconfig`（用户级）进行回退。邮箱建议使用与托管平台一致且长期有效的地址。

⚠️ Note: `--global` affects all repositories of the user, and `--system` affects all users on the machine and usually needs admin rights — use them cautiously and don't casually modify system-level settings. If a value is wrong, remove it with `git config --global --unset <key>`, or edit `~/.gitconfig` (user level) to roll back. Prefer a long-lived email that matches your hosting platform.

> 出处 First-Time Git Setup：https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup

## Getting Help

遇到不确定的命令或选项时，Git 内置了多种帮助方式，无需依赖网络即可查阅。

When you are unsure about a command or option, Git provides several built-in ways to get help without the network.

```
# 打开某条命令的完整手册页
git help <verb>
# 等价写法
git <verb> --help
# 查看该命令的简明选项列表
git <verb> -h
# 例如
git help config
git config -h
```

- `git help <verb>`：打开指定命令的完整帮助手册。Open the full manual for a command.
- `git <verb> --help`：与 `git help <verb>` 等价。Equivalent to `git help <verb>`.
- `git <verb> -h`：输出该命令的简明用法与常用选项。Print concise usage and common options.
- `git help config` / `git config -h`：以 config 命令为例，分别查看手册与简要选项。Examples for the config command: full manual vs. brief options.

示例：忘记 `git add` 的某个选项时，运行 `git add -h` 快速查看选项列表；需要完整说明时运行 `git help add`。

Example: when you forget an option of `git add`, run `git add -h` for a quick option list; run `git help add` for full documentation.

⚠️ 注意：`-h` 与 `--help` 并不相同——`-h` 给出简要选项，`--help` 打开完整手册。若手册页无法显示（例如 Windows 上未配置 man），可改用 `git <verb> -h` 或查阅在线文档 https://git-scm.com/doc。

⚠️ Note: `-h` and `--help` differ — `-h` gives brief options while `--help` opens the full manual. If the manual page won't render (e.g. man is not configured on Windows), use `git <verb> -h` or the online docs at https://git-scm.com/doc.

> 出处 Getting Help：https://git-scm.com/book/en/v2/Getting-Started-Getting-Help
