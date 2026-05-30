# Prompt: 既有项目只读审计

适合第一次给既有项目接入 agent harness。重点是先发现秩序，不要重排项目。

```text
我要给一个既有项目接入 agent harness 和 TDD 工作流。请按照当前这份 playbook 执行。

如果你已经能读取当前文档，请直接按本文执行。
如果你还没有看到这份文档，请先让我提供它的路径或内容，不要猜测流程。

项目名称：
项目类型：
技术栈：
我希望保留的现有约定：
暂时不要做的事：

请先只做只读审计，不要修改任何业务代码。

请检查：
1. 根目录有哪些入口文件：README、package.json、Makefile、justfile、Cargo.toml、pyproject.toml、go.mod、CI workflow 等。
2. 是否已有 AGENTS.md、CLAUDE.md、.cursor/rules、docs/、scripts/。
3. 现有开发命令、测试命令、lint/typecheck/build 命令分别是什么。
4. 当前测试覆盖和验证入口是否清楚。
5. 项目结构、主要模块和数据流大概是什么。
6. 哪些文档可以作为事实来源，哪些信息缺失。
7. 哪些目录或文件不应该让 agent 随便改。

然后先给我一份接入建议，不要直接修改：
- 当前项目已有的 harness 能力。
- 缺失的最小文件。
- 建议新增或修改的文件列表。
- 建议采用的验证命令。
- 风险和不确定点。
- 是否建议直接创建 AGENTS.md、docs/project-status.md、docs/testing.md、scripts/check。
```
