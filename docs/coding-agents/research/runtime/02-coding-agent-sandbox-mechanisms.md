# Codex CLI Sandbox 与 Coding Agent 沙箱机制详解

调研日期：2026-05-29

## 这篇文档回答什么

你问的是 Codex CLI 的 sandbox 是怎么实现的，以及进一步延伸到整个 coding CLI / coding agent 生态里，sandbox 通常是怎么实现的。

本文会分 4 层展开：

1. Codex CLI / Codex 本地 agent 的 sandbox 到底保护什么。
2. Codex 在 macOS、Linux / WSL2、Windows 上分别用了什么 OS 机制。
3. Claude Code、Cursor、Gemini CLI、OpenCode、Aider 等 coding CLI 的 sandbox 设计有什么异同。
4. 把这些实现抽象成一套可复用的 agent sandbox 设计框架。

本文里的 Codex 源码判断基于 `openai/codex` 仓库在 2026-05-29 的公开源码快照：

```text
commit: 1c55bb2702fa4fbbccbf21057d344240744b5041
subject: [codex] Improve built-in tool schema docs (#24794)
```

需要特别注意：sandbox 是变化很快的实现层。OpenAI 官方文档、源码、博客和 system card 在不同发布时间里可能会出现“历史实现”“高层描述”和“当前实现细节”并存的情况。例如 GPT-5.3-Codex system card 仍把 Linux 本地 sandbox 概括为 `seccomp + Landlock`，而当前公开源码已经把 `bubblewrap` 作为默认 filesystem sandbox，保留 Landlock 作为 legacy fallback。读这类资料时要把日期、接口和源码版本放在一起看。

核心结论：

> Coding agent 的 sandbox 不是一个单点功能，而是“文件系统隔离、网络隔离、进程树约束、权限审批、命令可观测性、外部工具边界”组合起来的执行安全层。Codex 的本地实现是典型的跨平台 OS-native sandbox：macOS 用 Seatbelt，Linux / WSL2 以 bubblewrap + seccomp 为主，Windows 用 restricted token / ACL / WFP 等 Windows 原生机制。

## 先区分几个容易混淆的概念

讨论 agent sandbox 前，先把几个词拆开：

| 概念 | 解决什么问题 | 是否是 OS 级边界 | 典型例子 |
|---|---|---:|---|
| sandbox | 限制命令真实能访问哪些文件、网络、系统资源 | 通常是 | Seatbelt、bubblewrap、Landlock、Docker、Windows restricted token |
| approval | 决定什么时候问用户要不要越权 | 不是 | `on-request`、`untrusted`、“是否允许联网” |
| permission / policy | 描述允许和禁止的动作 | 本身不是，需要执行器落实 | read-only、workspace-write、allow/deny bash command |
| rules / memory | 影响模型怎么计划和说话 | 不是 | `AGENTS.md`、`CLAUDE.md`、Cursor rules |
| container / VM | 一类 sandbox 承载方式 | 是 | Docker、Podman、Firecracker、E2B、Daytona |
| external sandbox | 告诉 agent 外部环境已经隔离 | 取决于外部系统 | Dev container、远程 disposable workspace |

一个 coding agent 最危险的能力通常不是“能生成代码”，而是它拿到了这些能力：

- 读本机文件。
- 改本机文件。
- 执行 shell 命令。
- 启动子进程。
- 安装依赖和运行包管理器脚本。
- 访问网络。
- 访问本机凭据、SSH agent、浏览器 cookie、云厂商配置、`.env`。

所以真正的问题不是“模型会不会犯错”，而是：

> 当模型犯错、被 prompt injection 诱导、运行了恶意依赖脚本，或者用户疲劳地批准了危险命令时，系统能不能把损失限制在一个很小的范围内？

## Codex 的 sandbox 总体模型

OpenAI 官方文档把 Codex 的本地安全控制拆成两类：

- **sandbox**：技术边界，限制命令能读写哪里、能不能联网、能不能碰受保护路径。
- **approval policy**：交互策略，决定什么时候让用户批准越过边界的动作。

这两个概念经常一起出现，但它们不是一回事。比如：

- `sandbox_mode = "workspace-write"` 表示命令默认只能写 workspace 和配置的 writable roots。
- `approval_policy = "on-request"` 表示 Codex 需要越权时可以请求用户批准。
- `sandbox_mode = "danger-full-access"` 加 `approval_policy = "never"` 才接近“完全不拦”的危险模式。

官方文档里常见的三个 sandbox mode：

| 模式 | 大致含义 | 适合场景 | 风险 |
|---|---|---|---|
| `read-only` | 只读，不能改文件，网络通常受限 | 代码阅读、审查、解释 | 很安全，但开发效率低 |
| `workspace-write` | 可写当前 workspace 和配置的 writable roots，网络默认受限 | 默认开发、测试、改代码 | 需要小心 workspace 内的敏感文件和配置注入 |
| `danger-full-access` | 不使用 Codex 内置 OS sandbox | 已经在外部容器 / VM 里 | 裸机上风险很高 |

常见 approval policy：

| 策略 | 大致含义 |
|---|---|
| `untrusted` | 只有非常安全的只读命令自动执行，其他动作询问 |
| `on-request` | sandbox 内动作自动执行，越界动作请求批准 |
| `never` | 不问用户，通常应只和强外部 sandbox 搭配 |

`--full-auto` 在 Codex 文档里相当于一种低摩擦自动化预设：`workspace-write` sandbox 加 `on-request` approval。它不是“完全无限制”，而是“能在 workspace 内自动干活，越界再问”。

## Codex 的执行链路

把 Codex 本地 shell command 简化成一条链路，大致是：

```text
模型提出要执行命令
  ↓
Codex harness 根据 config / session / cwd 生成 permission profile
  ↓
判断是否需要 approval
  ↓
把原始命令包装成平台 sandbox 命令
  ↓
启动进程，并让子进程继承同一个 sandbox 边界
  ↓
捕获 stdout / stderr / exit code / sandbox denial
  ↓
把结果回传给模型，必要时让模型请求越权或改用别的做法
```

这条链路有几个关键点：

