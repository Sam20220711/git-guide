# Git Branching

## Branches in a Nutshell

分支在 Git 中本质只是一个指向某次提交（commit）的可移动轻量指针。Git 默认分支名为 `master`（现在更多用 `main`），它会在每次提交后自动向前移动。创建分支只是新建一个指针，切换分支则把 `HEAD` 指向该指针，因此创建与切换的开销几乎为零。分支让我们可以在独立线路上并行开发，互不干扰。

A branch in Git is simply a lightweight, movable pointer to a commit. The default branch name is `master` (increasingly `main`), and it automatically advances on each commit. Creating a branch merely creates a new pointer; switching a branch just points `HEAD` at that pointer, so both operations are nearly free. Branches let you develop on independent lines of work in parallel without interfering with each other.

```bash
git branch testing          # 创建 testing 分支，指向当前提交
git checkout testing        # 切换 HEAD 到 testing 分支
git switch testing          # 新版等价命令，语义更清晰
git checkout -b testing     # 创建并切换（一步完成）
git switch -c testing       # 新版等价命令
```

示例：在 `testing` 分支上提交后，只有 `testing` 指针前移，`master` 仍停留在原地；切回 `master` 再做一次提交，两条线就自然分叉了。

Example: after committing on `testing`, only the `testing` pointer advances while `master` stays put; switching back to `master` and committing again naturally forks the two lines.

⚠️ 注意：`git checkout` 在旧版中同时承担「切换分支」与「恢复文件」两种职责，容易误操作；新项目建议统一使用 `git switch`（切分支）和 `git restore`（恢复文件）以避免歧义。分支名只是指针，删除分支不会删除提交本身。

⚠️ Note: older `git checkout` overloads both "switch branch" and "restore file", which is error-prone; prefer `git switch` (switch) and `git restore` (restore) in new work. A branch is only a pointer — deleting it does not delete the commits.

> 出处 Branches in a Nutshell：https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell

## Basic Branching and Merging

日常开发最常见的工作流是：在新分支上开发某个功能或修复，完成后再把改动合并回主线。合并（merge）有两种情况：一是「快进合并」（fast-forward），当目标分支没有新的提交时，指针直接前移即可；二是「三方合并」（three-way merge），当两条线都已各自前移时，Git 找到共同祖先并生成一个合并提交。

The everyday workflow is to develop a feature or fix on a new branch and then merge it back. Merging has two cases: a fast-forward merge, where the target has no new commits and the pointer simply advances; and a three-way merge, where both lines have moved and Git finds the common ancestor and creates a merge commit.

```bash
git checkout -b iss53          # 开新分支开发
git commit -am "fix issue"     # 提交改动
git checkout master            # 切回主线
git merge iss53                # 合并 iss53（可能快进）
git branch -d iss53            # 合并完成后安全删除分支
```

示例（快进）：`master` 没有新提交时，`git merge` 直接让 `master` 指针前移到 `iss53` 的最新提交，历史保持线性。

Example (fast-forward): when `master` has no new commits, `git merge` simply advances `master` to `iss53`'s tip, keeping history linear.

示例（三方合并）：若 `master` 与 `iss53` 都已各自前移，Git 会基于共同祖先 `C2` 做一次三方合并，自动生成一个新的「合并提交」，它有两个父提交。

Example (three-way merge): if both `master` and `iss53` advanced, Git performs a three-way merge on the common ancestor `C2` and automatically creates a merge commit with two parents.

### 解决冲突（Basic Merge Conflicts）

当两个分支改了同一文件的同一处内容时，合并无法自动完成，Git 会标记冲突并暂停。此时需要用 `git status` 找到冲突文件，手动编辑解决，再 `git add` 标记为已解决并提交。

When two branches change the same part of the same file, Git cannot merge automatically; it marks the conflict and pauses. Resolve it by locating conflicted files with `git status`, editing them by hand, then `git add` to mark resolution and commit.

```bash
git merge iss53                 # 触发冲突，Git 暂停
git status                      # 查看哪些文件处于冲突状态
# 编辑冲突文件，删除 <<<<<<< ======= >>>>>>> 标记并保留最终内容
git add <file>                  # 标记冲突已解决
git commit                      # 完成合并提交
git mergetool                   # 可选：启动可视化合并工具
```

⚠️ 注意：冲突标记形如 `<<<<<<< HEAD`、`=======`、`>>>>>>> iss53`，必须全部删除后再提交。解决冲突后若想放弃本次合并，可用 `git merge --abort` 回到合并前状态。提交前务必 `git status` 确认没有遗漏的冲突文件。

⚠️ Note: conflict markers look like `<<<<<<< HEAD`, `=======`, `>>>>>>> iss53` and must all be removed before committing. To abandon a conflicted merge, use `git merge --abort` to return to the pre-merge state. Always run `git status` before committing to confirm no conflicted file is missed.

> 出处 Basic Branching and Merging：https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging

## Branch Management

