# Customizing Git

本章讲如何按需定制 Git 的行为：通过三级配置项（system / global / local）设置用户信息、编辑器、别名、凭据等；用 `.gitattributes` 对特定文件指定差异比较、合并策略、关键字展开等；用 Git Hooks（客户端与服务器端）在关键时机自动执行脚本；最后以一个服务器端强制策略为例展示如何约束提交与提交说明格式。

This chapter covers how to customize Git's behavior to your needs: setting user info, editor, aliases, and credentials through the three config levels (system / global / local); using `.gitattributes` to specify diff, merge strategy, keyword expansion, and more per path; using Git Hooks (client- and server-side) to run scripts at key moments; and finally a server-side enforced-policy example showing how to constrain commits and commit-message format.

## Git Configuration

### 概念

Git 配置按优先级从低到高分为三级，冲突时低级别被高级别覆盖：

1. **system**（`--system`）：系统级，对所有用户生效。Linux/macOS 在 `/etc/gitconfig`，Windows 在 Git 安装目录（如 `C:\Program Files\Git\etc\gitconfig`）。
2. **global**（`--global`）：当前用户级，对改用户的所有仓库生效。位于 `~/.gitconfig` 或 `~/.config/git/config`。
3. **local**（`--local`，默认）：仓库级，只对当前仓库生效。位于仓库内 `.git/config`。

Git configuration has three levels, from lowest to highest precedence (higher overrides lower on conflict):

1. **system** (`--system`): machine-wide, applies to all users. On Linux/macOS it is `/etc/gitconfig`; on Windows it lives in the Git install directory (e.g. `C:\Program Files\Git\etc\gitconfig`).
2. **global** (`--global`): user-wide, applies to all of that user's repositories. Located at `~/.gitconfig` or `~/.config/git/config`.
3. **local** (`--local`, default): repository-specific, applies only to the current repo. Located in the repo's `.git/config`.

### 命令

```bash
# 查看所有生效的配置及其来源
git config --list --show-origin

# 读取单个配置项
git config user.name

# 写入（不指定级别时默认 --local）
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "code --wait"   # 设置提交说明编辑器
git config --global core.excludesfile ~/.gitignore_global   # 全局忽略文件
git config --global color.ui auto              # 开启彩色输出
git config --global help.autocorrect 30        # 打错命令时 3 秒后自动纠正

# 别名（alias.*）
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.st status
git config --global alias.last 'log -1 HEAD'   # 带参数的别名用单引号包裹
git config --global alias.unstage 'reset HEAD --'
git config --global alias.visual '!gitk'       # 以 ! 开头调用外部命令
```

每步说明：

- `--list --show-origin` 能看到每项配置来自哪个文件，便于排查冲突。
- 用户信息 `user.name` / `user.email` 是提交必须的；`core.editor` 指定写提交说明的编辑器。
- `alias.*` 创建命令别名；值带空格时用引号包裹，`!` 前缀表示调用外部命令而非 Git 子命令。

Step-by-step:

- `--list --show-origin` shows which file each setting comes from, handy for debugging conflicts.
- `user.name` / `user.email` are required for commits; `core.editor` sets the editor for commit messages.
- `alias.*` creates command aliases; quote values containing spaces, and the `!` prefix runs an external command instead of a Git subcommand.

### 示例

给 `git checkout` 建别名，并全局设置提交者信息：

```bash
git config --global alias.co checkout
git config --global user.name "Alice"
git config --global user.email "alice@example.com"
git config --list --show-origin | grep user.name
# file:C:/Users/alice/.gitconfig   user.name=Alice
```

此后 `git co feature` 等同于 `git checkout feature`。

After this, `git co feature` is equivalent to `git checkout feature`.

### ⚠️ 注意事项

- ⚠️ 风险：`git config` 覆盖同键旧值而不提示；如需一次性写入多个键或覆盖前确认，先 `git config --list --show-origin` 看现有来源。
- 优先级记忆：`local > global > system`。排查“为什么配置没生效”时，先用 `--show-origin` 确认是否被更低级配置覆盖。
- `core.autocrlf`（Windows 换行符）建议慎改：`true` 检出转 CRLF、提交转 LF，`input` 只在提交转 LF。改错可能导致整批文件被标记为已修改。
- 别名命名若与现有命令冲突会覆盖该命令名，起名时避开常用命令。

