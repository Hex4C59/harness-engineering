# Harness Design Log

这个文件记录 **Coding Agent Harness Toolkit** 的 workflow / prompt / checklist / skill 为什么需要演化。

它不是日常任务状态，不记录某个功能做到了哪一步；它记录 agent 使用中的失败模式、险些失败、流程摩擦和由此产生的 harness 设计决策。

## 为什么需要这个文件

不要因为看到某个外部工具很强，就直接复制它的完整流程。更稳的方式是：

```text
真实任务中的失败模式
-> 记录和归类
-> 先加 checklist / prompt / workflow
-> 用真实任务验证
-> 边界稳定后再升级成 skill
-> 多个 gate 稳定后再更新 orchestration map
```

这个文件用于保留“为什么要加这个 gate”的上下文，避免未来只看到一堆流程，却不知道它们解决过什么问题。

## 什么时候记录

遇到这些情况时，在这里补一条：

- agent 已经失败，或者差点失败。
- agent 开始跳过计划、测试、验证或 review。
- agent 一次修改太多，diff 不可 review。
- agent 因长上下文、旧计划或不完整状态而误判。
- 现有 workflow / prompt / skill 不知道该不该触发。
- 某个 gate 反复误伤小任务，流程显得过重。
- 从外部实践中看到一个值得吸收的模式，但还不确定是否适合本工具箱。

不要为了记录而记录。没有真实失败模式或明确设计问题，就不必新增条目。

## 决策漏斗

新增或修改 harness 资产时，默认按这个顺序升级：

```text
观察 / 失败记录
-> checklist 条目
-> prompt
-> workflow
-> skill
-> orchestration map
```

升级条件：

- 同类问题出现 2-3 次。
- 触发条件清楚。
- 输入和输出稳定。
- 失败时知道如何 stop。
- 证据可以被命令、diff、测试、review 或人工检查验证。
- 新增流程的成本低于继续靠临场提醒的成本。

降级条件：

- gate 经常误伤 Tiny / Small 任务。
- 需要加载太多上下文才能使用。
- 输出不稳定，无法被复用。
- 它只是原则解释，不是可执行流程。
- 它更适合 research / draft，而不是 playbooks / skills。

## 记录模板

```md
## YYYY-MM-DD：标题

任务：

agent 失败 / 险些失败：

失败阶段：

已有 gate 是否覆盖：

需要新增 / 修改的 gate：

先作为 checklist、prompt、skill 还是 workflow：

验证方式：

决策：
```

## 日志

### 2026-05-31：从 Superpowers 调研中补齐轻量编排层

任务：

研究 Superpowers 的 `brainstorming`、`writing-plans`、`test-driven-development` 和 `subagent-driven-development`，并判断哪些实践适合进入本项目的 Coding Agent Harness Toolkit。

agent 失败 / 险些失败：

不是一次具体代码失败，而是工具箱设计层面的风险：看到 Superpowers 很完整后，容易直接复制整套 skill，导致流程过重、上下文成本变高、小任务被固定流水线拖慢。

失败阶段：

Harness 设计阶段 / workflow 编排阶段。

已有 gate 是否覆盖：

部分覆盖。

- `daily-development.md` 已有日常任务循环。
- `tdd-and-verification.md` 已有 TDD 和验证原则。
- `reviewable-slices.md` 已有 diff 可 review 原则。
- 但缺少明确的任务路由图，也缺少从失败模式推动 harness 演化的记录机制。

需要新增 / 修改的 gate：

- 新增 `workflows/orchestration-map.md`，作为轻量路由表，而不是固定流水线。
- 新增 `skills/brainstorming`、`skills/writing-plans`、`skills/tdd-gate`，把成熟、可触发的阶段 gate skill 化。
- 保留 `review-gate`、`commit-gate` 的边界，不把所有流程都升级成 skill。
- 新增本文件，用于以后记录 gate 是否真的来自真实失败模式。

先作为 checklist、prompt、skill 还是 workflow：

- 任务路由：workflow。
- 需求澄清、写计划、TDD：skill + prompt。
- harness 演化记录：meta 文档。

验证方式：

- 后续真实任务中观察：
  - agent 是否能用 `orchestration-map.md` 选择轻量路径。
  - `writing-plans` 是否减少不可执行计划。
  - `tdd-gate` 是否减少先实现后补测试。
  - 新增 skill 是否误伤 Tiny / Small 任务。
  - 是否出现需要继续新增 skill 的真实失败模式。

决策：

不复制 Superpowers 的完整固定流程。先采用轻量编排：

```text
Tiny / Small 任务保留短路径；
Medium / Large 任务按风险升级；
需求不清 -> brainstorming；
多步骤 -> writing-plans；
行为变化 -> tdd-gate；
实现完成 -> review-gate；
准备提交 -> commit-gate。
```
