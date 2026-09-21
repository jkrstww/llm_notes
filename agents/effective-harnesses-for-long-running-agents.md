# Effective Harnesses for Long-Running Agents

> 来源：Anthropic Engineering  
> 原文：https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents  
> 发布日期：2025-11-26  
> 说明：以下为结构化中文整理，不是逐字翻译。

## 1. 核心问题

长时间运行的 Agent 往往需要跨越多个 context window。新的会话无法天然知道上一轮做了什么，因此仅依赖 context compaction 并不足以保证连续、稳定地完成复杂工程任务。

常见失败主要有两类：

- 一次尝试完成过多内容，导致上下文耗尽，留下半完成状态；
- 已完成部分功能后，后续 Agent 误以为整个任务已经结束。

因此，Harness 的关键目标是让每个新会话都能快速恢复项目状态，并持续做“小步、可验证”的增量工作。

## 2. 两阶段 Agent 设计

### Initializer Agent

首次运行负责建立后续会话需要的基础环境，包括：

- 初始化 Git 仓库；
- 创建 `init.sh`，统一启动开发环境；
- 创建进度记录文件，如 `claude-progress.txt`；
- 根据用户需求生成完整的 feature list；
- 将所有功能初始标记为未通过。

Anthropic 更倾向于使用 JSON 保存 feature list，因为结构更稳定，也更不容易被 Agent 意外重写。

### Coding Agent

后续会话不重新规划整个项目，而是：

1. 读取当前目录、Git 历史和进度文件；
2. 查看 feature list；
3. 选择一个尚未完成的高优先级功能；
4. 实现该功能；
5. 做端到端验证；
6. 验证成功后更新状态；
7. 提交 Git，并记录本轮进展。

核心原则是：**每一轮只推进有限范围的工作，并给下一轮留下清晰、可恢复的状态。**

## 3. Harness 中的关键机制

### Feature List

把高层需求拆成明确、可验证的功能项，并长期保存在结构化文件中。它既是任务清单，也是“是否真正完成”的外部状态。

### Incremental Progress

Agent 每轮只完成一个或少量功能，避免一次修改过多内容。完成后通过 Git commit 和 progress 文件保存交接信息。

### Testing

不能仅依赖单元测试或接口调用。对于 Web 应用，应尽可能让 Agent 使用浏览器自动化，从用户视角执行端到端测试，再将 feature 标记为通过。

### Clean State

每轮结束时，代码库应尽量保持类似可合并到主分支的状态：没有明显遗留错误，变更可理解，进展有记录，下一轮不需要先处理无关混乱。

## 4. 一个典型 Session

```text
恢复项目状态
    ↓
读取 progress / git log / feature list
    ↓
启动开发环境
    ↓
执行基础 smoke test
    ↓
选择一个未完成 feature
    ↓
实现
    ↓
端到端测试
    ↓
更新 feature 状态
    ↓
git commit + progress update
```

## 5. 常见失败与对应机制

| 失败模式 | Harness 对策 |
| --- | --- |
| Agent 过早宣布整个项目完成 | 使用完整 feature list，并要求每轮只选择未完成项 |
| 上一轮留下错误或未记录进度 | 使用 Git 历史、progress 文件和会话开始时的基础测试 |
| 功能未经验证就标记完成 | 只有端到端验证通过后才能修改 `passes` 状态 |
| 每轮都重新摸索如何启动项目 | 由 initializer 创建统一的 `init.sh` |

## 6. 文章的核心启示

对于 long-running agent，真正重要的不是单纯扩大 context，而是设计好跨 session 的“外部记忆”和工作协议：

```text
Persistent State
+ Explicit Task List
+ Incremental Execution
+ End-to-End Validation
+ Version Control
= 更稳定的长期 Agent 执行
```

这可以理解为把传统软件工程中的任务管理、交接记录、版本控制和回归测试，显式嵌入 Agent Harness。

## 7. 后续方向

Anthropic 认为仍有两个值得继续研究的问题：

- 单一通用 Coding Agent 与多 Agent 架构相比，哪种更适合跨上下文长期任务；
- 这些经验能否从全栈 Web 开发推广到科研、金融建模等其他 long-horizon agent 任务。
