# Prompt: 拆 Reviewable Slices

适合 agent 容易一次生成太多代码，或者你已经预感 diff 会很大。

```text
请把这个任务拆成 reviewable slices。

每个 slice 必须满足：
- 只对应一个行为变化。
- 能单独测试或验证。
- diff 足够小，人工 5-10 分钟内能 review。
- 有明确回滚方式。

每个 slice 请说明：
1. 目标行为。
2. 涉及文件。
3. 预计 diff 大小。
4. 测试方式。
5. 回滚方式。

现在只执行 slice 1。
如果你发现需要扩大范围，请停止并重新给我拆分方案。
```

如果 agent 已经改太多：

```text
停。这个 diff 太大，无法 review。
请不要继续实现。
请把当前改动拆成 3-5 个 reviewable slices，
说明每个 slice 的目标、涉及文件、测试方式。
然后只保留 slice 1，撤回其它无关改动。
```
