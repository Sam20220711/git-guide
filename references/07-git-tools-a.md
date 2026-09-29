# Git Tools（上）：选择修订 / 暂存 / 贮藏 / 签名 / 搜索

本章介绍 Git 高级工具中与日常操作最相关的一部分：如何精确选择某一个提交（revision）、如何交互式地暂存改动、如何临时贮藏和清理工作区、如何用 GPG 为提交签名，以及如何按内容搜索提交历史。这些工具能帮助你更精细地控制仓库状态，并在追溯代码来源时快速定位引发问题的那个提交。

This chapter covers a focused set of Git's advanced tools: how to select a specific commit (revision), how to stage changes interactively, how to stash and clean the working tree temporarily, how to sign commits with GPG, and how to search commit history by content. These tools help you control the repository state more precisely and quickly locate the commit behind a change.

## Revision Selection

### 概念

Git 允许你通过多种方式引用一个提交（revision）。除了完整的 40 位 SHA-1 哈希（新版仓库也可能是 SHA-256），你还可以使用短哈希、分支名、标签，以及引用日志（reflog）中的编号。理解这些记法能让你在任何命令中精确、简洁地指向目标提交，而无需每次都复制一长串哈希。

Git lets you refer to a commit (revision) in many ways. Besides the full 40-character SHA-1 hash (or SHA-256 in newer repositories), you can use a short hash, a branch or tag name, and entries in the reflog. Understanding these notations lets you point precisely and concisely at any target commit in any command, without copying a long hash every time.

### 命令

```bash
git rev-parse HEAD           # resolve HEAD to a commit hash
git rev-parse master         # resolve a branch to a commit hash
git log --oneline            # show short-hash history
git log -g --oneline         # show reflog history
git show HEAD@{2}            # refer to a commit by reflog index
```

`git rev-parse` 会把任意引用解析为提交哈希，是验证记法的好工具。`HEAD@{n}` 表示 HEAD 在 reflog 中第 n 步之前指向的位置，例如 `HEAD@{1}` 是“上一步所在位置”。

`git rev-parse` resolves any reference into a commit hash and is a good way to verify notation. `HEAD@{n}` means the position HEAD pointed to n steps ago in the reflog; for example `HEAD@{1}` is "where you were one step before".

### 祖先引用：`^` 与 `~`

`^` 用于选择某个合并提交的多个父提交；`~` 用于沿第一父提交向上走若干步。这两类记法可以组合，指向任意祖先。

`^` selects one of the parents of a merge commit; `~` walks up along the first-parent line for a number of steps. The two can be combined to point at any ancestor.

```bash
git show HEAD^       # first parent of HEAD
git show HEAD^2      # second parent of a merge commit
git show HEAD~2      # grandparent along the first-parent line
git show HEAD~3^2    # go up three steps, then take its second parent
```

记法要点：`HEAD^` 等价于 `HEAD^1`，`HEAD~` 等价于 `HEAD~1`；`HEAD~n` 等价于连续 n 个 `^`（始终沿第一父）。只有合并提交才有第二父，普通提交的 `HEAD^2` 会报错。

Notation notes: `HEAD^` equals `HEAD^1`, and `HEAD~` equals `HEAD~1`; `HEAD~n` is equivalent to n consecutive `^` (always along the first parent). Only merge commits have a second parent, so `HEAD^2` on a normal commit errors out.

### 范围引用：`..` 与 `...`

双点范围 `A..B` 表示“从 B 可达、但从 A 不可达的提交”，即 B 领先 A 的部分；三点范围 `A...B` 表示“从 A 或 B 任一侧可达、但不同时从两侧可达的提交”，用于查看分叉点之后两侧各自的提交。

The double-dot range `A..B` means "commits reachable from B but not from A", i.e. what B has over A; the triple-dot `A...B` means "commits reachable from either A or B but not both", useful for seeing each side's commits after a fork.

```bash
git log master..feature            # commits feature has over master
git log origin/master..HEAD        # local commits not yet pushed
git log --left-right master...feature   # show both sides, labeled left/right
git diff master...feature          # changes feature introduced since the merge base
```

### 示例

查看实验分支相对主分支多了哪些提交：

To see what commits an experiment branch has over main:

```bash
git log --oneline main..experiment
```

查看两个分支分叉后各自新增的提交（左侧 main，右侧 experiment）：

To see the commits each branch gained since they diverged (main on the left, experiment on the right):

```bash
git log --oneline --left-right main...experiment
```

借助 reflog 找回“刚刚还在”的提交（例如一次错误的 reset 之后）：

To recover the commit you were on moments ago via the reflog (e.g. after a bad reset):

