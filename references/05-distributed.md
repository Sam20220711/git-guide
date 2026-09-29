# Distributed Git

本章讲解分布式工作环境下的 Git 协作：如何选择合适的工作流、如何以贡献者身份提交补丁、以及如何以维护者身份整合贡献。所有内容源自 Pro Git 官方文档第五章。

This chapter covers Git collaboration in distributed environments: how to choose a suitable workflow, how to contribute patches as a contributor, and how to integrate contributions as a maintainer. All content is drawn from Chapter 5 of the official Pro Git book.

## Distributed Workflows

集中式工作流（Centralized Workflow）与传统的 SVN 类似：所有开发者都把代码推送到同一个中央仓库（hub），克隆后各自在本地开发，然后把变更推回中央仓库。它规则简单、易于上手，适合小团队，但缺点是中央仓库成为单点依赖，且开发者之间不能直接交换变更。

The centralized workflow resembles traditional SVN: every developer pushes to one central repository (the hub), clones it, works locally, and pushes changes back. It is simple and easy to adopt for small teams, but the central repository becomes a single point of dependency and developers cannot exchange changes directly with one another.

集成管理员工作流（Integration-Manager Workflow）为每个开发者分配一个公开仓库，并设有一个"神圣仓库"（blessed repository）代表正式项目。贡献者克隆神圣仓库、在本地开发、推送到自己的公开仓库，然后请求维护者把变更拉入神圣仓库。维护者可以本地测试后再合并，这是 GitHub、GitLab 等平台的典型模型。

In the integration-manager workflow each developer gets a public repository, plus one "blessed repository" representing the official project. Contributors clone the blessed repository, develop locally, push to their own public repository, and ask the maintainer to pull the changes in. The maintainer can test locally before merging — this is the model behind GitHub, GitLab, and similar platforms.

司令官与副官工作流（Dictator and Lieutenants Workflow）用于大型项目（如 Linux 内核）：司令官（dictator）只维护一个参考仓库，若干副官（lieutenant）各自负责一个子系统，开发者只与对应的副官交互，副官整合后再提交给司令官。这种层级结构把整合工作分散到多个层次。

The dictator and lieutenants workflow suits huge projects (like the Linux kernel): a dictator maintains one reference repository, several lieutenants each own a subsystem, developers interact only with their lieutenant, and each lieutenant integrates and forwards to the dictator. This hierarchy spreads integration work across multiple levels.

```bash
git clone <hub-url>        # 克隆中央/神圣仓库作为起点
git push origin <branch>   # 把本地分支推送到自己的公开仓库
git pull origin <branch>   # 从上游拉取并合并最新变更
```

`git clone` 初始化本地副本；`git push` 把本地提交发布到远端；`git pull` 是 `fetch` + `merge` 的组合，用于同步上游变更。选择工作流的关键在于：谁拥有权威仓库、谁能推送、变更如何汇总。

`git clone` initializes a local copy; `git push` publishes local commits to a remote; `git pull` combines `fetch` and `merge` to synchronize upstream changes. The essence of choosing a workflow is: who owns the authoritative repository, who can push, and how changes are aggregated.

**示例**：以集成管理员工作流为例，贡献者 A 维护自己的公开仓库，维护者 M 通过 `git remote add contributorA <A-url>` 添加远端，再执行 `git pull contributorA feature-x` 拉取并审查后合并。

**Example**: in the integration-manager workflow, contributor A keeps a public repository; maintainer M adds it with `git remote add contributorA <A-url>`, then runs `git pull contributorA feature-x` to pull, review, and merge.

⚠️ 注意事项：集中式工作流中多人同时强推容易互相覆盖，应养成先 `git pull --rebase` 再 `git push` 的习惯；集成管理员工作流要求维护者逐一审查，避免直接盲合未经测试的提交。

⚠️ Caution: in the centralized workflow, simultaneous force-pushes can overwrite each other, so pull (ideally `git pull --rebase`) before pushing; in the integration-manager workflow the maintainer must review each change rather than blindly merging untested commits.

