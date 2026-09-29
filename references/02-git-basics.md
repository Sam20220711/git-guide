# Git Basics

本章涵盖 Git 日常使用的核心工作流：如何获取仓库、记录变更、查看历史、撤销操作、管理远程与标签，以及如何用别名提高效率。每个概念先给出中文说明，再给出对等英文说明；命令一律放入代码块且只写英文。

This chapter covers the core workflows you use every day with Git: how to get a repository, record changes, view history, undo things, manage remotes and tags, and speed things up with aliases. Each concept is explained in Chinese first and then in equivalent English; commands always appear in code blocks and are written in English only.

---

## Getting a Git Repository

获取 Git 仓库有两种方式：在本地现有目录中初始化一个全新的仓库，或克隆一个已存在的远程仓库。初始化会创建一个 `.git` 子目录，其中保存所有版本元数据；克隆则会自动把源仓库配置为名为 `origin` 的远程。

There are two ways to get a Git repository: initialize a brand-new repository in an existing local directory, or clone an already existing remote repository. Initializing creates a `.git` subdirectory that stores all the version metadata; cloning automatically configures the source repository as a remote named `origin`.

### 命令 Commands

```bash
# 在当前目录初始化新仓库
git init

# 初始化并指定目录名
git init <directory>

# 克隆远程仓库到当前目录下的同名文件夹
git clone <url>

# 克隆并重命名本地目录
git clone <url> <local-directory>
```

- `git init`：在当前目录建立 `.git`，此时文件尚未被跟踪。Creates `.git` in the current directory; files are not tracked yet.
- `git clone <url>`：完整复制仓库的全部历史与每个文件的每个版本。Copies the full history and every version of every file.
- 克隆支持的协议：`https://`、`git://`、`ssh://`（`user@server:path`）以及本地路径。Clone supports these protocols: `https://`, `git://`, `ssh://` (`user@server:path`), and local paths.

### 示例 Example

```bash
git clone https://github.com/libgit2/libgit2 mylibgit
```

该命令把 `libgit2` 仓库克隆到本地目录 `mylibgit`，并自动设置好 `origin` 远程。This clones the `libgit2` repository into a local directory `mylibgit` and sets up the `origin` remote automatically.

### ⚠️ 注意事项 Notes

- `git clone` 不需要先在本地 `git init`，二者会重复。You do not need to run `git init` before `git clone`; doing both is redundant.
- 克隆会带下所有历史，仓库可能很大；若只想要浅历史可加 `--depth 1`（浅克隆）。A clone pulls the entire history and can be large; add `--depth 1` for a shallow clone when you only need recent history.

> 出处 Getting a Git Repository：https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository

---

## Recording Changes to the Repository

Git 中的每个文件都处于四种状态之一：未跟踪（untracked）、已修改（modified）、已暂存（staged）、未修改（unmodified）。工作区（working tree）是你在磁盘上看到的文件，暂存区（staging area）是提交前的中转站，提交（commit）是永久保存的快照。典型的提交流程是：修改文件 → `git add` 暂存 → `git commit` 提交。

Each file in Git is in one of four states: untracked, modified, staged, or unmodified. The working tree is what you see on disk, the staging area is the holding place before a commit, and a commit is a permanently saved snapshot. The typical flow is: modify files → stage with `git add` → commit with `git commit`.

### 命令 Commands

```bash
git status                 # 查看文件状态
git status -s              # 简短状态（两列：暂存区 + 工作区）
git add <file>             # 暂存某个文件
git add .                  # 暂存当前目录所有变更
git add -p                 # 交互式分块暂存
git diff                   # 工作区 vs 暂存区的差异（未暂存的改动）
git diff --staged          # 暂存区 vs 上一次提交的差异（已暂存的改动）
git commit -m "message"    # 提交暂存内容
git commit -a -m "message" # 跳过 add，直接提交所有已跟踪文件的修改
git rm <file>              # 从仓库和工作区删除文件
git rm --cached <file>     # 只从暂存区移除，保留工作区文件
git mv <old> <new>         # 重命名/移动文件（等价于 rm + add）
```