```bash
git reflog                 # list recent HEAD movements with HEAD@{n} indexes
git reset --hard HEAD@{1}  # move back to the previous position
```

### ⚠️注意事项

- reflog 是本地记录，克隆或推送不会携带它，因此 `HEAD@{n}` 只在本地有效，且旧条目可能被垃圾回收清理。
- 短哈希必须足够长以保证唯一，Git 通常至少取 4 位；冲突时会报错，此时多取几位即可。
- `A..B` 与 `A...B` 只是 `log`/`diff` 等命令的筛选范围，不是可独立使用的提交对象；`A...B` 必须配合 `--left-right` 或用于 `diff` 才容易看出两侧归属。

- The reflog is local-only and is not cloned or pushed, so `HEAD@{n}` is valid only locally, and old entries may be pruned by garbage collection.
- A short hash must be long enough to be unique; Git usually requires at least 4 characters, and it errors on ambiguity — just supply more characters.
- `A..B` and `A...B` are only filters for commands like `log`/`diff`, not standalone commit objects; pair `A...B` with `--left-right` (or use it in `diff`) to see which side each commit belongs to.

> 出处 Revision Selection：https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection

## Interactive Staging

### 概念

`git add -i`（交互模式）和 `git add -p`（补丁模式）让你逐块（hunk）决定是否暂存改动，而不是一次性 `git add` 全部。这对于把一个文件里的多处无关改动拆成多个逻辑提交非常有用，能保证每个提交只包含单一主题。

`git add -i` (interactive mode) and `git add -p` (patch mode) let you decide hunk-by-hunk whether to stage a change, instead of staging everything at once. This is useful for splitting unrelated changes in one file into multiple logical commits, keeping each commit focused on a single topic.

### 命令

```bash
git add -i           # enter the interactive staging menu
git add -p           # go straight to patch mode, confirm hunk by hunk
git reset -p         # unstage hunks interactively
git checkout -p      # discard working-tree hunks interactively
```

在补丁模式下，每个 hunk 都会提示操作：`y` 暂存该块、`n` 跳过、`s` 拆成更小块、`e` 手动编辑该块、`q` 退出、`?` 查看帮助。`git add -i` 的菜单还提供 status、update、revert、diff、quit 等子命令。

In patch mode, each hunk prompts for an action: `y` to stage it, `n` to skip it, `s` to split it into smaller hunks, `e` to edit the hunk manually, `q` to quit, and `?` for help. The `git add -i` menu also offers subcommands like status, update, revert, diff, and quit.

### 示例

只暂存某文件里的一半改动，分两次提交：

To stage only part of a file's changes and commit in two steps:

```bash
git add -p file.txt
# choose y / n / s / e for each hunk
git status              # confirm staged vs unstaged state
git commit -m "part 1"
git add file.txt        # stage the rest, then commit again
```

### ⚠️注意事项

- `s`（拆分）只对上下文分隔明确的 hunk 有效；改动太近无法拆分时用 `e` 手动编辑 diff，但要保留正确的 hunk 头行格式。
- `git checkout -p` 会丢弃未提交的工作区改动，属于不可恢复操作，操作前确认不再需要该内容。
- 交互模式下多数误操作可回退（如用 `git reset` 取消暂存），但手动编辑 diff 出错可能导致补丁无法应用。

- `s` (split) only works when hunks are separated by clear context; if changes are too close to split, use `e` to edit the diff manually while keeping valid hunk headers.
- `git checkout -p` discards uncommitted working-tree changes and is irreversible; confirm you no longer need the content before running it.
- Most interactive mistakes are recoverable (e.g. `git reset` to unstage), but a malformed manual hunk edit can make the patch fail to apply.

> 出处 Interactive Staging：https://git-scm.com/book/en/v2/Git-Tools-Interactive-Staging

## Stashing and Cleaning

### 概念

`git stash` 把你未提交的改动（已暂存和未暂存）临时保存起来，让工作区恢复干净，之后可以随时恢复。当你需要切换分支去处理别的事，却又不想提交半成品时，stash 是最方便的工具。`git clean` 则是删除工作区中未被跟踪的文件，用于彻底清理构建产物等杂物。

`git stash` temporarily saves your uncommitted changes (staged and unstaged) and returns the working tree to a clean state, so you can restore them later. It is the handiest tool when you need to switch branches for another task without committing half-finished work. `git clean` removes untracked files to thoroughly clean the working tree of build artifacts and other clutter.

### 命令

