# 怎么使用 Harness Engineering

下面是一套从小到大的落地方法。它不要求一次做完，而是把 agent 的失败逐步转化成更好的环境。

---

## Step 1：定义 agent 要交付什么

先不要急着选框架。先回答：

- Agent 的任务类型是什么？
- 输入来自哪里？
- 输出发到哪里？
- 什么算完成？
- 什么必须由人判断？
- 失败时应该保留什么证据？

例子：

```text
任务类型：实现 Rust 项目中的小功能。
输入：用户需求 + docs 中的计划。
输出：代码变更 + 测试结果 + 状态文档更新。
完成标准：cargo test 通过；计划 checkbox 更新；project-status 更新。
人工判断：架构方向变化、外部依赖选择、安全边界放宽。
证据：git diff、测试输出、决策记录。
```

---

## Step 2：把入口地图做小

入口文件应该短，负责告诉 agent 去哪里找信息。

推荐结构：

```text
AGENTS.md
docs/
  README.md
  project-status.md
  implementation-plan.md
  architecture/
  decisions/
  references/
```

入口文件里不要塞所有知识，只放：

- 代码风格。
- 测试策略。
- 必读文档。
- 不可破坏的约定。
- 计划如何更新。

这样可以保护上下文窗口，让 agent 逐步读取相关资料。

---

## Step 3：建立事实来源

把容易散落在对话里的内容版本化：

| 信息 | 建议位置 |
|---|---|
| 当前进度 | `project-status.md` |
| 全量任务 | `implementation-plan.md` |
| 架构边界 | `architecture.md` 或 `docs/architecture/` |
| 决策原因 | `decisions/` 或计划文档的“决策记录” |
| 外部参考 | `references.md` |
| 常见命令 | README 或 `docs/development.md` |
| 失败案例 | `docs/troubleshooting.md` 或技术债记录 |

事实来源越清楚，新窗口接手越稳。

---

## Step 4：给 agent 可执行工具，而不是只给说明

说明是软约束，工具和检查是硬反馈。

常见工具层：

- `rg` 搜索代码。
- `cargo fmt` 格式化。
- `cargo test` 验证。
- `cargo clippy` 静态检查。
- `git diff` 查看变更。
- 本地 dev server。
- 浏览器自动化。
- 日志查询。
- 数据库 fixture。

Harness 的目标不是让 agent 记住“要测试”，而是让测试成为流程的一部分。

---

## Step 5：设计权限边界

Agent 越强，越要明确边界：

- 哪些目录可写？
- 哪些文件只读？
- 哪些命令禁止？
- 哪些命令需要人确认？
- Secret 是否可能进入日志？
- 外部网络访问是否必要？
- 多 agent 是否会改同一文件？

对于 coding agent，最低限度应该有：

- 不覆盖用户未授权变更。
- 不运行 destructive git 命令。
- 不把 secret 写入文档、日志或 prompt。
- 不随便扩大依赖和架构范围。

对于运行在容器里的 agent，还要有：

- mount allowlist。
- readonly project root。
- per-group filesystem namespace。
- IPC 权限检查。
- 运行日志脱敏。

---

## Step 6：定义验证闭环

Agent 每次完成任务后应能回答：

```text
我做了什么？
我为什么这样做？
我怎么验证它？
还有什么风险？
下一步是什么？
```

对应 harness 机制：

- 任务计划保存“做什么”。
- 决策记录保存“为什么”。
- 测试和日志保存“怎么验证”。
- 项目状态保存“风险和下一步”。

更高级时可以加：

- 自动运行测试并把失败回灌给 agent。
- 独立 review agent。
- 结构化评审清单。
- UI 自动截图对比。
- 性能指标阈值检查。

---

## Step 7：记录失败并升级 harness

每次 agent 出错，都按这个模板处理：

```md
## Failure Record

- 日期：
- 任务：
- 表现：
- 根因：
- 缺失的 harness 能力：
- 修复方式：
- 是否需要测试/检查：
- 是否需要更新文档：
```

常见映射：

| 失败 | Harness 改进 |
|---|---|
| agent 找错文件 | 增加 docs 索引、模块地图、搜索约定 |
| agent 重复造轮子 | 增加架构边界、现有 helper 清单、lint |
| agent 忘记更新状态 | AGENTS.md 加必读和更新规则 |
| agent 不跑测试 | 完成清单强制验证命令 |
| agent 破坏用户改动 | 写入前检查 git status/diff |
| agent 输出不可验证 | 增加验收标准和测试 fixture |
| agent 做太大 | 增加里程碑切片和任务范围上限 |

---

## Step 8：按成熟度逐步升级

可以把 harness 成熟度分成四层：

| 层级 | 状态 | 典型能力 |
|---|---|---|
| H0 | 手工提示 | 人复制上下文，agent 临时执行 |
| H1 | 文档化 harness | `AGENTS.md`、状态快照、计划、README |
| H2 | 可执行 harness | 测试、lint、脚本、CI、fixture、自动校验 |
| H3 | 自我改进 harness | 失败分类、自动 review、doc gardening、质量评分、技术债清理 |

不要一开始追 H3。先让 H1 稳，再把重复的人为检查升级成 H2。

---

## Step 9：保持 human-in-the-loop

Harness engineering 不是取消人，而是把人放到更高杠杆的位置：

- 人定义目标。
- 人决定边界。
- 人做高风险判断。
- 人把品味和经验编码成规则。
- Agent 执行、验证、迭代、整理证据。

好的 harness 会让人少做重复沟通，多做方向判断。

---

## 最小落地清单

一个项目想开始使用 harness engineering，可以先做这 10 件事：

- [ ] 保持 `AGENTS.md` 简短，只做入口地图。
- [ ] 建立对应主题目录的 `README.md`。
- [ ] 建立 `project-status.md`。
- [ ] 建立 `implementation-plan.md`。
- [ ] 每个任务写验收标准。
- [ ] 规定完成后必须更新状态。
- [ ] 规定验证命令。
- [ ] 记录关键决策。
- [ ] 对危险操作设置明确禁止项。
- [ ] 把重复失败变成文档、测试或工具。
