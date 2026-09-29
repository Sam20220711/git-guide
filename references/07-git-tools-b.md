# Git Tools（下）：改写历史 / reset / 高级合并 / 调试 / 子模块

本章涵盖 Git 的高阶工具：改写历史、深入理解 `reset`、高级合并、`rerere`、`bisect` 调试、子模块、打包、替换对象与凭据存储。这些命令能显著提升日常效率，但也包含危险操作，务必先阅读各小节的 ⚠️ 风险提示。

This chapter covers Git's advanced tools: rewriting history, understanding `reset` in depth, advanced merging, `rerere`, debugging with `bisect`, submodules, bundling, object replacement, and credential storage. These commands can greatly boost daily productivity, but some are dangerous — always read the ⚠️ risk notes in each section first.

## Rewriting History

### 概念

改写历史指修改已经存在的提交（commit），例如合并多个提交、修改提交信息、调整提交顺序或拆分提交。它的核心命令是 `git rebase -i`（交互式变基）与 `git commit --amend`。改写历史会生成全新的提交对象（新的 SHA-1 哈希），因此旧提交并未消失，而是被新提交替代。

Rewriting history means modifying commits that already exist — for example squashing several commits, editing commit messages, reordering commits, or splitting a commit. The core commands are `git rebase -i` (interactive rebase) and `git commit --amend`. Rewriting history produces brand-new commit objects (new SHA-1 hashes), so the old commits do not vanish; they are simply replaced by new ones.

### 命令

```bash
# 修改最近一次提交的说明或内容
git commit --amend

# 对最近 3 个提交进行交互式改写
git rebase -i HEAD~3
```

交互式变基的指令（rebase todo 列表）如下：

```bash
pick   使用该提交（保留）
reword 使用该提交，但修改提交说明
edit   使用该提交，但停下来手动修改
squash 将该提交合并到上一个提交，并合并说明
fixup  将该提交合并到上一个提交，丢弃说明
drop   删除该提交
```

### 示例

把最近三次提交合并成一次：在 `git rebase -i HEAD~3` 打开的编辑器中，把第二、第三行由 `pick` 改为 `squash`：

```bash
pick  a1b2c3d 第一次提交
squash d4e5f6a 第二次提交
squash g7h8i9b 第三次提交
```

保存退出后 Git 会依次执行，并让你编辑合并后的提交说明。

Modify an older commit's message: run `git rebase -i HEAD~3`, change that line's `pick` to `reword`, save, and Git opens the editor again for the new message.

### ⚠️ 注意事项

- ⚠️ 风险：改写历史会重写所有后续提交的哈希。如果这些提交已被推送到远程并有人协作，改写会破坏他人的工作。**不要改写已推送的共享历史。**
- ⚠️ 风险：`git commit --amend` 会替换最近一次提交；若它已被推送，再次推送需要强制推送（见下方安全替代）。
- 安全替代：对已推送的提交，优先新建一个修复提交，而不是改写；确需改写时用 `git push --force-with-lease`（见 Credential/推送部分的安全说明）。
- 回退：改写前记录当前提交 `git rev-parse HEAD` 或用 `git branch backup` 打一个备份分支；任何意外都可用 `git reflog` 找回旧提交。
- 建议：`git rebase -i` 只在本地未推送的分支上使用。
- ⚠️ 风险：`git filter-branch`（批量改写历史，如从所有提交中删除某文件）是“核弹级”操作，会重写海量提交，容易出错且极慢。现代推荐用 `git filter-repo` 替代，并在运行前完整备份仓库。

> 出处 Rewriting History：https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History

## Reset Demystified

### 概念

`git reset` 与 `git checkout` 是最容易混淆的两条命令。理解 `reset` 的关键是“三棵树”模型：

- HEAD：当前所在分支指向的最近一次提交（上一次提交的快照）。
- Index / 暂存区（staging area）：下一次提交的候选内容。
- Working Directory（工作目录）：磁盘上实际的文件内容。

`reset` 以三种模式在这三棵树之间移动：

- `--soft`：只移动 HEAD（分支指针），暂存区与工作目录不变。相当于“撤销提交但保留全部改动在暂存区”。
- `--mixed`（默认）：移动 HEAD 并重置暂存区，工作目录不变。相当于“撤销提交并取消暂存，改动留在工作目录”。
- `--hard`：移动 HEAD、重置暂存区、并重置工作目录。三棵树全部对齐到目标提交，**未提交的改动会被彻底丢弃**。

Understanding `git reset` requires the "three trees" model:

- HEAD: the commit your current branch points to (snapshot of the last commit).
- Index / staging area: the candidate content of the next commit.
- Working directory: the actual files on disk.

`reset` moves among these three trees in three modes:

- `--soft`: moves only HEAD (the branch pointer); the index and working directory are untouched. "Undo the commit but keep all changes staged."
- `--mixed` (default): moves HEAD and resets the index; the working directory is untouched. "Undo the commit and unstage, leaving changes in the working directory."
- `--hard`: moves HEAD, resets the index, and resets the working directory. All three trees align to the target commit; **uncommitted changes are destroyed.**

### 命令

```bash
git reset --soft HEAD~1    # 撤销上一次提交，改动保留在暂存区
git reset --mixed HEAD~1   # 撤销上一次提交，改动保留在工作目录（默认）
git reset --hard HEAD~1    # 撤销上一次提交并丢弃所有改动（危险）
git reset <commit> <path>  # 取消指定文件的暂存（本质是重置 index 中的该文件）
```

### 示例

提交错了，想把改动拿回来重新分多次提交：

```bash
git reset --soft HEAD~1   # 撤销提交，改动回到暂存区
git status                # 确认文件仍在暂存区
git commit -m "拆分的第一个提交"
```

只想取消暂存（把文件从暂存区放回工作目录）：`git reset HEAD file.txt`（即 `--mixed` 的路径形式）。

### ⚠️ 注意事项

- ⚠️ 风险：`git reset --hard` 会**永久丢弃**工作目录与暂存区中未提交的改动，无法通过普通 `git status` 找回。
- 回退：`git reset --hard` 之后，仍可能通过 `git reflog` 找回被移动之前的提交（前提是改动曾提交过）；但纯未提交的工作目录改动无法找回。
- 安全替代：需要丢弃工作目录改动前，先 `git stash` 暂存起来；需要回退已推送提交时，优先用 `git revert`（生成反向提交，不改写历史）。
- 习惯：不确定时用 `--soft` 或 `--mixed`，把 `--hard` 留到最后。

> 出处 Reset Demystified：https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified

## Advanced Merging

### 概念

普通合并（`git merge`）在多数情况下能自动完成，但复杂冲突需要更精细的工具。本节的要点是：冲突发生时 Git 会把冲突双方标记在工作目录文件中，同时保留合并三方的“阶段”（stage）信息，可用 `git ls-files -u` 查看。高级合并还涉及“我们的/他们的”（ours/theirs）语义、忽略空白差异，以及合并前先使工作目录干净。

Ordinary merges usually complete automatically, but complex conflicts need finer tools. The key idea: on conflict, Git marks both sides inside the working-directory file and keeps three "stages" of each conflicted file, viewable with `git ls-files -u`. Advanced merging also covers the "ours/theirs" semantics, ignoring whitespace differences, and keeping the working directory clean before merging.

### 命令

```bash
git ls-files -u                 # 列出冲突文件及其三个 stage
git merge-file -p ours.$$ common.$$ theirs.$$ > merged  # 手动三路合并单个文件
git merge -Xignore-all-space     # 合并时忽略空白差异
git merge -Xours                 # 冲突时一律采用“我们的”版本
git merge -Xtheirs               # 冲突时一律采用“他们的”版本
git diff --ours / --theirs       # 查看冲突文件各自的一侧
```

### 示例

合并 `feature` 分支时希望所有冲突自动采用当前分支版本：

```bash
git merge -Xours feature
```

查看某个冲突文件在“我们这一侧”与合并基准之间的差异：

```bash
git diff --ours -- path/to/file
git diff --base -- path/to/file
git diff --theirs -- path/to/file
```

### ⚠️ 注意事项

- 语义陷阱：`-Xours`/`-Xtheirs` 对 `rebase` 而言是**反的**（rebase 中“theirs”指的是你正在变基到的分支）。合并冲突时务必确认方向。
- 合并前先提交或暂存工作目录改动，避免把未完成工作混入合并结果。
- `-X`（策略选项）与 `-s`（策略）不同：`-X` 只微调合并策略的行为。

> 出处 Advanced Merging：https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging

## Rerere

### 概念

`rerere` 是 “reuse recorded resolution”（重用已记录的解决方案）的缩写。启用后，Git 会记住你解决某个冲突的方式；当同样的冲突再次出现（例如反复 rebase 同一分支），Git 会自动套用上次的解决方案，避免重复手动解决。

`rerere` stands for "reuse recorded resolution". When enabled, Git remembers how you resolved a conflict; when the same conflict appears again (for example when repeatedly rebasing a branch), Git automatically applies the previous resolution, avoiding repetitive manual work.

### 命令

```bash
git config --global rerere.enabled true   # 启用 rerere
git rerere diff                           # 查看当前冲突与已记录解决方案的差异
git rerere status                         # 查看将被 rerere 处理的文件
git rerere forget <pathspec>              # 忘记某文件的已记录解决方案
```

### 示例

启用后，第一次解决冲突，Git 会记录解决方案；随后撤销合并再次合并时，`git status` 会显示冲突已被自动解决（`Resolved ... with previous resolution`），只需 `git add` 后提交即可。