```bash
git stash                        # same as "git stash push"; stash all changes
git stash push -m "msg"          # stash with a descriptive message
git stash push -- file.txt       # stash only the specified file
git stash push --keep-index      # stash working changes but keep the index staged
git stash push -u                # include untracked files in the stash
git stash list                   # list all stashes
git stash apply                  # restore the latest stash (keeps it)
git stash apply stash@{2}        # restore a specific stash
git stash pop                    # restore the latest stash and drop it
git stash drop stash@{0}         # delete a specific stash
git stash clear                  # delete all stashes
git stash show -p stash@{0}      # show a stash's full diff
git stash branch newbranch stash@{0}   # create a branch from a stash and apply it
```

`apply` 恢复但不删除贮藏，`pop` 恢复并删除；两者在遇到冲突时都会中止，并保留贮藏内容供后续处理。`stash branch` 会在贮藏所在的提交上新建分支、应用贮藏，成功后再删除该贮藏，是恢复旧贮藏最安全的方式。

`apply` restores without deleting the stash, while `pop` restores and deletes it; both abort and keep the stash if there is a conflict. `stash branch` creates a new branch at the stash's original commit, applies the stash, and drops it on success — the safest way to recover an old stash.

### 示例

切换分支前临时保存工作：

To save work temporarily before switching branches:

```bash
git stash push -m "WIP: login page"
git checkout other-branch
# handle the other task...
git checkout feature-branch
git stash pop
```

只贮藏某个文件：

To stash only a specific file:

```bash
git stash push -- src/auth.py
```

### ⚠️注意事项

- ⚠️ `git stash drop`、`git stash clear` 会永久删除贮藏内容，无法通过普通手段找回。删除前先用 `git stash show -p <stash>` 确认内容，或用 `git stash branch <name> <stash>` 先把内容落到分支上再决定。
- ⚠️ `git clean` 删除的未跟踪文件无法恢复。务必先用 `-n`（dry-run，只预览不删除）确认清单，再加 `-d` 删目录、`-f` 强制执行：先 `git clean -nd` 预览，确认无误后再 `git clean -fd`。
- stash 默认不贮藏未跟踪和忽略的文件；需要时加 `-u`（含未跟踪）或 `-a`（含忽略文件）。
- `pop` 遇到冲突不会删除贮藏，但一旦 `drop` 之后就没有回退余地，务必先验证。

- ⚠️ `git stash drop` and `git stash clear` permanently delete stashes with no ordinary way to recover them. Confirm content with `git stash show -p <stash>` first, or land it on a branch with `git stash branch <name> <stash>` before deciding.
- ⚠️ Files removed by `git clean` cannot be recovered. Always preview with `-n` (dry-run), then add `-d` for directories and `-f` to force: run `git clean -nd` first, then `git clean -fd` once you have confirmed the list.
- Stash skips untracked and ignored files by default; add `-u` (untracked) or `-a` (including ignored) when needed.
- `pop` keeps the stash on conflict, but there is no way back after a `drop` — verify first.

> 出处 Stashing and Cleaning：https://git-scm.com/book/en/v2/Git-Tools-Stashing-and-Cleaning

## Signing Your Work

### 概念

Git 支持用 GPG（GNU Privacy Guard）对提交和标签签名，让其他人能够验证该提交确实出自你。这对开源项目、发布流程和供应链安全很重要：签名能防止他人冒用你的身份伪造提交或篡改发布标签。

Git supports signing commits and tags with GPG (GNU Privacy Guard) so others can verify the work is genuinely yours. This matters for open-source projects, release processes, and supply-chain security: a signature prevents someone from forging commits under your identity or tampering with release tags.

### 命令

```bash
gpg --list-secret-keys --keyid-format=long   # list existing GPG keys
gpg --full-generate-key                       # generate a new key (follow prompts)
git config --global user.signingkey <KEYID>   # set the signing key
git config --global commit.gpgsign true       # sign all commits by default
git commit -S -m "signed commit"              # sign a single commit
git tag -s v1.0.0 -m "signed release"         # sign a tag
git log --show-signature                      # show signature verification for commits
git tag -v v1.0.0                             # verify a tag's signature
```

`-S` 表示对提交签名，`-s` 表示对标签签名；两者都需要预先配置 `user.signingkey`，或让 GPG 使用默认密钥。设置 `commit.gpgsign true` 后，所有提交会自动签名。

`-S` signs a commit, and `-s` signs a tag; both need `user.signingkey` configured or a default GPG key available. Once `commit.gpgsign true` is set, every commit is signed automatically.

### 示例

为一次提交签名并验证：

To sign a commit and verify it:

```bash
git commit -S -m "release 1.0"
git log --show-signature -1
# look for "Good signature" and the key fingerprint
```

对发布标签签名并验证：

To sign and verify a release tag:

```bash
git tag -s v1.0.0 -m "signed release"
git tag -v v1.0.0
```

