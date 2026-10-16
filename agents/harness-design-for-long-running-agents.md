# Harness Design for Long-Running Application Development

> Anthropic Engineering
> 发布时间：2026-03-24

## 1. 核心问题

Anthropic 关注两个彼此相关的问题：

* 如何让代码 Agent 生成更高质量、更有设计感的前端；
* 如何让 Agent 在数小时的自主运行中完成一个较完整的软件应用。

核心结论是：**模型能力并不是决定 Agent 最终表现的唯一因素，Harness 的结构设计同样重要。**

早期长时间运行的 Agent 主要存在两个问题：

1. **长任务中的上下文退化**

   * 随着上下文不断增长，Agent 容易逐渐偏离任务；
   * 一些模型在接近上下文限制时会提前结束工作；
   * Context compaction 可以压缩历史，但并不一定彻底解决这一问题。

2. **自我评价不可靠**

   * 让生成代码的 Agent 同时评价自己的工作，通常会产生过于乐观的评价；
   * 在前端设计这类主观任务中尤其明显；
   * 因此可以将“生成”和“评价”拆成不同 Agent。


## 2. Generator–Evaluator 结构

文章借鉴 GAN 的思想，将任务拆成两个主要角色：

```text
Generator
    │
    ▼
生成当前版本
    │
    ▼
Evaluator
    │
    ├── 发现问题
    ├── 评分
    └── 给出改进建议
    │
    ▼
Generator 修改
    │
    └───────────────↺
```

## 3. 前端设计实验

Anthropic 首先在前端设计任务中验证这种结构。

Evaluator 主要从四个方面评价前端：

| Criterion      | 关注内容                     |
| -------------- | ------------------------ |
| Design Quality | 整体设计是否协调、是否形成明确视觉语言      |
| Originality    | 是否存在独立设计决策，而不是模板化或默认组件拼接 |
| Craft          | 字体、间距、颜色、层级、对比度等实现质量     |
| Functionality  | 用户是否能够理解并正常使用界面          |

Evaluator 使用浏览器自动化工具直接操作页面，而不是仅通过代码或截图判断：


实验通常运行多个迭代轮次。Generator 每轮根据评价决定：

```text
当前方向有效
    → 继续 refinement

当前方向效果较差
    → pivot 到新的设计方向
```

作者观察到，评价标准本身不仅是“评分器”，也会反向塑造 Generator 的生成分布，因此 **Rubric 同时承担 evaluation signal 和 generation guidance 两种作用**。

---

# 4. 扩展到 Full-Stack Application

随后，Anthropic 将 Generator–Evaluator 模式扩展到完整应用开发，并形成三个 Agent：

```text
Planner
   ↓
Generator
   ↓
Evaluator
```

## 4.1 Planner

Planner 将用户非常简短的需求扩展成完整 Product Specification：

Planner 主要负责：

* 产品目标；
* 功能范围；
* 用户体验；
* 高层技术结构；
* 功能之间的关系。

一个重要原则是：

> Planner 不应该过早规定大量底层实现细节。

## 4.2 Generator

Generator 负责真正实现应用。

早期版本采用类似 Sprint 的开发方式：

```text
Feature 1
   ↓
Implementation
   ↓
Self-check
   ↓
QA

Feature 2
   ↓
Implementation
   ↓
Self-check
   ↓
QA
```

这种方式继承了 Anthropic 之前 long-running agent harness 的核心思想：

> 不要试图一次性完成整个应用，而是持续进行 incremental progress。

## 4.3 Evaluator

Evaluator 相当于独立 QA Agent。

它通过浏览器自动化真正使用应用，并检查：

```text
UI interaction
API behavior
Database state
Feature completeness
Visual quality
Code quality
```

# 5. Sprint Contract

在 Generator 开始编码之前：

```text
Generator
   │
   ├── 提议这一轮要实现什么
   └── 提议如何验证

        ↓

Evaluator
   │
   ├── 检查范围是否正确
   ├── 检查验收标准
   └── 要求补充或修改

        ↓

双方达成一致

        ↓

开始编码
```