### ⚠️ 注意事项

- 若某次解决是临时或错误的，记得用 `git rerere forget` 清除，否则错误方案会被反复套用。
- rerere 记录存储在 `.git/rr-cache` 中，可提交前用 `git diff` 检查自动解决的合理性。

> 出处 Rerere：https://git-scm.com/book/en/v2/Git-Tools-Rerere

## Debugging with Git

### 概念

Git 提供两类调试工具：

- `git blame`：逐行标注文件内容最后一次修改的提交与作者，用于定位“这行代码是谁、在哪个提交改的”。
- `git bisect`：二分查找定位引入 bug 的提交。你提供一个“好的”提交与一个“坏的”提交，Git 自动在中间选取提交让你标记好坏，从而以对数复杂度找到第一个坏提交。

Git provides two kinds of debugging tools:

- `git blame`: annotates each line of a file with the commit and author that last changed it, to find "who changed this line and in which commit".
- `git bisect`: binary-searches for the commit that introduced a bug. You supply a "good" and a "bad" commit; Git checks out the midpoint, you mark it good or bad, and it converges to the first bad commit in logarithmic time.

### 命令

```bash
git blame -L 10,20 file.txt    # 查看 file.txt 第 10–20 行的逐行标注
git blame -C file.txt          # 追踪被移动/复制的行来源
git bisect start               # 开始二分
git bisect bad                 # 标记当前提交为坏
git bisect good <commit>       # 标记某提交为好
git bisect reset               # 结束二分，回到原分支
git bisect run <script>        # 用脚本自动判断好坏
```

### 示例

定位引入 bug 的提交：

```bash
git bisect start
git bisect bad          # 当前版本有问题
git bisect good v1.0    # v1.0 是好的
# Git 检出中间提交，测试后标记：
git bisect good         # 或 git bisect bad
# 重复直到找到第一个坏提交
git bisect reset
```

自动化二分（脚本返回 0 表示好、非 0 表示坏）：

```bash
git bisect start HEAD v1.0
git bisect run ./test-script.sh
```

### ⚠️ 注意事项

- `git bisect` 会不断切换检出提交，务必先保证工作目录干净或已提交。
- `git blame` 对整体重写/换行的文件可能产生误导，`-C`/`-M` 可辅助追踪拷贝与移动。

> 出处 Debugging with Git：https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git

## Submodules

### 概念

子模块（submodule）让你在一个 Git 仓库中嵌入另一个 Git 仓库，并在父仓库里记录指向子仓库**某个特定提交**的指针。父仓库只保存子模块的 URL 与提交哈希，不保存子模块文件内容。适合依赖独立版本控制的第三方库。

A submodule lets you embed another Git repository inside a Git repository, recording a pointer to a **specific commit** of the submodule. The parent stores only the submodule's URL and commit hash, not its file contents. It suits third-party libraries kept under independent version control.

### 命令

```bash
git submodule add <url> [path]      # 添加子模块
git submodule update --init         # 初始化并检出子模块内容
git submodule update --remote       # 更新子模块到其远程最新提交
git submodule status               # 查看子模块状态
git submodule foreach <command>    # 对每个子模块执行命令
```

增（add）：

```bash
git submodule add https://github.com/example/lib.git libs/lib
git commit -m "Add lib submodule"
```

删（remove，需分步进行）：

```bash
git submodule deinit -f -- libs/lib   # 从配置中移除
rm -rf .git/modules/libs/lib          # 删除 Git 元数据目录
git rm -f libs/lib                    # 从索引与工作目录移除
git commit -m "Remove lib submodule"
```

改（在子模块内提交，再在父仓库更新指针）：

```bash
cd libs/lib
git checkout main && git pull          # 或直接修改后提交
cd ../..
git add libs/lib                       # 记录新的子模块提交指针
git commit -m "Update lib submodule"
```

克隆含子模块的仓库：

```bash
git clone --recurse-submodules <url>
# 或克隆后再拉取
git submodule update --init --recursive
```

### 示例

父仓库中，子模块显示为带哈希的“新提交”（`modified: libs/lib (new commits)`），表示子模块指向了新的提交，需 `git add libs/lib` 记录后提交，其他人才会拿到新指针。

### ⚠️ 注意事项

- ⚠️ 风险：误删子模块会连同其 Git 历史元数据一起丢失。删除前务必按“deinit → 删除 `.git/modules/<path>` → `git rm`”完整流程，并确认子模块内没有未推送的本地提交。
- 常见坑：克隆父仓库后忘记 `git submodule update --init`，子模块目录为空。
- 指针语义：父仓库记录的是子模块的**具体提交**而非分支；更新子模块后必须在父仓库提交新的指针，否则他人仍停留在旧提交。
- 嵌套子模块可用 `--recursive` 一并处理。

