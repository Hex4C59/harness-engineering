# Model Context Protocol 系统调研：从工具连接协议到 Agent Runtime 边界

调研日期：2026-05-31

建议路径：如果新建独立研究文档，可放在 `docs/research/2026-05-31-model-context-protocol.md`；本仓库已经有 coding agent runtime 主题入口，所以本次沿用 `docs/coding-agents/research/runtime/05-model-context-protocol.md`。

访问日期：2026-05-31

## 调研问题

这篇文档围绕三组问题调研 MCP（Model Context Protocol，模型上下文协议）：

1. 是什么：核心定义、关键术语、边界、容易混淆的概念。
2. 为什么：它解决什么问题、出现背景、适用场景、不适用场景、主要 trade-off。
3. 怎么做：实践步骤、最小例子、常见实现路径、验证方法、常见坑。

资料优先级：官方规范、官方文档、技术报告、论文、权威工程博客和一线产品文档。本文把结论区分为“事实”“作者观点”和“本文推断”。

## 核心结论

1. **事实：MCP 是 host / client / server 之间的开放协议，不是模型能力、agent runtime 或某个工具市场。** 最新稳定规范是 `2025-11-25`，协议基于 JSON-RPC 2.0、状态化 lifecycle、版本与能力协商，并把服务器能力抽象为 `tools`、`resources`、`prompts` 等 primitives。来源：[MCP Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)、[MCP Architecture](https://modelcontextprotocol.io/specification/2025-11-25/architecture)、[MCP Versioning](https://modelcontextprotocol.io/docs/learn/versioning)。
2. **事实 + 作者观点：MCP 解决的是“每个 AI 应用都要为每个外部系统写一套连接器”的碎片化问题。** Anthropic 在 2024-11-25 首发公告中把背景归因于数据孤岛和 legacy systems；官方架构文档也明确 MCP 只关注 context exchange，不规定 LLM 应用如何规划、记忆或使用上下文。来源：[Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)、[MCP Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)。
3. **事实 + 本文推断：MCP 的主要工程价值在“标准化连接层”，不是自动带来安全、可靠和好用。** 官方安全文档要求输入校验、访问控制、rate limit、敏感操作确认和审计；2025-2026 年多篇论文与安全博客把 tool poisoning、descriptor-level manipulation、STDIO command execution、OAuth confused deputy、SSRF、session hijacking 等列为主要风险。来源：[MCP Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)、[MCP Tools security considerations](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)、[Hou et al. 2025](https://arxiv.org/abs/2503.23278)、[Huang et al. 2026](https://arxiv.org/abs/2603.22489)、[Jamshidi et al. 2025/2026](https://arxiv.org/abs/2512.06556)、[OX Security 2026](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-technical-deep-dive/)。
4. **事实：不同客户端对 MCP 的支持是不完全一致的。** MCP 规范支持 tools、resources、prompts、sampling、elicitation 等能力；但 GitHub Copilot cloud agent 文档明确当前只支持 MCP tools，不支持 resources / prompts，且配置后可自主使用工具；Claude Code 与 VS Code 则暴露资源、prompt、workspace/user 配置、工具搜索等不同体验。来源：[GitHub Copilot cloud agent MCP](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/extend-cloud-agent-with-mcp)、[Claude Code MCP docs](https://code.claude.com/docs/en/mcp)、[VS Code MCP servers](https://code.visualstudio.com/docs/copilot/customization/mcp-servers)。
5. **本文推断：把 MCP 用进项目时，最小可 review slice 应该从“只读、低权限、少工具、可观测、可关闭”的本地或内部 server 开始。** 先验证工具 schema、权限边界、输出大小、错误语义、日志和 host 行为，再考虑远程 HTTP、OAuth、写入工具、registry / marketplace 分发。来源：[MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector)、[MCP Client Best Practices](https://modelcontextprotocol.io/docs/develop/clients/client-best-practices)、[Srinivasan 2026](https://arxiv.org/abs/2603.13417)、[MCP Registry](https://modelcontextprotocol.io/registry/about)。

## 是什么

### 工作定义

本文采用这个定义：

```text
MCP = AI host 与外部 context / tools / workflow 能力之间的标准化连接协议。
```

更具体地说：

```text
MCP Host      = 使用 LLM 的应用，例如 Claude Code、Claude Desktop、VS Code、Copilot、Cursor、Codex 类客户端。
MCP Client    = host 内部为每个 server 维护的一条协议连接，通常一个 client 对一个 server。
MCP Server    = 暴露资源、工具、prompt 或其它能力的进程/服务。
MCP Transport = 承载 JSON-RPC 消息的通道，例如 stdio 或 Streamable HTTP。
```

官方规范描述的是 client-host-server 架构：host 负责创建和管理多个 client，client 与 server 建立隔离的状态化会话；server 提供 focused responsibilities，可以是本地进程，也可以是远程服务。来源：[MCP Architecture](https://modelcontextprotocol.io/specification/2025-11-25/architecture)。

### 核心协议层

**Base protocol。** MCP 使用 JSON-RPC 2.0 消息，定义 request、response、notification，并要求 lifecycle management。初始化阶段由 client 发送 `initialize`，双方协商 protocol version、capabilities 和 implementation information；之后进入 operation，最后 shutdown。来源：[Base Protocol Overview](https://modelcontextprotocol.io/specification/2025-11-25/basic/index)、[Lifecycle](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)。

**Transport layer。** 最新规范定义两个标准传输：

- `stdio`：client 启动 server 子进程，通过 stdin/stdout 交换 JSON-RPC；server 不能往 stdout 写非 MCP 消息，日志应走 stderr。
- `Streamable HTTP`：server 作为独立 HTTP 服务，支持 POST / GET，可用 SSE 流式返回；远程场景通常需要认证、Origin 校验和 session 管理。

来源：[Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、[Debugging](https://modelcontextprotocol.io/docs/tools/debugging)。

**Server primitives。**

- `tools`：模型可调用的动作，例如查询数据库、调用 API、执行计算。工具需要 `name`、`description`、`inputSchema`，可选 `outputSchema`，调用方法是 `tools/list` 和 `tools/call`。来源：[Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)。
- `resources`：提供给模型或 host 的上下文数据，例如文件、数据库 schema、应用状态。资源用 URI 标识，通常由应用或用户选择是否加入上下文。来源：[Resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources)。
- `prompts`：server 暴露的 prompt templates，通常用户显式选择或以 slash command 形式触发。来源：[Prompts](https://modelcontextprotocol.io/specification/2025-11-25/server/prompts)。

**Client primitives。** MCP 也允许 server 请求 client 侧能力，例如 sampling（让 host LLM 生成）、elicitation（向用户要结构化输入）、roots（告诉 server 可访问的根目录）。这些能力必须通过 capability negotiation 明确声明。来源：[MCP Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)。

### 容易混淆的边界

**MCP 不是 function calling。**

Function calling 是模型 API 层的“模型输出某个函数调用请求”的机制；MCP 是 host/client/server 之间发现、描述和调用外部能力的协议。一个 host 可以把 MCP server 暴露的 tools 转换成模型 API 的 tool definitions，但这只是 host 的实现方式，不是 MCP 本身。来源：MCP `tools/list` / `tools/call` 的协议定义见 [Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)，OpenAI 对 remote MCP 的接入见 [OpenAI Connectors and MCP servers](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)。

**MCP 不是 RAG。**

RAG 主要是检索资料并塞入上下文；MCP 的 `resources` 可以承载 RAG 类上下文，但 MCP 同时包含 `tools`、`prompts`、sampling、elicitation、authorization、transport 等连接层能力。来源：[MCP Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)。

**MCP 不是 agent runtime。**

MCP 不定义 agent 的 planning、task decomposition、memory、approval policy、sandbox、review gate、CI 或 eval。官方架构文档明确 MCP focuses solely on context exchange，不规定 AI application 如何使用 LLM 或管理上下文。本文推断：MCP 是 runtime 的连接层，而不是 runtime 总体。来源：[MCP Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)。

**MCP server 不是 plugin。**

MCP server 是提供协议能力的进程或远程服务；plugin 可以打包和分发 MCP server、skills、hooks 或配置。Claude Code 文档就把 plugin MCP servers 描述为通过 plugin 自动安装和启动的 MCP servers。来源：[Claude Code MCP docs](https://code.claude.com/docs/en/mcp)。

**MCP Registry 不是安全背书。**

官方 MCP Registry 是公开 server metadata 仓库，当前仍是 preview；它做 namespace authentication 和 metadata hosting，但安全扫描委托给底层包 registry 和下游 aggregator / marketplace。来源：[MCP Registry](https://modelcontextprotocol.io/registry/about)。

**MCP resources 不是“server 能看完整对话”。**

官方设计原则之一是 server 不应读取完整对话，也不应看到其它 server；host 负责控制给每个 server 的上下文。来源：[MCP Architecture](https://modelcontextprotocol.io/specification/2025-11-25/architecture)。

## 为什么

### 出现背景

Anthropic 在 2024-11-25 发布 MCP 时指出：AI assistant 能力提升很快，但仍被隔离在数据孤岛和 legacy systems 之外；每接一个数据源都要 custom implementation，导致 connected systems 难以规模化。MCP 的设计目标是用一个开放标准替代碎片化集成。来源：[Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)。

从 agent harness 角度看，MCP 出现的背景是：

- LLM 应用从 chat 扩展到 coding agent、desktop assistant、IDE、自动化 workflow。
- agent 需要读 issue、PR、CI、监控、数据库、文档、设计稿、浏览器和本地文件。
- 工具越来越多，单个产品自建 connector 成本高。
- 同一个 connector 希望在 Claude、VS Code、Copilot、OpenAI Responses API、Cursor 等 host 间复用。
- 连接外部系统后，权限、认证、确认、日志、输出大小和错误恢复都成为 runtime 问题。

### 它解决的问题

**标准化工具和数据连接。** Server 只要实现 MCP primitives，多个 host 就可以通过相同协议发现和调用它。Anthropic 的首发公告把价值表述为用 single protocol 替代 fragmented integrations。来源：[Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)。

**降低 server 开发门槛。** 官方设计原则强调 server 应 easy to build，host 承担复杂 orchestration，server 聚焦明确能力。来源：[MCP Architecture](https://modelcontextprotocol.io/specification/2025-11-25/architecture)。

**支持组合多个 server。** 一个 host 可以连接多个 server，每个 client 与 server 建立 1:1 隔离连接；多个 server 可以组合，但边界由 host 管理。来源：[MCP Architecture](https://modelcontextprotocol.io/specification/2025-11-25/architecture)。

**把上下文和动作统一到可发现 primitives。** `tools/list`、`resources/list`、`prompts/list` 让 host 能动态发现能力；工具 schema 使用 JSON Schema，有利于校验和 UI 展示。来源：[Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)、[Resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources)、[Prompts](https://modelcontextprotocol.io/specification/2025-11-25/server/prompts)。

**为生态分发提供元数据层。** MCP Registry 提供公开 server metadata、DNS namespace verification、REST API 和标准安装/配置说明；但它不是私有 server 管理方案，私有组织应自建私有 registry 或内部分发。来源：[MCP Registry](https://modelcontextprotocol.io/registry/about)。

### 适用场景

MCP 适合：

- **读多写少的上下文接入**：内部文档、repo 知识、API 文档、数据库 schema、feature flags、日志和监控。
- **低风险工具调用**：查询 issue、列 PR、读 CI 状态、运行本地只读分析命令。
- **需要跨 host 复用的 connector**：同一 GitHub / Jira / Sentry / Postgres / docs server 想给多个 AI 客户端使用。
- **需要把工具描述和 schema 暴露给模型选择的场景**：host 能基于 `tools/list` 动态选择工具。
- **需要用户中途补充信息的 workflow**：server 可通过 elicitation 请求结构化输入，前提是 host 支持。来源：[Claude Code MCP elicitation](https://code.claude.com/docs/en/mcp)、[MCP Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)。

### 不适用场景

MCP 不适合：

- **只有一个固定调用点的普通后端集成。** 如果服务 A 永远以确定性方式调用服务 B，直接 API / SDK 往往更简单。
- **高权限写入默认开放。** 例如生产数据库写入、部署、转账、删除资源，不应先从 MCP 暴露给 agent 自主调用。官方工具安全建议要求敏感操作确认、访问控制和审计。来源：[Tools security considerations](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)。
- **无法隔离凭据和权限的个人脚本。** 如果 token、cookie、私钥和本机路径散落在配置里，MCP 只会扩大暴露面。
- **无法承受延迟的 tight loop。** 远程 MCP 叠加模型决策、工具发现、HTTP、认证和工具执行，可能比直接本地函数慢。生产化论文把 timeout budgeting 列为缺口之一。来源：[Srinivasan 2026](https://arxiv.org/abs/2603.13417)。
- **把所有工具一次性塞给模型的超大工具集。** 官方 client best practices 明确指出，几十个 server、上百个工具时，naive loading 会消耗大量上下文并降低性能，应考虑 progressive discovery。来源：[MCP Client Best Practices](https://modelcontextprotocol.io/docs/develop/clients/client-best-practices)。

### 主要 trade-off

**互操作性 vs 最小权限。** 标准协议降低接入成本，但也让工具更容易被更多 host 调用。每个 server 都要设计 scope、credential、tool allowlist 和确认策略。

**动态发现 vs metadata 攻击面。** `description`、schema、resource metadata 会进入模型上下文，安全论文将 tool poisoning、shadowing、rug pull 归为 descriptor-level manipulation。来源：[Jamshidi et al. 2025/2026](https://arxiv.org/abs/2512.06556)。

**stdio 简单 vs 本地命令执行风险。** stdio 的设计就是 client 启动 server 子进程；如果产品把 command/args 暴露给不可信输入，就可能变成任意命令执行。OX Security 2026 的案例属于这种实现风险。来源：[Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、[OX Security 2026](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-technical-deep-dive/)。

**远程 HTTP 可共享 vs 认证复杂度。** Streamable HTTP 支持远程 server 和标准 HTTP auth，但必须处理 OAuth、Origin、session hijacking、SSRF、token audience 等问题。来源：[Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)、[Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)。

**生态 registry vs 供应链治理。** Registry 有助于发现 server，但官方也说明它不扫描实际 server 代码，安全扫描依赖底层包 registry 和下游 aggregator。来源：[MCP Registry](https://modelcontextprotocol.io/registry/about)。

**标准能力 vs 客户端差异。** 规范支持 resources 和 prompts，但 GitHub Copilot cloud agent 当前只支持 tools；不同 host 的 UI、确认、权限、输出限制、tool search 都不一样。来源：[GitHub Copilot cloud agent MCP](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/extend-cloud-agent-with-mcp)、[Claude Code MCP docs](https://code.claude.com/docs/en/mcp)。

## 怎么做

### 实践步骤

1. **定义 use case 和 threat model。** 先写清楚 server 要让 agent 读什么、做什么、不能做什么。建议从只读开始，列出资源范围、工具范围、凭据来源、数据敏感级别和失败影响。
2. **选择 host 和 transport。** 本地开发优先 `stdio`；团队/企业共享优先 `Streamable HTTP`。远程 HTTP 要设计 OAuth / token、Origin 校验、session、rate limit 和日志。来源：[Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、[Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)。
3. **选择 primitives。** 查询和动作放 `tools`；长期上下文放 `resources`；用户可显式触发的任务模板放 `prompts`。不要把所有能力都做成 tool。
4. **收窄 schema 和输出。** 工具参数用 JSON Schema，输入校验在 server 端做；输出尽量结构化、分页、可截断。工具执行错误用 `isError: true` 返回可修复反馈，协议错误只用于 malformed request / unknown tool 等。来源：[Tools error handling](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)。
5. **实现 server。** 优先使用官方 SDK 或成熟框架。Python 文档用 `FastMCP`，TypeScript / Java / Kotlin / C# / Rust 也有 SDK 路径。来源：[Build an MCP server](https://modelcontextprotocol.io/docs/develop/build-server)。
6. **用 MCP Inspector 验证协议层。** 验证连接、capability negotiation、`tools/list`、`tools/call`、resources、prompts、invalid inputs、并发和错误处理。来源：[MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector)。
7. **接入真实 host 前做安全 review。** 检查 secrets、绝对路径、命令 allowlist、tool allowlist、敏感操作确认、日志、超时、rate limit、输出清洗。
8. **在 host 中做最小集成。** 只启用一个 server、少量 tools；记录一次真实任务从用户请求到 tool call 到结果的完整链路。
9. **做 eval / regression。** 设计 5-10 个任务：应该调用、应该拒绝、权限不足、错误输入、大输出、恶意 resource、工具名称相似、token 缺失等。
10. **再考虑远程化和分发。** 只有当本地 server 的权限、日志、错误和输出稳定后，再上 Streamable HTTP、OAuth、内部 registry 或 marketplace。

### 最小例子：给本仓库做一个只读 notes server

目标：让 agent 能读取本仓库 README，并搜索 `docs/` 中包含某个关键词的 Markdown 文件。这个 server 只读、无外部 token、无写入工具，适合做第一轮 MCP 实验。

```python
# examples/mcp-harness-notes/server.py
from pathlib import Path

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("harness-notes")

ROOT = Path(__file__).resolve().parents[2]
DOCS = ROOT / "docs"


@mcp.resource("notes://readme")
def read_project_readme() -> str:
    """Read the repository entry README."""
    return (ROOT / "README.md").read_text(encoding="utf-8")


@mcp.tool()
def search_docs(keyword: str, limit: int = 10) -> list[str]:
    """Find Markdown files under docs/ that contain the keyword."""
    keyword = keyword.strip()
    if not keyword:
        return []

    matches: list[str] = []
    for path in DOCS.rglob("*.md"):
        text = path.read_text(encoding="utf-8", errors="ignore")
        if keyword.lower() in text.lower():
            matches.append(str(path.relative_to(ROOT)))
        if len(matches) >= limit:
            break
    return matches


if __name__ == "__main__":
    mcp.run()
```

示例本地配置：

```json
{
  "mcpServers": {
    "harness-notes": {
      "command": "uv",
      "args": [
        "--directory",
        "examples/mcp-harness-notes",
        "run",
        "server.py"
      ]
    }
  }
}
```

验收标准：

- `npx @modelcontextprotocol/inspector uv --directory examples/mcp-harness-notes run server.py` 可以连上。
- `resources/list` 能看到 `notes://readme`，`resources/read` 能返回 README。
- `tools/list` 能看到 `search_docs`，schema 只接受 `keyword` 和 `limit`。
- `search_docs("MCP")` 只返回文件路径，不返回大段正文。
- server 不读取 `docs/` 之外的正文，不写文件，不需要 token。
- stdio server 不向 stdout 打日志。

### 常见实现路径

**路径 1：个人本地 stdio server。**

适合本地开发、coding agent 辅助、文件系统/浏览器/测试工具。配置通常放在用户级或项目级配置里。优点是启动简单、无网络；缺点是依赖本机路径、环境变量和命令安全。来源：[VS Code MCP servers](https://code.visualstudio.com/docs/copilot/customization/mcp-servers)、[Claude Code MCP docs](https://code.claude.com/docs/en/mcp)。

**路径 2：团队共享 Streamable HTTP server。**

适合公司内部 Jira、Sentry、GitHub Enterprise、文档库、数据库只读 gateway。优点是统一部署、统一审计；缺点是 OAuth、token audience、SSRF、session、rate limit 和网络边界更复杂。来源：[Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、[Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)。

**路径 3：API adapter / proxy server。**

把已有 REST / GraphQL / CLI / SDK 包一层 MCP。适合已有系统很多但不想改后端的情况。风险是 proxy 容易变成过宽权限的 confused deputy，尤其当一个 OAuth client 代表多个 MCP clients 时。来源：[Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)。

**路径 4：项目级 MCP 配置模板。**

项目只提交 `servers.example.json`、`.vscode/mcp.json` 示例或说明，不提交真实 token、本机绝对路径和高权限写入工具。这个仓库的 `docs/coding-agents/agent/mcp/` 就属于这类可复制的 MCP 配置入口，不放调研正文。

**路径 5：host 侧 progressive discovery / tool search。**

当工具很多时，不要把所有 tool definitions 一次性塞进模型上下文。官方 client best practices 建议使用 `search_tools` 类 meta-tool，按需把完整工具定义加载进上下文。来源：[MCP Client Best Practices](https://modelcontextprotocol.io/docs/develop/clients/client-best-practices)。

### 多客户端 compatibility matrix

访问日期：2026-05-31。MCP 客户端能力变化很快，下表只记录官方文档明确写出的能力；没有明确写出的项不推断为“支持”。

| Host / Client | 适合用途 | 配置位置 / 启用方式 | Transport | 支持能力 | Approval / trust | Secrets / auth | 主要限制 |
|---|---|---|---|---|---|---|---|
| Claude Code | 本地 coding agent 连接 issue、监控、数据库、浏览器、内部 API | `claude mcp add`；local / project / user 三个 scope；project scope 写入项目根目录 `.mcp.json`；`/mcp` 管理状态 | remote HTTP 推荐；SSE deprecated；stdio；WebSocket | tools、resources、prompts as commands、elicitation、dynamic tool updates、Tool Search、plugin-provided MCP servers | project-scoped `.mcp.json` server 使用前需要 approval；HTTP/SSE 自动重连；MCP 输出超过 10,000 token 警告，默认最大 25,000 token | OAuth 通过 `/mcp`；支持 header auth；`.mcp.json` 支持环境变量展开 | 功能最全，但配置面也最大；需要治理 scope、tool search、output limit 和插件自动启动。来源：[Claude Code MCP](https://code.claude.com/docs/en/mcp)。 |
| VS Code + GitHub Copilot Agent Mode | IDE 内本地/远程开发，适合团队共享低权限 MCP 配置 | workspace `.vscode/mcp.json`；user profile `mcp.json`；Command Palette；Dev Container `customizations.vscode.mcp`；可从其它应用发现配置 | 本地 command / stdio；remote HTTP；dev container 内运行 | tools、resources、prompts、MCP Apps；工具可在 chat 输入区开关 | 首次启动 server 时要求 trust；可能要求确认单次 tool invocation；server disabled 后 tools/prompts/resources/apps 都不会进 chat | 官方建议不要 hardcode API key，用 input variables 或 env files | macOS/Linux 可 sandbox 本地 stdio，Windows 暂不支持 sandbox；直接从 `mcp.json` 启动 server 时不会出现 trust prompt。来源：[VS Code MCP servers](https://code.visualstudio.com/docs/copilot/customization/mcp-servers)。 |
| GitHub Copilot cloud agent | GitHub 云端 coding agent 在 repo 任务中访问外部工具 | 仓库 Settings -> Copilot -> Cloud agent 中粘贴 JSON；组织/企业也可在 custom agents YAML frontmatter 配置 | `local`、`stdio`、`http`、`sse` | 只支持 MCP tools；不支持 resources 或 prompts | 配置后 agent 可自主使用工具，不会逐次请求 approval；官方强烈建议 allowlist read-only tools | 只有 `COPILOT_MCP_` 前缀的 Agents secrets / variables 会暴露给 MCP 配置；内置 GitHub MCP 默认使用只读当前 repo token | 不支持带 OAuth 的 remote MCP server；repository admin 配置，不是普通项目文件。来源：[GitHub Copilot cloud agent MCP](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/extend-cloud-agent-with-mcp)。 |
| Codex CLI / IDE extension | Codex 本地 CLI / IDE 扩展接入开发者工具、文档、浏览器、Figma 等 | `codex mcp add`；`~/.codex/config.toml`；trusted project 可用 `.codex/config.toml`；CLI 和 IDE extension 共享配置；TUI 用 `/mcp` 查看 | STDIO；Streamable HTTP | 官方明确：stdio、HTTP、server instructions、tool allowlist/denylist、per-server/per-tool approval mode；Codex 还可作为 experimental MCP server 暴露本地 Codex engine | `default_tools_approval_mode` 和 `tools.<tool>.approval_mode` 支持 `auto` / `prompt` / `approve`；可用 `enabled=false` 禁用 server | HTTP 支持 bearer token env var、static headers、env HTTP headers、OAuth login；stdio 支持 env、env_vars、cwd | streamable HTTP remote placement 尚未实现；Codex-as-MCP-server 接口标注 experimental。来源：[Codex MCP](https://developers.openai.com/codex/mcp)、[Codex config reference](https://developers.openai.com/codex/config-reference)、[Codex MCP Server Interface](https://github.com/openai/codex/blob/main/codex-rs/docs/codex_mcp_interface.md)。 |
| OpenAI Responses API / ChatGPT MCP integrations | 产品后端或 ChatGPT app 接入 remote MCP / connectors | API `tools` 数组使用 `{"type":"mcp"}`；ChatGPT custom remote MCP 在 ChatGPT settings / Apps & Connectors 配置 | remote Streamable HTTP 或 HTTP/SSE；API 不适合本地 stdio | 主要作为 API tool 调用 remote MCP tools；返回 `mcp_list_tools`、`mcp_call`、`mcp_approval_request` 等 output items；支持 `allowed_tools` 过滤 | Responses API 默认要求 approval 后才向 connector / remote MCP server 共享数据；可配置 `require_approval`；deep research API 场景要求 no approval | OAuth access token 通过 `authorization` 传入，API 不存储该值；ChatGPT 自定义 MCP 推荐 OAuth / CIMD | 这是 API/ChatGPT 集成路径，不是本地 coding agent；remote MCP server 必须是公网可达或通过安全隧道暴露。来源：[OpenAI MCP and Connectors](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)、[OpenAI MCP server guide](https://developers.openai.com/api/docs/mcp)。 |

项目落地时，不要写一份“通用 MCP 配置”假设所有客户端都能吃。更稳的做法是维护一个 compatibility matrix，并为每个 host 单独写最小配置：

| 决策项 | Claude Code | VS Code | GitHub Copilot cloud agent | Codex | OpenAI API / ChatGPT |
|---|---|---|---|---|---|
| 项目内可提交配置 | `.mcp.json` project scope | `.vscode/mcp.json` | 不直接提交到 repo；GitHub settings / custom agent 配置 | `.codex/config.toml` 仅 trusted project；更常见是用户级 `~/.codex/config.toml` | 不提交 client 配置；在应用代码或 ChatGPT app 配置中声明 |
| 默认推荐权限 | local scope 或只读 project scope | workspace 只读 server，必要时 sandbox | 只读 allowlisted tools | `enabled_tools` + `prompt` approval | `allowed_tools` + approval required |
| secrets 策略 | env expansion / OAuth，不提交真实值 | input variables / env files | `COPILOT_MCP_` Agents secrets | bearer token env var / env headers / OAuth | 每次 API 请求传 `authorization`，服务端自行保管 token |
| first slice | 一个只读 docs/search server | 一个 workspace docs/search server | 一个只读 tool allowlist | 一个只读 stdio 或 HTTP server | 一个 remote read-only MCP server + explicit approvals |

### 验证方法

**协议验证。**

- 用 MCP Inspector 连接 server。
- 检查 initialization、capabilities、protocol version。
- 检查 `tools/list`、`resources/list`、`prompts/list`。
- 调用正常输入、缺失参数、非法参数、大输出、超时、并发调用。

**安全验证。**

- 检查 server 是否能访问超出预期的文件、网络或环境变量。
- 检查工具参数是否有 allowlist，而不是把 shell command、SQL、URL、path 原样交给用户输入。
- 检查敏感工具是否需要 host 或用户确认。
- 检查日志是否包含 token、cookie、PII。
- 对 tool description、resource 内容、外部网页内容做 prompt injection 测试。
- 用 STRIDE / DREAD 轻量威胁建模覆盖 host/client、LLM、server、external data store、authorization server。来源：[Huang et al. 2026](https://arxiv.org/abs/2603.22489)。

**使用验证。**

- 让目标 host 完成一组固定任务：应该调用工具、应该不用工具、应该拒绝、应该请求更多信息、应该处理错误。
- 记录 tool call 轨迹、输入参数、输出大小、耗时和最终回答。
- 比较有 MCP / 无 MCP 的成功率、延迟、token、人工纠错次数。

**生产验证。**

- 超时预算：每个工具调用的 timeout、重试、fallback。
- 错误语义：哪些错误给模型自修复，哪些直接中断。
- 观测：tool call audit log、trace id、用户、server、scope、结果大小。
- 回滚：禁用 server、撤销 token、移除工具、降级到手工流程。
- 权限：按项目、用户、环境、工具级别做 allowlist。

## 关键概念和术语

| 术语 | 定义 | 注意点 |
|---|---|---|
| Host | 使用 LLM 并管理 MCP clients 的应用 | 安全策略、用户确认、上下文聚合通常在 host |
| Client | host 内部连接某个 MCP server 的协议对象 | 通常一个 client 对一个 server |
| Server | 暴露 MCP primitives 的进程或远程服务 | 可以本地 stdio，也可以远程 HTTP |
| Primitive | MCP 中可发现的能力类型 | server primitives 包括 tools/resources/prompts |
| Tool | 可调用动作 | 要有 schema、权限、错误语义和审计 |
| Resource | 可读取上下文 | 资源不是默认加入全部上下文，host 决定如何使用 |
| Prompt | 可复用 prompt template | 通常用户控制，不应替代所有项目规则 |
| Sampling | server 请求 host LLM 生成 | 让 server 保持模型无关，但需要 host 支持 |
| Elicitation | server 向用户请求结构化输入 | 用于缺少信息、确认或认证流程 |
| Roots | client 告知 server 可访问根目录 | 可用于限制文件范围 |
| Capability negotiation | 初始化时双方声明能力 | 不要假设所有 host 都支持全部规范能力 |
| stdio transport | client 启动本地子进程，通过 stdin/stdout 通信 | stdout 只能写 MCP 消息；不可信 command/args 是高风险 |
| Streamable HTTP | 远程 HTTP transport，可选 SSE | 要处理 Origin、OAuth、session、SSRF |
| Tool poisoning | 恶意工具 metadata 影响模型选择和行为 | 属于 prompt injection 的 MCP 变体 |
| Progressive discovery | host 按需加载工具定义 | 用于工具数量大、上下文预算紧张的场景 |
| Registry | MCP server metadata 目录 | 不是代码托管，也不是安全扫描承诺 |

## 常见坑

1. **把 `print()` 写到 stdout。** stdio server 的 stdout 是协议通道，非 JSON-RPC 输出会破坏连接。来源：[Build an MCP server](https://modelcontextprotocol.io/docs/develop/build-server)。
2. **在配置里使用相对路径。** 客户端启动 stdio server 时工作目录可能不确定，官方 debugging 文档建议用绝对路径。来源：[Debugging](https://modelcontextprotocol.io/docs/tools/debugging)。
3. **把用户输入拼成 shell command、SQL 或 URL。** 这是本地 RCE、SSRF 和数据泄漏的常见来源。来源：[Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)、[OX Security 2026](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-technical-deep-dive/)。
4. **工具过宽。** `tools: ["*"]`、全 repo PAT、生产数据库写权限会让 agent 的错误决策变成真实事故。GitHub 文档建议细粒度 token 和工具选择。来源：[GitHub Copilot cloud agent MCP](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/extend-cloud-agent-with-mcp)。
5. **假设 host 一定会让用户确认。** MCP 规范建议 human-in-the-loop，但不同 host 行为不同；GitHub Copilot cloud agent 文档明确配置后会自主使用 MCP tools，不会每次请求 approval。来源：[Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)、[GitHub Copilot cloud agent MCP](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/extend-cloud-agent-with-mcp)。
6. **把 registry 当可信安装源。** 官方 registry 当前 preview，并且安全扫描依赖外部 ecosystem。来源：[MCP Registry](https://modelcontextprotocol.io/registry/about)。
7. **忽略 output size。** 大输出会浪费上下文，甚至截断关键信息；应分页、摘要、结构化，并给 host 可控限制。Claude Code 文档也提到 MCP output limit 和 tool search。来源：[Claude Code MCP docs](https://code.claude.com/docs/en/mcp)。
8. **所有工具一次性进入上下文。** 工具多时要用 progressive discovery / tool search。来源：[MCP Client Best Practices](https://modelcontextprotocol.io/docs/develop/clients/client-best-practices)。
9. **错误语义不清。** 模型可修复的业务错误应放在 tool execution error；协议错误用于 malformed request 等结构问题。来源：[Tools error handling](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)。
10. **忽略版本协商。** MCP 使用 `YYYY-MM-DD` 版本标识，初始化时必须协商单一版本；当前版本是 `2025-11-25`。来源：[Versioning](https://modelcontextprotocol.io/docs/learn/versioning)。

## 资料综述

### 官方规范和文档

官方资料最适合回答“协议到底是什么”和“实现必须遵守什么”。`2025-11-25` 规范给出 JSON-RPC、lifecycle、transport、authorization、tools/resources/prompts、client features 和 schema 约束。`Architecture overview` 更适合快速建立 mental model。`Security Best Practices` 和 `Client Best Practices` 则说明官方已经承认 MCP 的安全和上下文规模问题不是协议本身能自动解决的。

优先级：最高。凡是 normative behavior，以官方规范为准。

### 产品文档和一线实现

Claude Code、VS Code、GitHub Copilot cloud agent、OpenAI Responses API 都说明 MCP 已经进入主流 agent 产品，但也暴露出客户端差异：

- Claude Code 支持 remote HTTP / stdio、项目/用户/local scope、resources 引用、prompts as commands、tool search 和 elicitation。
- VS Code 支持 user profile / workspace `.vscode/mcp.json`，并提醒不要 hardcode API key。
- GitHub Copilot cloud agent 当前只支持 tools，不支持 resources/prompts；配置后 Copilot 可自主使用工具。
- Codex CLI / IDE extension 支持共享 `config.toml` MCP 配置、stdio / Streamable HTTP、tool allowlist/denylist、server instructions 和 tool approval mode。
- OpenAI Responses API 把 remote MCP server 作为 API tool 接入，并有 approval request / response 流程。

优先级：高。用于判断“具体 host 实际怎么做”，但不能把某个 host 的限制误认为 MCP 规范限制。

### 论文

**Hou et al. 2025** 提供 MCP landscape 和安全威胁分类，适合做完整风险地图。它把 MCP server 生命周期拆成 creation、deployment、operation、maintenance，并给出不同攻击者类型和威胁场景。来源：[arXiv:2503.23278](https://arxiv.org/abs/2503.23278)。

**Srinivasan 2026** 从生产部署角度提出三个缺口：identity propagation、adaptive tool budgeting、structured error semantics。它的价值在于提醒：MCP 标准化的是连接协议，生产可靠性还需要 broker、timeout、error、observability 等 runtime 设施。来源：[arXiv:2603.13417](https://arxiv.org/abs/2603.13417)。

**Huang et al. 2026** 用 STRIDE / DREAD 分析 MCP 组件，并把 tool poisoning 作为主要 client-side 风险之一，对七个主要 MCP clients 做防护比较。来源：[arXiv:2603.22489](https://arxiv.org/abs/2603.22489)。

**Jamshidi et al. 2025/2026** 聚焦 descriptor-level manipulation，把攻击分成 Tool Poisoning、Shadowing、Rug Pull，并提出 descriptor integrity verification、semantic vetting、runtime guardrails 等防护。来源：[arXiv:2512.06556](https://arxiv.org/abs/2512.06556)。

优先级：中高。论文能补足官方文档不会展开的攻击模型和生产缺口，但很多结果仍是 preprint / controlled evaluation，应避免当成最终行业共识。

### 安全研究博客

OX Security 2026 的技术深挖把 STDIO command execution 与真实平台 RCE 串起来，强调当 MCP configuration 的 command/args 接受不可信输入时，会出现系统性风险。这个资料适合做工程红线：不要让用户或模型直接控制 stdio command/args；不要用 ad hoc string filtering 当安全边界。来源：[OX Security 2026](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-technical-deep-dive/)。

优先级：中高。它不是规范来源，但对真实 exploit path 有工程价值。

### 冲突点和判断

**冲突 1：MCP 是“secure two-way connections”还是新攻击面？**

- Anthropic 首发公告称 MCP enables developers to build secure, two-way connections。来源：[Anthropic 2024](https://www.anthropic.com/news/model-context-protocol)。
- 官方安全文档、论文和 OX Security 都列出大量风险。来源：[Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)、[Huang et al. 2026](https://arxiv.org/abs/2603.22489)、[OX Security 2026](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-technical-deep-dive/)。

判断：两者不完全冲突。MCP 可以提供比 ad hoc connector 更标准的安全接口，但不会自动实现安全。本文更信官方安全文档和实证安全研究对风险的描述，因为它们具体到攻击面和 mitigation。

**冲突 2：MCP 规范支持多 primitives，但某些产品只支持 tools。**

- 规范支持 tools、resources、prompts 等。来源：[Specification](https://modelcontextprotocol.io/specification/2025-11-25)。
- GitHub Copilot cloud agent 文档写明当前只支持 tools。来源：[GitHub Copilot cloud agent MCP](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/extend-cloud-agent-with-mcp)。

判断：这是实现子集，不是规范冲突。项目落地时必须按目标 host 写 compatibility matrix。

**冲突 3：STDIO 是标准 transport，但安全博客把 STDIO 作为 RCE 根源。**

- 规范定义 stdio 的行为就是 client 启动 server 子进程。来源：[Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)。
- OX Security 把不可信 MCP configuration 到 subprocess execution 的路径称为系统性漏洞。来源：[OX Security 2026](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-technical-deep-dive/)。

判断：规范描述的是能力；漏洞通常来自产品把 command/args 开给不可信输入，或让 agent 能修改 MCP config。本文更倾向把它定义为“高危实现模式”，而不是简单说 stdio transport 本身错误。

**冲突 4：Registry 提供信任，还是不提供安全保证？**

- Registry 提供 namespace authentication。来源：[MCP Registry](https://modelcontextprotocol.io/registry/about)。
- 同一文档也说 security scanning 依赖 package registries 和 downstream aggregators。

判断：Registry 解决“名字是谁的”，不解决“代码是否安全”。安装 server 仍需供应链 review。

## 参考资料

| 标题 | 作者或机构 | 发布日期 | 类型 | 链接 | 访问日期 |
|---|---|---:|---|---|---:|
| Introducing the Model Context Protocol | Anthropic | 2024-11-25 | blog | https://www.anthropic.com/news/model-context-protocol | 2026-05-31 |
| Model Context Protocol Specification 2025-11-25 | Model Context Protocol project | 2025-11-25 | docs | https://modelcontextprotocol.io/specification/2025-11-25 | 2026-05-31 |
| Architecture - Model Context Protocol | Model Context Protocol project | 2025-11-25 | docs | https://modelcontextprotocol.io/specification/2025-11-25/architecture | 2026-05-31 |
| Architecture overview | Model Context Protocol project | 未标明 | docs | https://modelcontextprotocol.io/docs/learn/architecture | 2026-05-31 |
| Versioning | Model Context Protocol project | 未标明 | docs | https://modelcontextprotocol.io/docs/learn/versioning | 2026-05-31 |
| Transports | Model Context Protocol project | 2025-11-25 | docs | https://modelcontextprotocol.io/specification/2025-11-25/basic/transports | 2026-05-31 |
| Authorization | Model Context Protocol project | 2025-11-25 | docs | https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization | 2026-05-31 |
| Tools | Model Context Protocol project | 2025-11-25 | docs | https://modelcontextprotocol.io/specification/2025-11-25/server/tools | 2026-05-31 |
| Resources | Model Context Protocol project | 2025-11-25 | docs | https://modelcontextprotocol.io/specification/2025-11-25/server/resources | 2026-05-31 |
| Prompts | Model Context Protocol project | 2025-11-25 | docs | https://modelcontextprotocol.io/specification/2025-11-25/server/prompts | 2026-05-31 |
| Build an MCP server | Model Context Protocol project | 未标明 | docs | https://modelcontextprotocol.io/docs/develop/build-server | 2026-05-31 |
| MCP Inspector | Model Context Protocol project | 未标明 | docs | https://modelcontextprotocol.io/docs/tools/inspector | 2026-05-31 |
| Debugging | Model Context Protocol project | 未标明 | docs | https://modelcontextprotocol.io/docs/tools/debugging | 2026-05-31 |
| Security Best Practices | Model Context Protocol project | 未标明 | docs | https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices | 2026-05-31 |
| Client Best Practices | Model Context Protocol project | 未标明 | docs | https://modelcontextprotocol.io/docs/develop/clients/client-best-practices | 2026-05-31 |
| The MCP Registry | Model Context Protocol project | 未标明，preview | docs | https://modelcontextprotocol.io/registry/about | 2026-05-31 |
| One Year of MCP: November 2025 Spec Release | MCP Core Maintainers | 2025-11-25 | blog | https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/ | 2026-05-31 |
| Connectors and MCP servers | OpenAI | 未标明 | docs | https://developers.openai.com/api/docs/guides/tools-connectors-mcp | 2026-05-31 |
| Model Context Protocol - Codex | OpenAI | 未标明 | docs | https://developers.openai.com/codex/mcp | 2026-05-31 |
| Configuration Reference - Codex | OpenAI | 未标明 | docs | https://developers.openai.com/codex/config-reference | 2026-05-31 |
| Docs MCP | OpenAI | 未标明 | docs | https://developers.openai.com/learn/docs-mcp | 2026-05-31 |
| Building MCP servers for ChatGPT Apps and API integrations | OpenAI | 未标明 | docs | https://developers.openai.com/api/docs/mcp | 2026-05-31 |
| Add and manage MCP servers in VS Code | Microsoft / VS Code | 2026-05-28 | docs | https://code.visualstudio.com/docs/copilot/customization/mcp-servers | 2026-05-31 |
| Connect Claude Code to tools via MCP | Anthropic | 未标明 | docs | https://code.claude.com/docs/en/mcp | 2026-05-31 |
| Connect agents to external tools | GitHub Docs | 未标明 | docs | https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/extend-cloud-agent-with-mcp | 2026-05-31 |
| Codex MCP Server Interface | OpenAI Codex repository | 未标明 | code | https://github.com/openai/codex/blob/main/codex-rs/docs/codex_mcp_interface.md | 2026-05-31 |
| Model Context Protocol (MCP): Landscape, Security Threats, and Future Research Directions | Xinyi Hou, Yanjie Zhao, Shenao Wang, Haoyu Wang | 2025-03-30；v3 2025-10-07 | paper | https://arxiv.org/abs/2503.23278 | 2026-05-31 |
| Bridging Protocol and Production: Design Patterns for Deploying AI Agents with Model Context Protocol | Vasundra Srinivasan | 2026-03-12 | paper | https://arxiv.org/abs/2603.13417 | 2026-05-31 |
| Model Context Protocol Threat Modeling and Analyzing Vulnerabilities to Prompt Injection with Tool Poisoning | Charoes Huang, Xin Huang, Ngoc Phu Tran, Amin Milani Fard | 2026-03-23 | paper | https://arxiv.org/abs/2603.22489 | 2026-05-31 |
| Semantic Attacks on Tool-Augmented LLMs: Securing the Model Context Protocol Against Descriptor-Level Manipulation | Saeid Jamshidi, Arghavan Moradi Dakhel, Kawser Wazed Nafi, Foutse Khomh | 2025-12-06；v2 2026-05-21 | paper | https://arxiv.org/abs/2512.06556 | 2026-05-31 |
| The Mother of All AI Supply Chains: Technical Deep Dive | Moshe Siman Tov Bustan, Mustafa Naamnih, Nir Zadok / OX Security | 2026-04-15 | blog | https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-technical-deep-dive/ | 2026-05-31 |

## 开放问题

1. **MCP governance 长期会如何演化？** `2025-11-25` changelog 已经 formalize governance、working groups、SDK tiering，但实际兼容性和变更节奏还需要继续观察。来源：[Key Changes](https://modelcontextprotocol.io/specification/2025-11-25/changelog)。
2. **Registry 从 preview 到 GA 后会承担多少安全责任？** 当前文档明确安全扫描委托给外部 ecosystem；企业是否会形成私有 registry / allowlist 的事实标准仍不确定。
3. **不同 host 的 confirmation model 会不会收敛？** 有的 host 每次请求 approval，有的 host 可以自主调用，安全体验差异会影响 MCP server 的默认权限设计。
4. **tool poisoning 的可用防护会落在 host、server、registry 还是第三方 security gateway？** 论文提出多层防御，但实际产品形态仍分散。
5. **MCP 与 skills、plugins、AGENTS.md、hooks 的边界会不会继续合并？** Claude Code 和 OpenAI / GitHub 生态已经把这些都放进 agent customization，但它们的治理对象不同。

## 下一步建议

1. **先做只读本地实验。** 在 `examples/mcp-harness-notes/` 做一个只读 server，暴露 `notes://readme` resource 和 `search_docs` tool，用 Inspector 验证。
2. **补一份项目级 MCP 配置模板。** 在 `docs/coding-agents/agent/mcp/` 放 `servers.example.json` 和安全边界说明，只写用途、权限、环境变量名和禁用方式，不写真实 token。
3. **做 MCP 安全 checklist。** 覆盖 tool allowlist、scope、secret、stdout、path、output size、prompt injection、approval、logs、timeout。
4. **做 host compatibility matrix。** 对 Claude Code、VS Code、Codex/OpenAI、GitHub Copilot 分别记录：支持哪些 primitives、配置位置、approval 行为、输出限制和 secrets 处理方式。
5. **把 MCP 放进 agent harness 分层图。** 建议定位为“tool/context connection layer”，与 skills（workflow package）、hooks（lifecycle guardrails）、AGENTS.md（repo rules）、sandbox（execution boundary）并列。

## 最值得先读的 3 篇资料

1. [Model Context Protocol Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)：先建立 host/client/server、data layer、transport layer、primitives 的 mental model。
2. [Model Context Protocol Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)：再看最新稳定规范，重点读 lifecycle、transports、tools、resources、authorization。
3. [MCP Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)：最后用安全视角校准，避免把“能连上”误认为“能安全上线”。

## 建议的最小实验或 reviewable slice

**Slice：`examples/mcp-harness-notes/` 只读 MCP server + `docs/coding-agents/agent/mcp/servers.example.json` 示例配置。**

范围：

- 一个 Python `FastMCP` server。
- 一个 `notes://readme` resource。
- 一个 `search_docs(keyword, limit)` tool。
- 不接外网、不需要 token、不写文件。
- 用 MCP Inspector 截图或日志验证。
- README 写清楚启用、禁用、权限、fallback。

Review 重点：

- server 是否只读。
- tool schema 是否收窄。
- 输出是否分页/限量。
- stdio 是否不污染 stdout。
- 配置是否不含真实 token 和本机绝对路径。
- host 行为是否记录清楚。
