# Git on the Server

本章讲解如何把 Git 仓库放到服务器上供多人协作：比较四种传输协议、解释裸仓库、生成 SSH 密钥、搭建服务器，并介绍 Git 守护进程、Smart HTTP、GitWeb 与 GitLab。内容源自 Pro Git 官方文档第四章。

This chapter explains how to host Git repositories on a server: comparing the four transport protocols, understanding bare repositories, generating SSH keys, setting up a server, and introducing the Git daemon, Smart HTTP, GitWeb, and GitLab. All content is drawn from Chapter 4 of the official Pro Git book.

## The Protocols

Git 支持四种传输协议：本地（Local）、HTTP（分"哑 HTTP"与"智能 HTTP"）、SSH 与 Git 原生协议。协议决定是否需要认证、是否支持匿名只读、是否加密以及搭建难度：本地最简，HTTP 易穿透防火墙，SSH 最易安全搭建，Git 协议最快但最难配置。

Git supports four transport protocols: Local, HTTP (dumb or smart), SSH, and the native Git protocol. The protocol determines whether authentication, anonymous read access, and encryption are available, plus setup difficulty: Local is simplest, HTTP crosses firewalls easily, SSH is the easiest to secure, and the Git protocol is fastest but hardest to configure.

```bash
git clone /srv/git/project.git                          # Local：直接以路径作为仓库
git clone file:///srv/git/project.git                   # Local：file:// 显式本地协议
git clone https://example.com/project.git               # HTTP(S)：智能/哑 HTTP 传输
git clone ssh://git@example.com/srv/git/project.git     # SSH：加密并认证
git clone git@example.com:project.git                   # SSH：scp 风格简写
git clone git://example.com/project.git                 # Git：原生守护进程协议
```

前缀决定协议：裸路径与 `file://` 是本地协议，`https://` 走 HTTP(S)，`ssh://` 或 `user@host:path` 走 SSH，`git://` 走 Git 原生协议（默认端口 9418）。

The prefix selects the protocol: a bare path or `file://` is Local, `https://` is HTTP(S), `ssh://` or `user@host:path` is SSH, and `git://` is the native Git protocol (default port 9418).

**示例**：内网团队用共享目录 `/srv/git/project.git` 作中央仓库，成员执行 `git clone /srv/git/project.git`；开源项目对外用 `git://` 提供匿名只读，用 SSH 供维护者推送。

**Example**: an intranet team uses `/srv/git/project.git` as the central repository and clones with `git clone /srv/git/project.git`; an open-source project offers anonymous read via `git://` and uses SSH for maintainer pushes.

⚠️ 注意事项：本地协议无认证且共享目录易被他人误删；`file://` 比裸路径慢。`git://` 无认证无加密，绝不能用于需保护的仓库。SSH 不提供匿名只读，所有人访问都需要账号。

⚠️ Caution: Local has no authentication and the shared directory can be deleted by mistake; `file://` is slower than a bare path. `git://` has no auth or encryption and must never protect repositories. SSH offers no anonymous read — every user needs an account.

> 出处 The Protocols：https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols

## Getting Git on a Server

要把仓库放到服务器，需要创建"裸仓库"（bare repository）：它没有工作目录（working tree），只存放 `.git` 里的内容，专作多人推送/拉取的目标。普通仓库有检出的文件，无法安全地作为推送目标。

To host a repository on a server you create a bare repository: it has no working tree and stores only what normally lives inside `.git`, serving purely as a push/pull target. A normal repository has checked-out files and cannot safely accept pushes.

```bash
git init --bare project.git                    # 直接创建一个空的裸仓库
git clone --bare project project.git           # 把已有仓库克隆成裸仓库
scp -r project.git user@gitserver:/srv/git/    # 把裸仓库拷贝到服务器
```

`git init --bare` 新建不含工作目录的裸仓库；`git clone --bare` 把已有仓库转换为裸仓库；`scp -r` 把裸仓库目录复制到服务器，他人即可克隆。

`git init --bare` creates a bare repository without a working tree; `git clone --bare` converts an existing repository into a bare one; `scp -r` copies the bare directory to the server, after which others can clone it.

