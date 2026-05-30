# Workflow: 新项目 Agent + TDD Bootstrap

这个流程用于从空目录初始化一个适合 coding agent 协作的新项目。目标不是马上写业务代码，而是先固定项目意图、上下文入口、验证命令和 TDD 工作流。

## 什么时候用

- 从零开始新项目。
- 想把 `AGENTS.md`、`docs/`、`scripts/` 和验证流程一次性立好。
- 希望后续每个功能都能按 Spec、Plan、TDD 和 Reviewable Slice 推进。

不适合既有项目接入；既有项目用 [`existing-project-onboarding.md`](existing-project-onboarding.md)。

## 最短路径

1. 技术栈没定时，先用 [`../prompts/tech-stack-selection.md`](../prompts/tech-stack-selection.md)。
2. 技术栈确定后，用 [`../prompts/new-project-bootstrap.md`](../prompts/new-project-bootstrap.md) 初始化 harness。
3. 按 [`../checklists/new-project-bootstrap.md`](../checklists/new-project-bootstrap.md) 检查是否漏项。
4. 模板来源见 [`../templates/project-harness-files.md`](../templates/project-harness-files.md) 和 [`../templates/scripts.md`](../templates/scripts.md)。

## 初始化后的工作方式

- 新功能：用 [`../prompts/new-feature-tdd.md`](../prompts/new-feature-tdd.md)。
- 按计划实现：用 [`../prompts/execute-plan-slice.md`](../prompts/execute-plan-slice.md)。
- 修 bug：用 [`bugfix.md`](bugfix.md)。
- 恢复上下文：用 [`../prompts/context-recovery.md`](../prompts/context-recovery.md)。
- Harness 自查：用 [`../prompts/harness-audit.md`](../prompts/harness-audit.md)。

## 核心分层

新项目初始化后，要坚持这四层：

```text
Roadmap 控制方向；
Spec 控制意图；
Execution Plan 控制行动；
Project Status 控制接手。
```

详细原则见 [`../principles/task-document-layers.md`](../principles/task-document-layers.md)。
