# Workflow: 日常 Coding Agent 开发

这个流程用于日常功能开发、重构、文档更新和一般维护任务。

## 先判断场景

| 当前情况 | 用什么 |
|---|---|
| 不熟悉代码，不知道从哪改 | [`../prompts/scout.md`](../prompts/scout.md) |
| 需求不清楚，方案不确定 | [`../prompts/solution-comparison.md`](../prompts/solution-comparison.md) |
| 要做新功能 | [`../prompts/new-feature-tdd.md`](../prompts/new-feature-tdd.md) |
| 已经有计划，要执行下一步 | [`../prompts/execute-plan-slice.md`](../prompts/execute-plan-slice.md) |
| 要修 bug | [`bugfix.md`](bugfix.md) |
| agent 一次改太多 | [`../prompts/reviewable-slices.md`](../prompts/reviewable-slices.md) |
| 已经有 diff，需要检查 | [`../prompts/review-gate.md`](../prompts/review-gate.md) |
| 想并行使用多个 agent | [`../prompts/parallel-research.md`](../prompts/parallel-research.md) |
| 会话太长或换新会话 | [`../prompts/context-recovery.md`](../prompts/context-recovery.md) |

## 五个硬规则

1. 不清楚怎么改时，先 scout，不写最终代码。
2. 非平凡任务先评估 blast radius。
3. 大任务必须拆成 reviewable slices，每轮只做一个 slice。
4. 完成前必须有测试/检查证据和 diff review。
5. 并行 agent 主要用于 research、POC、方案比较，不用于制造多个大 diff。

## 标准任务循环

1. 读取入口上下文：[`../prompts/context-entry.md`](../prompts/context-entry.md)。
2. 判断任务类型和 blast radius。
3. 必要时先 scout 或方案比较。
4. 需求不清时写 Spec。
5. 非平凡任务写 Execution Plan。
6. 拆 reviewable slices。
7. 本轮只执行一个 slice。
8. 先写失败测试并确认 RED。
9. 写最小实现并确认 GREEN。
10. 必要时重构。
11. 运行局部验证和 `./scripts/check`。
12. 做 review gate。
13. 更新 `docs/project-status.md` 和相关计划。
14. 完成前用 [`../prompts/final-check.md`](../prompts/final-check.md) 收口。

日常检查清单见 [`../checklists/daily-development.md`](../checklists/daily-development.md)。
