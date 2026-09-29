# GitHub

本章讲解基于 GitHub 的协作：账号与 SSH 配置、以贡献者身份参与项目、以维护者身份管理项目、组织（Organization）的团队与权限管理，以及通过 REST/GraphQL API 脚本化操作。所有内容源自 Pro Git 官方文档第六章。

This chapter covers collaboration on GitHub: account and SSH setup, contributing to a project, maintaining a project, managing teams and permissions in an Organization, and scripting via the REST/GraphQL API. All content is drawn from Chapter 6 of the official Pro Git book.

## Account Setup and Configuration

创建 GitHub 账号后，第一步是配置身份与认证。推荐使用 SSH 而非 HTTPS 密码：SSH 免密且更安全；同时应开启双重认证（2FA）保护账号。你的提交身份（姓名、邮箱）在本地 Git 配置中设置，它决定提交记录里显示的作者信息。

After creating a GitHub account, the first step is to configure identity and authentication. SSH is recommended over HTTPS passwords because it is passwordless and more secure; you should also enable two-factor authentication (2FA) to protect the account. Your commit identity (name and email) is set in local Git configuration and determines the author shown in commit history.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"   # 生成 SSH 密钥对
eval "$(ssh-agent -s)"                              # 启动 ssh-agent
ssh-add ~/.ssh/id_ed25519                           # 缓存私钥，避免重复输入密码短语
clip < ~/.ssh/id_ed25519                            # 复制公钥内容
ssh -T git@github.com                               # 测试 SSH 连接

git config --global user.name "Your Name"           # 设置本地提交身份
git config --global user.email "your_email@example.com"
```

`ssh-keygen` 生成一对 Ed25519 密钥，私钥留在本地，公钥粘贴到 GitHub 的 Settings → SSH and GPG keys 页面，让 GitHub 信任这台机器；`ssh -T` 成功时会输出 `Hi <username>! You've successfully authenticated`。提交身份则通过 `git config --global` 写入，决定未来所有提交的作者与邮箱。