**示例**：本地项目 `myproject` 执行 `git clone --bare myproject myproject.git`，再把 `myproject.git` 上传到服务器，团队即可用 `git clone git@server:/srv/git/myproject.git` 协作。

**Example**: run `git clone --bare myproject myproject.git`, upload `myproject.git` to the server, and the team can collaborate with `git clone git@server:/srv/git/myproject.git`.

⚠️ 注意事项：不要向"非裸"且检出了分支的仓库直接推送，Git 会拒绝以免破坏工作目录。`scp` 若目标已有同名目录会直接覆盖，⚠️ 风险——覆盖不可逆；安全替代：先 `ls` 确认目标不存在，或改用 `rsync -av --dry-run` 预览后再执行。

⚠️ Caution: never push to a non-bare repository with a branch checked out — Git refuses to avoid corrupting the working tree. `scp` silently overwrites an existing directory of the same name — ⚠️ risk: irreversible. Safer alternative: confirm with `ls` that the target is absent, or preview with `rsync -av --dry-run` first.

> 出处 Getting Git on a Server：https://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server

## Generating Your SSH Public Key

大多数 Git 服务器用 SSH 公钥认证：本地生成密钥对，把公钥交给服务器，之后推送/拉取就不必每次输密码。生成前先检查是否已有密钥，避免重复。

Most Git servers authenticate with SSH public keys: generate a key pair locally, give the public key to the server, and afterwards push/pull without typing a password each time. Check for an existing key before generating.

```bash
ls -al ~/.ssh                              # 检查是否已有密钥（看 *.pub 文件）
ssh-keygen -t ed25519 -C "you@example.com" # 生成 Ed25519 密钥（现代推荐）
ssh-keygen -t rsa -b 4096 -C "you@example.com"  # 旧系统：RSA 4096 位
cat ~/.ssh/id_ed25519.pub                  # 打印公钥内容，用于粘贴到服务器
```

`ls -al ~/.ssh` 查看是否已存在 `id_ed25519.pub` 或 `id_rsa.pub`；`ssh-keygen -t ed25519` 生成 Ed25519 密钥（比 RSA 更短更快更安全），`-C` 附加注释；`cat` 打印公钥供复制。

`ls -al ~/.ssh` shows whether `id_ed25519.pub` or `id_rsa.pub` exists; `ssh-keygen -t ed25519` generates an Ed25519 key (shorter, faster, and more secure than RSA) with `-C` adding a comment; `cat` prints the public key for copying.

**示例**：运行 `ssh-keygen -t ed25519 -C "alice@example.com"` 一路回车接受默认路径 `~/.ssh/id_ed25519`，再 `cat ~/.ssh/id_ed25519.pub`，把 `ssh-ed25519 AAAA...` 开头的一行粘贴到 GitHub/GitLab 或服务器的 `authorized_keys`。

**Example**: run `ssh-keygen -t ed25519 -C "alice@example.com"`, press Enter to accept the default path `~/.ssh/id_ed25519`, then `cat ~/.ssh/id_ed25519.pub` and paste the line beginning `ssh-ed25519 AAAA...` into GitHub/GitLab or the server's `authorized_keys`.

⚠️ 注意事项：⚠️ 风险——`ssh-keygen` 覆盖已有密钥会让旧公钥全部失效，导致无法登录已配好的服务器。安全替代：先 `ls -al ~/.ssh` 确认无同名文件，或指定新文件名（如 `-f ~/.ssh/id_ed25519_work`）。私钥可设 passphrase 保护；只分享 `.pub` 公钥，切勿外泄私钥。

⚠️ Caution: ⚠️ risk — overwriting an existing key with `ssh-keygen` invalidates every server configured with the old public key. Safer alternative: confirm with `ls -al ~/.ssh` that no same-named file exists, or use a new filename (e.g. `-f ~/.ssh/id_ed25519_work`). Protect the private key with a passphrase; share only the `.pub` public key, never the private key.

