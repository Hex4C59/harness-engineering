# Readonly Notes MCP Server

## 目标

这个实验用一个只读 MCP server，把本仓库的文档作为上下文源暴露给 agent。

第一版只验证两件事：

1. `notes://readme` resource：读取项目根目录 `README.md`。
2. `search_docs` tool：在 `docs/` 下搜索包含关键词的 Markdown 文件，只返回文件路径和少量元数据。

## 为什么从这里开始

这个实验贴近当前仓库的真实使用方式：你已经写了很多文档，下一步不是继续泛读，而是让 agent 能通过一个低风险 MCP server 检索和引用这些文档。

它也符合 MCP 落地的最小原则：

- 只读。
- 不接外网。
- 不需要 token。
- 不写文件。
- 输出有限制。
- 可以用 MCP Inspector 单独验证。

## 目录结构

```text
01-readonly-notes-server/
  README.md
  experiment-log.md
  .env.example
  src/
  tests/
```

后续实现时可以补：

```text
src/
  readonly_notes_server.py

tests/
  test_search_docs.py
```

## 预期 MCP 能力

Resource：

```text
notes://readme
```

Tool：

```text
search_docs(keyword: str, limit: int = 10) -> list[SearchResult]
```

`SearchResult` 初始可以很简单：

```json
{
  "path": "docs/coding-agents/research/runtime/05-model-context-protocol.md",
  "title": "Model Context Protocol 系统调研：从工具连接协议到 Agent Runtime 边界"
}
```

## 安全边界

- 只读取项目根目录 `README.md` 和 `docs/**/*.md`。
- 不读取 `.git/`、`.env*`、用户目录或项目外路径。
- 不执行 shell 命令。
- 不发网络请求。
- 不写文件。
- 不把完整大文档默认返回给 tool call。
- stdio 模式下不要向 stdout 写日志。

## 验证清单

- MCP Inspector 能连接 server。
- `resources/list` 能看到 `notes://readme`。
- `resources/read notes://readme` 能返回 README。
- `tools/list` 能看到 `search_docs`。
- `search_docs` 的参数 schema 只包含 `keyword` 和 `limit`。
- 空关键词返回空结果或明确错误。
- `limit` 有上限。
- 搜索结果不包含正文大段内容。
- server 不能读取 `docs/` 之外的任意文件。

## Host compatibility 记录

| Host | 状态 | 配置位置 | 备注 |
|---|---|---|---|
| Codex CLI / IDE | 未验证 | `~/.codex/config.toml` 或 trusted project `.codex/config.toml` | 优先验证 |
| Claude Code | 未验证 | `.mcp.json` project scope 或 local scope | 适合对照 |
| VS Code | 未验证 | `.vscode/mcp.json` 或 user profile | 注意 workspace trust |
| GitHub Copilot cloud agent | 暂不验证 | GitHub repository settings | 云端 agent 不适合第一版本地 stdio |
| OpenAI API / ChatGPT | 暂不验证 | remote MCP server | 需要公网或隧道，后续再做 |

## 下一步

1. 选择 Python 或 TypeScript 实现 server。
2. 用 MCP Inspector 跑通 resource 和 tool。
3. 把运行结果写入 `experiment-log.md`。
4. 再决定是否补 `docs/coding-agents/agent/mcp/servers.example.json`。
