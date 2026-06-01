# Examples

这里放阅读文档后的可运行实战实验。

目录约定：

- 每个实验按主题分组，例如 `mcp/`、`agent-loop/`、`context/`、`tool-use/`。
- 每个实验目录都应包含自己的 `README.md`，说明实验目标、运行方式、验证方法和安全边界。
- 运行记录、踩坑和复盘写在实验目录的 `experiment-log.md`。
- 不提交真实 token、私钥、cookie、生产地址或个人绝对路径。
- 如果实验依赖本机配置，提供 `.env.example` 或示例配置，并写清楚 fallback。

当前实验：

1. [`mcp/01-readonly-notes-server/`](mcp/01-readonly-notes-server/README.md)：只读 MCP notes server，用来验证 MCP 的最小 server、tools、resources、权限边界和 Inspector 调试流程。