> 出处 Distributed Workflows：https://git-scm.com/book/en/v2/Distributed-Git-Distributed-Workflows

## Contributing to a Project

贡献者最常见的起点是克隆仓库、在本地建一个主题分支（topic branch）开发，完成后用 `git format-patch` 生成补丁，通过邮件（`git send-email`）或平台提交给维护者。补丁方式适合没有推送权限或采用邮件列表流程的项目。

A contributor typically starts by cloning the repository, creating a topic branch locally, and when done generating patches with `git format-patch`, then sending them by email (`git send-email`) or through a platform. The patch route suits projects where you lack push access or that use a mailing-list workflow.

```bash
git clone <url>                              # 克隆上游仓库
git checkout -b topic-awesome                # 新建并切换到主题分支
git commit -am "Implement awesome feature"   # 提交变更
git format-patch -M origin/master            # 生成补丁文件（-M 检测重命名）
git send-email *.patch                       # 通过 SMTP 发送补丁
```

`git format-patch` 把每个提交转换成一个 mbox 格式的补丁文件，适合邮件投递；`-M` 让 Git 在补丁中检测重命名。`git send-email` 逐封发送补丁并自动保留补丁的提交信息与作者信息。

`git format-patch` turns each commit into an mbox-format patch file suitable for email; `-M` enables rename detection in patches. `git send-email` sends each patch while preserving commit messages and authorship.

若拿到的是普通 diff 补丁（非 mbox），维护者一侧用 `git apply`；若是 mbox 格式的邮件补丁，用 `git am`。两者差别在于 `git apply` 只改动工作区、不创建提交，而 `git am` 会把补丁当作提交逐个应用并保留作者与提交信息。

When the patch is a plain diff, the receiving side uses `git apply`; for mbox email patches it uses `git am`. The difference: `git apply` only modifies the working tree without creating commits, while `git am` applies each patch as a commit and preserves authorship and commit messages.

```bash
git apply /tmp/patch.diff      # 应用普通 diff 补丁到工作区
git apply --check patch.diff   # 先校验补丁能否干净应用，不实际修改
git am 0001-awesome.patch      # 将 mbox 补丁应用为一个提交
git am -3 0001-awesome.patch   # 冲突时尝试三方合并（-3 / --3way）
git am --abort                 # 应用中断时放弃并恢复原状
```

`git apply --check` 是安全的预检手段；`git am -3` 在出现冲突时尝试使用三方合并，失败后可手动解决冲突再 `git am --resolved` 继续。

`git apply --check` is a safe dry-run; `git am -3` attempts a three-way merge on conflict, and after resolving conflicts manually you continue with `git am --resolved`.

**示例**：贡献者在一个私有小组项目中拥有推送权限，可以直接在共享分支上工作；若项目规模较大，则应该采用"先拉取再变基"的流程，在推送前让本地提交线性地排在远端提交之后。

**Example**: in a private small-team project where contributors have push access, they may work directly on shared branches; in larger projects they should rebase onto the remote before pushing so local commits sit linearly on top of remote commits.

⚠️ 注意事项：不要直接改动历史中已经公开的提交；`git rebase` 会重写提交（见下节风险提示），应只对尚未推送的本地提交使用。提交信息应遵循项目规范：首行用祈使句简述、空行后再写详细说明。

⚠️ Caution: never rewrite commits that have already been made public; `git rebase` rewrites history (see the risk note below) and should only target commits that have not yet been pushed. Follow the project's commit-message convention: an imperative summary line, a blank line, then a detailed description.

> 出处 Contributing to a Project：https://git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project

## Maintaining a Project

维护者通常在专属的主题分支里工作，而不是直接在 `master` 上开发；当需要整合某个贡献时，先检出对应的远程分支到本地，再决定合并、变基或挑选提交。

A maintainer usually works in dedicated topic branches rather than directly on `master`; to integrate a contribution, they first check out the contributor's remote branch locally, then decide whether to merge, rebase, or cherry-pick.