> 出处 Submodules：https://git-scm.com/book/en/v2/Git-Tools-Submodules

## Bundling

### 概念

打包（bundle）把仓库（或其一段历史）封装成单个文件，用于在无网络或受限环境下传输 Git 数据。它可包含完整历史，也可只含某个区间，接收方用 `git clone` 或 `git fetch` 从 bundle 文件导入。

A bundle packages a repository (or a slice of its history) into a single file, for transferring Git data without network access. It can contain the full history or only a range; the receiver imports it with `git clone` or `git fetch`.

### 命令

```bash
git bundle create repo.bundle --all            # 打包全部分支历史
git bundle create range.bundle main~5..main    # 只打包最近 5 个提交
git bundle verify repo.bundle                  # 校验 bundle 完整性
git clone repo.bundle repo                     # 从 bundle 克隆
git fetch repo.bundle main:local-main          # 从 bundle 拉取到本地分支
```

### 示例

无网络时把 `main` 的最新改动打包发给同事：

```bash
git bundle create changes.bundle main~5..main
# 同事侧：
git fetch changes.bundle main:temp
git merge temp
```

### ⚠️ 注意事项

- 默认不包含未在指定引用范围内的对象；若 `--all` 之外还需某些 ref，用 `git bundle create f.bundle <refs...>` 显式列出。
- 接收方导入前可用 `git bundle verify` 确认文件完整、格式正确。

> 出处 Bundling：https://git-scm.com/book/en/v2/Git-Tools-Bundling

## Replace

### 概念

`git replace` 让 Git 在读取某个对象时，临时用另一个对象替换它，而**不实际改写历史**。常用于在不重写历史的前提下，把某个提交的父提交或树内容“虚拟替换”掉（例如一次性修复某条分支的合并基准）。它相当于在对象之上加了一层可逆的“重定向”。

`git replace` makes Git temporarily substitute another object when reading a given object, **without actually rewriting history**. It is often used to virtually swap a commit's parent or tree content without rewriting history (for example, to fix a branch's merge base in one shot). It adds a reversible "redirection" layer on top of objects.

### 命令

```bash
git replace <original> <replacement>   # 建立替换
git replace -l                        # 列出所有替换
git replace -d <original>             # 删除某个替换
git replace --graft <commit> <parent> # 改写某提交的父指针（虚替换）
```

### 示例

把 `feature` 分支的根提交重新接（graft）到 `main` 上，从而修正错误的合并基准：

```bash
git replace --graft <feature-root-commit> main
```

### ⚠️ 注意事项

- 替换只影响本仓库读取；要持久化或分享，需 `git push origin 'refs/replace/*'` 显式推送替换引用。
- 与真正改写历史不同，替换是**可逆**的：`git replace -d` 即可撤销。

> 出处 Replace：https://git-scm.com/book/en/v2/Git-Tools-Replace

## Credential Storage

### 概念

凭据存储（credential storage）用于缓存 HTTP(S) 认证的用户名与密码，避免每次操作重复输入。Git 提供多种存储后端：`cache`（内存中临时保存）、`store`（明文写入磁盘文件）、以及系统级助手（macOS Keychain、Windows Credential Manager、Linux 的 libsecret 等）。

Credential storage caches the username and password used for HTTP(S) authentication, avoiding repeated prompts. Git offers several backends: `cache` (temporary in-memory), `store` (plaintext on disk), and OS-level helpers (macOS Keychain, Windows Credential Manager, libsecret on Linux).

### 命令

```bash
git config --global credential.helper cache            # 内存缓存（默认约 15 分钟）
git config --global credential.helper 'cache --timeout=3600'  # 缓存 1 小时
git config --global credential.helper store            # 明文保存到 ~/.git-credentials
git config --global credential.helper manager-core     # Windows 凭据管理器
git config --global credential.helper osxkeychain      # macOS 钥匙串
git credential fill / approve / reject                 # 底层手动读写凭据
```

### 示例

让 Git 在 Windows 上使用系统凭据管理器保存 HTTPS 凭据：

```bash
git config --global credential.helper manager-core
```

之后首次推送输入用户名与令牌，Git 会交给凭据管理器保存，后续自动取用。

### ⚠️ 注意事项

- ⚠️ 风险：`store` 后端把密码**明文**写入 `~/.git-credentials`，任何能读该文件的用户都可获取密码。优先使用系统钥匙串/凭据管理器。
- 更安全的实践：HTTPS 下用个人访问令牌（PAT）代替密码，令牌可单独吊销。
- 强制推送安全：改写历史后需推送到已共享分支时，务必用 `git push --force-with-lease` 而非 `--force`。`--force-with-lease` 会先校验远程引用是否仍是你上次看到的状态，避免覆盖他人刚推送的提交。

> 出处 Credential Storage：https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage
