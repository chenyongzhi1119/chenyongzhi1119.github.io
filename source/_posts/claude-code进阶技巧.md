---
title: Claude Code 进阶技巧
date: 2026-03-21
categories: 工具
tags:
  - AI
  - Claude Code
  - 效率工具
description: 基于 Claude Code 官方文档整理的进阶使用技巧，涵盖 Plan Mode、上下文管理、并行 Session、非交互模式、CLAUDE.md 高级用法、Auto Memory 等，帮助你从"会用"到"用好"。
---

在掌握 Claude Code 的基本操作后，真正决定效率上限的是对上下文、计划模式、并行会话等进阶机制的理解。本文基于官方文档整理了最有价值的进阶技巧。

<!-- more -->

## 核心约束：上下文窗口

所有进阶技巧的出发点都是同一个约束：**上下文窗口会很快填满，填得越满，Claude 的表现越差**。

上下文里装着对话历史、读过的文件、命令输出，一次调试会话就可能消耗数万 token。理解这一点，才能理解为什么后面所有技巧都在围绕"如何保持上下文干净"展开。

---

## 技巧一：Plan Mode 深度用法

Plan Mode 让 Claude **只读不写**，先分析再执行，适合改动多个文件的复杂任务。

### 三种进入方式

```bash
# 1. 启动时直接进入
claude --permission-mode plan

# 2. 会话中按 Shift+Tab 切换（循环：普通 → 自动接受 → Plan Mode）

# 3. 非交互模式下使用
claude --permission-mode plan -p "分析认证系统并提出改进方案"
```

### 推荐四步工作流

```
[Plan Mode]
第一步：读 /src/auth，理解 session 和登录逻辑

↓

[Plan Mode]
第二步：我要加 Google OAuth，哪些文件需要改？给出完整计划

↓  按 Ctrl+G 在编辑器里直接修改计划内容

[Normal Mode]
第三步：按照计划实现 OAuth 流程，写测试，运行并修复失败

↓

[Normal Mode]
第四步：写一个规范的 commit message 并提 PR
```

> 小任务（改一行、重命名变量）不需要 Plan Mode，直接做更快。多文件改动、不熟悉的代码区域、方案不确定时，才值得开启。

---

## 技巧二：Extended Thinking（深度推理）

Claude 默认开启扩展思考，会在回答前进行内部推理。

### ultrathink 关键词

在 prompt 里加上 `ultrathink`，可以对当前这条请求触发最高强度的推理，不影响后续会话设置：

```
ultrathink，帮我设计这个分布式缓存系统的一致性方案
```

适合：复杂架构决策、棘手 bug 分析、多步骤实现规划、权衡不同方案。

### 查看推理过程

按 `Ctrl+O` 开启 verbose 模式，Claude 的内部推理会以灰色斜体显示。

---

## 技巧三：上下文管理进阶

### /compact 带指令压缩

```
/compact 只保留 API 接口改动和测试命令
```

比直接 `/compact` 更精准，告诉 Claude 压缩时保留什么。

### /rewind 检查点回溯

Claude 每次操作前都会自动创建检查点。双击 `Esc` 或运行 `/rewind` 可以回到任意历史状态：

- 只恢复对话
- 只恢复代码文件
- 两者都恢复
- 从某个节点重新总结

检查点跨 session 持久保存，关掉终端后依然可以回溯。

### /btw 侧边问题

```
/btw 这个项目用的是哪个测试框架？
```

用 `/btw` 提一个临时问题，答案会以浮层显示，**不进入对话历史**，不消耗主上下文。适合查个细节又不想污染当前对话。

### 何时 /clear

| 情况 | 处理方式 |
|------|---------|
| 换了一个不相关的任务 | `/clear` |
| 同一个问题已纠正两次以上 | `/clear` 后用更精确的 prompt 重来 |
| 上下文里充满无关文件内容 | `/clear` |

---

## 技巧四：Subagent 并行探索

Subagent 在独立上下文窗口里运行，**探索过程不消耗主对话上下文**，最终只返回摘要。

### 用 Subagent 做调研

```
用 subagent 调查我们的认证系统如何处理 token 刷新，
以及有没有现成的 OAuth 工具可以复用
```

Subagent 会读大量文件，但这些读操作不会出现在你的主对话里。

### Writer / Reviewer 模式

开两个 session，一个写代码，另一个做审查：

| Session A（Writer） | Session B（Reviewer） |
|--------------------|--------------------|
| 实现 API 限流中间件 | |
| | `review @src/middleware/rateLimiter.ts，找边界条件和竞争问题` |
| 根据 Review 意见修改 | |

两个 session 视角不同，Reviewer 没有"自己刚写了这段代码"的偏见。

---

## 技巧五：Git Worktree 并行 Session

多任务并行时，每个 session 需要独立的代码副本，否则改动会互相冲突。

```bash
# 为 feature 开一个独立 worktree
claude --worktree feature-auth

# 同时开另一个处理 bugfix
claude --worktree bugfix-123

# 不指定名字，自动生成
claude --worktree
```

Worktree 创建在 `<repo>/.claude/worktrees/<name>/`，有独立分支。无改动时退出自动清理，有改动时会询问是否保留。

> 建议在 `.gitignore` 里加上 `.claude/worktrees/`，避免出现在 git 状态里。

---

## 技巧六：非交互模式与脚本化

### 基本用法

