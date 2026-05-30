# Prompt: 恢复上下文

适合新开会话、长任务继续、上下文压缩后重新校准。

```text
请先恢复项目上下文，不要改代码。

请读取：
1. AGENTS.md
2. README.md
3. docs/README.md
4. docs/project-status.md
5. docs/roadmap.md
6. docs/development.md
7. docs/testing.md
8. 当前任务相关的 docs/specs/ 文档
9. 当前任务相关的 docs/plans/ 文档

然后只回复：
- 当前项目目标
- 当前 Roadmap 方向
- Active Spec
- Active Plan
- 当前 slice
- 已完成事项
- 当前进行中事项
- 下一步建议
- 当前验证命令
- 你还需要我补充什么信息
```
