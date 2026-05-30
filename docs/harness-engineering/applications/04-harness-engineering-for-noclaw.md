# Harness Engineering 如何映射到 Noclaw

Noclaw 不只是一个“聊天机器人项目”。从 harness engineering 的角度看，它更像一个 **agent harness runtime**：负责把不同通道的输入、持久状态、隔离环境、工具回调和任务调度组织成一个可靠的 agent 工作系统。

---

## Noclaw 的 harness 视角

当前参考架构可以这样理解：

```text
Channels -> SQLite -> Router -> GroupQueue -> Container Runner -> Agent -> Outbound
                         ^              |                  |
                         |              v                  v
                       State           IPC               Logs
```

对应 harness 组件：

| Noclaw 模块 | Harness 责任 |
|---|---|
| Channel | 输入/输出边界，统一不同平台消息 |
| SQLite | 事实来源、对话历史、任务状态 |
| Router | 触发规则、上下文选择、prompt 组装 |
| GroupQueue | 执行调度、并发限制、重试和生命周期 |
| Container Runner | sandbox、工具执行环境、超时和日志 |
| IPC | agent 到宿主的受控回调层 |
| Scheduler | 长期任务和自动唤醒 |
| Groups | 权限、记忆、文件系统隔离 |
| Docs + AGENTS.md | agent 可读知识库和操作约定 |
| Tests | 验证闭环 |

这个映射能帮助实现时避免模块混杂：每个模块都服务于 harness 的某个责任。

---

## 当前仓库已经具备的 harness 雏形

虽然代码还没开始展开，但文档层已经有了 H1 的基础：

- `AGENTS.md`：项目协作入口和规则。
- `docs/harness-engineering/README.md`：harness engineering 文档地图。
- `docs/coding-agents/README.md`：coding agent 工具、插件和个人工作流文档地图。
- `docs/ai-models/README.md`：模型能力、benchmark 和模型报告文档地图。
- `docs/noclaw/project-status.md`：新窗口接手快照。
- `docs/noclaw/implementation-plan.md`：长期任务计划、验收标准、决策记录。
- `docs/nanoclaw/`：上游参考资料。
- `docs/harness-engineering/`：agent harness 方法论说明。

这已经是在把仓库知识变成 agent 可读事实来源，而不是依赖聊天上下文。

---

## Noclaw 应优先构建哪种 harness

建议按实施计划继续走，但用 harness 视角重新解释优先级。

### 1. DevChannel + FakeRunner

这是最小可验证 harness。

目的不是模拟真实能力，而是先固定边界：

```text
输入消息 -> 入库 -> 路由 -> 队列 -> runner -> 出站
```

一旦这个闭环稳定，后续 Docker、Claude、Telegram 都只是替换某个 harness 部件。

### 2. SQLite Repository

SQLite 不只是存消息。它是 agent 系统的事实来源：

- 消息历史。
- 路由游标。
- 组注册。
- 会话 ID。
- 任务定义。
- 运行日志。

如果状态不可信，agent 就会靠猜。

### 3. Group Isolation

组隔离是权限 harness：

- 每组独立 `CLAUDE.md`。
- 每组独立 session。
- 每组独立 IPC。
- main 与非 main 权限不同。
- mount 规则不同。

这会直接决定系统是否能安全地让 agent 长时间工作。

### 4. IPC

IPC 是工具回调 harness。

容器里的 agent 不能直接随便碰宿主，它通过 JSON 文件表达意图：

- 发送消息。
- 创建任务。
- 暂停任务。
- 注册组。

宿主进程负责校验权限、执行操作、记录结果。

### 5. Docker Runner

Docker Runner 是执行环境 harness：

- 隔离文件系统。
- 注入有限环境变量。
- 挂载必要目录。
- 读取 stdout/stderr。
- 解析输出 marker。
- 控制超时和日志。

这部分决定 agent 的能力边界和安全边界。

---

## 对实施计划的具体建议

现有计划已经合理，但可以带着下面几条原则执行。

### 原则 1：每个里程碑都产出一个 harness 能力

不要只问“实现了哪个模块”，还要问“agent 因此多了什么可靠能力”。

例子：

- Config：让 harness 可配置、可复现。
- DB：让 harness 有事实来源。
- DevChannel：让 harness 可测试。
- FakeRunner：让 harness 不依赖真实模型也能验证流程。
- IPC：让 harness 有受控工具回调。
- Scheduler：让 harness 支持长期任务。

### 原则 2：先做可观察，再做自动化

不要急着自动修复所有问题。先让问题可见：

- 记录消息 ID、chat、group、timestamp。
- 记录 runner 输入摘要，而不是 prompt 明文。
- 记录输出长度、退出码、超时原因。
- 记录 IPC 操作和拒绝原因。
- 记录任务运行状态。

可观察之后，自动恢复和自动调度才有基础。

### 原则 3：失败要回写到文档或测试

如果实现过程中 agent 或人发现重复问题，应写回：

- `docs/noclaw/project-status.md`
- `docs/noclaw/implementation-plan.md`
- 未来的 `docs/noclaw/decisions/`
- 对应单元测试或集成测试

这就是把失败升级为 harness。

### 原则 4：AGENTS.md 继续保持短

不要把所有实现细节塞进 `AGENTS.md`。它应该继续只做入口：

- 项目风格。
- 测试策略。
- 必读文档。
- 状态更新约定。
- 禁止操作。

更细的知识放在 `docs/`。

---

## Noclaw 可以逐步达到的 harness 层级

| 阶段 | Noclaw 表现 |
|---|---|
| H1 | 文档、计划、状态快照、AGENTS 入口 |
| H2 | DevChannel/FakeRunner、测试、SQLite、IPC 权限、Docker Runner |
| H3 | 自动任务、运行日志分析、失败分类、doc gardening、质量评分、自动 review |

当前项目刚进入 H1。下一步不是追求复杂 agent，而是把 H2 的最小闭环做扎实。

---

## 一个适合 noclaw 的完成定义

每完成一个功能，不只看代码合并，还要检查：

- [ ] 是否更新实施计划 checkbox。
- [ ] 是否更新 project status。
- [ ] 是否有必要新增或更新决策记录。
- [ ] 是否有测试或手动验证记录。
- [ ] 是否影响安全边界。
- [ ] 是否影响 agent 可读文档。
- [ ] 是否引入新的 harness 约束或工具。

这个完成定义会让 noclaw 自己也成为一个更好被 agent 维护的项目。