`ssh-keygen` generates an Ed25519 key pair — the private key stays local while the public key is pasted into GitHub's Settings → SSH and GPG keys page so GitHub trusts this machine; on success `ssh -T` prints `Hi <username>! You've successfully authenticated`. The commit identity is set via `git config --global` and determines the author and email of all future commits.

**示例**：首次使用 GitHub CLI（`gh`）时直接完成认证，并选择 SSH 作为 Git 协议：

```bash
gh auth login     # 依次选择 GitHub.com → SSH → 浏览器授权或粘贴令牌
gh auth status    # 查看当前登录账号与所用协议
```

**Example**: on first use of the GitHub CLI (`gh`), authenticate directly and choose SSH as the Git protocol:

```bash
gh auth login     # GitHub.com → SSH → authorize in browser or paste token
gh auth status    # show the current account and protocol in use
```

⚠️ 注意事项：私钥（`~/.ssh/id_ed25519`）绝不能上传或分享，一旦泄露应立即在 GitHub 设置中移除对应公钥并重新生成；开启 2FA 后通过 HTTPS 推送需使用个人访问令牌（PAT）而非账号密码，因为 Git 无法在命令行交互式输入 2FA 验证码；不要把 `.env`、凭据或令牌提交进仓库——公开仓库中的任何密钥都应视为已泄露。

⚠️ Caution: never upload or share the private key (`~/.ssh/id_ed25519`); if it leaks, remove the corresponding public key in GitHub settings and regenerate immediately. After enabling 2FA, pushing over HTTPS requires a personal access token (PAT) instead of your password, because Git cannot prompt for 2FA codes interactively. Do not commit `.env`, credentials, or tokens — any secret in a public repository must be treated as compromised.

> 出处 Account Setup and Configuration：https://git-scm.com/book/en/v2/GitHub-Account-Setup-and-Configuration

## Contributing to a Project

贡献的核心模式是 fork + Pull Request（PR）：你没有目标仓库的写权限，因此先 fork 一份到自己账号下，在 fork 上创建特性分支、提交并推送，然后向原仓库发起 PR 请求合并。PR 是围绕一段提交的讨论线程，维护者可以评审、评论、请求改动，最终合并或关闭它。它既支持同一仓库内的分支，也支持跨 fork 的协作。

The core contribution pattern is fork + Pull Request (PR): because you lack write access to the target repository, you first fork a copy into your own account, create a feature branch on the fork, commit and push, then open a PR against the original repository to request a merge. A PR is a discussion thread around a set of commits; maintainers can review, comment, request changes, and finally merge or close it. It works both for branches in the same repository and for cross-fork collaboration.

```bash
gh repo fork octocat/Spoon-Knife --clone   # Fork 到自己的账号并克隆到本地
cd Spoon-Knife
git checkout -b my-feature                 # 基于最新主分支创建特性分支
git add .
git commit -m "Add my feature"             # 提交变更
git push -u origin my-feature              # 推送到 fork 并设置上游跟踪
gh pr create --title "Add my feature" --body "Description"   # 发起 PR
```

`gh repo fork --clone` 复制仓库到你的账号并克隆到本地，`origin` 指向你的 fork；`git checkout -b` 创建独立特性分支避免污染主分支；`git push -u` 推送并建立上游跟踪；`gh pr create` 打开 PR，网页会自动对比分支并生成标题与描述。

`gh repo fork --clone` copies the repository into your account and clones it locally, with `origin` pointing to your fork; `git checkout -b` creates an isolated feature branch to avoid polluting the main branch; `git push -u` pushes and sets up upstream tracking; `gh pr create` opens the PR, and the web UI auto-compares branches to draft a title and description.

**示例**：典型的 fork → PR 全流程，并跟踪上游更新、把他人 PR 拉到本地评审：

```bash
git remote add upstream https://github.com/octocat/Spoon-Knife.git   # 添加上游远程
git fetch upstream
git rebase upstream/main                                             # 在最新上游上变基
git push origin my-feature --force-with-lease                        # 安全地更新 fork 分支
gh pr checkout 123                                                   # 拉取某个 PR 到本地评审
```

**Example**: the full fork → PR flow, tracking upstream updates and pulling someone else's PR for local review:

```bash
git remote add upstream https://github.com/octocat/Spoon-Knife.git   # add upstream remote
git fetch upstream
git rebase upstream/main                                             # rebase onto latest upstream
git push origin my-feature --force-with-lease                        # safely update the fork branch
gh pr checkout 123                                                   # check out a PR for local review
```

⚠️ 注意事项：不要在已公开推送的 PR 分支上使用 `git push --force`——它会静默覆盖远程历史、可能摧毁协作者的提交；安全替代是 `--force-with-lease`，它在远程被他人改动时会拒绝覆盖。让每个 PR 聚焦单一目标并附清晰的描述与测试说明，可显著提高合并概率；合并方式（merge commit / squash / rebase）由维护者决定，提交前先 `git rebase upstream/main` 保持分支整洁。

⚠️ Caution: do not run `git push --force` on a PR branch whose history is already public — it silently overwrites remote history and can destroy collaborators' commits; the safe alternative is `--force-with-lease`, which refuses to overwrite if the remote changed. Keep each PR focused on a single goal with a clear description and test notes to greatly improve merge chances; the merge strategy (merge commit / squash / rebase) is the maintainer's call, so rebase onto `upstream/main` before submitting to keep the branch clean.

> 出处 Contributing to a Project：https://git-scm.com/book/en/v2/GitHub-Contributing-to-a-Project

## Maintaining a Project

维护者的日常工作包括：评审与合并 PR、管理议题（Issue）、用标签（Label）分类、用里程碑（Milestone）规划发布。GitHub 把 PR 的每个提交和每条评论都变成可引用的对象，维护者可以在网页或本地直接操作。关键是建立清晰的贡献指南与响应流程。

A maintainer's daily work includes reviewing and merging PRs, managing Issues, classifying them with Labels, and planning releases with Milestones. GitHub turns every commit and comment in a PR into an addressable object, and maintainers can act on the web or locally. The key is establishing clear contribution guidelines and a responsive workflow.

```bash
git fetch origin pull/123/head                      # 拉取 PR 的头部引用到本地
git checkout FETCH_HEAD                             # 检出该 PR 的提交
git merge --no-ff FETCH_HEAD -m "Merge PR #123"     # 创建合并提交

gh pr review 123 --approve                          # 表达通过
gh pr merge 123 --squash --delete-branch            # 压缩合并并清理分支

gh issue create --title "Bug: ..." --body "..." --label bug   # 新建议题
gh issue list --state open --label bug              # 列出未关闭的相关议题
gh issue close 123                                  # 关闭议题

