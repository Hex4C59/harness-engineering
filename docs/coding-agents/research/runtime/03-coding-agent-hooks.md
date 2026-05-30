# Coding Agent Hooks：Claude Code 与 Codex CLI 的生命周期扩展点

调研日期：2026-05-30

## 这篇文档回答什么

你问的是 Claude Code CLI 和 Codex CLI 里现在都有的 `/hook` / `/hooks` 相关能力：

```text
它是什么？
有什么用？
怎么使用？
为什么它会成为 coding agent 工作流里的重要机制？
```

先纠正一个小命名：官方文档里更常见的是 **`/hooks`**，复数形式。很多人聊天时会口头说 `/hook`，但真正要打开的通常是 `/hooks` 菜单，或者要配置的是 `hooks` 这个 feature。

核心结论：

> Hooks 不是“再写一段 prompt”，而是在 coding agent 的生命周期事件上挂确定性程序。它把“希望模型每次都记得做的事”，下沉成 harness 层的自动检查、拦截、补充上下文、通知和审计。

换句话说，`AGENTS.md` / `CLAUDE.md` 是告诉 agent 应该怎么做；hooks 是在 agent 准备行动、行动之后、压缩上下文、结束回合等关键节点，自动跑一段你定义的程序来检查和干预。

## 来源说明

本文以 2026-05-30 可查资料为准，并在本机确认：

- Codex CLI：`codex-cli 0.135.0`
- Claude Code：`2.1.143 (Claude Code)`

| 来源 | 类型 | 主要价值 |
|---|---|---|
| [Claude Code Hooks reference](https://code.claude.com/docs/en/hooks) | Anthropic 官方文档 | Claude Code hooks 的事件、配置位置、输入输出协议、matcher、HTTP / MCP / prompt / agent hooks |
| [Claude Code hooks guide](https://code.claude.com/docs/en/hooks-guide) | Anthropic 官方指南 | 常见用法：通知、自动格式化、阻止危险命令、补充 compaction 后上下文 |
| [Claude Code settings](https://code.claude.com/docs/en/settings) | Anthropic 官方文档 | `~/.claude/settings.json`、`.claude/settings.json`、managed settings、`disableAllHooks` 等配置面 |
| [Claude Code best practices](https://code.claude.com/docs/en/best-practices) | Anthropic 官方实践 | 明确建议把“必须每次发生的事”放进 hooks，而不是只靠 `CLAUDE.md` |
| [Codex Hooks](https://developers.openai.com/codex/hooks) | OpenAI 官方文档 | Codex hooks 的事件、信任审核、`hooks.json`、inline `[hooks]`、plugin hooks |
| [Codex Config basics](https://developers.openai.com/codex/config-basic) | OpenAI 官方文档 | Codex 配置层、feature flag、项目信任对 `.codex/` hooks 的影响 |
| [Codex Advanced Configuration](https://developers.openai.com/codex/config-advanced) | OpenAI 官方文档 | Codex hooks 在配置层中的加载位置和 inline TOML 示例 |
| [Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely) | OpenAI 技术博客 | sandbox、approval、rules、telemetry 如何组成 agent 安全控制面 |
| [Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness) | OpenAI 技术博客 | Codex harness 包含 agent loop、tool execution、sandbox、MCP、skills 等运行时层 |
| [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop) | OpenAI 技术博客 | agent loop 如何管理上下文、工具、sandbox instructions 和 compaction |
| [AI Harness Engineering](https://arxiv.org/abs/2605.13357) | 论文 | 把软件工程 agent 能力解释为 model-harness-environment 系统，强调 tool access、observability、verification、permissions |

本文区分三类内容：

| 类型 | 含义 |
|---|---|
| 事实 | 官方文档、CLI help、本机版本检查中明确出现的机制 |
| 推断 | 从 Claude Code、Codex 和 harness engineering 资料中共同归纳出的设计含义 |
| 个人整理框架 | 面向个人 coding agent 工作流的使用建议 |

## Hooks 到底是什么

Hooks 可以理解成 coding agent runtime 的生命周期回调。

一个 hook 通常由三层组成：

| 层级 | 含义 | 例子 |
|---|---|---|
| Event | 在什么时候触发 | `PreToolUse`、`PostToolUse`、`UserPromptSubmit`、`Stop`、`SessionStart` |
| Matcher | 只匹配哪些事件实例 | 只匹配 `Bash`，只匹配 `Edit|Write`，只匹配 `startup|resume` |
| Handler | 触发后执行什么 | shell script、HTTP endpoint、MCP tool、prompt、agent |

一个典型流程是：

```text
用户提交 prompt
-> UserPromptSubmit hook
-> 模型决定调用工具
-> PreToolUse hook
-> permission / approval flow
-> 工具真正执行
-> PostToolUse hook
-> 模型继续推理
-> Stop hook
-> 用户看到结果
```

hook handler 会收到一份 JSON 输入，里面包含当前事件、cwd、session id、工具名、工具参数等上下文。handler 可以：

- 什么都不返回，让流程继续。
- 返回一段补充上下文，喂给 agent。
- 阻止一次工具调用。
- 改写某些工具输入。
- 批准或拒绝一次权限请求。
- 在回合结束前要求 agent 继续处理。
- 写日志、通知人、调用外部系统。

这就是它和普通 slash command 的区别：

| 机制 | 谁触发 | 主要作用 |
|---|---|---|
| Slash command | 人主动输入 | 触发一个人类明确选择的工作流 |
| Skill / command | 模型或人按需调用 | 提供知识、模板、流程 |
| Hook | runtime 在生命周期节点自动触发 | 强制执行检查、审计、通知、上下文补充 |
| Permission / sandbox | runtime / OS 强制执行 | 限制 agent 能做什么 |
| CI / tests | 外部工程系统触发 | 验证最终结果 |

hooks 更接近 “runtime guardrail”，不是 “prompt recipe”。

## 为什么 hooks 重要

从 harness engineering 的角度看，coding agent 真正难的地方不是“模型会不会写代码”，而是：

- 它有没有看到正确上下文。
- 它能不能安全调用工具。
- 它能不能从失败中得到可读反馈。
- 它会不会在没有验证时过早宣布完成。
- 它的行为能不能被追踪、审计和复盘。
- 高风险动作能不能被自动阻止或要求审批。

`AI Harness Engineering` 论文把这些责任拆成 task specification、context selection、tool access、observability、verification、permissions、intervention recording 等组件。hooks 正好是这些组件里最容易落地的一种接口：你不用改模型，也不用改整个 CLI，只要在生命周期事件上挂脚本，就能把部分经验沉淀成可执行约束。

典型用途可以分成六类。

### 1. 自动化重复动作

例如：

- 文件被改后自动跑 formatter。
- 生成代码后自动跑 lint。
- Claude / Codex 等待输入时发桌面通知。
- session start 时加载某些本地上下文。

这些动作不应该靠模型“记得”。它们应该每次都发生。

### 2. 阻止危险动作

例如：

- 阻止 `rm -rf`、`git push --force`、`terraform apply`。
- 阻止编辑 `.env`、`.git/`、锁文件、迁移目录。
- 阻止把疑似 API key 粘进 prompt。
- 阻止访问不该访问的 MCP tool。

注意：hook 不是完整安全边界。真正的边界仍然是 sandbox、权限、OS、容器、CI 和审计。但 hook 能在 agent 行动前加一层更懂项目语义的拦截。

### 3. 给 agent 增加可读反馈

Post hook 可以把工具输出整理成更适合 agent 使用的反馈：

- 测试失败后提取关键错误。
- 编译失败后定位最相关文件。
- 发现格式化失败后告诉 agent 下一步该修哪里。
- 在 Stop 阶段检查是否缺少验证证据。

这类 hook 的价值不是“代替模型思考”，而是把环境反馈变得更可读。

### 4. 保护上下文和任务状态

长会话里，compaction 很容易丢掉关键细节。hooks 可以在 `PreCompact` / `PostCompact` 附近做：

- 保存修改文件列表。
- 保存已运行测试和结果。
- 重新注入项目当前状态。
- 检查 summary 是否保留验收标准。

这和本仓库前面关于 plan/status 文档的结论一致：长期工作需要外部状态，不要只靠对话上下文。

### 5. 记录审计和 telemetry

hooks 可以把事件写到本地日志或外部系统：

- 用户 prompt。
- 工具调用。
- 权限请求。
- 被阻止的动作。
- session start / stop。
- subagent start / stop。

OpenAI 的 Codex 安全实践强调 agent-native telemetry：传统日志能告诉你进程和文件发生了什么，但 agent 日志还能帮助解释“为什么发生”。

### 6. 把团队经验产品化

当你发现自己第 5 次提醒 agent “不要改这个目录”“改完要跑这个脚本”“不要用这个命令”时，这就不是 prompt 问题了，而是 harness 问题。

把它写成 hook，团队里每个 agent session 都能复用。

## Claude Code 里的 hooks

Claude Code 的 hooks 已经相当完整。官方文档把 hooks 定义为在 Claude Code 生命周期中特定点自动执行的 shell commands、HTTP endpoints、MCP tools、LLM prompts 或 agent checks。

### 配置位置

常见位置：

| 位置 | 作用范围 | 是否适合提交 |
|---|---|---|
| `~/.claude/settings.json` | 当前用户所有项目 | 不适合 |
| `.claude/settings.json` | 当前项目 | 适合 |
| `.claude/settings.local.json` | 当前项目的个人配置 | 不适合 |
| managed settings | 组织级策略 | 由管理员管理 |
| plugin `hooks/hooks.json` | 插件启用时 | 随插件分发 |
| skill / agent frontmatter | skill 或 agent 活跃时 | 随组件分发 |

Claude 的 `/hooks` 菜单主要用于浏览和确认当前配置。官方 guide 明确提示：`/hooks` 菜单是只读的，要新增、修改或删除 hooks，需要编辑 settings JSON，或者让 Claude 帮你改配置。

### 常见事件

Claude Code 支持的事件很多，常用的是：

| Event | 触发点 | 常见用途 |
|---|---|---|
| `SessionStart` | session 开始或恢复 | 注入项目状态、检查环境 |
| `UserPromptSubmit` | 用户 prompt 进入模型前 | 扫描 secret、改写或阻止不合规输入 |
| `PreToolUse` | 工具调用前 | 阻止危险命令、限制敏感文件 |
| `PermissionRequest` | 出现权限请求时 | 自动批准低风险动作、拒绝高风险动作 |
| `PostToolUse` | 工具成功执行后 | 格式化、lint、整理输出 |
| `PostToolUseFailure` | 工具失败后 | 提取错误、给 agent 反馈 |
| `PostToolBatch` | 一批并行工具调用完成后 | 批量检查 |
| `Stop` | Claude 准备结束本轮回复时 | 检查是否完成验证 |
| `PreCompact` / `PostCompact` | 上下文压缩前后 | 保存或补回关键状态 |
| `Notification` | Claude 发通知时 | 桌面提醒 |
| `SubagentStart` / `SubagentStop` | subagent 启动和结束 | 给子任务加规则或汇总 |

### Handler 类型

Claude Code 支持五种 handler：

| Handler | 作用 |
|---|---|
| `command` | 执行本地命令，JSON 从 stdin 传入 |
| `http` | 把 JSON 作为 POST body 发给 HTTP endpoint |
| `mcp_tool` | 调用已连接 MCP server 的工具 |
| `prompt` | 用一个模型做轻量 yes/no 判断 |
| `agent` | 启动一个子 agent 做检查 |

一般个人项目优先从 `command` 开始。它最直观、最可控、最容易 debug。

### 示例：阻止危险 rm

`.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-dangerous-rm.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/block-dangerous-rm.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

input="$(cat)"
command="$(printf '%s' "$input" | jq -r '.tool_input.command // ""')"

if printf '%s' "$command" | grep -Eq 'rm[[:space:]].*(-rf|-fr)'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Blocked destructive rm command by project hook."
    }
  }'
fi
```

这里有两个关键点：

- `matcher: "Bash"` 只匹配 Bash 工具。
- `if: "Bash(rm *)"` 先粗过滤，避免每个 Bash 命令都启动脚本。

### 示例：编辑文件后自动格式化

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path // empty' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

这类 hook 要谨慎控制范围。对大项目来说，编辑任意文件后跑全量 formatter 可能太慢，也可能产生额外 diff。更好的做法是只格式化本次编辑的文件，或者只在某些后缀上触发。

### Claude Code 使用步骤

一个稳妥流程：

1. 先用自然语言描述你想强制的规则。
2. 选事件：行动前用 `PreToolUse`，行动后用 `PostToolUse`，结束前用 `Stop`，压缩前后用 `PreCompact` / `PostCompact`。
3. 写一个小脚本，先只记录日志，不做阻断。
4. 用 `/hooks` 查看它是否被加载。
5. 用 `claude --debug` 或 `claude --debug-file <path>` debug。
6. 确认输入 JSON 后，再让脚本返回阻断或补充上下文。
7. 项目级 hook 放 `.claude/settings.json`，个人偏好放 `.claude/settings.local.json` 或 `~/.claude/settings.json`。

## Codex CLI 里的 hooks

Codex hooks 的定位和 Claude 类似：在 Codex 生命周期中运行确定性脚本，注入 agentic loop。

OpenAI 官方文档列出的用途包括：

- 把会话发到自定义 logging / analytics。
- 扫描用户 prompt，避免误贴 API key。
- 自动总结会话生成持久记忆。
- 在一轮对话停止时运行自定义验证。
- 在特定目录下自定义 prompting。

本机 `codex features list` 显示 `hooks` 是 stable 且默认开启。

### 开关

在 `~/.codex/config.toml` 或项目 `.codex/config.toml` 里：

```toml
[features]
hooks = false
```

`hooks` 是官方当前 canonical feature key。旧的 `codex_hooks` 仍可能作为 deprecated alias 存在，但新配置应使用 `hooks`。

### 配置位置

Codex 会在 active config layer 旁边找 hooks：

| 位置 | 说明 |
|---|---|
| `~/.codex/hooks.json` | 用户级 hooks |
| `~/.codex/config.toml` | 用户级 inline `[hooks]` |
| `<repo>/.codex/hooks.json` | 项目级 hooks |
| `<repo>/.codex/config.toml` | 项目级 inline `[hooks]` |
| plugin `hooks/hooks.json` | 插件随带 hooks |
| plugin `.codex-plugin/plugin.json` 的 `hooks` 字段 | 插件自定义 hook 配置入口 |

项目级 `.codex/` hooks 只有在项目被 Codex 信任时才会加载。用户级 hooks 不依赖项目是否可信。

### 信任审核

Codex 对 hooks 有明确的 trust 机制：

- 非 managed command hook 运行前需要 review 和 trust。
- Codex 会按 hook 当前定义的 hash 记录信任状态。
- 新 hook 或改过的 hook 会被标记为需要重新 review。
- CLI 里的 `/hooks` 可用于查看来源、review、trust、disable 非 managed hooks。
- managed hooks 由系统、MDM、cloud 或 `requirements.toml` 策略控制，用户不能在 hook browser 里禁用。

这点很重要：hook 本质上会执行本地代码，所以项目里新增 `.codex/hooks.json` 不能默认无条件运行。

自动化场景下，如果外部系统已经审查过 hook 来源，可以临时用：

```bash
codex --dangerously-bypass-hook-trust
```

这个 flag 名字很直白，意思也很直白：它绕过 hook trust，应该只用于你已经在别处验证过 hook 来源的自动化环境。

### Codex 当前支持的事件

Codex 官方文档列出的当前事件包括：

| Event | 触发点 | 常见用途 |
|---|---|---|
| `SessionStart` | session 启动、恢复、clear 后、compact 后 | 加载 session notes |
| `SubagentStart` | subagent 启动 | 给子 agent 注入额外上下文 |
| `PreToolUse` | 支持的工具执行前 | 拦截 Bash、`apply_patch`、MCP 工具 |
| `PermissionRequest` | Codex 要请求审批时 | 自动 allow / deny |
| `PostToolUse` | 支持的工具产生输出后 | 整理输出、反馈问题 |
| `PreCompact` | compaction 前 | 阻止或记录压缩前状态 |
| `PostCompact` | compaction 后 | 补充压缩后上下文 |
| `UserPromptSubmit` | 用户 prompt 提交时 | secret 扫描、prompt 审计 |
| `SubagentStop` | subagent 停止时 | 检查子任务结果 |
| `Stop` | 当前回合停止时 | 要求补验证、写审计 |

Codex 文档特别提醒：`PreToolUse` 是 guardrail，不是完整 enforcement boundary。当前它不拦截所有 shell 调用，也不拦截 `WebSearch` 等非 shell / 非 MCP 工具。`PostToolUse` 发生在工具之后，不能撤销已经发生的副作用。

### Handler 支持状态

截至 2026-05-30 的官方文档：

- Codex 当前真正运行的是 `type: "command"` handler。
- `prompt` 和 `agent` handler 会被解析，但会跳过。
- `async: true` 会被解析，但 async command hooks 尚不支持，会跳过。
- command hook 默认 timeout 是 600 秒。
- command 的工作目录是当前 session `cwd`。

这和 Claude Code 的 hooks 相比更保守，也更像一个正在稳定化的核心扩展点。

### 示例：Codex 拦截危险 Bash

`.codex/hooks.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$(git rev-parse --show-toplevel)/.codex/hooks/pre_tool_use_policy.py\"",
            "timeout": 30,
            "statusMessage": "Checking Bash command"
          }
        ]
      }
    ]
  }
}
```

`.codex/hooks/pre_tool_use_policy.py`：

```python
#!/usr/bin/env python3
import json
import re
import sys

data = json.load(sys.stdin)
tool_input = data.get("tool_input") or {}
command = tool_input.get("command") or ""

if re.search(r"\brm\s+.*(-rf|-fr)\b", command):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": "Blocked destructive rm command by project hook."
        }
    }))
```

项目级脚本建议从 git root 解析路径，而不是写相对路径 `.codex/hooks/...`。Codex 可能从子目录启动，git-root-based path 更稳定。

### 示例：inline TOML

也可以直接写在 `.codex/config.toml`：

```toml
[[hooks.PreToolUse]]
matcher = "^Bash$"

[[hooks.PreToolUse.hooks]]
type = "command"
command = 'python3 "$(git rev-parse --show-toplevel)/.codex/hooks/pre_tool_use_policy.py"'
timeout = 30
statusMessage = "Checking Bash command"
```

一个 layer 里同时有 `hooks.json` 和 inline `[hooks]` 时，Codex 会都加载并警告。个人建议一个项目只选一种形式，优先用 `hooks.json`，因为 JSON 更接近 Claude Code 的配置形状，也更适合被插件或脚本生成。

## Claude Code 和 Codex CLI 的差异

| 维度 | Claude Code | Codex CLI |
|---|---|---|
| 菜单 | `/hooks` 浏览配置，官方 guide 说只读 | `/hooks` 查看来源、review、trust、disable 非 managed hooks |
| 用户级配置 | `~/.claude/settings.json` | `~/.codex/hooks.json` 或 `~/.codex/config.toml` |
| 项目级配置 | `.claude/settings.json` | `.codex/hooks.json` 或 `.codex/config.toml` |
| 本地个人配置 | `.claude/settings.local.json` | 通常用用户级 config 或项目 local 约定 |
| Handler 类型 | `command`、`http`、`mcp_tool`、`prompt`、`agent` | 当前运行 `command`，`prompt` / `agent` parsed but skipped |
| async hook | 支持 async / asyncRewake | 当前 parsed but skipped |
| Tool matcher | `Bash`、`Edit|Write`、`mcp__.*` 等 | `Bash`、`apply_patch`、`Edit|Write` alias、MCP tool names 等 |
| Trust 模型 | settings / managed policy / plugin 等配置面，文档强调安全实践 | 非 managed command hook 需要 hash-based review / trust |
| 适合场景 | 自动化、拦截、HTTP 集成、MCP 集成、LLM 判断、subagent 检查 | 自动化、拦截、权限审批、上下文补充、插件随带 hook |

我的理解：Claude Code hooks 更像一个完整的 extension framework；Codex hooks 目前更像一个安全优先、逐步开放的 lifecycle command hook 系统。两者方向一致：把 agent 行为从“提示词建议”推进到“runtime 可执行约束”。

## 什么时候该用 hook

一个简单判断：

```text
这件事是否必须每次发生？
是否能由确定性程序判断？
是否发生在明确生命周期节点？
是否需要在工具执行前后拦截或反馈？
```

如果答案大多是“是”，就适合 hook。

### 适合 hooks

- 阻止危险 shell 命令。
- 阻止编辑敏感文件。
- 编辑后自动格式化当前文件。
- 用户 prompt 进入模型前扫描 secret。
- agent 停止前检查是否运行过指定验证。
- compaction 前后保存或补充任务状态。
- permission request 时自动批准明确低风险命令。
- 记录工具调用和审批日志。
- Claude / Codex 等待输入时通知人。

### 不适合 hooks

- 大段项目背景解释：放 `AGENTS.md` / `CLAUDE.md` / docs。
- 偶尔用的工作流：做 slash command 或 skill。
- 需要长时间探索和判断的任务：用 subagent 或 reviewer。
- 真正的安全边界：用 sandbox、权限、OS、容器、网络策略。
- 最终质量门禁：用测试、CI、review。
- 模糊审美判断：不要硬塞到 hook，容易变成噪音。

hooks 应该短、小、快、可解释。它们不是把整个 CI 搬进 agent loop。

## 推荐的个人 hooks 组合

如果你刚开始配置，我建议从这五类开始。

| 优先级 | Hook | 价值 | 风险 |
|---|---|---|---|
| P0 | Notification | agent 等你输入时提醒 | 低 |
| P0 | Secret scan on prompt | 避免误贴 token | 低到中 |
| P1 | Dangerous command blocker | 阻止明显破坏性命令 | 中，规则要避免误伤 |
| P1 | Protected file blocker | 防止改 `.env`、`.git/`、agent 配置 | 中 |
| P2 | Post-edit formatter | 降低格式 diff | 中，可能引入额外改动 |
| P2 | Stop verification reminder | 防止没跑测试就结束 | 中，过严会打断流畅性 |
| P3 | Audit log | 复盘 agent 行为 | 低到中，注意隐私 |

一个个人工作区可以这样分层：

| 层 | 放什么 |
|---|---|
| 用户级 hooks | 通知、个人日志、个人安全偏好 |
| 项目级 hooks | 项目约定、受保护路径、项目测试命令 |
| managed hooks | 公司安全策略、合规审计、统一网络/权限策略 |

项目级 hooks 不要依赖你个人电脑的特殊路径；用户级 hooks 不要提交进仓库。

## 安全注意事项

hooks 的能力很强，所以风险也实在。

### 1. Hook 本质是本地代码执行

一个来自陌生仓库的 `.claude/settings.json` 或 `.codex/hooks.json` 可能让 CLI 在你的机器上执行脚本。不要在不可信项目里随便 trust hooks。

Codex 之所以做 hook trust review，就是因为这不是普通配置，而是可执行代码入口。

### 2. Hook 不是 sandbox

hook 能拦截一些事件，但它不是完整安全边界。

特别是：

- `PostToolUse` 已经在工具执行之后，不能撤销副作用。
- Codex 官方说明 `PreToolUse` 当前不拦截所有 shell 路径，也不拦截 `WebSearch` 等工具。
- 复杂 shell 命令、MCP 工具、外部服务仍然需要权限和 sandbox。

真正的边界应该由 sandbox、permission profile、allow/deny rules、网络策略、CI 和操作系统承担。

### 3. 不要把 secret 写进项目 hook 配置

Claude HTTP hooks 支持 `allowedEnvVars`，只有列出的变量才会被插入 header。这个设计说明了一件事：hook 里处理 secret 要非常克制。

原则：

- repo 配置不写 secret。
- secret 从本机环境或 keychain 取。
- 明确 allowlist。
- 日志里不要输出完整输入 JSON，除非你确定没有敏感信息。

### 4. Hook 输出要少而清楚

给 agent 的反馈越多，不一定越好。坏 hook 会制造大量噪音，让模型忽略真正重要的错误。

好的反馈应该像这样：

```text
Blocked: command touches .env. Read docs/configuration.md and ask the user before changing secrets.
```

坏反馈是把整段脚本日志、全量环境变量、无关 warning 都塞回去。

### 5. Hook 应该可重复、可超时、可失败

hook script 最好满足：

- 幂等。
- 不依赖交互式 stdin。
- 有 timeout。
- 出错时给明确 stderr。
- 不修改大范围文件。
- 不在每次 tool call 都跑昂贵命令。

否则它会把 agent loop 卡住。

## 从 harness engineering 看 hooks

hooks 的意义不是“工具多了一个菜单”，而是 coding agent 从 prompt-driven 走向 runtime-governed 的信号。

用 `AI Harness Engineering` 的术语映射：

| Harness 责任 | Hooks 怎么承担一部分 |
|---|---|
| Tool access | `PreToolUse` 限制工具使用 |
| Permissions | `PermissionRequest` 自动 allow / deny |
| Observability | 记录 prompt、工具、审批、输出 |
| Verification | `PostToolUse` / `Stop` 检查测试和证据 |
| Task state | `PreCompact` / `PostCompact` 保存和恢复状态 |
| Failure attribution | 工具失败后提取错误并分类 |
| Intervention recording | 记录用户何时审批、拒绝、打断 |
| Maintenance state | 阻止生成物污染、锁文件误改、文档漂移 |

这也解释了为什么 hooks、rules、permissions、sandbox、MCP、skills、subagents 会一起出现。它们都是 harness 的不同面：

- skills 提供“如何做”的知识。
- hooks 提供“必须执行”的动作。
- permissions / sandbox 提供“能不能做”的边界。
- MCP 提供“能接触哪些外部系统”的工具面。
- subagents 提供“隔离上下文和职责”的结构。
- telemetry 提供“做过什么、为什么做”的证据。

## 最小落地流程

给一个项目加 hook，不要一上来写复杂系统。按这个顺序：

1. 写一句规则：例如“agent 不得编辑 `.env`”。
2. 选择事件：编辑前拦截就是 `PreToolUse`。
3. 选择 matcher：例如 `Edit|Write` 或 Codex 的 `apply_patch` / `Edit|Write` alias。
4. 写一个只读脚本：先把输入 JSON 保存到临时日志，理解字段。
5. 加阻断输出：只在命中规则时返回 deny / block。
6. 用 `/hooks` 查看是否加载。
7. 手动构造一次违规动作，确认被拦截。
8. 再让 agent 做一次正常动作，确认没有误伤。
9. 把脚本和配置写进项目文档，说明为什么存在。
10. 保留 CI / review 作为最终门禁。

如果一个 hook 让你经常想临时关掉它，它就太粗糙了。改 matcher、缩小范围、降低触发频率，而不是把所有 workflow 都塞进 hook。

## 一句话总结

`/hooks` 是 coding agent 从“听话的聊天模型”走向“可治理的软件工程 runtime”的关键入口。它让项目规则、验证、通知、审计和安全约束不再只停留在 prompt 或记忆里，而是在 agent 行动的生命周期中自动执行。

但 hooks 不是魔法，也不是安全边界本身。最好的使用方式是：用 hooks 固化高频、确定、可解释的小规则；用 sandbox 和权限限制真实能力；用测试和 CI 验证最终结果；用文档和 skills 解释复杂知识。

这样 agent 才不是靠“记性好”工作，而是被一个越来越成熟的 harness 托住。

## 参考资料

- Anthropic Claude Code Docs: [Hooks reference](https://code.claude.com/docs/en/hooks)
- Anthropic Claude Code Docs: [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide)
- Anthropic Claude Code Docs: [Claude Code settings](https://code.claude.com/docs/en/settings)
- Anthropic Claude Code Docs: [Best practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- OpenAI Developers: [Codex Hooks](https://developers.openai.com/codex/hooks)
- OpenAI Developers: [Codex Config basics](https://developers.openai.com/codex/config-basic)
- OpenAI Developers: [Codex Advanced Configuration](https://developers.openai.com/codex/config-advanced)
- OpenAI Developers: [Codex Configuration Reference](https://developers.openai.com/codex/config-reference)
- OpenAI Blog: [Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely)
- OpenAI Blog: [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness)
- OpenAI Blog: [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop)
- arXiv: [AI Harness Engineering: A Runtime Substrate for Foundation-Model Software Agents](https://arxiv.org/abs/2605.13357)
