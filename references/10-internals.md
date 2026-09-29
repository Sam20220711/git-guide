# Git Internals

本章深入 Git 底层：它本质上是一个「内容寻址文件系统」加一套面向该文件系统的版本控制用户界面。理解对象模型、引用与数据恢复机制，能让你在遇到"提交丢了""仓库变慢"等问题时从容应对。每个概念先中文、后英文对照；命令一律放入代码块且只写英文。

This chapter goes under Git's hood: at its core Git is a content-addressable filesystem with a VCS user interface written on top of it. Understanding the object model, references, and data-recovery mechanisms lets you calmly handle "my commit is gone" or "my repo is slow" problems. Each concept appears in Chinese first, then equivalent English; commands go in code blocks and are English only.

---

## Plumbing and Porcelain

Git 的命令分为两类：面向日常用户的「高层命令」（porcelain，如 `add`、`commit`、`branch`），以及由这些高层命令在内部组合调用的「底层命令」（plumbing，如 `hash-object`、`cat-file`、`update-ref`）。本章重点讲 plumbing，因为理解它们才能看懂 Git 真正存储了什么。

Git's commands fall into two groups: user-facing "porcelain" commands (such as `add`, `commit`, `branch`), and the lower-level "plumbing" commands that the porcelain ones invoke internally (such as `hash-object`, `cat-file`, `update-ref`). This chapter focuses on plumbing, because only by understanding them can you see what Git actually stores.

### 命令 Commands

```bash
git cat-file -t <object>     # show the type of an object
git cat-file -p <object>     # pretty-print the object's content
git hash-object -w <file>    # hash a file's content and store it as an object
git rev-parse <ref>          # resolve a ref (e.g. HEAD, master) to a commit SHA-1
```

- `git cat-file`：查看任意对象，是探查 `.git` 内部最常用的命令。Inspects any object; the most-used tool for peeking inside `.git`.
- `git rev-parse`：把引用解析成提交哈希，用于在脚本中拿到"当前提交"。Resolves a ref into a commit hash, handy for scripting "the current commit".

### 示例 Example

```bash
git rev-parse HEAD
# prints the 40-char SHA-1 of the current commit
```

该命令返回 `HEAD` 所指向的提交哈希，验证了 `HEAD` 本质上只是一个指针。This prints the commit hash that `HEAD` points to, confirming that `HEAD` is really just a pointer.

### ⚠️ 注意事项 Notes

- plumbing 命令通常直接读写 `.git` 内部，绕过正常检查，出错时不易察觉；除非在做底层修复，优先用 porcelain 命令。Plumbing commands read and write `.git` internals directly and skip normal safety checks; prefer porcelain commands unless you are doing low-level repairs.

> 出处 Plumbing and Porcelain：https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain

---

## Git Objects

Git 把一切数据都存成「对象」，存放在 `.git/objects` 里，共有四种：blob 保存文件内容、tree 保存目录结构与文件名、commit 保存一次提交（指向 tree 与父提交）、tag 保存附注标签（指向一个 commit）。对象以内容哈希（SHA-1）命名，因此内容相同必然得到相同对象——这就是"内容寻址"。

Git stores all data as objects in `.git/objects`, in four kinds: a blob holds file contents, a tree holds a directory structure and filenames, a commit holds one commit (pointing at a tree and its parents), and a tag holds an annotated tag (pointing at a commit). Objects are named by a SHA-1 hash of their content, so identical content always yields the identical object — this is "content-addressable" storage.

### 命令 Commands

```bash
echo 'hello world' | git hash-object -w --stdin   # write a blob from stdin
git cat-file -p <blob-sha>                        # read the blob back
git update-index --add --cacheinfo 100644 <blob-sha> hello.txt   # stage a blob as hello.txt
git write-tree                                   # write the index out as a tree
echo 'first commit' | git commit-tree <tree-sha>  # create a commit pointing at the tree
```