1. sandbox 主要约束 Codex 启动的本地命令及其子进程。
2. `git`、包管理器、测试 runner、构建脚本都会继承 sandbox。
3. Codex 内置工具、MCP tools、外部 app server 进程不一定自动处在同一个边界里，需要分别看实现。
4. OpenAI 在 agent loop 文章里特别提醒：Codex 给模型描述的 sandbox 边界主要适用于 Codex 提供的 shell tool，MCP tools 需要自己实现 guardrails。

所以不要把 “Codex 有 sandbox” 理解成“所有能调用的东西都天然被 Codex 接管”。对本地 coding agent 来说，MCP、浏览器自动化、外部 daemon、IDE 插件、云端 runner 都可能是另一个信任边界。

## Codex 的 policy 数据模型

从当前源码看，Codex 内部已经不只是简单的三个字符串 mode，而是把权限展开成更细的 profile：

```text
PermissionProfile
  ├─ FileSystemSandboxPolicy
  └─ NetworkSandboxPolicy
```

文件系统策略关注：

- 是否允许读整个磁盘。
- 哪些 roots 可读。
- 哪些 roots 可写。
- 哪些路径虽然在可写 root 内，但要强制只读。
- 哪些 glob 要不可读。
- 当前 workspace、额外 writable roots、用户 home、项目 metadata 如何处理。

网络策略关注：

- 是否完全允许网络。
- 是否完全限制网络。
- 是否走 managed proxy。
- 是否允许本机 loopback / local binding / Unix socket。

Codex 的 `workspace-write` 默认会把一些 metadata 路径保护起来，典型包括：

- `.git`
- `.codex`
- `.agents`

这是一个很重要的设计点。agent 如果能改 `.git/hooks`、agent 自己的配置、memory、rules、approval 配置，就可能通过“配置注入”影响未来会话，甚至把一次短暂越权变成长期后门。

## macOS：Seatbelt / sandbox-exec

Codex 在 macOS 上使用 Apple 的 Seatbelt sandbox，也就是 `/usr/bin/sandbox-exec` 背后的机制。

当前源码里的几个实现细节值得注意：

1. Codex 使用绝对路径 `/usr/bin/sandbox-exec`，而不是从 `PATH` 里找 `sandbox-exec`。这是为了避免 workspace 里放一个恶意同名二进制被优先执行。
2. Codex 动态生成 Seatbelt profile，并用 `-D` 参数把路径传进去。
3. 文件读写策略通过 Seatbelt 的 `file-read*`、`file-write*` 等能力表达。
4. `workspace-write` 会允许写 workspace / writable roots，同时把 `.git`、`.codex`、`.agents` 等 metadata 重新设为只读。
5. 对不可读 glob，Codex 会生成正则形式的 deny policy，用来限制 read 和 unlink。
6. 网络策略可以根据配置允许或限制 outbound / inbound / loopback / Unix socket / proxy 端口。

可以把 macOS 路径策略理解成：

```text
默认基线策略
  + 允许必要系统读取
  + 按 roots 放开读
  + 按 writable roots 放开写
  - 对受保护 metadata 重新 deny write
  - 对 unreadable globs deny read / unlink
  + 按网络配置放开网络能力
```

Seatbelt 的优势是：

- macOS 原生存在，不需要用户装 Docker。
- 能覆盖整个子进程树。
- policy 表达能力够 Codex / Claude Code / Cursor 这类工具使用。
- 启动成本比 VM / container 低。

局限也明显：

- `sandbox-exec` 是 Apple 标记为 deprecated 的命令行入口，虽然 2026 年仍然可用。
- 它是 path / profile 风格的 sandbox，不是完整虚拟机。
- 对很多本地系统服务、TCC 权限、钥匙串、Unix socket 等边界，需要非常小心地额外处理。
- 有些 path deny 会表现为 `EACCES`，并不能让文件“像不存在一样”完全隐身。

这也是为什么很多工具会把 Seatbelt 当作本机低成本 sandbox，而不是绝对安全边界。

## Linux / WSL2：bubblewrap + seccomp，Landlock legacy fallback

Codex Linux 的实现最容易被旧资料误解。基于 2026-05-29 的源码：

- 默认 filesystem sandbox 是 **bubblewrap**。
- 网络和部分危险 syscall 由 **seccomp** 处理。
- **Landlock** 仍在源码里，但主要作为 legacy fallback。
- WSL2 走 Linux sandbox 路线。
- WSL1 不支持当前 bubblewrap 路线。

### 为什么用 bubblewrap

`bubblewrap` 是 Flatpak 生态里常用的低层 sandbox 工具。它主要利用 Linux namespaces 构造一个新的挂载视图：

- user namespace
- mount namespace
- PID namespace
- network namespace
- IPC / UTS namespace
- 只读 bind mount
- 可写 bind mount
- tmpfs root
- seccomp filter

bubblewrap 自己不是“自动安全”的魔法盒子，它的安全性取决于传入的 mount / namespace / seccomp 参数。Codex 的工作就是根据 permission profile 生成一组相对保守的 bubblewrap 参数。

### Codex 的 Linux 执行序列

源码里的 Linux sandbox helper 大致分两阶段：

```text
codex-linux-sandbox 外层
  ↓
根据 permission profile 构造 bubblewrap 文件系统视图
  ↓
在 bubblewrap 内 re-exec 一个 inner helper
  ↓
inner helper 设置 PR_SET_NO_NEW_PRIVS
  ↓
安装 seccomp 网络 / syscall filter
  ↓
exec 用户原始命令
```

如果是完全 unrestricted filesystem 且网络完全允许，Codex 可能不需要 bubblewrap，只应用必要的 seccomp / no_new_privs 或直接执行。只要需要文件系统隔离、网络隔离或 proxy routing，就会走包装路径。

### 文件系统隔离策略

Codex Linux 当前实现的核心思路：

1. 默认把 `/` 以只读方式 bind 进去，或者在更收紧的策略里使用 `tmpfs /`，只挂载允许读的 roots。
2. 对 writable roots 使用 `--bind` 重新以可写方式挂载。
3. 对 writable root 内部需要保护的路径，再用 `--ro-bind` 覆盖回只读。
4. 对 `.git`、`.agents`、`.codex` 这类 metadata 做保护。
5. 对不可读 glob，启动前先展开匹配路径，再在 bubblewrap mount 视图里遮蔽。
6. 对 symlink-in-path 和不存在的受保护路径做额外处理，避免通过 symlink 或“先创建后利用”的方式绕过。