> 出处 Generating Your SSH Public Key：https://git-scm.com/book/en/v2/Git-on-the-Server-Generating-Your-SSH-Public-Key

## Setting Up the Server

搭建最小可用的 SSH Git 服务器只需三步：创建专用 `git` 用户、把每个开发者的公钥追加到该用户的 `authorized_keys`、在共享目录创建裸仓库。此后开发者用 `git@server:` 访问。

A minimal SSH Git server needs three steps: create a dedicated `git` user, append each developer's public key to that user's `authorized_keys`, and create bare repositories in a shared directory. Developers then access it as `git@server:`.

```bash
sudo adduser git                              # 创建专用 git 用户
su git                                        # 切换到 git 用户（或 sudo -u git）
mkdir -p ~/.ssh && touch ~/.ssh/authorized_keys   # 准备公钥存放文件
cat /tmp/id_ed25519.pub >> ~/.ssh/authorized_keys # 追加开发者公钥（每个一行）
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys  # 收紧权限
mkdir /srv/git && cd /srv/git                 # 建立仓库根目录
git init --bare project.git                   # 创建裸仓库
```

`adduser git` 建立共用低权限账号；`cat ... >> authorized_keys` 逐行追加公钥；`chmod 700/600` 收紧权限（过松会让 SSH 拒绝使用该文件）；`git init --bare` 在 `/srv/git` 下创建可推送的裸仓库。

`adduser git` creates a shared low-privilege account; `cat ... >> authorized_keys` appends each public key line by line; `chmod 700/600` tightens permissions (loose permissions make SSH refuse the file); `git init --bare` creates a pushable bare repository under `/srv/git`.

**示例**：把 Alice 的公钥追加进 `git` 用户的 `authorized_keys` 后，她即可 `git clone git@gitserver:/srv/git/project.git` 并正常 `git push origin main`，全程无需密码。

**Example**: after appending Alice's public key to the `git` user's `authorized_keys`, she can `git clone git@gitserver:/srv/git/project.git` and `git push origin main` with no password prompt.

⚠️ 注意事项：多人共用一个 `git` 账号无法区分"谁推送了什么"；要审计可把该用户登录 shell 设为 `git-shell` 限制其只能执行 Git 命令，或改用 GitLab 等带账号管理的方案。`chmod` 权限必须正确，否则 SSH 会拒绝公钥认证。

⚠️ Caution: a shared `git` account cannot tell who pushed what; for auditing, set `git-shell` as that user's login shell to restrict them to Git commands, or adopt GitLab-style per-user accounts. `chmod` permissions must be correct, or SSH refuses public-key authentication.

> 出处 Setting Up the Server：https://git-scm.com/book/en/v2/Git-on-the-Server-Setting-Up-the-Server

## Git Daemon

Git 守护进程（`git daemon`）通过 Git 原生协议在默认端口 9418 提供"匿名只读"访问，常用于给开源项目提供公开克隆，无需账号。它不能用于需要认证或写入的场景。

The Git daemon (`git daemon`) serves anonymous, read-only access over the native Git protocol on default port 9418, commonly used for public clones of open-source projects without accounts. It cannot serve scenarios needing authentication or write access.

```bash
git daemon --reuseaddr --base-path=/srv/git/ /srv/git/ & # 后台启动守护进程
touch /srv/git/project.git/git-daemon-export-ok           # 允许该仓库被公开导出
```

`--reuseaddr` 让服务器重启后能快速重新绑定端口；`--base-path` 与末尾路径指定允许导出的仓库根目录；`touch git-daemon-export-ok` 在裸仓库里放一个空文件作为"允许匿名导出"的开关。

`--reuseaddr` lets the server rebind the port quickly after a restart; `--base-path` and the trailing path set the root of repositories allowed for export; `touch git-daemon-export-ok` drops an empty file in a bare repository as the "allow anonymous export" switch.

**示例**：在 `/srv/git` 下放置裸仓库，对公开的仓库 `touch git-daemon-export-ok`，启动 `git daemon --reuseaddr --base-path=/srv/git/ /srv/git/`，外部用户即可 `git clone git://example.com/project.git` 匿名克隆。