`git branch` 不带参数会列出所有本地分支；带 `-v` 显示每个分支最后一次提交；带 `--merged` / `--no-merged` 分别列出已合并（可安全删除）与尚未合并的分支。合并完成后用 `git branch -d` 删除已合并分支，用 `-D` 强制删除未合并分支（需谨慎）。

Running `git branch` alone lists local branches; `-v` shows each branch's last commit; `--merged` / `--no-merged` list branches already merged (safe to delete) versus those not yet merged. After merging, delete merged branches with `git branch -d`; use `-D` to force-delete an unmerged branch (with care).

```bash
git branch                  # 列出本地分支
git branch -v               # 显示分支与各自最新提交
git branch --merged         # 列出已合并到当前分支的分支
git branch --no-merged      # 列出尚未合并的分支
git branch -d iss53         # 删除已合并分支
git branch -D iss53         # 强制删除（即使未合并）
```

示例：完成合并后运行 `git branch --merged`，输出中仍带 `*` 的是当前分支，其余未带星号的通常可以 `-d` 删除。

Example: after merging, run `git branch --merged`; the entry marked with `*` is the current branch, and other unstarred entries can usually be deleted with `-d`.

⚠️ 注意：`-D` 会丢弃尚未合并的提交，属不可逆操作。删除前先 `git branch --no-merged` 核对，或用 `git log <branch>` 确认该分支的提交是否真的不再需要。被删除分支的提交若仍被 reflog 或其它引用指向，仍可找回（见 Rebasing 一节）。

⚠️ Note: `-D` discards unmerged commits irreversibly. Before deleting, verify with `git branch --no-merged` or `git log <branch>` that the commits are truly no longer needed. Commits of a deleted branch can still be recovered if reflog or another ref points to them (see the Rebasing section).

> 出处 Branch Management：https://git-scm.com/book/en/v2/Git-Branching-Branch-Management

## Branching Workflows

因为分支成本极低，Git 支持多种协作工作流。常见的有：长期分支（long-running branches），如 `master` 只放稳定代码、`develop` 或 `next` 放待稳定代码；主题分支（topic branches），每个功能或修复用独立短命分支；以及只维护旧版本的维护分支。核心原则是：稳定代码只通过合并进入更稳定的分支，而不是直接在上面开发。

Because branches are cheap, Git supports several collaborative workflows: long-running branches, such as `master` for stable code and `develop` or `next` for code being stabilized; topic branches, one short-lived branch per feature or fix; and maintenance branches for older releases. The core principle is that stable code only enters more stable branches through merges, not by committing directly on them.

```bash
git checkout master        # 长期稳定分支
git merge develop          # 将 develop 的稳定改动合并进 master
git checkout -b feature-x  # 为单个功能创建主题分支
# ... 开发并提交 ...
git checkout master
git merge feature-x        # 完成后合并回主线
git branch -d feature-x    # 删除已合并的主题分支
```

示例：一个项目可以同时拥有 `master`（发布）、`develop`（集成）以及多个 `feature-*`（进行中的功能），并让更稳定的分支只通过合并吸收下游改动，从而保持各层级代码质量。

Example: a project may hold `master` (release), `develop` (integration), and several `feature-*` branches in progress, with more stable branches absorbing downstream changes only via merge, preserving quality at each level.

⚠️ 注意：工作流没有银弹，关键是团队一致约定分支的命名、寿命与合并方向。避免在一个分支上堆积过多无关改动，主题分支应保持小而聚焦、尽快合并并删除。

⚠️ Note: no workflow is a silver bullet; the key is team-wide agreement on naming, lifetime, and merge direction. Avoid piling unrelated changes onto one branch — topic branches should stay small, focused, merged, and deleted promptly.

> 出处 Branching Workflows：https://git-scm.com/book/en/v2/Git-Branching-Branching-Workflows

## Remote Branches

远程分支（remote-tracking branches）是本地对远程仓库状态的只读引用，形如 `origin/main`，不能直接在其上提交，只能在本地分支上工作后再推送到远程。拉取用 `git fetch`（只更新远程引用，不影响工作目录）或 `git pull`（等于 fetch + merge）。推送用 `git push origin <branch>`；想删除远程分支则用 `git push origin --delete <branch>`。

Remote-tracking branches are local read-only references to the state of a remote repository, such as `origin/main`. You cannot commit on them directly; work on local branches and then push. Fetch with `git fetch` (updates remote refs only, without touching your working tree) or `git pull` (fetch + merge). Push with `git push origin <branch>`; delete a remote branch with `git push origin --delete <branch>`.

```bash
git fetch origin                      # 更新所有远程跟踪分支
git checkout -b serverfix origin/serverfix  # 基于远程分支创建本地分支
git switch -c serverfix origin/serverfix    # 新版等价命令
git push origin serverfix             # 推送本地分支到远程
git push origin serverfix:serverfix   # 显式指定本地:远程 名称
git push origin --delete serverfix    # 删除远程分支
git branch -r                         # 列出远程跟踪分支
git branch -vv                        # 查看本地分支与上游的追踪关系
```