可以抽象成：

```text
read view:
  /                 -> read-only
  readable roots    -> read-only

write view:
  cwd / writable roots -> read-write

re-protect:
  .git / .codex / .agents / protected subpaths -> read-only or masked
  unreadable globs -> masked
```

这类 mount-view sandbox 的好处是，进程看到的文件系统本身已经被改造了。相比“进程每次访问时再问 supervisor”，它更接近静态内核边界，性能和稳定性更好。

### 网络隔离策略

Codex Linux 把网络分成几个模式：

| 模式 | 大致含义 |
|---|---|
| `FullAccess` | 不隔离网络 |
| `Isolated` | 使用新的 network namespace，基本没有外部网络 |
| `ProxyOnly` | 网络 namespace 内只允许走 Codex 管理的 proxy / bridge |

在 restricted / proxy 场景下，bubblewrap 会使用 `--unshare-net` 创建独立网络 namespace。然后 Codex 再用 seccomp 限制进程创建某些 socket 或绕过 proxy。

源码里的 seccomp 还会限制一些高风险 syscall，例如：

- `ptrace`
- `process_vm_readv`
- `process_vm_writev`
- `io_uring_setup`
- `io_uring_enter`
- `io_uring_register`

这些限制不是文件读写策略的替代品，而是减少进程逃逸、窥探其他进程或绕过网络策略的攻击面。

### no_new_privs 的意义

Linux sandbox 里经常会看到 `PR_SET_NO_NEW_PRIVS`。它的意思是：当前进程及其子进程不能通过 `execve` 获得新的特权。

这可以防止一些 setuid / file capability 路径让 sandboxed command 获得超出预期的权限。Landlock 和 seccomp 这类机制也通常要求或强烈依赖 `no_new_privs`。

### Landlock 为什么仍然值得理解

Landlock 是 Linux 的 unprivileged LSM sandbox。它允许普通进程给自己和子进程加一层不可撤销的访问限制，典型用于：

- 限制文件系统访问。
- 在较新 ABI 里限制 TCP bind / connect 端口。
- 让限制随子进程继承。

Landlock 的优点是它是内核 LSM，概念上很适合“进程主动给自己降权”。但在 coding agent 场景里，它也有一些现实约束：

- ABI 版本和发行版支持差异较大。
- 早期 ABI 主要覆盖文件系统，网络能力较晚才加入。
- 对“隐藏一整套复杂路径视图”的表达不如 mount namespace 直观。
- 对 unreadable glob、受保护子路径、workspace 可写叠加只读子路径这类需求，实现上会更绕。

所以 Codex 当前把 bubblewrap 作为默认 FS sandbox，并保留 Landlock fallback，是一个比较务实的选择。

## Windows：restricted token、ACL、WFP

Windows 是最值得单独看的部分，因为它没有一个和 `sandbox-exec` 或 `bubblewrap` 完全对应的简单入口。OpenAI 在 2026-05-13 发布的 Windows sandbox 技术博客里解释了取舍。

他们评估过几条路线：

| 方案 | 为什么不理想 |
|---|---|
| AppContainer | 对开放式开发工作流太窄，很多 CLI / 构建工具不适配 |
| Windows Sandbox | 更像轻量 VM，和 host workspace 集成差，而且 Windows Home 不一定可用 |
| Mandatory Integrity Control | 需要给真实文件系统打 label，容易产生过宽或难回收的状态 |

Codex 最终走的是 Windows 原生访问控制路线：

- restricted token
- write-restricted token
- capability SID / synthetic SID
- ACL allow / deny rule
- 专用 sandbox account
- Windows Filtering Platform

### restricted token 的核心思路

Windows access check 不只是看“用户是谁”，还看 token 里有哪些 SID、哪些 privilege、哪些 restricted SID。Codex 可以用 `CreateRestrictedToken` 创建一个降权 token：

- 禁用大部分 privilege。
- 使用 `LUA_TOKEN` 类似 UAC 降权。
- 使用 `WRITE_RESTRICTED`。
- 加入用于 sandbox 的 synthetic SID / capability SID。
- 只给特定目录 ACL 授予写权限。

一个直观模型是：

```text
真实用户 token
  ↓ CreateRestrictedToken
降权 token
  + sandbox capability SID
  + limited privileges
  + write-restricted behavior
  ↓
只在 ACL 明确允许的 roots 内可写
```

这样做的好处是，进程仍然能作为“当前用户语境”运行很多开发工具，但写权限被缩小到了 workspace / writable roots。

### ACL 保护 workspace 内部路径

仅仅允许 workspace 可写还不够，因为 workspace 里有些路径不应该让 agent 改：

- `.git`
- `.codex`
- `.agents`
- 未来可能还有 hooks、agent config、memory、secrets cache 等。

Windows 实现会通过 ACL 规则给允许路径加 allow ACE，同时给受保护路径加 deny / read-only 规则。相比 macOS / Linux 的 mount 或 Seatbelt profile，Windows 这边更像是在 token 和 NTFS ACL 的交叉点上构造边界。

### elevated sandbox 与专用账号

当前 Windows 源码还包含更完整的 elevated sandbox setup：

- 创建或刷新 sandbox 相关目录。
- 使用 `CodexSandboxOffline` / `CodexSandboxOnline` 这类专用账号。
- 设置 platform read roots，例如 `C:\Windows`、`Program Files`、`ProgramData`。
- 保护用户 profile 下常见敏感目录，例如 `.ssh`、`.aws`、`.azure`、`.kube`、`.docker`、`.gnupg` 等。
- 通过 command runner / IPC 启动受限进程。

这说明 Windows 的“强一点”的实现更接近“专用低权限身份 + ACL 文件系统视图 + 网络过滤”的组合，而不是单个 API 调用。

### 网络：WFP 与 proxy

Windows 不能简单照搬 Linux network namespace。Codex Windows 源码里出现了 Windows Filtering Platform 相关实现，用 provider / sublayer / filter 针对 sandbox account 施加网络规则。