### ⚠️注意事项

- 签名的安全性完全取决于私钥的保管；私钥泄露意味着他人可以伪造你的签名。
- 验证方必须在本地导入并信任你的公钥，否则会显示“无法验证”，而非“坏签名”。
- 签名提交不会加密改动内容，只证明作者身份与内容完整性。
- 丢失私钥后旧签名仍可验证，但你无法再签新的提交；需生成新密钥并更新 `user.signingkey`。

- The security of a signature depends entirely on keeping your private key safe; a leaked key lets others forge your signature.
- Verifiers must import and trust your public key locally, otherwise it shows as unverifiable rather than "bad signature".
- A signed commit does not encrypt the changes; it only proves authorship and content integrity.
- If you lose the private key, old signatures remain verifiable but you cannot sign new commits; generate a new key and update `user.signingkey`.

> 出处 Signing Your Work：https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work

## Searching

### 概念

`git log` 和 `git grep` 提供了强大的搜索能力。按提交元数据搜索可用 `--author`、`--since`、`--grep` 等选项；按代码内容的历史变化搜索则用 `-S`（统计某字符串出现次数的变化）和 `-G`（用正则匹配补丁中新增或删除的行）。`git grep` 用于在任意历史版本中快速搜索代码，而 `-L` 可以追踪某个函数从引入到现在的完整历史。

`git log` and `git grep` provide powerful search. Metadata can be filtered with `--author`, `--since`, and `--grep`; to search how code content changed over history use `-S` (tracks changes in a string's occurrence count) and `-G` (regex-matches lines added or removed in a patch). `git grep` quickly searches code in any historical version, and `-L` traces a function's full history from introduction to the present.

### 命令

```bash
git log --author="Name" --since="2024-01-01"   # filter by author and date
git log --grep="fix" -i                        # search commit messages (case-insensitive)
git log -S "function_name" --oneline           # commits that changed the string's occurrence count
git log -G "regex_pattern" --oneline           # commits whose patch lines match the regex
git log -L :funcname:file.c                    # trace the history of a function
git grep "pattern"                             # search the current working tree
git grep "pattern" HEAD~5                      # search a specific historical version
git grep -n "pattern" $(git rev-list --all)    # search across all commits
```

`-S` 关注“某字符串在某次提交前后出现次数不同”，`-G` 关注“补丁里新增/删除的行是否匹配正则”。两者定位的是不同问题：要回答“字符串何时被引入或删除”用 `-S`；要回答“哪些提交改动了匹配某模式的行”用 `-G`。

`-S` focuses on "the string's occurrence count changed across a commit", while `-G` focuses on "whether the lines added or removed in a patch match the regex". They answer different questions: use `-S` for "when was a string introduced or removed", and `-G` for "which commits touched lines matching a pattern".

### 示例

找出某函数被删除的那次提交：

To find the commit that removed a function:

```bash
git log -S "def old_function" --oneline -- path/to/file.py
```

找出所有改动过包含 `TODO` 的行的提交：

To find all commits that changed lines containing `TODO`:

```bash
git log -G "TODO" --oneline
```

追踪某函数从引入到现在的完整历史：

To trace a function's full history:

```bash
git log -L :my_function:src/module.c
```

### ⚠️注意事项

- `-S` 后跟的是要统计出现次数变化的字符串（不加 `--pickaxe-regex` 时按字面匹配）；`-G` 后跟的是正则，二者行为不同，别混用。
- `-S` 按“净变化”计数：若一次提交里增删数量相同，它可能漏报，此时 `-G` 更可靠。
- 多个 `--grep` 默认是“或”的关系，要“与”需加 `--all-match`；`--author`、`--committer`、`--since`/`--until` 可与 `--grep` 组合做精确过滤。
- `git grep` 搜索范围大时（如遍历全部历史）会很慢，尽量限定路径或版本。
- 合并提交默认可能被跳过，需要时加 `--full-history` 或 `-m`。

- `-S` takes a string whose occurrence count is being tracked (a literal match unless `--pickaxe-regex` is added); `-G` takes a regex — the two behave differently, so do not confuse them.
- `-S` counts net change: it can miss a commit where additions equal deletions, so `-G` is more reliable in that case.
- Multiple `--grep` flags are OR-ed by default; add `--all-match` for AND. Combine `--author`, `--committer`, and `--since`/`--until` with `--grep` for precise filtering.
- `git grep` over a large scope (e.g. the entire history) is slow; restrict the path or revision when possible.
- Merge commits may be skipped by default; add `--full-history` or `-m` when needed.

> 出处 Searching：https://git-scm.com/book/en/v2/Git-Tools-Searching