- `git status`：查看哪些文件被跟踪、哪些已暂存。Shows which files are tracked and which are staged.
- `git add`：把变更加入暂存区；它是多用途命令，也用于标记合并冲突已解决。Stages changes; it is multipurpose and also marks merge conflicts as resolved.
- `git diff --staged`（旧写法 `git diff --cached`）：查看将要提交的内容。Shows exactly what will be committed (older form: `git diff --cached`).
- `git commit -a`：只暂存并提交“已跟踪”文件的修改，不包含新文件（untracked）。Stages and commits changes to already-tracked files only; new untracked files are not included.

### 示例 Example

```bash
git add README.md
git commit -m "Add project readme"
git status -s
# M  README.md   （M 在第一列 = 已暂存，第二列 = 工作区已修改）
```

### ⚠️ 注意事项 Notes

- `.gitignore` 用于忽略不想跟踪的文件（日志、编译产物、密钥等）；已跟踪的文件需要先 `git rm --cached` 才能被忽略。Use `.gitignore` to ignore files you don't want tracked (logs, build artifacts, secrets); already-tracked files must be removed with `git rm --cached` first.
- 提交前养成 `git diff --staged` 的习惯，避免把意外内容提交进去。Review `git diff --staged` before committing to avoid committing unintended content.
- ⚠️ `git rm <file>` 会同时删除工作区文件且无法从文件系统恢复；若只想停止跟踪但保留文件，用 `git rm --cached <file>`。`git rm <file>` deletes the working-tree file irrecoverably from the filesystem; to only untrack while keeping the file, use `git rm --cached <file>`.

> 出处 Recording Changes to the Repository：https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository

---

## Viewing the Commit History

`git log` 是查看提交历史的主力命令，拥有大量选项用于筛选和格式化输出。理解各选项能让你快速定位某次提交、某个作者或某个文件的历史。

`git log` is the workhorse for inspecting commit history, with many options to filter and format output. Understanding the options lets you quickly locate a specific commit, author, or a file's history.

### 命令 Commands

```bash
git log                     # 默认：倒序列出全部提交
git log -p -2               # 显示最近 2 次提交及其完整补丁（diff）
git log --stat              # 显示每次提交的改动统计
git log --oneline           # 每行一个提交，精简输出
git log --pretty=format:"%h %an %ar %s"   # 自定义格式
git log --graph             # 用 ASCII 图形展示分支与合并历史
git log --since=2.weeks     # 按时间筛选（--until 同理）
git log -S "function_name"  # 查找某字符串增删的提交
git log --author="Alice"    # 按作者筛选
git log --grep="fix"        # 按提交信息筛选
git log --all --decorate --oneline --graph   # 全分支可视化
```

- `git log -p`：显示每次提交引入的补丁；`-2` 限制只显示两条。Shows the patch each commit introduces; `-2` limits output to two entries.
- `--pretty=format` 常用占位符：`%h`（短哈希）、`%an`（作者名）、`%ar`（相对时间）、`%s`（标题）。Common placeholders for `--pretty=format`: `%h` (abbreviated hash), `%an` (author name), `%ar` (relative date), `%s` (subject).
- `--graph`：以文本图形展示分支分叉与合并。Draws a text-based graph of branching and merging.

### 示例 Example

```bash
git log --oneline --graph --all -n 10
# * 3f2a1c1 (HEAD -> main) Merge feature branch
# |\
# | * 8d5c0e2 Add login page
# * | 1b9d4f7 Update README
```

### ⚠️ 注意事项 Notes

- `git log` 是只读操作，永远安全。`git log` is read-only and always safe.
- 组合 `--graph --decorate --all` 是理解分支结构的常用起点；很多用户把它配成别名。Combining `--graph --decorate --all` is a common starting point for understanding branch topology; many users alias it.

> 出处 Viewing the Commit History：https://git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History

---

## Undoing Things

撤销是 Git 中最需要谨慎的部分，因为部分操作会丢弃内容。总体原则：只要改动还在暂存区或已提交，多数情况都能通过 reflog 找回；一旦丢弃了“未提交且未暂存”的工作区改动，通常无法恢复。

Undoing is where you must be most careful in Git, because some operations discard content. The general rule: as long as changes are staged or committed, most things can be recovered via the reflog; but once you discard working-tree changes that were neither committed nor staged, they are usually gone for good.