**Example**: place bare repositories under `/srv/git`, run `touch git-daemon-export-ok` in the public ones, start `git daemon --reuseaddr --base-path=/srv/git/ /srv/git/`, and outsiders can clone anonymously with `git clone git://example.com/project.git`.

⚠️ 注意事项：⚠️ 风险——默认只读，但加 `--enable=receive-pack` 会开放"无认证匿名推送"，任何能连上端口的人都能改写仓库，切勿对非公开环境使用。守护进程无加密无认证，只能对外公开只读内容；需认证或写入时改用 SSH 或 Smart HTTP。

⚠️ Caution: ⚠️ risk — read-only by default, but adding `--enable=receive-pack` opens unauthenticated anonymous push: anyone who can reach the port can rewrite the repository. Never use it outside a public read-only setting. The daemon has no encryption or auth, so it serves only public read-only content; use SSH or Smart HTTP when auth or writes are needed.

> 出处 Git Daemon：https://git-scm.com/book/en/v2/Git-on-the-Server-Git-Daemon

## Smart HTTP

智能 HTTP（Smart HTTP）通过现有 Web 服务器（如 Apache）运行 `git-http-backend`，以 HTTP(S) 暴露 Git 读写，并复用 Web 服务器的认证（Basic Auth 等）。它既支持匿名读，也支持认证后的写，是 GitLab 等平台 HTTP 端的机制。

Smart HTTP runs `git-http-backend` behind an existing web server (such as Apache), exposing Git read/write over HTTP(S) and reusing the web server's authentication (Basic Auth, etc.). It supports anonymous reads and authenticated writes, and is the mechanism platforms like GitLab use on the HTTP side.

```bash
sudo apt install apache2                        # 安装 Apache（Git 自带 git-http-backend）
sudo a2enmod cgi alias env                      # 启用 CGI、alias、env 模块
sudo systemctl restart apache2                  # 重启使配置生效
```

`apt install apache2` 准备 Web 服务器；`a2enmod cgi alias env` 开启 CGI 等必要模块；核心是在站点配置里用 `SetHandler git-http-backend` 把仓库路径请求交给 Git 的 CGI 后端；最后重启 Apache。

`apt install apache2` prepares the web server; `a2enmod cgi alias env` enables the CGI and related modules; the core is a site configuration using `SetHandler git-http-backend` to hand repository requests to Git's CGI backend; finally restart Apache.

```apache
# /etc/apache2/conf-available/git.conf（片段）
SetEnv GIT_PROJECT_ROOT /srv/git
SetEnv GIT_HTTP_EXPORT_ALL
ScriptAlias /git/ /usr/lib/git-core/git-http-backend/
<LocationMatch "^/git/.*/git-receive-pack$">
    AuthType Basic
    Require valid-user
</LocationMatch>
```

`GIT_PROJECT_ROOT` 指定仓库根目录；`GIT_HTTP_EXPORT_ALL` 允许匿名导出所有仓库；`ScriptAlias` 把 `/git/` 映射到 `git-http-backend`；`LocationMatch` 只对写操作 `git-receive-pack` 要求 Basic 认证。

`GIT_PROJECT_ROOT` sets the repository root; `GIT_HTTP_EXPORT_ALL` allows anonymous export of all repositories; `ScriptAlias` maps `/git/` to `git-http-backend`; `LocationMatch` requires Basic auth only for the write operation `git-receive-pack`.

**示例**：配置完成后，用户 `git clone https://example.com/git/project.git` 匿名克隆；`git push https://example.com/git/project.git` 时提示输入用户名密码，认证通过即可写入。

**Example**: once configured, users clone anonymously with `git clone https://example.com/git/project.git`; `git push https://example.com/git/project.git` prompts for credentials, and writes proceed after authentication.

⚠️ 注意事项：⚠️ 风险——用明文 HTTP 而非 HTTPS 会让账号密码和仓库内容明文传输、可被窃听。安全替代：全程使用 HTTPS（配置 TLS 证书），并让 `LocationMatch` 的认证覆盖写操作；切勿在生产环境用明文 HTTP 承载写权限。