- `git hash-object -w`：计算内容哈希并把对象写入数据库，返回该哈希。Hashes content and writes the object to the database, returning the hash.
- `git update-index --add --cacheinfo`：直接把一个已有 blob 以指定模式/路径写进暂存区，是 `git add` 的底层版。Places an existing blob into the index at a given mode/path — the plumbing form of `git add`.
- `git write-tree`：把当前暂存区写成 tree 对象并返回其哈希。Writes the current index out as a tree object and returns its hash.
- `git commit-tree`：以给定 tree 创建 commit，可指定父提交，是 `git commit` 的底层版。Creates a commit from a tree (with optional parents) — the plumbing form of `git commit`.

### 示例 Example

```bash
git hash-object -w test.txt
# d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

该哈希是 `test.txt` 内容的唯一标识；只要内容不变，无论文件名或位置如何，哈希都相同。That hash uniquely identifies the content of `test.txt`; as long as the content is unchanged, the hash is the same regardless of filename or location.

### ⚠️ 注意事项 Notes

- 单个对象一旦写入就不会被修改；"修改文件"实际上是写入一个全新的 blob。Objects are immutable once written; "changing a file" actually writes a brand-new blob.
- 手动用 `update-index`/`commit-tree` 拼提交很容易造出与 porcelain 不一致的状态，仅用于理解原理。Assembling commits by hand with `update-index`/`commit-tree` easily produces states inconsistent with porcelain; use it only to understand the model.

> 出处 Git Objects：https://git-scm.com/book/en/v2/Git-Internals-Git-Objects

---

## Git References

引用（ref）是指向提交的指针，存放在 `.git/refs` 下：分支在 `refs/heads/`，标签在 `refs/tags/`，远程分支在 `refs/remotes/`。`HEAD` 是一个符号引用（symbolic ref），通常指向某个分支。这些文件里的内容往往只是一个 40 字符的哈希。

References (refs) are pointers to commits, stored under `.git/refs`: branches live in `refs/heads/`, tags in `refs/tags/`, and remote-tracking branches in `refs/remotes/`. `HEAD` is a symbolic ref that usually points at a branch. The content of these files is often just a 40-character hash.

### 命令 Commands

```bash
git update-ref refs/heads/test <commit-sha>   # create/update a branch ref
git symbolic-ref HEAD                         # show what HEAD points to
git symbolic-ref HEAD refs/heads/test         # switch HEAD to a branch (low-level)
git rev-parse test                            # resolve a ref to a hash
```

- `git update-ref`：直接写一个引用，是 `git branch` 的底层版。Writes a ref directly — the plumbing form of `git branch`.
- `git symbolic-ref`：读取或设置 `HEAD` 这类符号引用。Reads or sets symbolic refs like `HEAD`.

### 示例 Example

```bash
git update-ref refs/heads/hotfix 1a410efbd13591db07496601ebc7a059dd55cfe9
```

该命令创建（或更新）名为 `hotfix` 的分支，让它指向该提交，相当于 `git branch hotfix 1a410ef`。This creates (or updates) a branch named `hotfix` pointing at that commit, equivalent to `git branch hotfix 1a410ef`.

### ⚠️ 注意事项 Notes

- 引用只是小文本文件；删掉 `.git/refs/heads/<branch>` 就等于删掉分支，但这通常不删除底层提交对象，可用 reflog 找回。Refs are just tiny text files; deleting `.git/refs/heads/<branch>` removes the branch, but usually not the underlying commits, which the reflog can still recover.
- ⚠️ 直接用文件系统或 `update-ref` 覆盖 `refs/heads/main` 前先记录旧哈希，否则当前分支位置不可逆丢失。Before overwriting `refs/heads/main` via the filesystem or `update-ref`, record the old hash, or the branch position is lost irreversibly.

> 出处 Git References：https://git-scm.com/book/en/v2/Git-Internals-Git-References

---

## Packfiles

对象最初以「松散对象」（loose objects，每个文件一个对象）形式保存，比较浪费空间。`git gc` 会把松散对象打包进 packfile，并利用 delta 压缩存储相近版本，同时生成索引文件（`.idx`）以快速定位。Git 会在松散对象达到一定数量时自动执行打包。

Objects are first stored as loose objects (one file per object), which is wasteful. `git gc` packs loose objects into a packfile, storing similar versions with delta compression, and writes an index file (`.idx`) for fast lookup. Git packs automatically once loose objects reach a threshold.

### 命令 Commands

```bash
git count-objects -v       # count loose objects and estimate size
git verify-pack -v <packfile>   # inspect a packfile's contents
git gc                     # pack objects and run housekeeping
```

- `git count-objects -v`：显示松散对象数量、占用空间与是否触发自动打包。Shows loose object count, disk usage, and auto-pack threshold info.
- `git verify-pack -v`：列出 packfile 里的对象与它们的 delta 关系。Lists the objects in a packfile and their delta relationships.

### 示例 Example

```bash
git count-objects -v
# count: 17
# size: 8
# in-pack: 130
# packs: 1
```

输出显示当前有 17 个松散对象，其余 130 个对象已被打进 1 个 pack。The output shows 17 loose objects while the remaining 130 objects have been packed into 1 pack.

### ⚠️ 注意事项 Notes

- 打包通常安全且可逆（会先写新 pack 再删旧松散对象），但会重写磁盘布局；首次在大型仓库上运行 `git gc` 前建议先备份或确认磁盘空间。Packing is usually safe and recoverable (it writes new packs before removing old loose objects), but it rewrites disk layout; on a large repo, back up or confirm free disk space before the first `git gc`.
- ⚠️ 不要手动删除 `.git/objects/pack/` 或 `.idx` 文件；那会直接破坏仓库，且不是"清理"。Never delete `.git/objects/pack/` or `.idx` files by hand; that corrupts the repository and is not "cleanup".

> 出处 Packfiles：https://git-scm.com/book/en/v2/Git-Internals-Packfiles

---

## The Refspec

refspec 是 `<src>:<dst>` 形式的映射规则，定义 fetch/push 时本地引用与远程引用如何对应，可选的前缀 `+` 表示"强制覆盖，不做快进检查"。远程仓库的默认抓取 refspec 通常形如 `+refs/heads/*:refs/remotes/origin/*`，写在 `.git/config` 里。

A refspec is a `<src>:<dst>` mapping rule that defines how local and remote refs correspond during fetch/push; the optional leading `+` means "overwrite without a fast-forward check". A remote's default fetch refspec is usually `+refs/heads/*:refs/remotes/origin/*`, stored in `.git/config`.

### 命令 Commands

```bash
git fetch origin refs/heads/main:refs/remotes/origin/main   # fetch one branch into a specific ref
git push origin main:refs/heads/qa/main                     # push local main to remote qa/main
git fetch origin +refs/heads/*:refs/remotes/origin/*        # force-update all remote-tracking refs
```

- `git fetch <remote> <src>:<dst>`：按 refspec 抓取，可只取某个分支并放到指定位置。Fetches per the refspec, optionally taking just one branch into a chosen spot.
- `+`：允许非快进覆盖，等价于对 refspec 施加强制更新。Allows a non-fast-forward overwrite, i.e. a forced update for that refspec.

### 示例 Example

```bash
git push origin master:qa/master
```

把本地 `master` 推送到远程的 `qa/master` 分支，二者名字可以不同。This pushes local `master` to the remote branch `qa/master`; the names need not match.

### ⚠️ 注意事项 Notes

- ⚠️ 带 `+` 的 refspec 会无条件覆盖目标引用，可能丢掉远端提交；推送时优先用 `git push --force-with-lease` 而不是写 `+` 的强制 refspec。A `+`-prefixed refspec overwrites the target unconditionally and can drop remote commits; prefer `git push --force-with-lease` over force refspecs when pushing.

> 出处 The Refspec：https://git-scm.com/book/en/v2/Git-Internals-The-Refspec

---

## Transfer Protocols

Git 支持四种传输协议：哑 HTTP（dumb HTTP，直接读裸仓库文件，旧式）、智能 HTTP（smart HTTP，服务端运行 Git）、SSH（`ssh://` 或 `user@host:path`）以及原生 Git 协议（`git://`，由 `git daemon` 提供，无鉴权）。智能 HTTP 与 SSH 是当今最常用且支持鉴权/加密的方案。

Git supports four transfer protocols: dumb HTTP (serves the bare repo as plain files, legacy), smart HTTP (server-side Git process), SSH (`ssh://` or `user@host:path`), and the native Git protocol (`git://`, served by `git daemon`, with no authentication). Smart HTTP and SSH are the common modern choices, offering authentication and encryption.

### 命令 Commands

```bash
git daemon --base-path=/srv/git --export-all   # serve git:// read-only access
git clone git://host/repo.git                  # clone over the native protocol
git clone ssh://user@host/path/repo.git        # clone over SSH
```

- `git daemon`：启动只读的 `git://` 服务；`--export-all` 让所有仓库可被访问。Starts a read-only `git://` server; `--export-all` makes all repos available.
- 智能 HTTP 需要 `git-http-backend` 作为 CGI 后端，配合 Web 服务器部署。Smart HTTP requires `git-http-backend` as a CGI backend, deployed with a web server.

### 示例 Example

```bash
git clone https://github.com/libgit2/libgit2
```

这是典型的智能 HTTP 克隆：客户端与服务端协商后只传输需要的对象。A typical smart-HTTP clone: client and server negotiate and transfer only the objects needed.

### ⚠️ 注意事项 Notes

- `git://` 无鉴权无加密，只适合只读的公开镜像，切勿用于私有仓库。`git://` has no auth or encryption; use it only for read-only public mirrors, never private repos.
- 在公网暴露 `git daemon` 前务必用 `--base-path` 限制可访问目录，防止越权读取。Always restrict `git daemon` with `--base-path` before exposing it publicly to prevent reading outside the intended tree.

> 出处 Transfer Protocols：https://git-scm.com/book/en/v2/Git-Internals-Transfer-Protocols

---

## Maintenance and Data Recovery

Git 的维护主要靠 `git gc`：打包松散对象、清理不可达对象并压缩 reflog。数据恢复依赖两条线索：`git reflog`（记录 HEAD 与分支的每一次移动，即使提交已不被任何分支引用）与 `git fsck`（扫描对象库找出不可达/悬空对象）。这两者是找回"误删分支、`reset --hard` 过头"等丢失提交的关键。

Git's maintenance centers on `git gc`, which packs loose objects, prunes unreachable objects, and compresses the reflog. Data recovery relies on two clues: `git reflog` (records every move of HEAD and branches, even for commits no branch points to) and `git fsck` (scans the object database for unreachable/dangling objects). Together they are the key to recovering lost commits after a deleted branch or an overzealous `reset --hard`.

### 命令 Commands

```bash
git reflog                                  # list where HEAD has been
git reflog show <branch>                    # list where a branch ref has been
git branch recover <sha>                    # resurrect a commit by its reflog SHA
git fsck --full --unreachable               # list unreachable objects
git fsck --lost-found                       # write dangling objects to .git/lost-found
git gc --auto                               # pack only if thresholds are met (safe)
git gc --prune=<date>                       # drop objects older than <date>
git count-objects -v                        # inspect loose object counts
```

- `git reflog`：列出本地指针的历史位置，是回退误操作的第一手信息。Lists the history of local pointers — the first thing to check after a mistake.
- `git branch recover <sha>`：用 reflog/fsck 找到的哈希新建分支，从而"找回"提交。Creates a branch at a hash found via reflog/fsck, thereby "recovering" the commit.
- `git fsck --unreachable`：找出没有任何引用可达、但仍存在于对象库中的对象。Finds objects not reachable from any ref but still present in the database.

### 示例 Example

```bash
git reflog
# 1a410ef HEAD@{0}: reset: moving to HEAD~3
# ab1afef HEAD@{1}: commit: add feature X
git branch recovered ab1afef
```

`reset` 之前的提交 `ab1afef` 已不被任何分支引用，但仍能从 reflog 找到并恢复为新分支。The pre-reset commit `ab1afef` is no longer on any branch, yet the reflog still finds it and restores it as a new branch.

### ⚠️ 注意事项 Notes

- ⚠️ `git gc --prune=now` 或 `git reflog expire --expire=now --all` 会立即删除不可达对象与 reflog 记录，此后提交无法找回；除非确需立即回收空间，否则不要加 `now`。`git gc --prune=now` or `git reflog expire --expire=now --all` deletes unreachable objects and reflog entries immediately, after which commits are unrecoverable; never use `now` unless you must reclaim space right away.
- 默认保护期：对象在约两周内、reflog 记录在约 90 天内不会被 prune，给了误删后的恢复窗口。By default objects survive roughly two weeks and reflog entries roughly 90 days before pruning, giving a recovery window after mistakes.
- 恢复前不要再运行 `git gc`，以免把悬空对象提前清理掉。Do not run `git gc` before recovering, or the dangling objects may be pruned early.
- `git fsck` 报出的 dangling/不可达对象未必都是坏数据；先确认确实需要它们再重建引用。Dangling/unreachable objects reported by `git fsck` are not necessarily corruption; confirm you actually need them before rebuilding refs.

> 出处 Maintenance and Data Recovery：https://git-scm.com/book/en/v2/Git-Internals-Maintenance-and-Data-Recovery

---

## Environment Variables

Git 的行为受大量环境变量控制，多为 `GIT_*` 前缀，优先级通常高于配置文件。常用的有：`GIT_EDITOR`/`core.editor` 指定编辑器，`GIT_PAGER` 指定分页器，`GIT_AUTHOR_NAME`、`GIT_AUTHOR_EMAIL`、`GIT_COMMITTER_NAME`、`GIT_COMMITTER_EMAIL` 覆盖作者/提交者身份，`GIT_EXTERNAL_DIFF` 指定外部 diff 程序。

Git's behavior is controlled by many environment variables, mostly prefixed `GIT_*`, which generally take precedence over config files. Common ones: `GIT_EDITOR`/`core.editor` sets the editor, `GIT_PAGER` sets the pager, `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`, `GIT_COMMITTER_EMAIL` override author/committer identity, and `GIT_EXTERNAL_DIFF` sets an external diff program.

### 命令 Commands

```bash
GIT_EDITOR=nano git commit          # use nano for this commit's message
GIT_PAGER=cat git log               # disable paging for this command
GIT_AUTHOR_NAME="Alice" GIT_AUTHOR_EMAIL="a@example.com" git commit
```

- `GIT_EDITOR`：本次命令使用指定编辑器，覆盖 `core.editor`。Uses the given editor for this command, overriding `core.editor`.
- `GIT_PAGER=cat`：把分页器设为 `cat`，使输出一次性打印。Sets the pager to `cat`, printing output all at once.
- 身份变量只覆盖本次提交，适合在共享机器上以他人身份提交。The identity variables override only this commit, useful for committing under another identity on a shared machine.

### 示例 Example

```bash
GIT_PAGER=cat git log --oneline -5
```

直接打印最近 5 条提交而不会进入分页器，适合脚本或复制粘贴。Prints the last five commits without opening a pager, handy for scripts or copy-paste.

### ⚠️ 注意事项 Notes

- 环境变量是临时覆盖，不写入仓库配置；若希望长期生效应改用 `git config`。Environment variables are transient overrides and are not written to repo config; use `git config` for lasting effect.
- 设置 `GIT_AUTHOR_*`/`GIT_COMMITTER_*` 会改变历史里的身份信息，多人共享环境时注意不要误标提交者。Setting `GIT_AUTHOR_*`/`GIT_COMMITTER_*` changes recorded identity in history; on shared environments be careful not to mislabel committers.

> 出处 Environment Variables：https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables
