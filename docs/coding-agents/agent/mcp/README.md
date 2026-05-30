# MCP

这里预留项目级 MCP 配置示例、启用说明和安全边界。

## 定位

MCP 适合描述 agent 可以连接哪些外部工具或服务，例如：

- browser / Playwright：验证本地 Web 应用。
- docs search：查询项目文档或官方文档。
- database：只读查看本地开发数据库。
- GitHub：查看 issue、PR、CI 状态。

## 使用原则

- 默认只提交示例配置，不提交真实 token、私钥或个人路径。
- 高权限、写入型、部署型 MCP 不默认启用。
- 项目级 MCP 配置应说明权限、数据范围、启用方式和禁用方式。
- 如果 MCP 依赖本机环境，必须写清楚 fallback，不要让项目验证强依赖个人机器。

## 注入方式

把 `docs/agent/mcp/` 复制进项目后，里面应只放示例和说明。真实启用通常发生在个人级或环境级配置里。

推荐流程：

1. 在项目中提交 `docs/agent/mcp/README.md` 和 `servers.example.json`。
2. 在示例里只写 server 名称、用途、权限、需要的环境变量名，不写真实值。
3. 开发者在自己的 MCP 客户端配置里复制示例并填入本地路径或凭据。
4. 在 `docs/development.md` 中说明哪些功能依赖 MCP，以及没有 MCP 时的 fallback。

不要提交：

- token、私钥、cookie、生产数据库地址
- 本机绝对路径
- 会写入生产系统的默认配置
- 高权限 GitHub / cloud / database 写入配置

如果某个 MCP 只是辅助能力，项目验证命令不应该强依赖它。

## 状态

当前目录先作为占位入口。后续可以补充 `servers.example.json` 和按工具分组的配置说明。