```bash
git remote add contributor <url>             # 添加贡献者仓库为远端
git fetch contributor                        # 拉取贡献者所有分支
git checkout -b feature-x contributor/feature-x   # 检出贡献者分支到本地
git log --no-merges master..feature-x        # 查看该分支相对 master 新增的提交
git diff master...feature-x                  # 三点语法：只看分支自分离点以来的改动
```

`git log --no-merges master..feature-x` 列出 feature-x 有而 master 没有的非合并提交；`git diff master...feature-x`（三个点）只显示 feature-x 从两者共同祖先之后引入的变化，是审查补丁最常用的命令。

`git log --no-merges master..feature-x` lists the non-merge commits in feature-x not in master; `git diff master...feature-x` (three dots) shows only the changes feature-x introduced since the common ancestor — the most common review command.

当只想把某一次提交引入另一个分支时用 `git cherry-pick`；当需要对一批提交做压缩、编辑、重排时用交互式变基 `git rebase -i`。

Use `git cherry-pick` to bring a single commit into another branch; use interactive rebase `git rebase -i` to squash, edit, or reorder a series of commits.

```bash
git cherry-pick e43a6fd3      # 将指定提交复制到当前分支
git cherry-pick -x e43a6fd3   # -x 在提交信息里记录原提交，便于溯源
git rebase -i HEAD~5          # 交互式变基最近 5 个提交（可 squash/edit/reorder）
git rebase --onto newbase oldbase feature   # 把 feature 中 oldbase 之后的提交接到 newbase 上
```

`git cherry-pick` 会生成一个新提交（哈希不同），适合把一个修复移植到多个分支；`git rebase -i` 打开编辑器，可把多个提交压缩为一个、修改提交信息或调整顺序；`git rebase --onto` 用于把一段提交"搬迁"到新的基底。

`git cherry-pick` produces a new commit (different hash) and suits porting one fix to several branches; `git rebase -i` opens an editor to squash several commits into one, edit messages, or reorder; `git rebase --onto` moves a range of commits onto a new base.

发布版本时，维护者常打带签名的标签并用 `git archive` 打包源代码。

When releasing, maintainers often create a signed tag and package the source with `git archive`.

```bash
git tag -s v1.0 -m "Release 1.0"   # 创建 GPG 签名标签
git archive --prefix=proj-1.0/ v1.0 | gzip > proj-1.0.tar.gz   # 打包源码
git shortlog --no-merges master..release   # 生成按作者分组的变更摘要
```

`git tag -s` 创建带签名的标签用于验证发布完整性；`git archive` 生成不含 `.git` 目录的源码快照；`git shortlog` 汇总各作者的提交，常用于撰写发布说明。

`git tag -s` creates a signed tag to verify release integrity; `git archive` produces a source snapshot without the `.git` directory; `git shortlog` summarizes commits by author, useful for release notes.

⚠️ 注意事项：`git rebase` 属于历史重写，⚠️ 风险——它会改写提交哈希，若这些提交已被他人克隆或合并，会造成历史分叉，且之后只能强制推送（`git push --force`）覆盖远端，破坏协作。安全替代：对未公开的本地提交才使用 rebase；对已公开的提交改用 `git merge` 或 `git cherry-pick`。若确需强推，用 `git push --force-with-lease` 代替 `--force`，它会在远端已被他人更新时拒绝推送，避免误覆盖。同样地，`git am` 与 `git cherry-pick` 会复制提交（哈希不同），长期维护时可能产生重复内容，应保持单一整合入口并记录来源。

⚠️ Caution: `git rebase` rewrites history — ⚠️ risk: it changes commit hashes, and if those commits were already cloned or merged by others it causes divergent history and forces a `git push --force` that overwrites the remote and breaks collaboration. Safer alternative: rebase only local, unpublished commits; for published commits use `git merge` or `git cherry-pick` instead. If you must force-push, use `git push --force-with-lease` rather than `--force` — it refuses to push when the remote has been updated by others, preventing accidental overwrites. Likewise `git am` and `git cherry-pick` duplicate commits (new hashes), which over time can create duplicated content; keep a single integration point and record provenance.

> 出处 Maintaining a Project：https://git-scm.com/book/en/v2/Distributed-Git-Maintaining-a-Project
