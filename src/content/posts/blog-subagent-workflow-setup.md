---
title: 用 Subagent 构建博客发布流水线：归档、审查、发布一条龙
published: 2026-07-15
description: "为博客维护工作流创建 3 个 subagent（mdoc-archiver / doc-validator / blog-publisher）和 blog-pipeline skill，实现从代码改动到博客发布的完全自动化"
tags: [subagent, 工作流, 博客, 自动化, skill, Claude]
category: 工具搭建
draft: false
---

日常维护博客时，常常需要反复执行"代码改动 -> 归档到 /mdoc -> 写成博客 -> 发布"这一流程。手动操作不仅繁琐，而且容易遗漏步骤——有时候修完 bug 就直接切去干别的事了，改完的配置、排查的思路全忘了归档。更烦人的是，单个 agent 处理这么长的链条，上下文一长就容易出现幻觉。

为了解决这个问题，我基于 Claude Code 的 subagent 机制搭了一套三步流水线，把归档、审查、发布全自动化了。

## 两层架构：Skill 编排 + Subagent 执行

系统分为两层，各司其职：

| 层 | 组件 | 职责 | 存放位置 |
|---|------|------|---------|
| 编排层 | `blog-pipeline` skill | 检测修复场景、调度 subagent、展示用户提示 | `~/.claude/skills/blog-pipeline/SKILL.md` |
| 执行层 | `mdoc-archiver` subagent | 将代码改动归档到 /mdoc 文档 | `~/.claude/agents/mdoc-archiver.md` |
| 执行层 | `doc-validator` subagent | 审查文档准确性，防幻觉 | `~/.claude/agents/doc-validator.md` |
| 执行层 | `blog-publisher` subagent | 将 /mdoc 文档转为博客并发布 | `~/.claude/agents/blog-publisher.md` |

每个 subagent 都有独立的 context window，互不干扰。skill 层只负责编排调度，不参与具体的文档处理。

## blog-pipeline Skill：编排入口

blog-pipeline skill 是整个工作流的调度大脑，承担四项职责：

1. **自动检测修复场景** — 判断当前对话是否包含代码改动、问题排查、修复完成等信号
2. **轻量用户提示** — 检测到修复场景时在回复末尾显示三段式提示
3. **三步流水线执行** — 按顺序调用三个 subagent
4. **安全规则** — push 前必须展示 git diff 并等待用户确认

当工作流检测到修复场景时，会在回复末尾显示这样的提示：

```
[r] 一条龙（归档→审查→博客→push）
[a] 仅归档到 /mdoc
[s] 跳过
```

无侵入设计：未检测到修复场景时完全不打扰用户。

## 三步流水线

完整的流程如下图：

```
代码改动/修复完成
      │
      ▼
[blog-pipeline skill 检测到修复场景]
      │
      ├── 用户选 [r] ──→ Step 1: mdoc-archiver 归档到 /mdoc
      │                         │
      │                         ▼
      │                   Step 2: doc-validator 审查文档
      │                         │
      │                    ┌─────┴─────┐
      │                    ▼           ▼
      │                  通过         不通过 → 用户决定修正/跳过/取消
      │                    │
      │                    ▼
      │                   Step 3: blog-publisher 写博客
      │                         │
      │                         ▼
      │                   git diff 展示给用户确认
      │                         │
      │                    ┌─────┴─────┐
      │                    ▼           ▼
      │                  确认         拒绝 → 不 push，保留本地
      │                    │
      │                    ▼
      │                   git commit + git push → GitHub Actions 自动部署
      │
      ├── 用户选 [a] ──→ 仅执行 Step 1（归档）
      │
      └── [s] 跳过 ──→ 什么也不做
```

### 关键设计原则

- **每步独立 context window** — 每个 subagent 有独立的上下文，减少长上下文带来的幻觉。归档时不用关心博客格式，写博客时不用关心代码细节
- **doc-validator 只读** — 审查者与写文档者分离，交叉验证控制幻觉。validator 只有 Read/Grep/Glob 三个工具，没有写入权限
- **push 前确认** — git diff 必须展示给用户，防止误推送。这是最重要的安全防线
- **无侵入提示** — 未检测到修复场景时完全不打扰用户，避免无效干扰