### 命令 Commands

```bash
git commit --amend              # 修改上一次提交（补充暂存内容或改提交信息）
git restore <file>              # 丢弃工作区改动，还原到暂存区/HEAD 版本
git restore --staged <file>     # 取消暂存（旧写法 git reset HEAD <file>）
git reset --hard <commit>       # 将分支指针与工作区强制回到指定提交
git revert <commit>             # 新建一个“反向”提交来撤销某次提交
git reflog                      # 查看 HEAD 的移动历史，用于找回“丢失”的提交
```

- `git commit --amend`：把新的暂存内容并入上一次提交，或只改提交信息（不改内容时无需 add）。Amends the last commit with newly staged content, or just rewrites the message (no need to add if content is unchanged).
- `git restore <file>`：撤销工作区修改，等价于旧命令 `git checkout -- <file>`。Discards working-tree changes; equivalent to the older `git checkout -- <file>`.
- `git reset --hard`：同时移动分支指针并重写工作区，会删除之后提交的可达引用。Moves the branch pointer and rewrites the working tree, removing reachability of later commits.
- `git revert`：安全撤销，通过追加一个抵消提交来保留完整历史，适合已推送的分支。A safe undo that appends an inverse commit and preserves history, suitable for already-pushed branches.

### 示例 Example

```bash
# 修改提交信息（不改内容）
git commit --amend -m "Better message"

# 误用 reset --hard 后找回提交
git reflog
# 5f3a1c1 HEAD@{0}: reset: moving to HEAD~1
# 9d2b7e4 HEAD@{1}: commit: WIP feature
git reset --hard 9d2b7e4
```

### ⚠️ 注意事项 Notes

- ⚠️ `git restore <file>` 会永久丢弃工作区中未提交的改动，无法恢复；执行前先用 `git diff` 确认。`git restore <file>` permanently discards uncommitted working-tree changes and cannot be recovered; review with `git diff` first.
- ⚠️ `git reset --hard` 会移除提交并覆盖工作区，属于不可逆操作；误操作后用 `git reflog` 找回提交哈希再 `git reset --hard <hash>` 回退。`git reset --hard` removes commits and overwrites the working tree and is irreversible; after a mistake, find the commit hash with `git reflog` and recover with `git reset --hard <hash>`.
- ⚠️ 不要对已推送到共享仓库的提交使用 `git commit --amend` 或 `git reset`（会改写历史导致他人冲突）；已推送的改动请用 `git revert`。Do not use `git commit --amend` or `git reset` on commits already pushed to a shared repository (rewriting history causes conflicts for others); use `git revert` for pushed changes.

> 出处 Undoing Things：https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things

---

## Working with Remotes

远程仓库（remote）是托管在网络或本机其他位置上的仓库版本，用于协作与备份。默认克隆得到的远程名为 `origin`。与远程协作的核心动作是：拉取（fetch/pull）与推送（push）。

A remote is a version of your repository hosted elsewhere on the network or local machine, used for collaboration and backup. The default remote after a clone is named `origin`. The core remote actions are fetching/pulling and pushing.

### 命令 Commands

```bash
git remote                     # 列出远程名称
git remote -v                  # 列出远程及其 URL（fetch 与 push）
git remote add <name> <url>    # 添加远程
git remote show <name>         # 查看远程详情（分支、跟踪关系等）
git remote rename <old> <new>  # 重命名远程
git remote remove <name>       # 删除远程（旧写法 git remote rm）
git fetch <remote>             # 拉取远程数据，但不合并到当前分支
git pull                       # 等价于 fetch + merge（到当前分支）
git push <remote> <branch>     # 推送本地分支到远程
git push -u origin <branch>    # 首次推送并设置上游跟踪分支
```

- `git fetch`：下载远程新数据到本地远程引用，但不改动工作区，需要手动合并。Downloads new remote data into local remote refs but leaves your working tree untouched; you merge manually.
- `git pull`：`fetch` 之后自动把远程分支合并进当前分支。Fetches and then automatically merges the remote branch into your current branch.
- `git push -u`：`-u` 设置上游分支，之后可省略参数直接 `git push`/`git pull`。`-u` sets the upstream branch so later you can just run `git push`/`git pull`.

