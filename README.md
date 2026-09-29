# git-guide

基于 Pro Git 官方文档的 DeepSeek Harness 技能：你描述一个 Git 使用场景，它给出具体命令、分步说明、示例与风险提示（中英对照）。
A DeepSeek Harness skill built on the official Pro Git book: describe a Git scenario and it returns concrete commands, step-by-step explanations, examples, and risk notes (bilingual Chinese/English).

## 安装 / Install

把本目录放到你的技能目录，例如：
Copy this folder into your skills directory, for example:

- 项目级 Project-level：`<project>/.dsh/skills/git-guide/`
- 用户级 User-level：`~/.dsh/skills/git-guide/`

## 使用 / Usage

直接描述你的 Git 场景即可，例如：
Just describe your Git scenario, for example:

- “怎么撤销最近一次提交但保留改动？”
- “我合并冲突了，怎么解决？”
- “误删了一个分支，能找回吗？”

技能会先澄清关键信息，再按「目标 → 前置检查 → 步骤 → 示例 → 注意事项 → 出处」作答。
The skill first clarifies key facts, then answers as Goal → Pre-flight checks → Steps → Example → Risks & notes → Source.

## 结构 / Structure

```
git-guide/
├── SKILL.md                   # 主技能：流程 + 回答模板 + 章节索引
└── references/                # Pro Git 各章知识（中英对照）
    ├── 01-getting-started.md … 10-internals.md
    ├── 11-appendix.md
    └── scenarios-index.md     # 场景速查表
```

## 来源 / Source

Pro Git（第二版）— https://git-scm.com/book/en/v2
