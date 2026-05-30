# Prompt: Prototype 模式

适合只想快速验证想法、不追求长期维护时使用。

```text
这是一个 prototype，不是 production。

目标：
非目标：
允许偷懒的地方：
不允许偷懒的地方：
风险边界：

请帮我快速实现，但必须遵守：
1. 不操作生产数据。
2. 不写入真实外部服务。
3. 不保存 secret。
4. 关键路径至少有 smoke test。
5. 在 README 或项目状态里标注这是 AI-assisted prototype，不保证 personally maintainable。
6. 完成后列出：如果要生产化，必须补哪些测试、文档、review 和安全检查。

先给计划，不要直接实现。
```
