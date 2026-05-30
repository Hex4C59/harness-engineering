# 既有项目最小 `AGENTS.md` 模板

既有项目的 `AGENTS.md` 不应该重写项目百科。它只需要告诉 agent：先读哪里、不要碰哪里、常用命令是什么、完成前必须跑什么验证。

```md
# AGENTS.md

这个项目是一个既有项目。修改前先尊重已有结构和约定，不要按新项目模板重排目录。

## 入口地图

1. `README.md`：项目目标、安装和运行方式。
2. `docs/README.md`：项目文档地图和渐进式读取顺序。
3. `docs/project-status.md`：当前状态、验证命令、未决问题。
4. `docs/development.md`：本地开发命令。
5. `docs/testing.md`：测试策略、TDD 和 bug 修复流程。

## 工作规则

- 修改前先阅读相关文档和现有代码，不要猜测项目结构。
- 非平凡任务先写 `docs/plans/` 计划，并等待确认。
- Bug 修复先复现，再写失败回归测试，再最小修复。
- 不做无关重构。
- 不删除或覆盖已有资料，除非用户明确要求。
- 完成前运行 `./scripts/check` 或本文档指定的验证命令。

## 禁止默认修改

- 生成物、vendor、lockfile、迁移文件、生产配置、secret、部署脚本等，除非当前任务明确需要。
```

如果项目已经有 `AGENTS.md`、`CLAUDE.md`、`.cursor/rules`、`.github/copilot-instructions.md` 或贡献指南，只做增量补充，不要复制出互相冲突的规则。