### 示例 Example

```bash
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git push -u origin main
```

### ⚠️ 注意事项 Notes

- `git fetch` 是安全的只读拉取；`git pull` 会合并，可能产生冲突。`git fetch` is a safe read-only fetch; `git pull` merges and may produce conflicts.
- `git remote remove` 只删除本地对远程的引用配置，不删除远程服务器上的仓库本身。`git remote remove` only deletes the local reference to the remote, not the repository on the server.

> 出处 Working with Remotes：https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes

---

## Tagging

标签（tag）用于给历史中的某个提交打上不可变（通常如此）的标记，最常用于标记发布版本（如 `v1.0.0`）。标签有两种：轻量标签（lightweight，仅一个指向提交的指针）与附注标签（annotated，保存为完整对象，含打标者、日期与说明，并支持 GPG 签名）。

Tags mark a specific commit in history as a fixed point, most commonly for releases (e.g. `v1.0.0`). There are two kinds: lightweight tags (just a pointer to a commit) and annotated tags (stored as full objects with tagger, date, message, and optional GPG signature).

### 命令 Commands

```bash
git tag                        # 列出所有标签
git tag -l "v1.*"              # 通配符筛选标签
git tag <tagname>              # 创建轻量标签
git tag -a <tagname> -m "msg"  # 创建附注标签
git tag -a <tagname> <commit>  # 为历史中的某提交补打标签
git show <tagname>             # 查看标签信息及其指向的提交
git push origin <tagname>      # 推送单个标签到远程
git push origin --tags         # 推送所有标签
git tag -d <tagname>           # 删除本地标签
git push origin --delete <tagname>   # 删除远程标签
```

- 附注标签是发布推荐的默认选择，因为保留了完整元数据。Annotated tags are the recommended default for releases because they retain full metadata.
- 标签默认不随 `git push` 推送，需要显式推送或使用 `--tags`。Tags are not pushed by default; push them explicitly or use `--tags`.

### 示例 Example

```bash
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0
git tag -l "v1.*"
```

### ⚠️ 注意事项 Notes

- ⚠️ 删除标签前确认它没有指向唯一的发布点；误删远程标签会让他人失去该版本引用，请先核对 `git tag -l` 输出。Before deleting a tag, confirm it isn't the sole release marker; deleting a remote tag removes that version reference for others, so check `git tag -l` output first.
- 检查某个标签对应的内容用 `git checkout <tagname>` 会进入“分离头指针”（detached HEAD）状态；若只是查看，用 `git show <tagname>` 更安全。Checking out a tag with `git checkout <tagname>` puts you in a detached HEAD state; use `git show <tagname>` if you only want to inspect it.

> 出处 Tagging：https://git-scm.com/book/en/v2/Git-Basics-Tagging

---

## Git Aliases

别名（alias）让你把常用或冗长的命令缩写成短命令，提升效率。别名通过 `git config` 定义，推荐写入全局配置（`--global`），这样对所有仓库生效。

Aliases let you shorten frequently used or verbose commands. They are defined via `git config`, and it is recommended to store them globally (`--global`) so they apply to every repository.

### 命令 Commands

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual '!gitk'   # 调用外部命令（以 ! 开头）
```

- 之后 `git co` 等价于 `git checkout`，`git st` 等价于 `git status`。Afterwards `git co` equals `git checkout`, `git st` equals `git status`.
- 以 `!` 开头的别名会作为 shell 外部命令执行，可调用 Git 之外的命令。Aliases beginning with `!` run as external shell commands and can invoke things outside Git.

### 示例 Example

```bash
git config --global alias.unstage 'reset HEAD --'
git unstage file.txt     # 等价于 git reset HEAD -- file.txt
```

### ⚠️ 注意事项 Notes

- 别名可能与其他 Git 子命令同名而产生歧义，命名时避免与现有命令冲突。Aliases can collide with existing Git subcommands, so choose names that avoid conflicts.
- 别名的值里有空格时要用引号包住。Wrap alias values that contain spaces in quotes.

> 出处 Git Aliases：https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases
