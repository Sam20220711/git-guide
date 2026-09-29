# Git and Other Systems

Git 并非只能与 Git 协作。本章讲解两种场景：把 Git 当作其他版本控制系统（SVN、Mercurial、Perforce、TFS）的"客户端"，以及把其他系统上的整个项目历史迁移到 Git。核心工具是 `git svn`，其次是 `git-remote-hg` 桥接器与 `git p4`；迁移时则常借助 `git svn`、`hg-fast-export` 或自定义脚本配合 `git fast-import`。

Git does not only work with Git. This chapter covers two scenarios: using Git as a client for other version control systems (SVN, Mercurial, Perforce, TFS), and migrating an entire project history from those systems into Git. The core tool is `git svn`, followed by the `git-remote-hg` bridge and `git p4`; migrations typically use `git svn`, `hg-fast-export`, or a custom script feeding `git fast-import`.

## Git as a Client

`git svn` 让你把 Git 当作 SVN 服务器的前端：你在本地享受 Git 的分支、合并、暂存等能力，团队却仍通过中央 SVN 仓库协作。它的本质是在本地维护一个 Git 仓库，每次与 SVN 同步时把 SVN 的线性提交映射为 Git 提交。这是渐进引入 Git 的常见方式——你可以边学 Git，团队边继续用 SVN。

`git svn` lets you use Git as a front end for an SVN server: you enjoy Git's local branching, merging, and staging while the team still collaborates through a central SVN repository. Under the hood it keeps a local Git repository and maps SVN's linear commits into Git commits on each sync. This is a common gradual path into Git — you can learn Git while your team stays on SVN.

克隆 SVN 仓库；若布局是标准 trunk/branches/tags，直接用 `--stdlayout`：

```bash
git svn clone <svn-url> -T trunk -b branches -t tags   # explicit layout
git svn clone <svn-url> --stdlayout                    # standard trunk/branches/tags
git svn clone <svn-url> --stdlayout --authors-file=authors.txt
```

`-T/-b/-t` 显式指定主干/分支/标签目录；`--stdlayout` 等价于 `-T trunk -b branches -t tags`；`--authors-file` 用 `用户名 = 姓名 <邮箱>` 格式把 SVN 登录名映射为 Git 作者。克隆后 SVN 的远程分支会映射为 `remotes/origin/*` 这类远程引用。

`-T/-b/-t` name the trunk/branches/tags directories explicitly; `--stdlayout` is shorthand for `-T trunk -b branches -t tags`; `--authors-file` maps SVN usernames to "Name <email>" authors. After cloning, SVN's remote branches are mapped to remote refs like `remotes/origin/*`.

日常协作只有两条命令——`git svn rebase` 拉取 SVN 最新提交并把本地提交变基到其上，`git svn dcommit` 把本地提交逐个提交回 SVN：

```bash
git svn rebase     # pull latest from SVN and rebase local commits on top
git svn dcommit    # push local commits back to SVN, one by one
```

`git svn rebase` 等价于先 `git svn fetch` 再对当前分支做变基，保证本地历史保持线性；`git svn dcommit` 把每个本地 Git 提交按顺序提交到 SVN，再把本地提交改写为指向新生成的 SVN 版本。关键规则：SVN 无法表达合并提交，所以绝对不要先合并本地分支再 dcommit，必须用变基保持线性。

`git svn rebase` is effectively `git svn fetch` followed by a rebase onto the current branch, keeping local history linear; `git svn dcommit` commits each local Git commit to SVN in order, then rewrites your local commits to point at the new SVN revisions. The key rule: SVN cannot represent merge commits, so never merge local branches before dcommitting — rebase instead to stay linear.

分支与标签在 SVN 侧也要用 `git svn` 创建，而不是 `git branch`/`git tag`：

```bash
git svn branch <name>    # create a branch on the SVN server
git svn tag <name>       # create a tag on the SVN server
git svn log              # view SVN history (like svn log)
git svn create-ignore    # write .gitignore from svn:ignore
git svn info             # show SVN info
```

`git svn branch/tag` 会在 SVN 服务器上建立对应目录并同时生成远程引用；`git svn log`/`info` 分别对应 `svn log`/`svn info`；`git svn create-ignore` 把 SVN 的 `svn:ignore` 属性转成 `.gitignore`，省去手工维护。

`git svn branch/tag` create the corresponding directory on the SVN server and generate a remote ref; `git svn log`/`info` mirror `svn log`/`svn info`; `git svn create-ignore` converts the `svn:ignore` property into `.gitignore`, saving manual upkeep.

