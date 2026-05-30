# Prompt: Rust Review

适合专门审查 Rust diff。

```text
请作为克制的 Rust reviewer 审查当前 diff。

重点检查：
1. ownership 边界是否自然。
2. 是否用 clone 绕过 borrow checker。
3. lifetime 或引用关系是否过度复杂。
4. 错误类型和错误上下文是否足够表达调用方需要处理的语义。
5. 是否有 unwrap / expect / panic 泄漏到不该出现的位置。
6. async task、锁、channel 是否有泄漏、死锁或取消问题。
7. trait / generic 抽象是否过早。
8. 测试是否覆盖错误路径和边界条件。
9. 是否引入 unsafe；如果有，safety invariant 是否完整。

输出规则：
- 只报告 P0/P1/P2。
- 每条必须有文件行号、失败场景、Rust 语义依据和验证方式。
- 不要给纯风格建议。
- 如果只是可以以后改善，请标为 P2。
```