当前源码中的过滤规则包括阻断一些典型通道：

- ICMP v4 / v6
- DNS 53
- DNS-over-TLS 853
- SMB 445 / 139

再结合 offline / online sandbox account、proxy port、环境变量等机制，形成“默认受限，必要时经受控通道出去”的模式。

这里的设计重点不是“把 Windows 伪装成 Linux”，而是用 Windows 自己的安全模型表达同一个目标：

> 命令可以像正常 CLI 一样跑，但它的 token、ACL、网络过滤都已经降权。

## Codex Cloud 和 App Server 的边界

Codex 不只有本地 CLI。还有云端 coding agent、IDE / app server 等形态。

官方资料里提到：

- Codex cloud task 运行在隔离容器里。
- 云端默认禁用网络，除非用户配置。
- App server 里的 `command/exec` 可以带 `sandboxPolicy`，例如 `workspaceWrite`。
- App server 里的某些 process API 是 unsandboxed，需要调用方自己负责边界。
- `externalSandbox` 可以告诉 Codex 外部已经有 sandbox，不再由 Codex 自己套 OS sandbox。

这给我们一个很重要的工程判断：

> “Codex”不是一个单一执行环境。CLI、本地 app、IDE、云端 task、MCP、app server 的边界不完全一样。调研某个安全能力时，必须问清楚是哪个入口、哪个平台、哪个 command path。

## Claude Code：Seatbelt / bubblewrap 的开源 sandbox runtime

Anthropic 在 2026 年公开写过 Claude Code sandboxing。它的设计和 Codex 非常相近：

- macOS 使用 Seatbelt。
- Linux 使用 bubblewrap。
- 同时强调 filesystem isolation 和 network isolation。
- sandbox bash tool 以及所有脚本 / 子进程。
- 内部使用 sandbox 后，Claude Code 因 permission prompt 停下来的次数减少了约 84%。

Claude Code 文档里还能看到更细的配置面：

- `sandbox.enabled`
- `failIfUnavailable`
- `autoAllowBashIfSandboxed`
- `allowWrite`
- `denyWrite`
- `allowRead`
- `denyRead`
- `network.allowedDomains`
- `network.deniedDomains`
- `allowUnsandboxedCommands`
- `excludedCommands`
- `bwrapPath`
- `socatPath`

Anthropic 的一个判断很值得记下来：

> 只做文件系统隔离不够，因为被 compromise 的 agent 可以把数据从网络发出去；只做网络隔离也不够，因为恶意命令可以改文件或留下后门。coding agent 需要同时考虑 filesystem 和 network。

这和 Codex 的 `workspace-write + restricted network` 默认方向是一致的。

## Cursor：动态 policy、Landlock + seccomp、WSL2

Cursor 也公开写过 agent sandboxing。它的实现路径：

- macOS：评估 App Sandbox、container、VM、Seatbelt 后，选择 Seatbelt / `sandbox-exec`。
- Linux：使用 Landlock + seccomp。
- Windows：通过 WSL2 运行 Linux sandbox，而不是先做完整 Windows native sandbox。
- policy 会结合 workspace、管理员配置和 `.cursorignore`。
- agent 的系统提示和 shell description 会告诉模型 sandbox 约束。
- 当 sandbox 拒绝某个操作时，错误会被明确渲染给 agent，引导 agent 请求 escalation 或换方案。

Cursor 报告的产品结果是：sandboxed agents 停下来的次数少约 40%。这说明 sandbox 不只是安全功能，也是体验功能。只要边界设计得好，agent 不需要对每个 `mkdir`、`npm test`、`go test` 都问用户。

Cursor 和 Codex 的差异主要在 Linux：

- Codex 当前默认是 bubblewrap + seccomp。
- Cursor 博客描述的是 Landlock + seccomp。

两者都合理，只是工程取舍不同：

- Landlock 更像“进程给自己套 LSM policy”。
- bubblewrap 更像“构造一个新的 mount / namespace 视图”。

## Gemini CLI：Seatbelt 与 Docker / Podman

Gemini CLI 官方 sandbox 文档里列了几种启用方式：

- `-s` / `--sandbox`
- `GEMINI_SANDBOX=true`
- `GEMINI_SANDBOX=docker`
- `GEMINI_SANDBOX=podman`
- `GEMINI_SANDBOX=sandbox-exec`
- settings 里的 `tools.sandbox`

它的路线更偏“多后端可选”：

| 后端 | 平台 | 特点 |
|---|---|---|
| Seatbelt / `sandbox-exec` | macOS | 轻量、本机、profile 可切换 |
| Docker | macOS / Linux / Windows | 隔离更强，依赖容器环境 |
| Podman | macOS / Linux / Windows | 类似 Docker，daemonless 体验更好 |

Gemini CLI 的 Seatbelt profile 还区分：

- `permissive-open`
- `permissive-closed`
- `permissive-proxied`
- `restrictive-open`
- `restrictive-closed`

名字里的 `open / closed / proxied` 主要对应网络策略，`permissive / restrictive` 主要对应文件系统策略。

这类设计适合一个事实：不同用户对“方便”和“隔离”的偏好差别很大。研究、开源项目、企业代码库、带生产凭据的本机环境，不应该用同一档 sandbox。

## OpenCode、Aider、Goose：权限系统不等于 OS sandbox

OpenCode 文档有比较完整的 permission 配置：

- `bash`
- `edit`
- `read`
- `webfetch`
- `external_directory`
- allow / ask / deny
- command pattern
- `.env` 等敏感文件默认 deny read

但这更像 agent permission / approval 层，不等同于 OS-level sandbox。公开资料里也能看到用户询问 OpenCode 是否有类似 Codex / Gemini 的 macOS Seatbelt sandbox，而官方文档主线仍然是 permission 配置。

Aider 的核心安全模型更偏：

- diff 驱动修改。
- git commit / undo。
- 用户确认变更。
- 可以用 Docker 运行。

这对可恢复性和审查很有帮助，但也不是本机 OS sandbox 的替代品。

Goose、OpenHands、各种本地 / 云端 agent runner 也类似：有些提供权限提示，有些鼓励 Docker / remote workspace，有些由企业平台提供隔离。判断时不要只看“会不会问我确认”，而要看：

