# Prompt: Writing Plans

适合需求、Spec 或 brainstorming 方向已经清楚，但还不能直接实现，需要先写 `docs/plans/*.md` 执行计划时使用。这个 prompt 只写计划，不改业务代码。

```text
请使用 writing-plans 方式写一个 Execution Plan，不要实现代码。

目标：

非目标：

关联 Spec / brainstorming 结果 / issue：

已知约束：

请先读取 AGENTS.md、README.md、docs/project-status.md、docs/development.md、docs/testing.md、相关 Spec / ADR / 代码 / 测试。

计划请写到：
docs/plans/YYYY-MM-DD-feature-name.md

计划必须包含：
1. Goal / Non-goals。
2. Linked Context。
3. Spec Summary。
4. Blast Radius。
5. Affected Files，每个文件说明责任和预期改动。
6. Acceptance Criteria。
7. Reviewable Slices。
8. 每个 slice 的 TDD 步骤、验证命令、回滚方式和 stop conditions。
9. Full Validation。
10. Risks、Done Criteria 和 Plan Change Log。

写完后请做 plan self-review：
- spec coverage
- scope control
- file consistency
- TDD coverage
- validation
- reviewability
- rollback
- placeholder scan
- name consistency

计划完成后停下来，等我确认后再执行第一个 slice。
```