gh label create "high-priority" --color "d93f0b"    # 新建标签
gh api repos/{owner}/{repo}/milestones -f title=v1.0 -f due_on=2025-06-01T00:00:00Z   # 创建里程碑
```

`git fetch origin pull/123/head` 直接拉取 `refs/pull/123/head`，无需先 clone 贡献者的 fork；`git merge --no-ff` 创建合并提交，把 PR 作为独立单元记录在历史中；`gh pr review --approve` 表达通过，`gh pr merge --squash --delete-branch` 压缩合并并删除分支；`gh issue` 系列管理议题，`gh label create` 新建标签，`gh api` 直接调用 REST 端点创建里程碑。

`git fetch origin pull/123/head` pulls the `refs/pull/123/head` ref directly without first cloning the contributor's fork; `git merge --no-ff` creates a merge commit that records the PR as a distinct unit; `gh pr review --approve` signals approval, and `gh pr merge --squash --delete-branch` squashes and deletes the branch; the `gh issue` family manages issues, `gh label create` adds a label, and `gh api` calls the REST endpoint to create a milestone.

**示例**：在合并前把 PR 变基到最新主分支以保持线性历史：

```bash
git fetch origin
git checkout feature-branch
git rebase main                                    # 在最新主分支上变基
git push origin feature-branch --force-with-lease  # 安全地更新 PR 分支
gh pr merge 123 --merge                            # 用 merge commit 方式合并
```

**Example**: rebase the PR onto the latest main branch before merging to keep a linear history:

```bash
git fetch origin
git checkout feature-branch
git rebase main                                    # rebase onto latest main
git push origin feature-branch --force-with-lease  # safely update the PR branch
gh pr merge 123 --merge                            # merge with a merge commit
```

⚠️ 注意事项：`git push --force` 或删除远程分支会不可逆地丢失他人引用的历史，删除分支前应确认 PR 已合并或已备份，安全替代是 `--force-with-lease` 或归档而非删除。合并策略应全仓库统一：`--merge` 保留完整历史，`--squash` 得到干净的线性历史但丢失单个提交粒度，`--rebase` 保留提交但需改写已推送历史。建议在仓库中放置 `CONTRIBUTING.md` 与 PR/Issue 模板，减少来回沟通成本。

⚠️ Caution: `git push --force` or deleting a remote branch irreversibly loses history that others may reference; confirm the PR is merged or backed up before deleting, and prefer `--force-with-lease` or archiving over deletion. Keep the merge strategy consistent across the repository: `--merge` preserves full history, `--squash` yields clean linear history but loses per-commit granularity, and `--rebase` keeps commits but rewrites pushed history. Add a `CONTRIBUTING.md` and PR/Issue templates to reduce back-and-forth.

> 出处 Maintaining a Project：https://git-scm.com/book/en/v2/GitHub-Maintaining-a-Project

## Managing an organization

组织（Organization）是共享仓库、成员和权限的容器，适合团队与企业。组织内可以创建团队（Team），把成员分组并授予对特定仓库的读（read）、写（write）或管理（admin）权限。通过组织可以集中管理账单、SSO、审计日志与权限策略，而不必逐个仓库配置。

An Organization is a container for shared repositories, members, and permissions, suited to teams and enterprises. Within an organization you can create Teams, group members, and grant read, write, or admin access to specific repositories. Through the organization you can centrally manage billing, SSO, audit logs, and permission policies instead of configuring each repository individually.

```bash
gh org list                                        # 列出你所属的组织
gh api orgs/{org}/members --jq '.[].login'         # 查看组织成员（jq 精简输出）