- 命令是否真的在低权限环境中执行。
- 子进程是否继承边界。
- 网络是否被内核 / 容器 / 防火墙约束。
- 敏感文件是否被 OS 层 deny，而不是仅靠模型“不要读”。

## 主流 coding CLI sandbox 对比

| 工具 | macOS | Linux | Windows | 网络隔离 | 备注 |
|---|---|---|---|---|---|
| Codex CLI | Seatbelt / `sandbox-exec` | bubblewrap + seccomp，Landlock legacy fallback | restricted token / ACL / WFP | restricted / proxy / full | OS-native 跨平台实现，源码公开 |
| Claude Code | Seatbelt | bubblewrap + seccomp | 依赖平台实现和配置 | 支持 allow / deny / proxy | sandbox runtime 开源，配置面较细 |
| Cursor | Seatbelt | Landlock + seccomp | WSL2 Linux sandbox | 支持策略化限制 | policy 结合 workspace / admin / `.cursorignore` |
| Gemini CLI | Seatbelt 或 Docker / Podman | Docker / Podman 等 | Docker / Podman 等 | profile 区分 open / closed / proxied | 多后端可选 |
| OpenCode | 未见官方 OS sandbox 主线 | 未见官方 OS sandbox 主线 | 未见官方 OS sandbox 主线 | permission 层为主 | 可用外部 wrapper / container |
| Aider | Docker / 外部隔离 | Docker / 外部隔离 | Docker / 外部隔离 | 取决于外部环境 | diff / git workflow 强，但不是 sandbox |
| Hosted agents | 容器 / VM / microVM | 容器 / VM / microVM | 云端隔离环境 | 通常默认关或 allowlist | E2B、Daytona、Modal、Firecracker 等路线 |

这个表不是“谁更安全”的排名。更准确的读法是：

- Codex / Claude / Cursor 代表本地 CLI 直接集成 OS sandbox。
- Gemini CLI 代表本地 Seatbelt + 容器后端的混合方案。
- OpenCode / Aider 更依赖 permission、diff、git、外部容器或用户自己的 wrapper。
- 云端 agent 更像把问题推到 disposable workspace / container / VM 上。

## OS sandbox 原语速查

### Seatbelt

Seatbelt 是 macOS 的 sandbox policy 机制。`sandbox-exec` 可以通过 profile 描述允许或拒绝：

- 文件读写。
- 网络。
- 进程相关能力。
- Mach / Unix socket 等系统资源。

优点：

- macOS 原生可用。
- 启动轻。
- 很适合包装本地 CLI 命令。

缺点：

- `sandbox-exec` 命令入口 deprecated。
- policy 语义和系统服务交互复杂。
- 不等同于 VM。

### bubblewrap

bubblewrap 是基于 Linux namespaces 的低层 sandbox 工具。它常用于 Flatpak，也适合 coding agent 这种“临时启动一个受限命令”的场景。

常见能力：

- 新 mount namespace。
- 新 user namespace。
- 新 PID namespace。
- 新 network namespace。
- 只读 bind mount。
- 可写 bind mount。
- tmpfs root。
- seccomp filter。

优点：

- 能构造清晰的文件系统视图。
- 对子进程天然继承。
- 不需要完整 Docker daemon。

缺点：

- 依赖 unprivileged user namespace，某些发行版 / AppArmor 配置会挡。
- 安全性高度依赖参数。
- 如果把敏感 socket、D-Bus、host path 挂进去，边界会变弱。

### Landlock

Landlock 是 Linux 内核提供的 unprivileged LSM。应用可以创建 ruleset，然后对自己和子进程施加不可撤销的访问限制。

优点：

- 内核级、非特权、可叠加。
- 很适合“进程主动降权”。
- 较新 ABI 支持部分网络限制。

缺点：

- ABI 版本差异需要处理。
- 复杂 path view / glob / overlay 需求不如 mount namespace 直接。
- 不能单独解决所有 network / IPC / syscall 风险。

### seccomp

seccomp-bpf 用来过滤 Linux syscall。它很适合限制：

- 创建 socket。
- `ptrace`。
- `process_vm_readv/writev`。
- `io_uring`。
- 其他高风险 syscall。

但 seccomp 不适合表达“这个路径能读，那个路径不能读”，因为 syscall filter 通常看不到完整 path 语义。它更适合作为 filesystem sandbox 的补充。

### Windows restricted token / ACL / WFP

Windows 的思路和 Unix 不一样：

- token 决定进程身份、SID、privilege。
- ACL 决定对象允许哪些 SID 做什么。
- restricted token 可以让进程即使是同一个用户，也被降权。
- WFP 可以按用户 / 协议 / 端口过滤网络。

优点：

- 原生 Windows 安全模型。
- 不需要强行要求 Docker / WSL。
- 可以和 NTFS ACL 细粒度结合。

缺点：

- 实现复杂。
- ACL 状态管理、回滚、继承、deny 优先级都容易出错。
- 很多开发工具对“低权限 token + 特殊 ACL”组合没有专门适配。

### Docker / Podman / dev container

容器是 coding agent sandbox 里最容易理解的一类：

- 文件系统隔离。
- 网络 namespace。
- cgroups。
- capabilities。
- seccomp / AppArmor / SELinux。
- 镜像可复现。

优点：

- 跨平台心智模型更统一。
- 适合高风险任务和 disposable workspace。
- 企业和云端平台容易管理。

缺点：

- 本地开发集成成本高。
- 需要处理 volume mount、UID/GID、SSH、包缓存、浏览器、IDE、数据库等体验问题。
- 如果把 host home、Docker socket、SSH agent 直接挂进去，隔离会大幅削弱。

### VM / microVM

Firecracker、Kata、完整 VM、云端 ephemeral VM 是更强的隔离层。

优点：

- 内核级隔离更强。
- 适合运行不可信代码。
- 云端可以配合快照、重建、审计。

缺点：

- 启动和资源成本更高。
- 本地编辑体验更复杂。
- 文件同步、凭据、网络、调试都需要额外产品化。

## 一套通用的 coding agent sandbox 设计框架