- ⚠️ Risk: `git config` overwrites an existing key silently; if you need to check before overwriting, run `git config --list --show-origin` first to see the current source.
- Precedence to remember: `local > global > system`. When a setting "doesn't take effect," confirm with `--show-origin` whether a lower-level value is overriding it.
- `core.autocrlf` (Windows line endings) should be changed carefully: `true` converts to CRLF on checkout and LF on commit, `input` converts only on commit. Getting it wrong can mark whole files as modified.
- An alias whose name collides with an existing command overrides that command name — avoid common command names.

> 出处 Git Configuration：https://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration

## Git Attributes

### 概念

`.gitattributes` 是仓库根目录下的按路径配置文件，可为匹配的文件指定属性，例如：标记二进制文件、指定差异/合并策略、做关键字展开、导出规则等。规则优先级与 `.gitignore` 类似：后写的匹配规则覆盖先写的，每目录可有自己的 `.gitattributes`。

`.gitattributes` is a per-path config file at the repo root that assigns attributes to matching files — for example marking binary files, specifying diff/merge strategies, doing keyword expansion, and export rules. Precedence works like `.gitignore`: later matching rules override earlier ones, and each directory can have its own `.gitattributes`.

### 命令

```bash
# 标记为二进制，不做差异比较与换行转换
*.pbxproj binary

# 指定差异驱动（word diff / 语言高亮）
*.md diff
*.java diff=java

# 关键字展开（配合 ident 过滤器，在文件里写 $Id$）
*.txt ident

# 合并策略：此文件冲突时简单拼接 union
*.rb merge=union
database.xml merge=ours

# 导出规则（git archive 打包时）
.gitignore export-ignore
README.md export-subst
```

每步说明：

- `binary` 等价于 `-diff -merge -text`：不做 diff、不合并、不转换换行。
- `diff` 开启差异比较；`diff=java` 等内置/自定义驱动可改善语言识别与高亮。
- `ident` 配合文件内的 `$Id$` 占位符，在检出时替换为 blob 哈希。
- `merge=union` 让冲突时把两侧内容都保留；`merge=ours` 无条件采用本侧。
- `export-ignore` 打包时排除该文件；`export-subst` 允许 `$Format:` 占位符替换。

Step-by-step:

- `binary` is equivalent to `-diff -merge -text`: no diff, no merge, no line-ending conversion.
- `diff` enables diffing; `diff=java` and other built-in/custom drivers improve language detection and highlighting.
- `ident` plus an `$Id$` placeholder in the file expands to the blob hash on checkout.
- `merge=union` keeps both sides' content on conflict; `merge=ours` always takes our side.
- `export-ignore` excludes the file from `git archive`; `export-subst` allows `$Format:` placeholder substitution.

### 示例

让 Markdown 文件在 `git diff` 时按单词而非整行比较（显示更直观）：

```bash
echo '*.md diff' >> .gitattributes
git add .gitattributes
git commit -m "enable diff for markdown"
git diff   # 修改处会高亮到具体单词
```

也可以用 `git config --global diff.markdown.textconv ...` 自定义外部转换器，但简单场景直接 `*.md diff` 即可。

You could also define a custom converter with `git config --global diff.markdown.textconv ...`, but for simple cases `*.md diff` is enough.

### ⚠️ 注意事项

- ⚠️ 风险：`merge=ours` 会**静默丢弃**其他分支对同一文件的修改，仅在确知应保留本侧时使用；否则冲突更安全。
- `binary` 会让该文件在 `git show`/`git diff` 里不再显示可读内容，只报“Binary files differ”。
- `.gitattributes` 的改动需要提交后才能被其他协作者使用；它只影响之后的操作，不会回溯改写历史文件。
- 关键字展开（`ident`/`export-subst`）会改变检出内容，可能干扰 diff 与校验和，官方建议谨慎使用。