## 三个 Subagent 的职责分工

| Subagent | 职责 | 工具 | 模型 |
|----------|------|------|------|
| mdoc-archiver | 分析代码改动/修复方案，提取结构化信息（问题描述、根因、解决方案、涉及文件、命令），按 /mdoc 格式写入 memory 目录。会先 Glob 检查是否已有同名文档，决定新建还是补充更新 | Read, Grep, Glob, Bash, Write | sonnet |
| doc-validator | 纯只读审查：文件路径真实存在、命令合理、代码引用与实际一致、跨文档链接有效。输出审查结论：通过 / 有警告 / 不通过 | Read, Grep, Glob | sonnet |
| blog-publisher | 将 /mdoc 文档转为 Astro 博客文章，写入 `src/content/posts/`。确保 frontmatter 完整，适配博客风格（概述、小标题、代码块标注语言），删除内部文档专用字段。创建后显示 git diff 待用户确认 | Read, Grep, Glob, Bash, Write | sonnet |

### mdoc-archiver 设计要点

从代码改动中提取结构化信息，包括问题描述、根因分析、解决方案、涉及的文件和命令。写入前会用 Glob 搜索是否已有相关文档，如果有则询问覆盖还是追加。

### doc-validator 设计要点

纯只读，只有三个工具（Read, Grep, Glob），没有 Write 和 Bash 权限。验证内容：

- 文件中引用的路径在文件系统中真实存在
- 文档中的命令语法正确、参数合理
- 代码引用与实际代码一致
- 跨文档链接有效

### blog-publisher 设计要点

将 /mdoc 文档转化为博客文章时做三件事：

1. 确保 frontmatter 完整（title、published、tags、category、draft）
2. 适配博客风格：开头有概述段落，分节用小标题，代码块标注语言
3. 删除内部文档专用的 frontmatter 字段（name、description、metadata）

## 特殊情况处理

| 情况 | 处理方式 |
|------|---------|
| 无可归档内容 | 提示用户，问是否手动提供 |
| 文档已存在 | 问覆盖还是追加，再执行 |
| 审查不通过 | 展示问题，让用户决定修正/跳过/取消 |
| push 前发现不对 | 不确认即可，保留本地不推送 |

## 使用方式

### 自动流程（推荐）

日常修复完代码后，blog-pipeline skill 会自动检测到修复场景，在回复末尾显示轻量提示：

- 输入 `r` — 自动走完归档→审查→博客→push 完整流程
- 输入 `a` — 仅归档到 /mdoc
- [s] — 跳过

### 手动触发

也可以直接对 skill 或 subagent 下达指令：

- "用 mdoc-archiver 把这次修复归档" — 归档到 /mdoc
- "用 doc-validator 审查刚才那篇文档" — 交叉验证
- "用 blog-publisher 把 nginx 配置那篇写成博客发出去" — 发布
- "执行 blog-pipeline" — 触发完整流水线

## 变更内容

无代码变更。创建了 1 个 skill 定义文件和 3 个 subagent 定义文件：

- `~/.claude/skills/blog-pipeline/SKILL.md` — 编排层，包含修复场景检测、用户提示模板、三步流水线调度
- `~/.claude/agents/mdoc-archiver.md` — 归档 subagent
- `~/.claude/agents/doc-validator.md` — 审查 subagent
- `~/.claude/agents/blog-publisher.md` — 发布 subagent

注意：subagent 仅存放在用户级目录 `~/.claude/agents/`，项目级目录 `~/Desktop/myFireflyBlog/.claude/agents/` 目前留空，仅供项目配置预留。

## 总结

这套流水线的核心思路是**分治 + 交叉验证**：把长流程拆成三个独立步骤，每步由专门的 subagent 处理，中间加一层只读审查来兜底幻觉。从实际使用来看，doc-validator 确实抓出过几次归档文档里的路径错误和命令拼写问题，起到了预期的把关作用。
