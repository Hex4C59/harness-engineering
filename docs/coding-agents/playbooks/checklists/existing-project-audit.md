# Checklist: 既有项目审计

先只读审计，不要改业务代码。

- [ ] 根目录入口文件：`README`、`CONTRIBUTING`、manifest、Makefile、justfile、CI workflow。
- [ ] 是否已有 `AGENTS.md`、`CLAUDE.md`、`.cursor/rules`、Copilot instructions 或其它 agent 配置。
- [ ] 现有开发命令。
- [ ] 现有测试命令。
- [ ] 现有 lint / typecheck / build 命令。
- [ ] 当前测试覆盖和验证入口是否清楚。
- [ ] 主要代码入口和测试入口。
- [ ] 主要模块和数据流。
- [ ] 哪些文档是事实来源。
- [ ] 哪些文档可能过期或与命令冲突。
- [ ] 哪些目录是 generated、vendor、snapshot、迁移或部署资产。
- [ ] 哪些命令有副作用，不应进入默认 `scripts/check`。
- [ ] 最小接入建议：新增或更新哪些文档和脚本。

遇到文档和现实冲突时，优先级建议：

```text
当前可运行命令 > CI 配置 > package/manifest scripts > README > 旧文档 > agent 推断
```