```bash
# 单次查询
claude -p "解释这个项目做什么"

# 结构化输出
claude -p "列出所有 API 端点" --output-format json

# 流式输出（实时处理）
claude -p "分析这个日志文件" --output-format stream-json
```

### 集成到 CI/CD

```json
// package.json
{
  "scripts": {
    "lint:claude": "claude -p '你是一个 linter。检查与 main 分支的差异，报告 typo 问题。格式：文件名:行号 + 问题描述。不返回其他内容。'"
  }
}
```

### Pipe 管道

```bash
# 分析构建错误
cat build-error.txt | claude -p '简洁解释这个构建错误的根本原因' > output.txt

# 批量文件迁移
for file in $(cat files.txt); do
  claude -p "将 $file 从 React 迁移到 Vue，返回 OK 或 FAIL" \
    --allowedTools "Edit,Bash(git commit *)"
done
```

---

## 技巧七：CLAUDE.md 高级用法

### @import 拆分文件

```markdown
# CLAUDE.md

参考 @README.md 了解项目概述，@package.json 查看可用命令。

## 额外规范
- Git 工作流：@docs/git-instructions.md
- 个人偏好（不提交）：@~/.claude/my-project-instructions.md
```

### .claude/rules/ 按路径生效

在 `.claude/rules/` 目录下创建规则文件，可以**只在特定文件被访问时加载**，节省上下文：

```markdown
<!-- .claude/rules/api-design.md -->
---
paths:
  - "src/api/**/*.ts"
---

# API 开发规范
- 所有端点必须做入参校验
- 使用统一的错误响应格式
- 必须包含 OpenAPI 注释
```

规则文件只在 Claude 读取匹配路径的文件时才进入上下文，避免每次都装载全部规则。

### 有效 CLAUDE.md 的原则

| 写进去 | 不要写 |
|--------|--------|
| Claude 猜不到的构建命令 | Claude 能从代码里读出来的信息 |
| 与默认不同的代码风格 | 标准语言约定 |
| 测试方式和首选测试器 | 详细 API 文档（链接即可） |
| 分支命名、PR 规范 | 频繁变化的信息 |
| 项目特有的架构决策 | 显而易见的做法（"写整洁代码"） |

> 文件超过 200 行后，Claude 会开始忽略部分规则。定期精简，对每一行问：「去掉这行 Claude 会犯什么错误？」不会犯就删掉。

---

## 技巧八：Auto Memory（自动记忆）

Claude 会在会话中自动记录有价值的信息：构建命令、调试经验、架构备注、你的偏好习惯。

存储位置：`~/.claude/projects/<project>/memory/`

```
memory/
├── MEMORY.md          # 索引，每次会话开始时加载（前 200 行）
├── debugging.md       # 调试经验
├── api-conventions.md # API 设计决策
└── ...
```

### 使用 /memory 管理

- `/memory` 列出当前会话加载的所有 CLAUDE.md 和规则文件
- 可以查看、编辑自动记忆文件
- 可以开关 Auto Memory

你也可以主动要求 Claude 记住某件事：

```
记住：这个项目的 API 测试需要本地启动一个 Redis 实例
```

Claude 会把它写入 auto memory，下次会话自动知晓。

---

## 技巧九：Session 管理

### 命名和恢复

```bash
# 启动时命名
claude -n oauth-migration

# 会话中重命名
/rename auth-refactor

# 恢复最近会话
claude --continue

# 选择会话恢复
claude --resume

# 按名字恢复
claude --resume oauth-migration
```

### /resume 选择界面快捷键

| 快捷键 | 操作 |
|--------|------|
| `↑` / `↓` | 浏览会话 |
| `P` | 预览会话内容 |
| `R` | 重命名会话 |
| `/` | 搜索过滤 |
| `B` | 只看当前 git 分支的会话 |

---

## 技巧十：让 Claude 面试你

处理大型功能时，不要急着写 spec，先让 Claude 来问你问题：

```
我想做 [功能简述]。用 AskUserQuestion 工具深度面试我。

问技术实现、UI/UX、边界情况、权衡与隐患。
跳过显而易见的问题，挖掘我可能没想到的难点。

面试完后，把结论写入 SPEC.md。
```

Claude 会追问你没考虑到的地方，面试完后生成完整规格文档。然后**开一个新 session** 执行这个 spec，新 session 上下文干净，专注于实现。

---

## 小结

| 场景 | 技巧 |
|------|------|
| 多文件复杂改动 | Plan Mode + `Ctrl+G` 编辑计划 |
| 棘手问题需要深度推理 | prompt 里加 `ultrathink` |
| 上下文快满了 | `/compact <保留什么>` 或 `/clear` |
| 想回撤某次操作 | 双击 `Esc` 或 `/rewind` |
| 探索代码但不想消耗主上下文 | 让 Claude 用 subagent 调研 |
| 多任务并行 | `--worktree` 独立代码副本 |
| 集成到脚本/CI | `claude -p` + `--output-format` |
| 规则只对某类文件生效 | `.claude/rules/` + `paths` frontmatter |
| 跨 session 记住某件事 | Auto Memory 或主动告知 Claude 记录 |
| 大功能需求不清晰 | 让 Claude 先面试你，生成 spec 再执行 |

进阶使用的本质是：**把上下文当成最贵的资源来管理**，用正确的工具在正确的时机干预 Claude 的工作流。