从 Codex、Claude Code、Cursor、Gemini CLI 和相关论文看，一个比较完整的 coding agent sandbox 至少包括 8 层。

### 1. 文件系统边界

最低要求：

- workspace 可写。
- workspace 外默认不可写。
- 常见系统目录只读。
- 用户 home 下敏感目录不可读或只读。
- agent 自身配置和 metadata 只读。
- `.git` 和 hooks 受保护。

推荐进一步做：

- 支持 `writable_roots`。
- 支持 `readable_roots`。
- 支持 `deny_read` / unreadable globs。
- 支持 symlink 防绕过。
- 支持 missing path 防“先创建后利用”。

### 2. 网络边界

最低要求：

- 默认禁止 outbound。
- 明确允许包管理器 registry、Git remote、公司内网、OpenAI / Anthropic API 等需要的域名。
- 禁止访问 metadata service、localhost 敏感服务、SSH agent、Docker socket 等。

更成熟的实现：

- network namespace。
- managed proxy。
- 域名 allowlist / denylist。
- DNS 代理。
- OpenTelemetry 记录网络事件。
- 按 approval 临时放开。

### 3. 进程树继承

sandbox 必须覆盖整棵进程树。

危险模式是：

```text
agent shell 被限制
  ↓
npm install 启动 postinstall
  ↓
postinstall 子进程逃出限制
```

如果子进程不继承边界，sandbox 就会在真实 coding workflow 里失效。Codex / Claude / Cursor 都强调命令和子进程要在同一个边界里。

### 4. 系统调用和进程窥探限制

这层主要是减少逃逸面：

- `ptrace`
- `process_vm_readv/writev`
- `io_uring`
- 某些 raw socket / packet socket。
- 某些 namespace / mount / keyring syscall。

它不能替代文件系统策略，但能降低“读别的进程内存”“绕过 socket 限制”“利用内核攻击面”的风险。

### 5. 配置和 metadata 防篡改

这是 coding agent 特有的重要层。

应该保护：

- `.git`
- `.git/hooks`
- `.codex`
- `.agents`
- `.claude`
- `.cursor`
- agent memory 文件。
- approval / permission 配置。
- MCP server 配置。
- shell profile。
- package manager token config。

原因是 agent 很容易被诱导“为了完成任务，先改一下自己的配置”。一旦配置可写，一次 prompt injection 就可能变成长期驻留。

### 6. Approval 与可解释 denial

sandbox 拦住动作后，系统应该给 agent 和用户一个清楚的错误：

```text
命令失败，因为 sandbox 禁止写入 /Users/name/.ssh/config
当前 writable roots: /repo
如确实需要，请请求用户批准或改写到 workspace 内
```

这比单纯返回 `Permission denied` 好很多，因为模型可以据此调整计划。

好的 approval 系统应该支持：

- approve once。
- approve for session。
- deny。
- explain why。
- 自动批准低风险动作。
- 对高风险动作强制人工批准。

OpenAI 的 `auto_review` 和 Claude / Cursor 的 sandbox prompt reduction，本质上都是在解决“安全和流畅性”的张力。

### 7. 外部工具边界

MCP、浏览器、数据库、IDE、云端 runner、app server、local daemon 都可能绕开 shell sandbox。

设计时要逐一问：

- 这个 tool 是不是由 agent harness 启动？
- 它是否继承同一个 sandbox？
- 它能不能读 host 文件？
- 它能不能联网？
- 它能不能拿到用户 token？
- 它的日志和结果是否可审计？

如果一个 MCP server 自己可以任意读写文件，那么 Codex shell sandbox 再严格也挡不住它。

### 8. 可观测性和审计

对个人用户来说，日志能帮助 debug。对团队来说，日志是安全审计基础。

应该记录：

- 命令。
- cwd。
- sandbox profile。
- approval 决策。
- 被拒绝的路径和网络目标。
- tool call。
- MCP call。
- 网络 proxy 事件。
- 文件改动摘要。

OpenAI 内部 Codex 使用经验里提到用 OpenTelemetry 记录 prompts、approvals、tool results、MCP events、network proxy events，这就是把 agent sandbox 当成工程系统而不是单个功能来管理。

## 论文和研究视角

### Sandlock：面向 AI agent 的 Linux sandbox

`Sandlock: Confining AI Agent Code with Unprivileged Linux Primitives` 是一篇 2026 年的 agent sandbox 相关论文。它的出发点非常贴近 coding agent：

- AI agent 会动态生成和执行代码。
- 传统容器隔离较重。
- 只靠 prompt / approval 不可靠。
- 需要低开销、可组合、非特权的内核边界。

Sandlock 的设计组合包括：

- Landlock 做静态文件系统 / TCP / IPC 策略。
- seccomp-bpf / user notification 做动态决策。
- copy-on-write / reversible filesystem effects。
- HTTP-level access control。

论文报告的方向是：接近裸机性能，启动开销约毫秒级，比 Docker setup 更轻。这类研究说明，未来 coding agent sandbox 很可能不会只在“Docker vs 无 sandbox”之间二选一，而是会出现更多专门面向 agent workload 的轻量内核策略层。

### Reframing LLM Agent Security as Agent-Human Interaction

这类研究强调：LLM agent security 不只是模型推理问题，也是不可靠 agent 和人类之间的交互协议问题。

对 sandbox 的启发是：

- 高可靠的约束应该尽量交给 OS / runtime enforcement。
- 人类表达的 policy 应该被系统直接执行，而不是完全交给模型理解。
- approval UI 必须减少歧义和疲劳。
- agent 的解释不能代替强制边界。

### Engineering Pitfalls in AI Coding Tools

`Engineering Pitfalls in AI Coding Tools` 系统分析了 Claude Code、Codex、Gemini CLI 等工具公开 issue 中的工程问题。它对 sandbox 的意义在于：coding agent 的故障高发点并不只在模型回答质量，而是在 tool invocation、command execution、环境集成、权限与状态管理这些 harness 层。

这说明 sandbox 文档不能只写“用了什么 OS API”，还要覆盖：

- 命令如何包装。
- 错误如何回传给模型。
- permission denial 会不会被误判成普通命令失败。
- agent 会不会在失败后反复尝试更危险的路径。

### Overeager Coding Agents

