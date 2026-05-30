# Prompt: 新功能 TDD

适合中等复杂度功能。计划确认前不要写实现代码。

```text
我要做一个新功能。请遵守 TDD 和 reviewable slices。

功能：
背景：
验收标准：
暂时不做：

请先读取 AGENTS.md、README.md、docs/README.md、docs/testing.md 和相关代码。
然后先写计划，不要改实现代码。

计划必须包含：
1. 目标和非目标。
2. blast radius：预计影响文件、是否改公共接口、是否涉及权限/数据/网络/部署。
3. reviewable slices：每个 slice 的目标、涉及文件、测试方式、回滚方式。
4. 先写哪些失败测试。
5. 验证命令。

等我确认计划后，只执行 slice 1。
```