- ⚠️ Risk: `merge=ours` **silently discards** other branches' changes to the file; use it only when you are certain our side should win — otherwise leaving the conflict is safer.
- `binary` makes the file show as "Binary files differ" instead of readable content in `git show`/`git diff`.
- `.gitattributes` changes must be committed before collaborators benefit; it only affects future operations and does not rewrite history.
- Keyword expansion (`ident`/`export-subst`) alters checked-out content and can interfere with diff and checksums — the book advises caution.

> 出处 Git Attributes：https://git-scm.com/book/en/v2/Customizing-Git-Git-Attributes

## Git Hooks

### 概念

Hooks 是存放在 `.git/hooks/` 目录下的可执行脚本，Git 在特定时机自动运行它们。钩子分为两类：

- **客户端钩子（client-side）**：在提交/合并/推送等本地操作前后触发，用于格式检查、测试、自动生成提交说明等。注意：客户端钩子**不会随克隆复制**到别人机器，不能作为强制约束。
- **服务器端钩子（server-side）**：在服务器接收推送时触发（`pre-receive` / `update` / `post-receive`），能真正强制执行团队策略。

Hooks are executable scripts in `.git/hooks/` that Git runs automatically at specific moments. There are two kinds:

- **Client-side hooks**: fired around local operations like commit/merge/push, used for style checks, tests, or auto-generating commit messages. Note that client-side hooks are **not copied by clone**, so they cannot enforce rules on others.
- **Server-side hooks**: fired when the server receives a push (`pre-receive` / `update` / `post-receive`), and they can truly enforce team policy.

### 命令

```bash
# 查看示例钩子
ls .git/hooks/
# applypatch-msg.sample  pre-commit.sample  pre-rebase.sample  update.sample ...

# 启用一个钩子：去掉 .sample 后缀并赋予可执行权限
cp .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit   # Windows 上通常无需 chmod，改名即生效
```

钩子通过**退出码**控制是否中止：返回非零（`exit 1`）即中止本次操作，返回 0 放行。

各钩子时机（客户端常用）：

| 钩子 | 触发时机 | 能否中止 |
|---|---|---|
| `pre-commit` | 提交前（编辑器打开前） | 是，非零即中止提交 |
| `prepare-commit-msg` | 默认提交说明生成后、编辑器打开前 | 是 |
| `commit-msg` | 用户保存提交说明后、真正提交前 | 是 |
| `post-commit` | 提交完成后 | 否（仅通知） |
| `post-checkout` | `git checkout` 完成后 | 否 |
| `post-merge` | 合并成功后 | 否 |
| `pre-rebase` | 变基开始前 | 是 |
| `pre-push` | 推送前 | 是 |

服务器端钩子：

| 钩子 | 触发时机 | 能否拒绝 |
|---|---|---|
| `pre-receive` | 服务器收到推送、更新引用前 | 是，非零拒绝整批推送 |
| `update` | 对每个被更新的分支逐个触发 | 是，非零拒绝该分支 |
| `post-receive` | 全部引用更新完成后 | 否（仅通知/部署） |

Hooks control whether to abort via **exit code**: returning non-zero (`exit 1`) aborts the operation; returning 0 allows it to continue.

Client-side hook timing:

| Hook | When it runs | Can abort? |
|---|---|---|
| `pre-commit` | Before the commit (before the editor opens) | Yes — non-zero aborts the commit |
| `prepare-commit-msg` | After the default message is generated, before the editor opens | Yes |
| `commit-msg` | After you save the message, before the actual commit | Yes |
| `post-commit` | After the commit completes | No (notification only) |
| `post-checkout` | After `git checkout` completes | No |
| `post-merge` | After a successful merge | No |
| `pre-rebase` | Before a rebase starts | Yes |
| `pre-push` | Before a push | Yes |

Server-side hooks:

