# Skills

这里预留 **Coding Agent Harness Toolkit** 中可复制到项目级或个人环境的 coding agent skills。

## 定位

Skills 适合沉淀固定时刻、固定判断框架、跨项目重复使用的操作能力，例如：

- Commit Gate：判断当前工作区是否到达 commit 边界，并生成提交拆分建议。
- Checklist Self Audit：按 checklist 检查当前结果、补齐遗漏并给出验证证据。
- Review And Finish：运行验证、检查 diff、更新状态文档并收口。

## 使用原则

- Skill 应优先读取项目内 `AGENTS.md` 和 `docs/agent/playbooks/`，以项目规则为准。
- Skill 默认只给出计划、判断和建议；执行高风险操作前必须得到用户明确授权。
- 不要把 token、私钥、本机绝对路径或真实 MCP 凭据写入 skill 模板。
- 如果某个 skill 还没有经过真实项目验证，先保留为草稿，不要升级成默认入口。

## 注入方式

把 `docs/agent/skills/` 复制进项目后，skill 还只是项目内资产。要让 agent 自动触发，必须注入到具体 agent runtime。

### Codex

个人级安装：

```text
<project>/docs/agent/skills/commit-gate/
-> $CODEX_HOME/skills/commit-gate/
```

安装后，新会话会通过 `SKILL.md` 的 frontmatter `description` 自动判断是否触发。

临时使用：

```text
请阅读 docs/agent/skills/commit-gate/SKILL.md，
然后按这个 skill 判断当前工作区是否 commit-ready。
不要执行 git commit。
```

项目级使用：

- 在 `AGENTS.md` 中登记可用 skill 路径。
- 在相关 workflow 中写明何时调用该 skill。
- 如果团队成员没有安装 skill，也能通过直接阅读项目内 `SKILL.md` 执行同一套规则。

### 其他 agent runtime

不同客户端的 skill 目录和启用方式不同。迁移时只复制 `SKILL.md` 和必要资源，不复制个人凭据、本机路径或运行时缓存。

## 状态

当前已收录：

| Skill | 用途 |
|---|---|
| [`brainstorming`](brainstorming/SKILL.md) | 在计划或实现前澄清模糊需求，比较方案，并产出可确认的方向、验收标准和下一步 |
| [`writing-plans`](writing-plans/SKILL.md) | 将已确认需求或 Spec 写成 `docs/plans/*.md` 执行计划，拆 reviewable slices、TDD 步骤、验证和回滚 |
| [`commit-gate`](commit-gate/SKILL.md) | 判断当前 Git 工作区是否到达提交边界，并生成提交拆分和 commit message 建议 |