⚠️ Caution: ⚠️ risk — plain HTTP rather than HTTPS sends credentials and repository contents in clear text that can be eavesdropped. Safer alternative: use HTTPS throughout (configure a TLS certificate) and keep the `LocationMatch` guard over writes; never carry write access over plain HTTP in production.

> 出处 Smart HTTP：https://git-scm.com/book/en/v2/Git-on-the-Server-Smart-HTTP

## GitWeb

GitWeb 是一个用 CGI 脚本生成的 Web 界面，用于在浏览器里浏览仓库的提交历史、文件与差异，但只是"只读查看器"，不提供托管或推送能力。快速体验可用 `git instaweb` 临时起一个 Web 服务器。

GitWeb is a CGI script that generates a web interface for browsing a repository's commit history, files, and diffs in a browser, but it is only a read-only viewer — no hosting or push capability. For a quick try, `git instaweb` starts a temporary web server.

```bash
git instaweb --httpd=apache2      # 用 Apache 临时启动 GitWeb（也可 lighttpd/webrick）
git instaweb --stop               # 停止 git instaweb 启动的服务器
```

`git instaweb --httpd=apache2` 在仓库里临时启动 Web 服务器并打开 GitWeb 界面；`git instaweb --stop` 停止它。生产环境需把 `gitweb.cgi` 接入 Apache/Nginx 并配置 `/etc/gitweb.conf`。

`git instaweb --httpd=apache2` temporarily starts a web server in the repository and opens the GitWeb interface; `git instaweb --stop` stops it. In production, wire `gitweb.cgi` into Apache/Nginx and configure `/etc/gitweb.conf`.

**示例**：进入任一仓库执行 `git instaweb --httpd=lighttpd`，浏览器自动打开 `http://127.0.0.1:1234/` 显示 `summary`、`shortlog` 与 `tree` 页面；看完后 `git instaweb --stop` 关闭。

**Example**: inside any repository, run `git instaweb --httpd=lighttpd`; the browser opens `http://127.0.0.1:1234/` showing the `summary`, `shortlog`, and `tree` pages; run `git instaweb --stop` when done.

⚠️ 注意事项：⚠️ 风险——`git instaweb` 监听本机回环地址时较安全，但若对外网开放且未加认证，等于把源码公开给任何能访问的人。安全替代：对外部署时套上 Web 服务器认证或 VPN，且只把它当只读浏览工具，不作权限管理。

⚠️ Caution: ⚠️ risk — `git instaweb` is relatively safe bound to loopback, but opened to the internet without authentication it publishes the source to anyone who can reach it. Safer alternative: put it behind web-server auth or a VPN when exposing externally, and treat it strictly as a read-only viewer, not access control.

> 出处 GitWeb：https://git-scm.com/book/en/v2/Git-on-the-Server-GitWeb

## GitLab

GitLab 是一个功能完整的自托管 Git 托管平台，集成仓库管理、用户/权限、Issue、合并请求与 CI/CD，相当于自建 GitHub。官方推荐 Omnibus 方式，一条命令即可装好并自动配置所需服务。

GitLab is a full-featured, self-hosted Git hosting platform integrating repository management, users/permissions, issues, merge requests, and CI/CD — essentially a self-hosted GitHub. The officially recommended Omnibus method installs and configures everything with a single command.

```bash
sudo apt-get install -y curl openssh-server ca-certificates postfix  # 安装依赖
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash  # 添加仓库
sudo EXTERNAL_URL="https://gitlab.example.com" apt-get install gitlab-ce  # 安装并设定访问域名
sudo gitlab-ctl reconfigure           # 应用配置，启动各组件
```

`apt-get install ... postfix` 安装依赖与邮件服务；`curl ... | sudo bash` 添加官方软件源；`EXTERNAL_URL=... apt-get install gitlab-ce` 安装社区版并指定外部地址；`gitlab-ctl reconfigure` 应用 `/etc/gitlab/gitlab.rb` 并启动组件。

