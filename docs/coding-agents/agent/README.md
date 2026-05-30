# Coding Agent Assets

这里保存可以复制到具体项目里的 agent 协作资产。目标项目中推荐落位到 `docs/agent/`。

## 目录

| 目录 | 用途 |
|---|---|
| [`playbooks/`](playbooks/README.md) | 工作流、可复制 prompt、检查清单、模板和稳定原则 |
| [`skills/`](skills/README.md) | 可安装或可改写成项目级能力的 skill 模板 |
| [`mcp/`](mcp/README.md) | MCP server 配置示例、启用说明和安全边界 |

## 使用方式

新项目初始化时，可以把本目录下的资产复制到目标项目：

```text
docs/coding-agents/agent/playbooks/ -> <project>/docs/agent/playbooks/
docs/coding-agents/agent/skills/    -> <project>/docs/agent/skills/
docs/coding-agents/agent/mcp/       -> <project>/docs/agent/mcp/
```

默认只把项目专属的 `AGENTS.md`、`docs/project-status.md`、`docs/development.md`、`docs/testing.md` 和 `docs/agent/README.md` 追踪进 Git。复制进去的 `playbooks/`、`skills/` 和 `mcp/` 可以按项目需要决定是否 vendoring。

## 注入方式

复制到项目内只是保存资产，不等于运行时自动启用。

| 资产 | 项目内保存位置 | 运行时启用方式 |
|---|---|---|
| Playbooks | `docs/agent/playbooks/` | 在任务 prompt 或 `AGENTS.md` 中要求 agent 阅读对应 workflow / prompt |
| Skills | `docs/agent/skills/` | 复制或安装到具体 agent runtime 的 skills 目录；未安装时也可以让 agent 直接阅读对应 `SKILL.md` 执行 |
| MCP | `docs/agent/mcp/` | 把示例配置改成本地真实配置，再按具体客户端方式启用；不要提交真实凭据 |

推荐节奏：

1. 新项目先复制 `playbooks/`，建立稳定工作流。
2. 只把成熟、低风险、经常重复的流程升级为 `skills/`。
3. MCP 只放示例和说明；真实启用在个人或环境级配置里完成。

## 分层

```text
playbooks 定义规则；
skills 执行规则；
mcp 提供工具连接。
```
