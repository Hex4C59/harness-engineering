# Scripts 模板

每个项目至少提供稳定的脚本入口，让人和 agent 不需要猜命令。

```text
scripts/bootstrap  安装依赖、准备本地环境
scripts/dev        启动开发服务
scripts/test       跑测试
scripts/check      完整验证：format/lint/typecheck/test/build
```

脚本应该满足：

- 可以被人运行。
- 可以被 agent 运行。
- 输出清晰。
- 失败时 exit code 非 0。
- 不依赖 agent 记忆。

## `scripts/check`

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "==> format check"
# npm run format:check / cargo fmt --check / gofmt check / ruff format --check

echo "==> lint"
# npm run lint / cargo clippy / ruff check

echo "==> typecheck"
# npm run typecheck / cargo check / pyright

echo "==> test"
./scripts/test

echo "==> build"
# npm run build / cargo build / go build ./...
```

不一定每个项目都有所有步骤，但接口要稳定：agent 永远先找 `./scripts/check`。

## 既有项目包装已有命令

如果项目已经有 `make check` 或 `just check`：

```bash
#!/usr/bin/env bash
set -euo pipefail

make check
```

Node 项目示例：

```bash
#!/usr/bin/env bash
set -euo pipefail

npm run lint
npm run typecheck
npm test
npm run build
```

如果某些命令暂时跑不通，不要假装成功。写到 `docs/project-status.md`：

```text
当前 scripts/check 暂不可用，原因：
- 缺少数据库。
- 缺少 env var。
- 现有测试依赖外部服务。

当前最小可用验证命令：
- npm run lint
- npm test -- path/to/unit.test.ts
```
