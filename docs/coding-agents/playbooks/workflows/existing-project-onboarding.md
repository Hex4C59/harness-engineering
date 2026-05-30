# Workflow: 既有项目 Agent 接入

既有项目接入 agent harness 时，第一目标不是把项目改成新模板，而是让 agent 看懂项目、尊重现有约定、能运行验证命令，并且不会扩大改动范围。

核心流程：

```text
只读审计 -> 找到现有事实来源 -> 建立最小 AGENTS.md
-> 补 docs/status 和 testing -> 包装已有命令
-> 小步验证 -> 再开始写业务代码
```

## 什么时候用

- 接手一个已有仓库。
- 给老项目补 `AGENTS.md`。
- 想把已有项目改成更适合 Codex / Claude Code / Cursor 协作。
- 项目没有统一验证入口，想补 `scripts/check`。
- 项目文档散落，想让 agent 有渐进式读取路径。

## 最短路径

1. 先用 [`../prompts/existing-project-audit.md`](../prompts/existing-project-audit.md) 做只读审计。
2. 用 [`../checklists/existing-project-audit.md`](../checklists/existing-project-audit.md) 检查审计是否完整。
3. 审计确认后，用 [`../prompts/existing-project-minimal-onboarding.md`](../prompts/existing-project-minimal-onboarding.md) 做最小接入。
4. `AGENTS.md` 可参考 [`../templates/existing-project-agents.md`](../templates/existing-project-agents.md)。
5. 验证入口可参考 [`../templates/scripts.md`](../templates/scripts.md)。

## 接入原则

- 不改业务代码。
- 不重排目录结构。
- 不删除或覆盖已有文档。
- 优先包装已有命令，不发明一套新命令。
- 信息不确定时标注“待确认”，不要编造。
- 如果已有 `AGENTS.md`、`CLAUDE.md`、`.cursor/rules` 或贡献指南，只做增量补充。

## 接入完成标准

1. 新会话能通过 `AGENTS.md` 找到入口。
2. `docs/project-status.md` 能说明当前项目状态。
3. `docs/development.md` 能说明本地开发命令。
4. `docs/testing.md` 能说明测试和 TDD 流程。
5. 有一个可信的最小验证命令。
6. agent 知道哪些目录和文件不要默认修改。
7. 第一个小验证改动可以被顺利完成和 review。

最短记忆版：

```text
既有项目不要重建秩序，先发现秩序。
先只读审计，再最小补入口；
先包装现有命令，再谈 TDD；
先做小验证改动，再做大功能。
```
