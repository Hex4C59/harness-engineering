# Workflow: Agent Orchestration Map

这个文件用于在任务开始时选择最轻但足够可靠的 workflow / prompt / skill。它不是固定流水线，不要求所有任务都完整经过 brainstorming、writing-plans、TDD、review 和 commit。

核心规则：

```text
Use the lightest workflow that preserves correctness.
```

也就是：任务越小、越清楚、越容易验证，流程越短；任务越模糊、影响越大、风险越高，越需要升级到更完整的 gates。

## 先判断四件事

开始前先快速判断：

1. 需求是否清楚？
2. 是否会改变行为？
3. blast radius 多大？
4. 需要什么证据才能说完成？

如果任何答案不清楚，先不要写最终实现。

## 风险分层

| 层级 | 典型任务 | 推荐路径 |
|---|---|---|
| Tiny | typo、链接、小文案、单行 docs、明显配置说明 | 直接改 -> 局部验证 -> final-check |
| Small | 明确的小 bug、小行为变化、单文件局部调整 | tdd-gate 或 bugfix -> 局部验证 -> 必要时 review-gate |
| Medium | 新功能 slice、非平凡 bug、多个文件、需要计划的 docs/tooling 改动 | brainstorming 可选 -> writing-plans -> 执行一个 slice -> tdd-gate -> review-gate |
| Large / Risky | 架构、迁移、权限、安全、数据模型、公共 API、跨模块重构、部署 | brainstorming -> writing-plans -> 每次一个 slice -> tdd-gate / bugfix -> review-gate -> commit-gate |

## 路由表

| 当前情况 | 进入哪里 |
|---|---|
| 不熟悉代码，不知道从哪改 | [`../prompts/scout.md`](../prompts/scout.md) |
| 需求、目标用户、非目标或验收标准不清 | [`../prompts/brainstorming.md`](../prompts/brainstorming.md) 或 `skills/brainstorming` |
| 有多个可行方案，需要比较取舍 | [`../prompts/solution-comparison.md`](../prompts/solution-comparison.md) |
| 需求已清楚，但任务多步骤或影响多个文件 | [`../prompts/writing-plans.md`](../prompts/writing-plans.md) 或 `skills/writing-plans` |
| 已有计划，要执行下一步 | [`../prompts/execute-plan-slice.md`](../prompts/execute-plan-slice.md) |
| 行为会变化 | `skills/tdd-gate` |
| 修 bug | [`bugfix.md`](bugfix.md)；非琐碎 bug 必须先复现和回归测试 |
| diff 变大或 scope creep | [`../prompts/reviewable-slices.md`](../prompts/reviewable-slices.md) |
| 实现完成，需要判断是否正确可 review | [`../prompts/review-gate.md`](../prompts/review-gate.md) 或 `skills/review-gate` |
| 准备提交、需要拆 commit 或写 message | `skills/commit-gate` |
| 会话太长或换新会话 | [`../prompts/context-recovery.md`](../prompts/context-recovery.md) |

## 可跳过条件

可以跳过 brainstorming，当：

- 用户目标、非目标和验收标准已经清楚。
- 任务不涉及产品方向、架构方向或多方案取舍。

可以跳过正式 writing-plans，当：

- 任务很小。
- 改动影响 1-2 个文件。
- 验证命令明确。
- 不涉及公共 API、数据、权限、部署、依赖或跨模块重构。

可以跳过 tdd-gate，当：

- 任务不改变行为，例如纯 docs、格式、链接、注释或说明。
- UI 视觉微调用截图/浏览器验证更合适，并已记录验证方式。
- 原型 spike 已明确标记为非最终实现。

可以跳过独立 review-gate，当：

- diff 很小、低风险、验证新鲜且通过。
- 任务不触及公共接口、数据、安全、权限、部署、依赖或多个模块。
- 用户没有要求 review。

不要跳过 final-check。即使是小任务，也要能说明改了什么、如何验证、剩余风险是什么。

## 升级条件

遇到这些情况，升级到更重的流程：

- 需求解释出现分歧。
- 预计修改超过 2-3 个文件。
- 需要新增依赖、迁移数据、改公共接口或改变部署。
- 需要修改权限、安全、认证、支付、数据持久化或并发逻辑。
- 需要跨模块重构。
- 测试策略不明确。
- agent 开始顺手改无关文件、重命名、格式化或扩大 scope。
- 局部修复连续失败。

升级方式：

```text
不清楚 -> brainstorming
范围变大 -> writing-plans / reviewable-slices
行为变化 -> tdd-gate
bug 难复现 -> bugfix workflow
diff 难 review -> review-gate
准备提交 -> commit-gate
```

## 推荐主路径

### 小任务

```text
读相关文件
-> 改最小范围
-> 局部验证
-> final-check
```

### 中等功能

```text
context-entry
-> brainstorming 或 solution-comparison（如果需要）
-> writing-plans
-> execute-plan-slice
-> tdd-gate
-> review-gate
-> final-check
```

### Bug 修复

```text
bugfix workflow
-> 复现
-> 回归测试
-> 最小修复
-> 验证
-> review-gate
-> final-check
```

### 高风险任务

```text
scout
-> brainstorming
-> writing-plans
-> 每次只执行一个 slice
-> tdd-gate 或 bugfix
-> review-gate
-> commit-gate（如果用户要提交）
```

## 输出要求

当 agent 使用这个 map 做分流时，先给一个短判断：

```text
任务层级：Tiny / Small / Medium / Large
选择路径：
跳过的 gate：
跳过原因：
升级条件：
```

如果后续发现判断错了，停下来升级流程，不要在实现中暗自扩大范围。