`apt-get install ... postfix` installs dependencies and mail service; `curl ... | sudo bash` adds the official repository; `EXTERNAL_URL=... apt-get install gitlab-ce` installs the Community Edition and sets the external URL; `gitlab-ctl reconfigure` applies `/etc/gitlab/gitlab.rb` and starts components.

**示例**：装好后浏览器打开 `https://gitlab.example.com`，用 root 完成初始密码设置、创建用户与项目，团队成员即可 `git clone https://gitlab.example.com/team/project.git` 或 SSH 方式克隆推送。

**Example**: after installation, open `https://gitlab.example.com`, set the root password, create users and a project, and teammates can clone and push via `git clone https://gitlab.example.com/team/project.git` or SSH.

⚠️ 注意事项：GitLab 会捆绑并管理自己的 PostgreSQL、Redis、Nginx，⚠️ 风险——可能与机器上既有同名服务端口冲突，调整配置可能影响现有服务。安全替代：安装前先查端口占用（80/443、5432、6379），并优先用独立机器或 Docker 部署。Omnibus 版资源占用较高，小团队可考虑 Gitea/Gogs 等轻量替代。

⚠️ Caution: GitLab bundles and manages its own PostgreSQL, Redis, and Nginx — ⚠️ risk: they may conflict on ports with existing services, and later adjustments can disrupt them. Safer alternative: check port usage first (80/443, 5432, 6379) and prefer a dedicated machine or Docker. The Omnibus edition is resource-heavy; small teams may consider lighter options like Gitea or Gogs.

> 出处 GitLab：https://git-scm.com/book/en/v2/Git-on-the-Server-GitLab

## Third Party Hosted Options

不愿自维护服务器时，可直接用第三方托管平台：GitHub 是最流行的开源协作平台，GitLab.com 提供托管版 GitLab，Bitbucket 与 Atlassian 工具链集成良好，还有 Gitea、Gogs 等可自托管轻量方案。托管平台省去运维，还附带代码审查、Issue、CI。

If you prefer not to maintain your own server, use third-party hosting: GitHub is the most popular open-source collaboration platform, GitLab.com offers hosted GitLab, Bitbucket integrates with the Atlassian toolchain, and lighter self-hosted options like Gitea and Gogs exist. Hosted platforms remove ops burden and bundle code review, issues, and CI.

```bash
git remote add origin https://github.com/user/project.git   # 关联 GitHub 远端（HTTPS）
git remote add origin git@github.com:user/project.git       # 关联 GitHub 远端（SSH）
git push -u origin main                                      # 首次推送并设置上游
```

`git remote add origin` 把平台空仓库关联为本地 `origin` 远端（HTTPS 或 SSH 地址皆可）；`git push -u origin main` 首次推送并把 `main` 设为上游，之后 `git push` 即可。

`git remote add origin` links the platform's empty repository as the local `origin` remote (HTTPS or SSH address); `git push -u origin main` pushes the first time and sets `main` as upstream, after which plain `git push` suffices.

**示例**：在 GitHub 新建空仓库后，本地 `git remote add origin https://github.com/alice/myapp.git`，再 `git push -u origin main`，浏览器即可看到代码，团队成员随后 `git clone` 协作。

**Example**: after creating an empty repository on GitHub, run `git remote add origin https://github.com/alice/myapp.git`, then `git push -u origin main`; the code appears in the browser and teammates `git clone` to join.

⚠️ 注意事项：托管平台的免费/私有策略与配额会随时间变化，公开仓库对外可见，切勿提交密钥、凭证等敏感信息。选择平台时权衡：是否接受代码放在第三方、私有仓库费用、协作与 CI 功能是否满足，以及是否需要数据自主可控（后者回归自托管）。

⚠️ Caution: hosting platforms' free/private policies and quotas change over time, and public repositories are visible to the world — never commit secrets or credentials. When choosing, weigh whether you accept code living with a third party, private-repo costs, whether collaboration/CI features fit, and whether you need full data control (in which case return to self-hosting).

> 出处 Third Party Hosted Options：https://git-scm.com/book/en/v2/Git-on-the-Server-Third-Party-Hosted-Options
