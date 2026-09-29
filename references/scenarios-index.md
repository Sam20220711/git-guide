# 场景速查索引 / Scenario Quick Index

把用户描述的「我想做 X」或「出问题的现状」映射到对应参考文件与小节。回答时先用本表定位，再读目标文件。
Map the user's "I want to X" (or a problem state) to the right reference file and section. Locate with this table first, then read the target file.

## 入门与配置 / Setup & config

| 场景 | 位置 | 出处 |
|---|---|---|
| 安装 Git、首次配置 user.name/user.email | `01-getting-started.md` | /Getting-Started-First-Time-Git-Setup |
| 查看命令帮助 | `01-getting-started.md` | /Getting-Started-Getting-Help |
| 三级配置（system/global/local）、常用 core.*/alias | `08-customizing.md` | /Customizing-Git-Git-Configuration |
| 配置行尾、diff 驱动、.gitattributes | `08-customizing.md` | /Customizing-Git-Git-Attributes |

## 日常提交 / Daily commit

| 场景 | 位置 | 出处 |
|---|---|---|
| 新建仓库 git init / 克隆 git clone | `02-git-basics.md` | /Git-Basics-Getting-a-Git-Repository |
| 查看状态 / 差异 git status、git diff | `02-git-basics.md` | /Git-Basics-Recording-Changes-to-the-Repository |
| 暂存与提交 git add、git commit | `02-git-basics.md` | /Git-Basics-Recording-Changes-to-the-Repository |
| 忽略文件 .gitignore | `02-git-basics.md` | /Git-Basics-Recording-Changes-to-the-Repository |
| 查看提交历史 git log | `02-git-basics.md` | /Git-Basics-Viewing-the-Commit-History |
| 只暂存部分改动 git add -p | `07-git-tools-a.md` | /Git-Tools-Interactive-Staging |

## 撤销与回退 / Undo & rollback

| 场景 | 位置 | 出处 |
|---|---|---|
| 修改最近一次提交 git commit --amend | `02-git-basics.md` | /Git-Basics-Undoing-Things |
| 撤销暂存 git restore --staged / reset | `02-git-basics.md` | /Git-Basics-Undoing-Things |
| 撤销工作区改动 | `02-git-basics.md` | /Git-Basics-Undoing-Things |
| reset 三种模式（soft/mixed/hard）详解 | `07-git-tools-b.md` | /Git-Tools-Reset-Demystified |
| 找回丢失的提交（reflog + fsck） | `10-internals.md` | /Git-Internals-Maintenance-and-Data-Recovery |
| 回退远程已推送的提交 | `07-git-tools-b.md`（revert/reset）+ `03-branching.md` | /Git-Tools-Reset-Demystified |

## 分支与合并 / Branch & merge

| 场景 | 位置 | 出处 |
|---|---|---|
| 创建/切换/删除分支 git branch、checkout、switch | `03-branching.md` | /Git-Branching-Basic-Branching-and-Merging |
| 合并分支 git merge | `03-branching.md` | /Git-Branching-Basic-Branching-and-Merging |
| 解决合并冲突 | `03-branching.md` + `07-git-tools-b.md` | /Git-Branching-Basic-Branching-and-Merging |
| 变基 git rebase、rebase 与 merge 的区别 | `03-branching.md` | /Git-Branching-Rebasing |
| 交互式变基 rebase -i（合并/改写提交） | `07-git-tools-b.md` | /Git-Tools-Rewriting-History |
| 远程分支跟踪、fetch/pull/push | `03-branching.md` | /Git-Branching-Remote-Branches |
| 分支管理、查看已合并分支 | `03-branching.md` | /Git-Branching-Branch-Management |
| 分支工作流（长期/特性分支） | `03-branching.md` | /Git-Branching-Branching-Workflows |

## 贮藏与清理 / Stash & clean

| 场景 | 位置 | 出处 |
|---|---|---|
| 临时保存未提交改动 git stash | `07-git-tools-a.md` | /Git-Tools-Stashing-and-Cleaning |
| 清理未跟踪文件 git clean | `07-git-tools-a.md` | /Git-Tools-Stashing-and-Cleaning |

## 远程与协作 / Remote & collaboration

| 场景 | 位置 | 出处 |
|---|---|---|
| 管理远程仓库 git remote | `02-git-basics.md` | /Git-Basics-Working-with-Remotes |
| 打标签 git tag（轻量/附注） | `02-git-basics.md` | /Git-Basics-Tagging |
| 集中式/集成管理员/司令官与副官工作流 | `05-distributed.md` | /Distributed-Git-Distributed-Workflows |
| 生成补丁 format-patch、apply、am | `05-distributed.md` | /Distributed-Git-Contributing-to-a-Project |
| 维护者挑选提交 cherry-pick、维护分支 | `05-distributed.md` | /Distributed-Git-Maintaining-a-Project |
| fork + Pull Request | `06-github.md` | /GitHub-Contributing-to-a-Project |
| gh CLI 常用命令 | `06-github.md` | /GitHub-Scripting-GitHub |
| 协议区别、SSH 密钥、搭服务器、裸仓库 | `04-server.md` | /Git-on-the-Server-* |

## 查找与调试 / Search & debug

| 场景 | 位置 | 出处 |
|---|---|---|
| 选择特定提交/记法（^ ~ 范围 …） | `07-git-tools-a.md` | /Git-Tools-Revision-Selection |
| 在历史中搜索代码变化 git log -S/-G | `07-git-tools-a.md` | /Git-Tools-Searching |
| 二分定位引入 bug 的提交 git bisect | `07-git-tools-b.md` | /Git-Tools-Debugging-with-Git |
| blame 逐行追溯 | `07-git-tools-a.md` | /Git-Tools-Searching |

## 高级/底层 / Advanced & internals

| 场景 | 位置 | 出处 |
|---|---|---|
| 子模块 git submodule 增删改 | `07-git-tools-b.md` | /Git-Tools-Submodules |
| 签署提交/标签 GPG | `07-git-tools-a.md` | /Git-Tools-Signing-Your-Work |
| 凭据存储 credential | `07-git-tools-b.md` | /Git-Tools-Credential-Storage |
| 离线打包 bundle | `07-git-tools-b.md` | /Git-Tools-Bundling |
| rerere 复用冲突解决 | `07-git-tools-b.md` | /Git-Tools-Rerere |
| 钩子 git hooks | `08-customizing.md` | /Customizing-Git-Git-Hooks |
| git svn / 迁移到 Git | `09-other-systems.md` | /Git-and-Other-Systems-* |
| 对象模型 blob/tree/commit/tag、packfile、refspec | `10-internals.md` | /Git-Internals-* |
| 数据恢复、gc 维护、环境变量 | `10-internals.md` | /Git-Internals-Maintenance-and-Data-Recovery |

## 使用说明 / How to use

1. 在左列找到最贴近用户场景的一行。
2. 打开「位置」列指向的参考文件，读取对应小节。
3. 按 SKILL.md 的「回答模板」组织最终回答，并附「出处」列的链接。