| Hook | When it runs | Can reject? |
|---|---|---|
| `pre-receive` | Server receives the push, before updating refs | Yes — non-zero rejects the whole push |
| `update` | Runs once per branch being updated | Yes — non-zero rejects that branch |
| `post-receive` | After all refs are updated | No (notification/deploy only) |

### 示例

一个检查尾随空白的 `pre-commit` 钩子（写成 `.git/hooks/pre-commit`）：

```bash
#!/bin/sh
if git diff --cached --check; then
  exit 0
else
  echo "存在尾随空白或冲突标记，提交被中止。" >&2
  exit 1
fi
```

A `pre-commit` hook that checks for trailing whitespace (written to `.git/hooks/pre-commit`):

```bash
#!/bin/sh
if git diff --cached --check; then
  exit 0
else
  echo "Trailing whitespace or conflict markers found; commit aborted." >&2
  exit 1
fi
```

保存为可执行后，再 `git commit` 时若暂存区有尾随空白，提交会被拒绝。

Once saved and made executable, any commit whose staged content has trailing whitespace will be rejected.

### ⚠️ 注意事项

- ⚠️ 风险：客户端钩子不随 `git clone` 分发，别人可以绕过；**不要**把客户端钩子当作团队硬约束，真正的规则应放在服务器端钩子。
- 钩子脚本可能被跳过：`git commit --no-verify`（或 `-n`）会跳过 `pre-commit`/`commit-msg`，`git push` 也可绕过部分钩子。
- 钩子脚本出错或返回非零会**中止**对应操作；写钩子时要保证脚本自身稳定，避免误伤正常提交。
- 安全：钩子脚本会以你的权限执行，切勿克隆陌生仓库后直接信任其中的示例脚本；钩子本身不随克隆复制，但仓库内容仍应审慎。

- ⚠️ Risk: client-side hooks are not distributed by `git clone` and can be bypassed; do **not** treat them as a team-wide hard constraint — put real rules in server-side hooks.
- Hooks can be skipped: `git commit --no-verify` (or `-n`) skips `pre-commit`/`commit-msg`, and some hooks can also be bypassed on push.
- A failing or non-zero hook **aborts** the operation; keep hook scripts robust so they don't block legitimate commits.
- Security: hooks run with your privileges; don't blindly trust scripts from an untrusted clone — though hooks themselves aren't copied by clone, still review repository content carefully.

> 出处 Git Hooks：https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks

## An Example Git-Enforced Policy

### 概念

若要在服务器上强制执行团队规范（如只允许在指定分支提交、强制提交说明格式），可以在裸仓库的 `update` / `commit-msg` 钩子里实现。本示例演示三层策略：

1. 只有 `refs/heads/master` 和 `refs/heads/develop` 两个分支允许被直接推送（可扩展为按用户 ACL 授权）。
2. 提交说明必须匹配指定格式（如“问题编号 + 描述”），由 `commit-msg` 钩子校验。
3. 只允许快进（fast-forward）合并，禁止在服务器端改写已存在的提交。

To enforce team rules on the server (e.g. only allowing commits to certain branches, or enforcing commit-message format), implement them in the bare repo's `update` / `commit-msg` hooks. This example demonstrates a three-layer policy:

1. Only `refs/heads/master` and `refs/heads/develop` may be pushed directly (extensible to per-user ACLs).
2. Commit messages must match a required format (e.g. "issue number + description"), checked by a `commit-msg` hook.
3. Only fast-forward merges are allowed; rewriting commits already on the server is forbidden.

### 命令

服务器端 `update` 钩子的核心逻辑（`.git/hooks/update`）：

```bash
#!/usr/bin/env ruby
$refname = ARGV[0]
$oldrev  = ARGV[1]
$newrev  = ARGV[2]
$user    = ENV['USER']

# 1) 只允许推送 master / develop
unless $refname =~ %r{^refs/heads/(master|develop)$}
  puts "只能推送到 master 或 develop 分支。"
  exit 1
end

# 2) 禁止删除分支（newrev 为全零）
if $newrev == "0000000000000000000000000000000000000000"
  puts "禁止删除分支。"
  exit 1
end

# 3) 只允许快进：new 的祖先必须是 old
commits = `git rev-list #{$oldrev}..#{$newrev}`.split("\n")
commits.each do |c|
  msg = `git cat-file commit #{c} | sed '1,/^$/d'`
  unless msg =~ /\[ISSUE-\d+\]/
    puts "提交 #{c[0,7]} 缺少 [ISSUE-xxx] 标记。"
    exit 1
  end
