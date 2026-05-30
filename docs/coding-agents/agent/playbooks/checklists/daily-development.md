# Checklist: 日常开发

## 先判断任务类型

| 类型 | 流程 |
|---|---|
| 小修 | 读相关文件 -> 修改 -> 局部测试 -> check |
| 新功能 | 必要时写 Spec -> 写 Execution Plan -> TDD -> 实现当前 slice -> 完整验证 -> 更新状态 |
| bug | 复现 -> 根因分析 -> 回归测试 -> 修复 -> 验证 |
| 重构 | 行为测试保护 -> 小步改 -> 每步验证 |
| 架构变化 | 写 decision -> 用户确认 -> 写 Execution Plan -> 实现 |
| 文档 | 更新相关 docs -> 检查链接和目录 |

## 开始前

- [ ] 读取 `AGENTS.md`、`README.md` 和 `docs/README.md`。
- [ ] 读取 `docs/project-status.md`、`docs/development.md`、`docs/testing.md`。
- [ ] 判断是否需要 scout。
- [ ] 判断是否需要 Spec。
- [ ] 判断是否需要 Execution Plan。
- [ ] 评估 blast radius。
- [ ] 拆 reviewable slices。
- [ ] 明确本轮只执行哪个 slice。

## 实现中

- [ ] 先写失败测试。
- [ ] 运行测试并确认失败原因正确。
- [ ] 写最小实现。
- [ ] 运行局部测试。
- [ ] 在测试保护下重构。
- [ ] 不做无关重构、改名、格式化或依赖升级。

## 收口

- [ ] 运行本轮相关验证命令。
- [ ] 必要时运行 `./scripts/check`。
- [ ] 检查 diff 是否可 review。
- [ ] 更新 `docs/project-status.md` 和相关 plan。
- [ ] 总结修改、验证、风险和下一步。
