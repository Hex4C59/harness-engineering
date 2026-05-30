# Prompt: 既有项目最小接入

适合已经看过只读审计报告，确认可以改文档和脚本之后使用。

```text
请按照刚才的审计结果，对这个既有项目做最小 agent harness 接入。

约束：
1. 不要改业务代码。
2. 不要重排目录结构。
3. 不要删除或覆盖已有文档。
4. 如果已有 AGENTS.md / CLAUDE.md / docs / scripts，请在原有内容基础上增量修改。
5. 如果已有项目命令，请优先包装已有命令，不要发明一套新命令。
6. 如果信息不确定，在文档里标注“待确认”，不要编造。

请优先完成：
1. 创建或更新 AGENTS.md，让它只作为入口地图和硬约定。
2. 创建或更新 docs/README.md，写清渐进式读取路径。
3. 创建或更新 docs/project-status.md，记录当前项目状态、验证命令和未决问题。
4. 创建或更新 docs/testing.md，记录现有测试策略、TDD 约定和 bug 修复流程。
5. 创建或更新 docs/development.md，记录本地开发命令。
6. 创建或更新 scripts/check，把现有 format/lint/typecheck/test/build 命令串起来。
7. 如果没有足够信息，不要强行创建完整 architecture/decisions/plans 目录，只写建议。

完成后请运行最小安全验证命令，并告诉我：
- 新增或修改了哪些文件。
- 沿用了哪些现有命令。
- 哪些命令还没法运行。
- 哪些信息需要我确认。
- 下一步建议。
```