其他系统的桥接也很简洁。访问 Mercurial（hg）时，`git-remote-hg` 让 Git 直接克隆并推送 hg 仓库；访问 Perforce 时，`git p4` 提供 clone/rebase/submit 三件套：

```bash
git clone hg::<hg-repo-url>                    # clone a Mercurial repo as Git (git-remote-hg)
git push                                       # push local commits back to Mercurial

git p4 clone //depot/project/main@all <dir>    # clone full Perforce history
git p4 rebase                                  # pull latest and rebase local commits
git p4 submit                                  # push local commits back to Perforce
```

`hg::` 前缀让 Git 通过 `git-remote-hg` 桥接器访问 Mercurial，克隆后即可像普通 Git 仓库一样提交、推送；`git p4` 依赖 `p4` 命令行客户端，`@all` 表示导入全部历史，提交前需用 `git config git-p4.user <user>` 设置 Perforce 账号。

The `hg::` prefix routes Git through the `git-remote-hg` bridge to access Mercurial; after cloning you commit and push like any Git repo. `git p4` depends on the `p4` command-line client, `@all` imports the full history, and you must set your Perforce account with `git config git-p4.user <user>` before submitting.

**示例**：团队仍用 SVN，你想用 Git 提交一个修复。先 `git svn clone <svn-url> --stdlayout` 克隆；在本地建分支 `git checkout -b fix` 开发并提交；合回主干前用 `git rebase trunk`（或 `git checkout master && git svn rebase` 后合并），再 `git svn dcommit` 推送。整个过程 SVN 侧同事看到的都是线性提交。

**Example**: your team still uses SVN and you want to commit a fix with Git. First `git svn clone <svn-url> --stdlayout`; develop on a local branch with `git checkout -b fix`; before pushing, rebase onto trunk (`git rebase trunk`, or `git checkout master && git svn rebase` then merge), then `git svn dcommit`. SVN-side colleagues see linear commits throughout.

⚠️ 注意事项：`git svn dcommit` 会改写你已 dcommit 过的本地提交（哈希变化），因此绝对不要基于已 dcommit 的历史再建本地合并——那会产生 SVN 无法表达的合并提交，导致 dcommit 失败或历史错乱；正确做法是始终 `git rebase` 保持线性，且 dcommit 前先 `git svn rebase`。`git svn clone` 在大仓库上很慢，必要时用 `-r <rev>:HEAD` 只克隆部分历史。克隆前先备份 SVN 仓库或确认网络稳定，避免中途失败。

⚠️ Caution: `git svn dcommit` rewrites the local commits you have already dcommitted (their hashes change), so never create local merge commits on top of dcommitted history — SVN cannot represent them and dcommit will fail or corrupt the history; always `git rebase` to stay linear and run `git svn rebase` before dcommit. `git svn clone` is slow on large repositories, so clone only partial history if needed (e.g. `-r <rev>:HEAD`). Back up the SVN repository or confirm a stable network before cloning to avoid mid-way failures.

> 出处 Git as a Client：https://git-scm.com/book/en/v2/Git-and-Other-Systems-Git-as-a-Client

## Migrating to Git

把整个项目从 SVN、Mercurial、Perforce 等系统迁到 Git，思路统一：把旧系统里的每一次提交按顺序重放成 Git 提交。最省事的是用现成工具；没有现成工具时，写一个自定义脚本输出 `git fast-import` 格式，再交给 `git fast-import` 一次性导入。

Migrating a whole project from SVN, Mercurial, Perforce, or similar systems into Git follows one idea: replay every commit from the old system, in order, as a Git commit. The easiest route is a ready-made tool; when none exists, write a custom script that emits `git fast-import` format and feed it to `git fast-import` in one pass.

从 SVN 迁移最直接：用 `git svn clone` 拉全历史，再用 `--authors-file` 保证作者信息正确：

```bash
git svn clone <svn-url> --stdlayout --authors-file=authors.txt proj
cd proj
git svn create-ignore          # preserve svn:ignore as .gitignore
```

克隆完成后 `trunk` 会变成 `remotes/origin/trunk`，用 `git branch -a` 查看，再把它设为 `master`；分支/标签也在远程引用里，用 `git branch -r` 逐个转成本地分支或标签。若历史中夹杂 SVN 的"分支复制"（branch copy），迁移后的 Git 历史可能不如预期简洁，需要人工整理。