gh repo create {org}/my-repo --public --clone      # 在组织命名空间下新建仓库
gh api orgs/{org}/teams -f name=backend -f privacy=closed          # 创建团队
gh api -X PUT orgs/{org}/teams/backend/repos/{org}/my-repo -f permission=push   # 授予写权限
```

`gh org list` 列出组织；`gh api orgs/{org}/members --jq` 拉取成员并用 jq 只取登录名；`gh repo create {org}/my-repo` 直接在组织下建库；创建团队用 `gh api ... -f name=... -f privacy=...`；授权用 `-X PUT` 配合 `-f permission=push`（`pull` 为读、`push` 为写、`admin` 为管理）。

`gh org list` lists your organizations; `gh api orgs/{org}/members --jq` fetches members and extracts just the logins with jq; `gh repo create {org}/my-repo` creates a repo directly under the organization; create a team with `gh api ... -f name=... -f privacy=...`; grant access with `-X PUT` plus `-f permission=push` (`pull` = read, `push` = write, `admin` = admin).

**示例**：批量查看某团队拥有的仓库及其权限：

```bash
gh api orgs/{org}/teams/backend/repos --jq '.[] | "\(.name) \(.permissions)"'
```

**Example**: list the repositories owned by a team along with their permissions:

```bash
gh api orgs/{org}/teams/backend/repos --jq '.[] | "\(.name) \(.permissions)"'
```

⚠️ 注意事项：`admin` 权限可以删除仓库、清空历史、修改保护规则，只应授予信任的维护者，安全替代是默认授予 `read`/`triage`、按需升级到 `push`。删除组织、团队或成员是不可逆的治理操作，操作前应导出审计日志并二次确认。组织内默认成员可见内部仓库，敏感仓库应设为 `internal` 或 `private` 并限制团队可见范围；变更权限前先运行只读的 `gh api ... --jq` 查询现状，避免误改。

⚠️ Caution: `admin` permission can delete repositories, purge history, and modify protection rules, so grant it only to trusted maintainers — the safe alternative is to grant `read`/`triage` by default and upgrade to `push` as needed. Deleting an organization, team, or member is an irreversible governance action; export the audit log and double-check before acting. By default members can see internal repositories, so set sensitive repos to `internal` or `private` and restrict team visibility; before changing permissions, run a read-only `gh api ... --jq` query of the current state to avoid mistakes.

> 出处 Managing an organization：https://git-scm.com/book/en/v2/GitHub-Managing-an-organization

## Scripting GitHub

GitHub 提供两类 API：REST（按资源分端点，适合简单脚本）和 GraphQL（单端点按需取数，适合聚合查询与减少往返）。所有脚本化操作都通过个人访问令牌（PAT）或 OAuth 认证。`gh api` 封装了 REST 调用的认证与分页，是脚本化最快的入口；`curl` 则适合没有 `gh` 的环境。

GitHub offers two APIs: REST (endpoints per resource, good for simple scripts) and GraphQL (a single endpoint that fetches exactly what you need, good for aggregate queries and fewer round trips). All scripted operations authenticate via a personal access token (PAT) or OAuth. `gh api` wraps authentication and pagination for REST calls and is the fastest scripting entry point; `curl` suits environments without `gh`.

```bash
gh api repos/{owner}/{repo}                        # 读取仓库信息（自动认证）
gh api --paginate "repos/{owner}/{repo}/issues?state=open&per_page=100" --jq '.[].number'   # 分页 + 字段过滤
gh api repos/{owner}/{repo}/issues -f title="New issue" -f body="Details"   # 创建议题

curl -H "Authorization: token $GH_TOKEN" https://api.github.com/repos/{owner}/{repo}   # curl 调用 REST

gh api graphql -f query='query { viewer { login } repository(owner:"octocat" name:"Spoon-Knife") { stargazerCount } }'   # GraphQL 聚合查询
```

`gh api` 自动读取 `GH_TOKEN` 或 `gh auth` 的令牌并处理 `Link` 分页；`--paginate` 遍历所有页，`--jq` 用 jq 表达式提取字段；`-f` 发送表单字段用于 REST 写操作，`gh api graphql` 发送 GraphQL 查询；`curl` 需手动携带 `Authorization: token` 头。

`gh api` automatically reads the token from `GH_TOKEN` or `gh auth` and handles `Link` pagination; `--paginate` walks all pages and `--jq` extracts fields with a jq expression; `-f` sends form fields for REST write operations and `gh api graphql` issues a GraphQL query; `curl` requires manually adding the `Authorization: token` header.

**示例**：列出某仓库所有未关闭 PR 的编号、标题与作者：

```bash
gh api --paginate "repos/{owner}/{repo}/pulls?state=open&per_page=100" \
  --jq '.[] | "\(.number)\t\(.title)\t\(.user.login)"'
```

**Example**: list the number, title, and author of every open PR in a repository:

```bash
gh api --paginate "repos/{owner}/{repo}/pulls?state=open&per_page=100" \
  --jq '.[] | "\(.number)\t\(.title)\t\(.user.login)"'
```

⚠️ 注意事项：令牌等同于密码，泄露后他人可冒充你操作（包括删除仓库），安全替代是使用细粒度 PAT（限定仓库与权限）、短期令牌、环境变量注入与 `gh auth` 管理，绝不硬编码。REST 单页上限通常是 100，未加 `--paginate` 会静默漏数据；GraphQL 用游标分页，需循环 `endCursor`。注意速率限制：REST 未认证 60 次/小时、认证 5000 次/小时，GraphQL 按点数计费，超限返回 403/429。测试写操作前先用只读端点或 `-X GET` 验证，避免脚本误创建或误删资源。

⚠️ Caution: a token is equivalent to a password — if leaked, others can act as you (including deleting repositories); the safe alternatives are fine-grained PATs (scoped to repos and permissions), short-lived tokens, environment-variable injection, and `gh auth` management — never hardcode. REST's per-page limit is usually 100, and without `--paginate` you silently lose data; GraphQL uses cursor pagination and requires looping over `endCursor`. Mind rate limits: REST allows 60 requests/hour unauthenticated and 5000/hour authenticated, GraphQL is metered in points, and exceeding them returns 403/429. Before testing write operations, verify with a read-only endpoint or `-X GET` to avoid accidentally creating or deleting resources.

> 出处 Scripting GitHub：https://git-scm.com/book/en/v2/GitHub-Scripting-GitHub
