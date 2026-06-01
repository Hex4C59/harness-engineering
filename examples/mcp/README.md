# MCP Examples

这里放 Model Context Protocol 相关实战实验。

实验顺序：

1. [`01-readonly-notes-server/`](01-readonly-notes-server/README.md)：从只读本地 server 开始，验证 MCP tools/resources 的最小闭环。

原则：

- 默认只读。
- 默认不接生产系统。
- 默认不提交真实 token 或个人路径。
- 先用 MCP Inspector 验证，再接入具体 host。
- 每个实验都记录 host compatibility：Claude Code、VS Code、Codex、GitHub Copilot、OpenAI API / ChatGPT 哪些能跑，哪些不能跑。