`Overeager Coding Agents: Measuring Out-of-Scope Actions on Benign Tasks` 把问题定义为 authorization，而不只是 capability 或 prompt injection。它关注 agent 在 benign task 中做出超出用户授权范围的动作。

这对 sandbox 的启发是：

- sandbox 不只是防恶意代码，也是在约束 agent 的任务边界。
- approval UI 应该表达“这个动作是否仍属于当前任务”，不只是“这个命令是否能执行”。
- framework / harness 的默认策略会显著影响越界行为。

### Capsicum、seccomp、container hardening

传统 OS sandbox 论文虽然不是为 coding agent 写的，但概念仍然很有用：

- Capsicum 强调 capability mode，把进程降到只能访问已授予 capability 的资源。
- seccomp-bpf 强调 syscall 级过滤，减少内核攻击面。
- container hardening 研究强调 namespaces、capabilities、cgroups、seccomp、LSM 组合，而不是只开 Docker 就万事大吉。

coding agent sandbox 本质上是在把这些老问题放进一个新场景：

```text
过去：用户主动运行不可信程序
现在：模型可能替用户运行不完全理解的程序
```

## 常见绕过和失败模式

### 1. 只做 approval，不做 sandbox

模型被 prompt injection 诱导后，可能会把危险命令解释得很合理。用户在连续批准几十个正常命令后，也容易批准一个危险命令。

approval 是必要的人机协作层，但不是强安全边界。

### 2. 只保护 workspace 外，不保护 workspace 内 metadata

如果 agent 能改：

- `.git/hooks`
- `.codex/config`
- `.agents`
- `.claude/settings`
- `.cursor/rules`
- package manager scripts

那么它可能改变下一次会话的行为。这是 coding agent 特有的持久化攻击面。

### 3. 只禁网络，不禁文件

恶意命令虽然不能 exfiltrate，但仍然能：

- 删除代码。
- 修改配置。
- 植入后门。
- 改测试让问题看起来通过。
- 改 agent rules。

### 4. 只禁文件，不禁网络

恶意命令可能读取允许范围内的源码、业务逻辑、测试数据，然后发到外部服务。很多 prompt injection 攻击真正想要的就是 exfiltration。

### 5. 忽略 package manager scripts

`npm install`、`pip install`、`cargo build`、`go generate`、`make test` 都可能运行项目提供的脚本。coding agent 不是只执行你看到的那一行命令，而是会触发一串子进程。

### 6. PATH 劫持

如果 sandbox wrapper、shell、git、node 等关键命令从不可信 `PATH` 解析，workspace 里放一个同名文件就可能劫持执行链。Codex macOS 使用 `/usr/bin/sandbox-exec` 绝对路径就是在规避这一类问题。

### 7. symlink / mount / glob 边界错误

文件系统 sandbox 常见坑：

- writable root 内放 symlink 指向外部敏感路径。
- protected path 启动时不存在，命令运行时创建。
- glob 展开不完整。
- bind mount 顺序错误，先保护后又被 writable mount 覆盖。
- `.git` 是 gitdir pointer，真实 gitdir 在 workspace 外。

Codex Linux README 里专门提到 symlink-in-path、non-existent protected paths、resolved gitdir 等处理，说明这些都不是理论问题。

### 8. MCP / external tool 绕过

如果 shell 被 sandbox，但 MCP server 可以直接读文件或发网络请求，攻击者会诱导 agent 改用 MCP。agent harness 必须把所有 tool 都纳入 threat model。

## 对个人使用 Codex CLI 的建议

如果是在自己的主力电脑上跑 Codex CLI，我会优先用这个心智模型：

```text
默认：workspace-write + on-request
高风险：外部 container / VM + danger-full-access
低信任仓库：read-only 或 workspace-write + no network
需要联网：临时批准或配置 allowlist / proxy
```

实践建议：

1. 不要在裸机上长期使用 `danger-full-access + never`。
2. 只有在外部 dev container / VM / disposable workspace 里，才考虑关闭 Codex 内置 sandbox。
3. 把 `writable_roots` 控制得越少越好。
4. 不要把 home 目录、SSH key、云厂商凭据目录直接暴露给 agent。
5. 对陌生仓库先用 read-only 让 agent 阅读和计划。
6. 运行安装脚本、迁移脚本、下载脚本、curl pipe shell 前，把网络和写权限想清楚。
7. 把项目规则写进 `AGENTS.md`，但不要把它当安全边界。
8. 用 `codex sandbox macos ...` 或 `codex sandbox linux ...` 这类测试命令验证本机 sandbox 行为。

一个比较稳的个人配置方向：

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"

