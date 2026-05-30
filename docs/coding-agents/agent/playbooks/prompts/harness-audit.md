# Prompt: Harness 自查

适合怀疑项目文档、脚本或 agent 约定变乱时使用。

```text
请审查这个项目的 agent harness 配置，不要改业务代码。

请检查：
1. AGENTS.md 是否只作为入口地图，是否过长。
2. docs/README.md 是否能指导渐进式读取。
3. docs/project-status.md 是否能让新会话接手。
4. docs/roadmap.md 是否只记录长期方向、Now/Next/Later 和暂时不做，没有混入执行细节。
5. docs/specs/ 和 docs/plans/ 是否分工清楚：Spec 控制意图，Execution Plan 控制行动。
6. docs/development.md 是否写清本地开发命令。
7. docs/testing.md 是否写清 TDD 和验证命令。
8. scripts/bootstrap、scripts/dev、scripts/test、scripts/check 是否存在且可运行。
9. README.md 是否写清项目目标、运行方式和验证方式。

请输出：
- 已经合格的部分。
- 缺失或混乱的部分。
- 建议修复顺序。

先只给审查报告，不要直接修改。
```