end

# 快进检查
if `git merge-base #{$oldrev} #{$newrev}`.strip != $oldrev
  puts "只允许快进推送。"
  exit 1
end
exit 0
```

每步说明：

- `$refname` / `$oldrev` / `$newrev` 是 Git 传给 `update` 的三个参数：引用名、旧提交、新提交。
- 分支白名单：只匹配 `master`/`develop`，其余分支直接 `exit 1` 拒绝。
- `newrev` 为全零表示删除分支，可据此禁止删除。
- 遍历新提交，用 `git cat-file commit` 取出提交说明，检查是否含 `[ISSUE-xxx]` 格式。
- `git merge-base` 判断 `oldrev` 是否为新提交的祖先，若否则拒绝非快进推送。

Step-by-step:

- `$refname` / `$oldrev` / `$newrev` are the three arguments Git passes to `update`: the ref name, old commit, and new commit.
- The branch whitelist matches only `master`/`develop`; any other branch is rejected with `exit 1`.
- A `newrev` of all zeros means a branch deletion, which you can reject.
- Loop over new commits, extract the message with `git cat-file commit`, and check for a `[ISSUE-xxx]` marker.
- `git merge-base` verifies `oldrev` is an ancestor of the new commit; otherwise the non-fast-forward push is rejected.

### 示例

把上述脚本放入裸仓库 `repo.git/hooks/update` 并 `chmod +x` 后：

- 某人 `git push origin feature-x` → 被拒：“只能推送到 master 或 develop 分支。”
- 某人提交说明为 `fix bug`（无 `[ISSUE-]` 标记）→ 被拒。
- 某人 `git push --force` 覆盖已推送提交 → 被拒（非快进）。

Put the script into the bare repo's `repo.git/hooks/update` and `chmod +x` it. Then:

- Someone runs `git push origin feature-x` → rejected: "只能推送到 master 或 develop 分支。"
- Someone commits with `fix bug` (no `[ISSUE-]` marker) → rejected.
- Someone force-pushes over an already-pushed commit → rejected (not a fast-forward).

### ⚠️ 注意事项

- ⚠️ 风险：服务器端钩子直接拒绝推送，配置错误可能把团队**锁死**在仓库外；上线前先在测试裸仓库验证，并保留紧急通道。
- ⚠️ 风险：`update` 钩子逐分支执行，若策略需全局一致，`pre-receive`（整批一次拒绝）更合适；两者混用需理解其触发差异。
- 建议：客户端加 `pre-push`/`commit-msg` 钩子做“提前反馈”，但规则最终仍以服务器端钩子为准，客户端钩子可被绕过。
- 保持钩子脚本幂等、无副作用，并写日志便于排查被拒原因；脚本用 Git 自带命令（`rev-list`、`cat-file`、`merge-base`）而非手动解析，更可靠。

- ⚠️ Risk: server-side hooks reject pushes outright; a misconfiguration can **lock the team out** of the repo. Test on a staging bare repo first and keep an emergency path.
- ⚠️ Risk: `update` runs per-branch; if a policy must be global, `pre-receive` (which rejects the whole push at once) is more appropriate. Understand the trigger difference if you combine them.
- Suggestion: add client-side `pre-push`/`commit-msg` hooks for early feedback, but the server-side hook remains authoritative — client hooks can be bypassed.
- Keep hook scripts idempotent and side-effect-free, and log rejections for diagnosis; prefer Git's own commands (`rev-list`, `cat-file`, `merge-base`) over manual parsing for reliability.

> 出处 An Example Git-Enforced Policy：https://git-scm.com/book/en/v2/Customizing-Git-An-Example-Git-Enforced-Policy