示例：`git checkout -b serverfix origin/serverfix` 会创建本地 `serverfix` 并自动设置上游（tracking），之后在本地提交、`git push` 即可同步到远程同名分支。

Example: `git checkout -b serverfix origin/serverfix` creates a local `serverfix` and sets up tracking automatically, after which you commit locally and `git push` to sync to the remote branch of the same name.

⚠️ 注意：`git fetch` 只下载数据、更新 `origin/*` 引用，不会合并到你的工作分支；想真正同步需再 `git merge` 或用 `git pull`。推送前若远程已有他人提交，push 会被拒绝，需先 fetch 并合并（或变基）再推送。删除远程分支是不可逆的对外操作，删除前务必与团队确认。

⚠️ Note: `git fetch` only downloads data and updates `origin/*` refs — it does not merge into your working branch; to truly sync, merge or use `git pull`. If others have pushed first, your push is rejected — fetch and merge (or rebase) first. Deleting a remote branch is an irreversible outward action; confirm with the team first.

> 出处 Remote Branches：https://git-scm.com/book/en/v2/Git-Branching-Remote-Branches

## Rebasing

变基（rebase）是另一种整合改动的方式：把当前分支「重放」到目标分支的最新提交之上，得到线性历史；与之相比，merge 会保留两条线的分叉并生成合并提交。Rebase 让历史更整洁，但会改写提交（生成新提交对象），因此对已公开、他人可能基于其工作的分支做 rebase 是危险的。

Rebasing is another way to integrate work: it replays your branch onto the tip of the target branch, producing linear history; merge, by contrast, preserves the fork and creates a merge commit. Rebase yields cleaner history but rewrites commits (creating new commit objects), so rebasing branches that are already public and possibly used by others is dangerous.

```bash
git checkout experiment
git rebase master          # 把 experiment 的提交重放到 master 之上
git checkout master
git merge experiment       # 此时为快进合并，历史保持线性
git rebase --onto master server client   # 只把 client 中不在 server 上的提交重放到 master
git rebase --continue      # 解决冲突后继续变基
git rebase --abort         # 放弃变基，回到变基前状态
git pull --rebase          # 拉取时先变基再合并，保持历史线性
```

示例：在 `experiment` 上 `git rebase master` 后，`experiment` 的提交 `C4` 被重放到 `master` 的 `C3` 之上成为 `C4'`（新提交），再切回 `master` 合并即为快进，历史变成一条直线。

Example: after `git rebase master` on `experiment`, its commit `C4` is replayed onto `master`'s `C3` as a new commit `C4'`; switching back to `master` and merging is then a fast-forward, producing a linear history.

### 变基的风险与回退（The Perils of Rebasing）

⚠️ 风险：不要对已经推送到公共仓库、他人可能已拉取并基于其工作的分支执行 rebase，因为改写会生成新提交，导致协作者的历史与你「分叉」，产生大量重复与混乱。也不要 rebase 已经发布过的提交。

⚠️ Risk: never rebase commits that exist outside your repository and that people may have based work on, because rewriting creates new commits and forks your collaborators' history, causing duplication and chaos. Do not rebase commits you have already pushed.

安全替代：对公共分支用 merge 而非 rebase；只有在本地私有分支上做 rebase 整理历史。若要更新已推送分支的改动，用 `git push --force-with-lease`（而非 `--force`），它会在远程分支未被他人改动时才强制覆盖，更安全。

Safe alternatives: use merge instead of rebase on public branches; only rebase private local branches to tidy history. To update an already-pushed branch, use `git push --force-with-lease` (not `--force`), which force-overwrites only if the remote branch has not been changed by others — a safer option.

回退手段：若 rebase 出错或误操作，可借助 reflog 找回改写前的提交，用 `git reset --hard <sha>` 或新建分支恢复。

Recovery: if a rebase goes wrong, use the reflog to locate the pre-rewrite commits and recover with `git reset --hard <sha>` or by creating a new branch.

```bash
git reflog                          # 查看 HEAD 的历史记录，找到改写前的 SHA
git reset --hard <sha>              # 回退到该提交（丢弃工作区改动）
git branch recover <sha>            # 更安全：基于旧提交新建分支
git push --force-with-lease         # 更新远程时更安全的强推
```

⚠️ 注意：`git reset --hard` 会丢弃未提交的工作区改动，操作前确保已提交或暂存重要内容。强推会覆盖远程历史，属不可逆对外操作；优先用 `--force-with-lease`，并确认协作者已知情。

⚠️ Note: `git reset --hard` discards uncommitted working-tree changes — commit or stash important work first. Force-pushing overwrites remote history and is an irreversible outward action; prefer `--force-with-lease` and make sure collaborators are informed.

> 出处 Rebasing：https://git-scm.com/book/en/v2/Git-Branching-Rebasing
