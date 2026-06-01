# Experiment Log

实验：Readonly Notes MCP Server

开始日期：2026-05-31

## 当前问题

我已经阅读了 MCP 调研文档，但需要用代码验证：

1. MCP server 最小实现需要哪些文件和配置。
2. `resource` 和 `tool` 在真实 host 中的体验差异。
3. 只读权限边界如何设计和验证。
4. MCP Inspector 能发现哪些问题。

## 假设

- 一个只读本地 MCP server 足够验证 MCP 的核心协议闭环。
- 先返回文件路径和标题，比直接返回完整文档正文更容易控制上下文。
- Host 差异会主要体现在配置位置、approval 行为和 output limit。

## 实验记录

### 2026-05-31

- 创建实验目录。
- 尚未实现 server。

## 观察

待补充。

## 踩坑

待补充。

## 结论

待补充。

## 下一步

- 实现最小 server。
- 添加搜索函数测试。
- 用 MCP Inspector 验证。