[sandbox_workspace_write]
writable_roots = []
network_access = false
```

如果需要更自动化，可以考虑 OpenAI 文档里的 auto-review 路线：

```toml
approval_policy = "on-request"
approvals_reviewer = "auto_review"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
writable_roots = ["~/development"]
```

但这里仍然要看你的实际环境。如果 `~/development` 下面有很多公司仓库、密钥、私有材料，把整个目录设为 writable root 就不一定合适。

## 对 coding agent 产品 / runtime 设计的建议

如果要设计一个 coding CLI 的 sandbox，我会按下面顺序做。

### MVP

1. 默认 workspace-write。
2. workspace 外不可写。
3. 网络默认关。
4. 命令和所有子进程继承 sandbox。
5. `.git`、agent config、agent memory 只读。
6. denial 信息清楚反馈给模型和用户。
7. approval 支持 once / session / deny。

### 进阶

1. macOS Seatbelt。
2. Linux bubblewrap + seccomp，必要时支持 Landlock。
3. Windows restricted token + ACL，网络用 WFP 或受控 proxy。
4. 支持 readable / writable / unreadable roots。
5. 支持 network allowlist / proxy。
6. 支持 OpenTelemetry / audit log。
7. MCP tools 也接入统一 permission model。

### 高安全场景

1. disposable container / VM / microVM。
2. 无 host home mount。
3. 无 host SSH agent / Docker socket。
4. 短期 token。
5. 仓库 clone 到临时 workspace。
6. 任务结束销毁环境。
7. 所有网络走 egress proxy。
8. 记录完整 command / file / network audit trail。

## 最后用一句话概括

Codex CLI 的 sandbox 不是“模型自觉听话”，而是 harness 把模型要执行的命令转换成平台原生的低权限进程：macOS 用 Seatbelt profile，Linux / WSL2 用 bubblewrap mount namespace 加 seccomp，Windows 用 restricted token、ACL 和网络过滤。整个 coding CLI 生态也在朝同一个方向收敛：把权限审批留给人机协作，把真正的安全边界交给 OS、container、VM 和可审计的 runtime。

## 参考资料

### OpenAI / Codex

- [OpenAI Developers: Codex sandboxing](https://developers.openai.com/codex/concepts/sandboxing)
- [OpenAI Developers: Agent approvals and security](https://developers.openai.com/codex/agent-approvals-security)
- [OpenAI Developers: Codex permissions](https://developers.openai.com/codex/permissions)
- [OpenAI Developers: Codex app server](https://developers.openai.com/codex/app-server)
- [OpenAI: Running Codex safely at scale](https://openai.com/index/running-codex-safely/)
- [OpenAI: Building the Codex Windows sandbox](https://openai.com/index/building-codex-windows-sandbox)
- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop)
- [OpenAI: GPT-5-Codex system card](https://cdn.openai.com/pdf/97cc5669-7a25-4e63-b15f-5fd5bdc4d149/gpt-5-codex-system-card.pdf)
- [OpenAI: GPT-5.3-Codex system card](https://deploymentsafety.openai.com/gpt-5-3-codex/gpt-5-3-codex.pdf)
- [openai/codex source snapshot: codex-rs](https://github.com/openai/codex/tree/1c55bb2702fa4fbbccbf21057d344240744b5041/codex-rs)
- [openai/codex: core README sandbox notes](https://github.com/openai/codex/blob/1c55bb2702fa4fbbccbf21057d344240744b5041/codex-rs/core/README.md)
- [openai/codex: Linux sandbox README](https://github.com/openai/codex/blob/1c55bb2702fa4fbbccbf21057d344240744b5041/codex-rs/linux-sandbox/README.md)
- [openai/codex: macOS Seatbelt implementation](https://github.com/openai/codex/blob/1c55bb2702fa4fbbccbf21057d344240744b5041/codex-rs/sandboxing/src/seatbelt.rs)
- [openai/codex: Linux bubblewrap implementation](https://github.com/openai/codex/blob/1c55bb2702fa4fbbccbf21057d344240744b5041/codex-rs/linux-sandbox/src/bwrap.rs)
- [openai/codex: Linux seccomp / legacy Landlock implementation](https://github.com/openai/codex/blob/1c55bb2702fa4fbbccbf21057d344240744b5041/codex-rs/linux-sandbox/src/landlock.rs)
- [openai/codex: Windows sandbox crate](https://github.com/openai/codex/tree/1c55bb2702fa4fbbccbf21057d344240744b5041/codex-rs/windows-sandbox-rs)

### 其他 coding agent / coding CLI

- [Anthropic Engineering: Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Anthropic Engineering: How we contain Claude Code](https://www.anthropic.com/engineering/how-we-contain-claude)
- [Anthropic Docs: Claude Code settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [anthropic-experimental/sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime)
- [Cursor Blog: Agent sandboxing](https://cursor.com/blog/agent-sandboxing)
- [Gemini CLI Docs: Sandbox](https://google-gemini.github.io/gemini-cli/docs/cli/sandbox.html)
- [OpenCode Docs: Permissions](https://opencode.ai/docs/permissions)
- [OpenCode issue: question about sandboxing](https://github.com/anomalyco/opencode/issues/2242)
- [Aider Docs: Usage with Docker](https://aider.chat/docs/usage/docker.html)
- [Rivet sandbox-agent](https://github.com/rivet-dev/sandbox-agent)
- [NAV IT cplt](https://github.com/navikt/cplt)
- [Sandvault](https://github.com/webcoyote/sandvault)
- [AixGate](https://github.com/aixgo-dev/aixgate)

### OS primitives / 安全基础

- [Linux kernel docs: Landlock](https://docs.kernel.org/userspace-api/landlock.html)
- [Landlock project](https://landlock.io/)
- [containers/bubblewrap](https://github.com/containers/bubblewrap)
- [Linux kernel docs: seccomp BPF](https://docs.kernel.org/userspace-api/seccomp_filter.html)
- [Apple Developer: App Sandbox](https://developer.apple.com/documentation/security/app_sandbox)
- [Reverse Apple Sandbox Design Guide](https://reverse.put.as/wp-content/uploads/2011/09/Apple-Sandbox-Guide-v1.0.pdf)
- [Microsoft Learn: CreateRestrictedToken](https://learn.microsoft.com/en-us/windows/win32/api/securitybaseapi/nf-securitybaseapi-createrestrictedtoken)
- [Microsoft Learn: Windows Filtering Platform](https://learn.microsoft.com/en-us/windows/win32/fwp/windows-filtering-platform-start-page)

### 论文 / 研究

- [Sandlock: Confining AI Agent Code with Unprivileged Linux Primitives](https://arxiv.org/abs/2605.26298)
- [Reframing LLM Agent Security as Agent-Human Interaction](https://arxiv.org/abs/2605.24309)
- [Engineering Pitfalls in AI Coding Tools: An Empirical Study of Bugs in Claude Code, Codex, and Gemini CLI](https://arxiv.org/abs/2603.20847)
- [Overeager Coding Agents: Measuring Out-of-Scope Actions on Benign Tasks](https://arxiv.org/abs/2605.18583)
- [Capsicum: practical capabilities for UNIX](https://www.usenix.org/legacy/event/sec10/tech/full_papers/Watson.pdf)
- [Confine: Automated System Call Policy Generation for Container Attack Surface Reduction](https://www.usenix.org/conference/raid2020/presentation/ghavamnia)

### 安全案例和社区讨论

- [Ona: How Claude Code escapes its own denylist and sandbox](https://ona.com/stories/how-claude-code-escapes-its-own-denylist-and-sandbox)
- [Cymulate: The race to ship AI tools left security behind](https://cymulate.com/blog/the-race-to-ship-ai-tools-left-security-behind-part-1-sandbox-escape)