这实际上把模糊需求转化成了**可测试的局部成功标准**。

# 6. Agent 之间如何通信

Agent 并不主要依赖共享完整 conversation history，而是通过结构化文件交换信息。

大致过程：

```text
Planner
   │
   └── product_spec.md
             ↓
Generator
   │
   └── sprint_contract / implementation state
             ↓
Evaluator
   │
   └── QA report / bug report
             ↓
Generator
```

这和 Anthropic 之前的 long-running agent harness 一致：旧版系统同样通过 progress file、feature list 和 Git history 让不同 session 恢复任务状态。

# 7. Full Harness 与 Single Agent

文章用一个浏览器内的 2D Retro Game Maker 做实验。

大致比较为：

| System       | Duration |   Cost |
| ------------ | -------: | -----: |
| Single Agent |  约 20 分钟 |   约 $9 |
| Full Harness |   约 6 小时 | 约 $200 |

Full Harness 的成本高很多，但 Planner 会把简单需求扩展为更丰富的产品规格，而 Evaluator 会逐项验证 Sprint Contract，并发现许多只查看代码时很难发现的真实交互问题。

# 8. QA Agent 并不会天然做好 QA

一个非常重要的发现是：

> 单纯增加一个 Evaluator Agent 并不足以获得可靠评价。

未经调优的 QA Agent 可能：

```text
发现 Bug
   ↓
认为问题“不严重”
   ↓
自己说服自己
   ↓
PASS
```

或者只进行表面测试，而没有深入测试边界情况。


Evaluator 本身也是需要被训练 / 调优的 Agent Component

# 9. Harness 不应该无限复杂

```text
每增加一个 Harness Component

实际上都隐含一个假设：

“模型本身无法可靠完成这件事。”
```

当模型能力提升以后，就应该重新检查这些假设。

Anthropic 更通用的 Agent 工程原则也是：

```text
从最简单的系统开始
        ↓
只有性能确实需要时
        ↓
再增加 Agent / Workflow Complexity
```

# 10. Opus 4.6 后 Harness 的简化

随着模型能力增强，文章中的第二版 Harness 删除了固定 Sprint decomposition。

结构变为：

```text
Planner
   ↓
Generator 长时间连续开发
   ↓
Evaluator
   ↓
Generator Repair
   ↓
Evaluator
```

# 11. Evaluator 是否必要取决于任务难度

文章提出一个很重要的动态观点：

```text
Evaluator 并不是永远必要。
```

可以理解为：

```text
Task Difficulty
        ↑

        │      Evaluator 有明显价值
        │        ┌──────────────
        │       /
        │      /
        │─────● Model Capability Boundary
        │
        │ Evaluator 收益较小
        └────────────────────────→ Model Capability
```

# 12. V2 Harness 实验

新版 Harness 用浏览器中的 Digital Audio Workstation（DAW）进行测试。

总体运行大约：

```text
3 h 50 min
$124.70
```

其中主要时间花在 Generator 的长时间 Build 阶段。

尽管 Generator 已能持续开发两个小时以上，Evaluator 仍然发现了一些重要问题，例如：

```text
功能表面存在
        ↓
但核心交互没有真正实现
```

这再次说明：

```text
Feature Presence
        ≠
Feature Completeness
        ≠
Feature Usability
```

# 13. Harness Design 的核心原则

整篇文章可以归纳为以下设计思想。

## 13.1 分离生成与评价

## 13.2 将模糊目标转换成可验证标准

## 13.3 使用真实环境验证

## 13.4 使用 Persistent Artifact 管理状态

## 13.5 Incremental Progress

## 13.6 Harness 应随模型能力变化

# 14. 一句话总结

这篇文章的核心并不是“多 Agent 一定比单 Agent 好”，而是：

> 对于超出单模型稳定能力范围的长周期软件开发任务，可以通过明确规划、独立评价、真实环境验证、结构化状态传递和迭代修复，把模型组织成一个更可靠的软件工程闭环；随着模型能力提升，再逐步移除不再必要的 Harness scaffold。