After cloning, `trunk` becomes `remotes/origin/trunk`; view it with `git branch -a` and make it `master`. Branches and tags also live in remote refs — list them with `git branch -r` and convert each into a local branch or tag. If the history contains SVN branch copies, the resulting Git history may be less clean than expected and needs manual tidying.

从 Mercurial 迁移用 `hg-fast-export`（把 hg 变更集转成 fast-import 流）：

```bash
git clone https://github.com/frej/fast-export.git
hg clone <hg-repo-url> /tmp/hg-repo
/tmp/fast-export/hg-fast-export.sh -r /tmp/hg-repo
git checkout HEAD
```

`hg-fast-export.sh -r` 读取 hg 仓库并生成一个 Git 仓库，完成后 `git checkout HEAD` 检出最新提交。若要双向同步，可用 Mercurial 的 `hg-git` 扩展。

`hg-fast-export.sh -r` reads the Mercurial repository and produces a Git repository; afterwards `git checkout HEAD` checks out the latest commit. For bidirectional sync, use Mercurial's `hg-git` extension.

从 Perforce 迁移可用 `git p4 clone`（导入全部历史）或 Perforce 官方的 Git Fusion。没有现成工具时，写自定义脚本输出 fast-import 流：

```bash
git init proj
cd proj
/path/to/my-exporter.sh | git fast-import
git checkout -f master
```

fast-import 是一种极简的文本流格式，脚本按顺序输出 commit/blob 命令即可表达任意历史：

```bash
blob
mark :1
data 7
# hello

commit refs/heads/master
mark :2
author Scott Chacon <schacon@example.com> 1234567890 -0700
committer Scott Chacon <schacon@example.com> 1234567890 -0700
data 12
first commit

M 100644 :1 README
```

`blob` 定义一段文件内容并标为 `mark :1`；`commit refs/heads/master` 在指定引用上开一个提交，`mark :2` 给它内部编号；`author`/`committer` 记录姓名、邮箱、时间戳与时区；`data <N>` 后跟 N 字节的提交信息；`M 100644 :1 README` 表示以 100644 权限把 blob `:1` 写成 README。小内容也可用 `M 100644 inline <path>` + `data` 内联。脚本把旧系统逐条提交翻译成这些指令即可。

`blob` defines file content and labels it `mark :1`; `commit refs/heads/master` starts a commit on the given ref and `mark :2` labels it; `author`/`committer` record name, email, timestamp, and timezone; `data <N>` is followed by N bytes of commit message; `M 100644 :1 README` writes blob `:1` as README with mode 100644. Small content can be inlined with `M 100644 inline <path>` plus `data`. The script simply translates each old commit into these directives.

**示例**：把一个小型 SVN 仓库迁到 Git 并保留作者。准备 `authors.txt`（每行 `svnuser = Name <email>`），执行 `git svn clone <svn-url> --stdlayout --authors-file=authors.txt`；克隆后 `git branch -a` 看到 `remotes/origin/trunk`，执行 `git checkout -b master remotes/origin/trunk` 建立主干；`git svn create-ignore` 生成 `.gitignore`；确认 `git log` 历史完整后，推送到新的 Git 远端。

**Example**: migrate a small SVN repository to Git while preserving authors. Prepare `authors.txt` (one `svnuser = Name <email>` per line), run `git svn clone <svn-url> --stdlayout --authors-file=authors.txt`; after cloning, `git branch -a` shows `remotes/origin/trunk`, so run `git checkout -b master remotes/origin/trunk` to establish the main line; generate `.gitignore` with `git svn create-ignore`; once `git log` confirms the history is complete, push to the new Git remote.

⚠️ 注意事项：迁移是一次"重放"，会生成全新的提交哈希，旧系统上之后的任何提交都不会自动出现在 Git 里——迁移前应冻结旧仓库或规划好同步窗口。`git fast-import` 直接写入 `.git` 对象库，格式错误可能导致导入失败或产生脏历史，务必先在一次性副本仓库里试跑，验证 `git log` 与文件树一致后再推广。大规模迁移前先完整备份旧仓库。

⚠️ Caution: migration is a "replay" that produces brand-new commit hashes, and any later commits made to the old system will not automatically appear in Git — freeze the old repository or plan a sync window before migrating. `git fast-import` writes directly into the `.git` object database and malformed input can fail the import or produce dirty history, so always test in a throwaway copy first and verify `git log` matches the file tree before rolling out. Back up the old repository fully before any large migration.

> 出处 Migrating to Git：https://git-scm.com/book/en/v2/Git-and-Other-Systems-Migrating-to-Git
