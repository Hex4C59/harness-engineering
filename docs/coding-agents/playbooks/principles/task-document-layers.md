# Roadmap / Spec / Plan / Status 分层

非平凡任务开始前，先分清四层文档：

| 层次 | 文件 | 作用 |
|---|---|---|
| Roadmap | `docs/roadmap.md` | 决定长期方向、Now/Next/Later 和暂时不做 |
| Spec | `docs/specs/*.md` 或计划中的 Spec 章节 | 固定需求意图、非目标、验收标准和设计边界 |
| Execution Plan | `docs/plans/*.md` | 拆 reviewable slices、TDD 步骤、验证命令和回滚方式 |
| Project Status | `docs/project-status.md` | 记录当前 active work、最近验证、下一步和接手点 |

不要用一个 `plan.md` 同时承担这四件事。

一句话：

```text
Roadmap 控制方向；Spec 控制意图；Execution Plan 控制行动；Project Status 控制接手。
```

## 写入边界

| 情况 | 更新位置 |
|---|---|
| 当前目标、进度、风险变化 | `docs/project-status.md` |
| 长期方向、优先级、Now/Next/Later 变化 | `docs/roadmap.md` |
| 需求意图、非目标、验收标准变化 | `docs/specs/` 或当前 plan 的 Spec 章节 |
| 新增开发命令或环境要求 | `docs/development.md` |
| 新增测试策略或验证命令 | `docs/testing.md` |
| 架构边界变化 | `docs/architecture/overview.md` |
| 做出重要技术选择 | `docs/decisions/` |
| 复杂任务执行计划和 slice 状态 | `docs/plans/` |
| 遇到可复用失败经验 | `docs/troubleshooting.md` |

如果下次新会话还需要知道，就写进 docs。
