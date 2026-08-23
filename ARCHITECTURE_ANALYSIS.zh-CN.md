# Codex CLI 技术架构分析

> 分析对象：`openai/codex` 仓库（本仓库），核心为 `codex-rs/` 下的 Rust 工作区（150+ crate）。
> 生成日期：2026-08-23。所有路径均相对仓库根目录。

---

## 目录

1. [总体概览](#1-总体概览)
2. [顶层架构设计](#2-顶层架构设计)
3. [核心引擎 codex-core](#3-核心引擎-codex-core)
4. [协议层 protocol](#4-协议层-protocol)
5. [服务层 app-server / exec-server / mcp-server](#5-服务层)
6. [入口层 cli / tui / exec](#6-入口层)
7. [沙箱与安全体系](#7-沙箱与安全体系)
8. [模型接入层](#8-模型接入层)
9. [配置系统](#9-配置系统)
10. [持久化：rollout / SQLite / thread-store](#10-持久化)
11. [扩展体系：extension / plugin / skills / hooks / MCP / code-mode](#11-扩展体系)
12. [可观测性与其他支撑设施](#12-可观测性与支撑设施)
13. [构建系统与发布](#13-构建系统与发布)

---

## 1. 总体概览

Codex 是 OpenAI 的本地编码 Agent（CLI / IDE 插件 / 桌面 App 共用一套引擎）。产品代码几乎全部为 Rust：

- **单一多态二进制**：`codex` 二进制通过 clap 子命令 + **arg0 技巧**（同一物理文件硬链接为 `codex-linux-sandbox`、`codex-execve-wrapper`、apply-patch 模式等）承载约 30 个子命令。
- **客户端/服务端解耦**：TUI 不直接内嵌 core，而是作为 app-server 的 JSON-RPC 客户端；app-server 可运行于进程内、stdio、Unix socket、WebSocket 四种形态。
- **执行即服务**：进程/文件系统操作统一走 exec-server（本地沙箱化或远程 Noise 加密环境）。
- **双轨 MCP**：既作为 host 消费外部 MCP 服务器（`codex-mcp` + `rmcp-client`），又把 Codex 自身暴露为 MCP 工具（`mcp-server`，提供 `codex` / `codex-reply` 工具）。
- **纵深防御配置**：项目级 config 层不可信（有 denylist），管理员 requirements 层可约束用户配置。

---

## 2. 顶层架构设计

### 2.1 分层视图

```
┌──────────────────────────────────────────────────────────────┐
│  客户端：TUI (ratatui) │ IDE 插件 │ 桌面 App (codex app)      │
│         sdk/typescript │ sdk/python │ codex exec (无头)       │
├──────────────────────────────────────────────────────────────┤
│  API 网关层                                                   │
│   app-server (JSON-RPC v1 冻结 / v2 活跃开发)                  │
│   传输：stdio │ unix socket │ WebSocket │ 进程内 │ daemon      │
├──────────────────────────────────────────────────────────────┤
│  核心引擎层                                                    │
│   core (会话/turn 循环/上下文/工具编排/审批)                     │
│   协议总线：protocol (Op → EventMsg)                           │
├──────────────┬───────────────┬───────────────────────────────┤
│ 执行层        │ 模型接入层     │ 支撑层                         │
│ exec-server  │ model-provider│ config / login / state        │
│ linux-sandbox│ models-manager│ rollout / thread-store        │
│ network-proxy│ codex-api     │ ext/* / plugin / skills       │
└──────────────┴───────────────┴───────────────────────────────┘
```

### 2.2 crate 分组（codex-rs/ 下约 150 个 crate）

| 分组 | 代表 crate |
|---|---|
| 引擎 | `core`, `core-api`, `protocol`, `context-fragments`, `tools` |
| 服务 | `app-server*`(6个), `exec-server*`(2个), `mcp-server`, `codex-mcp`, `rmcp-client` |
| 入口 | `cli`, `tui`, `exec`, `arg0` |
| 沙箱/安全 | `sandboxing`, `linux-sandbox`, `windows-sandbox-rs`, `execpolicy`, `network-proxy`, `process-hardening`, `bwrap`, `apply-patch`, `shell-command`, `shell-escalation` |
| 模型 | `codex-api`, `model-provider`, `model-provider-info`, `models-manager`, `websocket-client`, `responses-api-proxy` |
| 配置/身份 | `config`, `login`, `keyring-store`, `features`, `workload-identity` |
| 持久化 | `rollout`, `state`, `thread-store`, `history`, `codex-home` |
| 扩展 | `ext/*`(15个), `plugin`, `core-plugins`, `code-mode*`(4个), `skills`, `hooks`, `prompts`, `memories/*`, `connectors` |
| 观测 | `otel`, `analytics`, `diagnostics`, `feedback` |
| 云 | `cloud-tasks*`, `chatgpt`, `app-server-daemon` |
| 工具库 | `utils/*`(约30个小crate) |

---

## 3. 核心引擎 codex-core

### 3.1 关键类型与职责

| 类型 | 位置 | 职责 |
|---|---|---|
| `ThreadManager` | `core/src/thread_manager.rs` | 进程级线程管理：`start_thread` / `resume_thread_from_rollout` / `fork_prepared_thread` / `spawn_subagent`（旧名 `ConversationManager` 已废弃为别名） |
| `CodexThread` | `core/src/codex_thread.rs:197` | 单个会话的公共句柄：`submit(Op)`、`next_event()`、steer/recover turn、shutdown |
| `SessionIo` | `core/src/session/mod.rs:367` | 队列端点：有界 submission channel（async_channel, 容量 512）、unbounded event channel、`watch<AgentStatus>` |
| `Session` | `core/src/session/session/session.rs` | 会话运行时：持有 `Mutex<SessionState>`、活动 turn、输入队列及约 30 项 `SessionServices`（MCP runtime、exec policy、auth、model client、网络代理、state db、extensions、hooks…） |

### 3.2 Op/Event 循环

```
Client ──Op──▶ [bounded(512) submission channel] ──▶ submission_loop ──▶ Event ──▶ [unbounded event channel] ──▶ Client
                                                        │
                                        handlers.rs 按 Op 分发：
                                        TurnInput / Interrupt / Compact /
                                        ExecApproval / PatchApproval /
                                        UserInputAnswer / Shutdown ...
```

- Submission ID 为 UUIDv7，兼作公开的 turn ID。
- 审批应答、turn 输入回执使用 `tokio::oneshot`；agent 状态用 `tokio::watch`；模型流用 `tokio::mpsc(1600)`。

### 3.3 Turn 生命周期（`run_turn`，`core/src/session/turn.rs:153`）

1. **准入**：`turn_input::handle`（turn_input.rs:141）判定 start / steer / reject；steering 将消息入队，在采样步骤之间排空。
2. **预处理**：预采样压缩（上下文接近上限时）；按 skill/plugin/@mention 计算所需 MCP 服务器；注入 skills/plugins 片段；跑 UserPromptSubmit hooks。
3. **采样循环**（每步冻结一个 `StepContext` 快照——model info、审批策略、工具路由器、AGENTS.md 等）：
   - `build_prompt` = `history.for_prompt()` + tool specs + base instructions；
   - `client_session.stream(...)` 消费 `ResponseEvent`；
   - `OutputItemDone` 写入历史；function/custom-tool 调用生成工具 future 推入 `FuturesOrdered` 并行执行；
   - delta 流式转发给 UI（ItemStarted / ContentDelta）；
   - 每步后检查 token 上限触发自动压缩（mid-turn roll-over）、Stop hooks 可阻断完成强制续跑。
4. **收尾**：`on_task_finished` 发出 TurnComplete、flush rollout、更新 token 用量指标。

### 3.4 模型客户端（`core/src/client.rs`）

- 底层 HTTP/WS/SSE 在 `codex-api` crate；`ModelClientSession::stream`（client.rs:1883）做传输选择。
- **仅支持 Responses API**：Chat Completions 已移除（`WireApi::Chat` 反序列化直接报错并指向 openai/codex#7782）。
- **WebSocket 优先**：Responses WebSocket（beta header `responses_websockets=2026-02-06`），按 diff 增量发送请求 payload、`x-codex-turn-state` 粘性路由、prewarm `response.create generate=false`；失败自动回退 HTTPS SSE（会话级一次性降级，client.rs:1942）。
- 重试：指数退避封顶 60s，WS→HTTP 回退后通用退避（responses_retry.rs）；`ContextWindowExceeded` / `UsageLimitReached` 单独处理。

### 3.5 上下文管理

- `ContextManager`（context_manager/history.rs:94）：内存中的模型可见历史（`Vec<ResponseItemEnvelope>`），逐 item token 估算、函数输出截断、用户 turn 边界检测。
- **`ContextualUserFragment` trait**（context-fragments/src/fragment.rs:14）：所有注入上下文的统一抽象（角色、标记符、body、render→ResponseItem）。`core/src/context/` 下约 40 种片段实现：AGENTS.md（UserInstructions）、EnvironmentContext、时间提醒、token 预算提醒、world-state diff、skill/plugin 指令、Guardian 审查证据等。
- 压缩：本地 inline 压缩 + 远程 `/responses/compact` 端点（compact_remote*.rs）。
- 架构约束（AGENTS.md 明确规定）：历史只增不重写、注入项必须有界且硬上限、单项 >10K tokens 需 P0 评审。

### 3.6 工具编排（core/src/tools/ + tools/ crate）

- `build_tool_router`（spec_plan.rs:117）按步注册工具到 `ToolRegistry`：unified_exec（含持久 shell 会话）、apply_patch、update_plan、view_image、web_search（hosted）、request_user_input、动态工具、multi-agents V1/V2、扩展工具适配器、MCP 工具追加。
- **Tool Mode**：Direct / CodeMode / CodeModeOnly —— Code Mode 将工具 schema 渲染为 TS 定义，在嵌入式 V8 JS 运行时中由模型写代码批量调用（见 §11.5）。
- `ToolOrchestrator::run`（orchestrator.rs:125）标准序列：计算 `ExecApprovalRequirement` → 请求审批（带缓存）→ 沙箱内首试 → 被拒且策略允许时再次审批后提权重试 → PreToolUse/PostToolUse hooks → 输出截断。

### 3.7 审批流

```
工具调用 ──▶ Session::request_approval(approvals.rs:439)
              ├─ 优先级1：PermissionRequest hook（Allow/Deny）
              ├─ 优先级2a：Guardian 自动审查子会话（严格模式下即 Skip 也强制审查）
              └─ 优先级2b：用户审批
                    ├─ ExecCommand → EventMsg::ExecApprovalRequest + oneshot
                    ├─ ApplyPatch  → EventMsg::ApplyPatchApprovalRequest
                    └─ McpToolCall → 结构化问题 UI
用户回答 Op::{ExecApproval,PatchApproval}(ReviewDecision)
  → 持久化 execpolicy/网络 amendment → resolve oneshot
```

---

## 4. 协议层 protocol

`codex-rs/protocol` 是轻依赖的共享词汇表（`protocol/src/protocol.rs` 约 6000 行）：

- **`Op`**（:544）：client→agent 提交枚举。注意枚举内嵌 `oneshot::Sender` 回执通道——这是进程内提交队列协议而非纯 serde DTO。
- **`Event { id, msg: EventMsg }`**（:1271/:1289）：agent→client 事件大枚举，覆盖 turn 生命周期（`TurnStarted` 序列化为遗留名 `task_started`）、流式 delta、工具/MCP/web-search begin/end、各类审批请求、TokenCount、压缩/回滚等。
- 配套类型：`AskForApproval`、`SandboxPolicy`（legacy）、`FileSystemSandboxPolicy`/`NetworkSandboxPolicy`/`PermissionProfile`（permissions.rs，新权限模型）、`ReviewDecision`、`ResponseItem`/`ResponseInputItem`（历史与提示词的通用货币）、`ThreadId`/`RolloutId`。
- v1/v2 版本划分不在本 crate，而在 `app-server-protocol`；此处通过 serde alias 与 `legacy_events.rs` 做兼容。

---

## 5. 服务层

### 5.1 app-server 家族

| Crate | 职责 |
|---|---|
| `app-server-protocol` | JSON-RPC API 定义。`common.rs` 用宏生成 `ClientRequest`（每个变体声明方法名、参数/响应类型、**串行化作用域**（全局锁 vs per-thread/per-process）、实验性门控 `#[experimental]`）。v1（`v1.rs`）全部冻结/废弃，活跃开发全在 `v2/`（38 个文件）。命名约定 `<resource>/<method>`，camelCase（config RPC 例外用 snake_case）。通过 ts-rs 生成 TS 绑定 + schema fixtures（`just write-app-server-schema`） |
| `app-server` | 服务本体。`message_processor.rs` 中央分发器 + `request_processors/`（thread/turn/config/account/apps/mcp/fs/git/command_exec/remote_control/marketplace 等每资源一个处理器）。支持 `in_process.rs` 进程内嵌入供库消费方/测试使用 |
| `app-server-transport` | 连接接收器：stdio / UDS（带启动锁）/ WebSocket（bearer auth）/ remote-control（向 OpenAI 出站连接，让手机等远程设备触达本地 daemon） |
| `app-server-daemon` | `$CODEX_HOME/app-server-daemon/` 下的常驻 daemon（pid 文件、settings.json），Start/Restart/Stop/bootstrap（托管安装+自更新循环），remote-control 开关 |
| `app-server-client` | 统一客户端门面：`AppServerClient::{InProcess, Remote}`（remote.rs 约 1000 行，UDS/WebSocket JSON-RPC + 远程路径翻译）。TUI 即通过它访问 daemon 或远程 server |

关键 API 面（节选）：`initialize`、`thread/start|resume|fork|archive|list|read|search|rollback|compact/start|queue/*`、`turn/start|steer|interrupt`、`review/start`、`fs/*`、`command/exec(+write|terminate|resize)`、`config/read|value/write|batchWrite`、`account/login/*|rateLimits/read`、`mcpServer/oauth/login|tool call`、约 80 个 ServerNotification（`item/started|completed`、`item/agentMessage/delta`、`turn/completed`…）。注意 `rawResponseItem/*` 为破坏性敏感面。

### 5.2 exec-server（执行即服务）

- 独立服务（`ws://IP:PORT` 默认或 stdio），JSON-RPC 方法面：`process/start|read|write|signal|terminate`、完整 fs API（FsReadFile/FsOpen/FsWalk/FsCopy…）、HTTP 代理转发、能力发现（扫描插件/skill 文件）、shell 快照。
- **环境模型**：LOCAL vs REMOTE 环境；远程环境经 WebSocket + **Noise 协议加密中继**注册（token 绑定 ChatGPT 账号，`CODEX_EXEC_SERVER_NOISE_AUTH_TOKEN_ENV_VAR`）。
- 每进程沙箱（process_sandbox.rs）、fs 沙箱 + symlink no-follow 保护；`codex exec-server forward --connect URL` 可将已有 WS server 注册为远程环境。
- core 与 mcp-server 通过 `ExecServerClient` 把所有命令执行统一路由至此，而非临时 spawn。

### 5.3 MCP 双向集成

- **host 侧（消费外部 MCP）**：`codex-mcp` 提供 `McpConnectionSet`（聚合所有连接、启动状态事件、必需/可选服务器截止时间、工具目录修订、名称前缀策略、elicitation 路由）；`rmcp-client` 提供 stdio/streamable-HTTP/进程内传输 + OAuth 登录（keyring 存储）+ elicitation client。
- **server 侧（暴露 Codex 为 MCP 工具）**：`mcp-server`（stdio，`codex mcp-server`，已标废弃倾向）暴露 `codex`（带 model/cwd/sandbox/approval 配置发起 turn）与 `codex-reply` 工具；审批以 MCP elicitation 形式回传给 MCP 客户端。

---

## 6. 入口层

### 6.1 cli（`codex-rs/cli`）

- 根解析器 `MultitoolCli`：无子命令 = 交互式 TUI。
- `arg0_dispatch_or_else`（arg0 crate）：单二进制多身份（`codex-linux-sandbox`、`codex-execve-wrapper`、`--codex-run-as-apply-patch`），`Arg0DispatchPaths` 携带各 helper 路径贯穿子系统——沙箱 helper 无需独立安装。
- 子命令全景（节选）：默认 TUI / `exec(e)` / `review` / `resume` / `fork` / `apply(a)` / `login|logout` / `mcp` / `plugin` / `mcp-server`(deprecated) / `app-server`(含 `daemon`、`proxy`) / `app`(桌面启动器，缺失时下载安装) / `sandbox` / `doctor` / `cloud` / `queue` / `debug` / `features`。
- 远程模式：`--remote ws://|wss://|unix://[PATH]` 让 TUI 连接共享 app-server 而非本地内嵌。

### 6.2 tui（`codex-rs/tui`）

- ratatui 实现，严格模块化（repo 规范要求文件 <500 行）：`app.rs` + `app/`(~50 文件)、`chatwidget.rs`(编排) + `chatwidget/`(~40 文件)、`bottom_pane/`(composer/footer/审批 overlay/命令弹窗)。
- **后端访问**：`app_server_session.rs` 门面——TUI 通过 `AppServerClient` 发送 typed JSON-RPC、消费 `AppServerEvent`，不再直连 core。
- 大量 insta 快照测试保障 UI 变更可视化评审。

### 6.3 exec（无头模式）

- `codex exec`：CI/脚本自动化面。`--json`（稳定 schema 的 JSONL 事件流 + CodexStatus）、`--output-last-message`、`--output-schema`（JSON Schema 约束最终响应）、`--ephemeral`、`--skip-git-repo-check`。事件泵接 pluggable EventProcessor（人类可读渲染 or JSONL）。

---

## 7. 沙箱与安全体系

### 7.1 平台抽象（sandboxing/ crate）

`SandboxManager`（manager.rs）为核心入口：

- `SandboxType { None, MacosSeatbelt, LinuxSeccomp, WindowsRestrictedToken }`；按 `cfg!` 选择平台后端。
- `transform()`（manager.rs:331）管线：解析有效 `PermissionProfile` → 规范化 cwd（PathUri→绝对路径）→ 注入托管 MITM CA 可读根 → 产出前缀化的 `SandboxExecRequest`。
- `should_sandbox()`：网络受限或 FS 策略 Restricted 时强制沙箱；ExternalSandbox 永不要求。

### 7.2 macOS Seatbelt

- 固定路径 `/usr/bin/sandbox-exec -p <profile> -D KEY=path`（防 PATH 注入）。
- SBPL 由模板拼装：deny-by-default 基础策略（借鉴 Chrome）+ 网络/偏好策略；文件规则从 `FileSystemSandboxPolicy` 生成（可读/可写 subpath + `require-not` 排除；`.git`/`.codex` 等受保护元数据经 regex deny；根锚点 deny-write-unlock 防 rename 迁移权威边界）。
- 网络动态策略：受限模式仅允许 loopback 代理端口（+可选 DNS :53）；托管网络开启但无可用代理时**失败关闭**（空策略）。

### 7.3 Linux（linux-sandbox/ crate，两阶段）

```
外层：构造 bwrap 命令（--ro-bind / /、可写根 --bind、保护路径重挂 RO、
      --unshare-user/pid[/net]、新 /proc、glob deny 文件 mask /dev/null）
        ↓ 自我重进入 --apply-seccomp-then-exec
内层：验证 fd mount → capget 断言零 capability → netns 内激活代理桥
      （TCP→UDS→TCP）→ 安装 seccomp → fork/exec 用户命令
```

- seccomp 两模式：Restricted（禁 connect/bind/sendto…，AF_UNIX only；即使 DangerFullAccess 在托管网络下也强制安装，fail-closed；另禁 ptrace/io_uring）与 ProxyRouted（仅放行 AF_INET 到本地桥）。
- Landlock ABI V5 只读规则集保留为 legacy fallback（`features.use_legacy_landlock`）。
- WSL1 不支持（bwrap 场景）；捆绑 fallback bwrap 二进制带摘要校验。

### 7.4 Windows（windows-sandbox-rs/）

非 AppContainer，而是**受限令牌 + Capability SID + ACL** 双后端：

- Legacy：`CreateRestrictedToken`（LUA_TOKEN/WRITE_RESTRICTED）+ 每 workspace root 的 capability SID + DACL deny ACE + Job 对象 + ConPTY。
- Elevated：一次性管理员配置创建专用账户 `CodexSandboxOffline`/`CodexSandboxOnline` + 防火墙规则，命令经命名管道 runner 服务（framed IPC）以这些账户执行；支持 legacy 无法实现的 deny-read ACE。
- 网络：WFP 过滤器绑定沙箱 SID，流量限定至托管代理。

### 7.5 Starlark 策略引擎（execpolicy/）

- `prefix_rule(...)` / `host_executable(...)` 自定义 Starlark 模块；规则为有序 token 前缀（token 可为备选数组）；决策序 Allow > Prompt > Forbidden，多重匹配取最严；无匹配时可挂 heuristics fallback（危险命令启发式）。
- 同时携带网络规则（域名 allow/deny 编译进代理配置）。
- "总是允许"审批 → amendment 追加到 `$CODEX_HOME/rules/*.rules`。
- CLI：`codex execpolicy check --rules f.rules cmd...`。

### 7.6 出站网络控制（network-proxy/）

- 本地 HTTP 代理 `127.0.0.1:3128`（CONNECT）+ SOCKS5 `:8081`（含 UDP ASSOCIATE）；Windows 用端口段。
- 策略：精确主机 / 通配 `*.x.com`、`**.x.com`（拒绝全局 `*`）；DNS-rebinding 防护（私网 IP 解析保持封锁除非显式允许）。
- HTTPS MITM（limited 模式隐含开启）：CA 私钥驻留代理内存，信任束写入 `$CODEX_HOME/proxy/` 注入子进程 env；mitm_hook 可按 method/host/path 匹配执行如 strip_auth。
- 凭证 broker：github/openai provider 以假凭证换真凭证（子进程 env 中是 dummy，服务侧替换）。

### 7.7 其他安全组件

| Crate | 说明 |
|---|---|
| `process-hardening` | pre-main 加固：PR_SET_DUMPABLE=0、PT_DENY_ATTACH、RLIMIT_CORE=0、清除 LD_*/DYLD_* |
| `shell-command` | tree-sitter-bash/-powershell AST 解析命令 → ParsedCommand 摘要；危险命令启发式分类器 |
| `shell-escalation` | 补丁版 zsh EXEC_WRAPPER → 每次 execve 经 `codex-execve-wrapper` → fd 传递连 EscalateServer → Run/Escalate/Deny 三决策（提权命令如何在沙箱外忠实执行的机制） |
| `apply-patch` | 自定义补丁格式（`*** Begin Patch`/Add/Update/Delete/Move + `@@` hunk），宽松模式兼容 gpt-4.1；拦截 shell 内 heredoc 补丁（tree-sitter-bash/PowerShell/cmd 解析）转为结构化 ApplyPatchAction 并对照沙箱策略校验 |

### 7.8 模式 × 平台矩阵

| 模式 | macOS | Linux | Windows |
|---|---|---|---|
| DangerFullAccess | 无 sandbox-exec | 托管网络下仍装 seccomp | 无 |
| ReadOnly | 全盘读/禁写/断网 SBPL | bwrap RO + seccomp 断网 | 受限令牌 + deny-write ACL |
| WorkspaceWrite | 写限 cwd/tmp/writable_roots（`.git`/`.codex` 除外） | bind-mount 可写根 | CapSID ACL |
| 网络 | 全开或仅代理端口 | netns 隔离 + ProxyRouted | WFP SID 过滤 |

---

## 8. 模型接入层

| Crate | 职责 |
|---|---|
| `codex-api` | 原始传输：Responses HTTP/SSE/WebSocket 端点、请求构造 |
| `model-provider-info` | config.toml `[model_providers.*]` 反序列化：base_url/env-key/query/header 参数/重试超时/WS 设置；内置工厂 openai/oss(Ollama、LM Studio)/Bedrock |
| `model-provider` | 运行时 trait `ModelProvider`：capabilities、审批/记忆任务的偏好模型、attestation、provider 作用域 auth |
| `models-manager` | `ModelsManager` trait：OpenAiModelsManager（拉取/刷新线上 `/models` 目录 + 缓存）vs StaticModelsManager（打包 models.json）；app-server 定期刷新 |
| `websocket-client` | 代理感知的 TLS WS 拨号共享组件 |
| `responses-api-proxy` | tiny_http 调试代理，转发 `/responses` 到上游，可选 dump 报文（测试/debug 用） |
| Auth | 见 §9.2 |

---

## 9. 配置系统

### 9.1 八层叠加（config/src/loader/mod.rs，低→高优先级）

1. package（编译进二进制的 defaults.toml）
2. admin（macOS 托管设备偏好）
3. system（`/etc/codex/config.toml` / `%ProgramData%`）
4. cloud（企业云配置片段）
5. user（`$CODEX_HOME/config.toml`）
6. profile（`$CODEX_HOME/<name>.config.toml`）
7. cwd/tree/repo（目录树上溯的 `.codex/config.toml`；**目录不受信时禁用**，且有 PROJECT_LOCAL_CONFIG_DENYLIST 禁止项目层设置 `model_provider`、`notify`、otel、base URL 等）
8. runtime（CLI `--config` 覆盖、UI 选择）

独立的 **requirements 层**（system/cloud/managed_config 组合）为管理员强制约束，凌驾于普通合并之上。schema 经 `just write-config-schema` 从 JsonSchema derive 生成 `core/config.schema.json`。

### 9.2 认证（login/ + keyring-store/）

- `AuthManager` / `CodexAuth` 多模态：ApiKey（OPENAI_API_KEY/CODEX_API_KEY/CODEX_ACCESS_TOKEN）、ChatGPT OAuth、PAT、workload identity、Bedrock key、外部 bearer。
- ChatGPT OAuth 双路径：localhost callback（PKCE）+ 设备码轮询；JWT ID-token 解析 plan/user/account。
- 存储：`$CODEX_HOME/auth.json` 或 OS keyring（`cli_auth_credentials_store = file|keyring|auto`）。

---

## 10. 持久化

三层结构：

1. **Canonical 历史 = rollout JSONL**（`$CODEX_HOME/sessions/rollout-<threadid>[_<rolloutid>].jsonl`）：每行 `{timestamp, ordinal?, item}`，item 为 tagged union `RolloutItem`（SessionMeta / ResponseItem / InterAgentCommunication / Compacted / TurnContext / WorldState / SecurityRiskScore / EventMsg）——模型上下文与协议事件同录。异步 mpsc 写入；后台 zstd 压缩 worker 处理旧文件；反向 JSONL 扫描器支持 resume；列表/搜索/游标分页。
2. **SQLite 投影**（state/ crate，50 个版本化 migration）：threads/projects/goals/queues/logs/memories 元数据索引，由 rollout backfill 同步（`$CODEX_SQLITE_HOME`）。
3. **thread-store 抽象缝**：`ThreadStore` trait（append_items 为规范原始历史追加）+ `LiveThread`（活动会话元数据同步 + 持久化策略）+ `LocalThreadStore`（rollout+SQLite 实现），为未来非本地后端预留。

Resume 路径：rollout → `InitialHistory::{New, Cleared, Resumed, Forked}` → `session/rollout_reconstruction.rs` 重建内存历史/world state/token 用量。`Op::ThreadRollback` 支持 truncate-after-turn 回滚并打 ThreadRolledBack 标记。TurnContextItem 冻结每 turn 配置保证 resume 复现决策。

---

## 11. 扩展体系

三管齐下：

### 11.1 编译期 Extension（ext/ + ext/extension-api/）
`ExtensionRegistryBuilder` 注册 `Arc<dyn ...Contributor>` 到各生命周期点：thread/turn lifecycle、turn-input、turn-item、tool、context、MCP-server、config、token-usage、skill-invocation、approval-review contributor + host capabilities（agent spawner、event sink、metrics、response-item 注入）。具体扩展：agent、connectors、git-attribution、goal、guardian-v2、history-notes、image-generation、items、mcp、memories、queue、skills、web-search——各自导出 `install(...)`。

### 11.2 打包 Plugin（plugin/ + core-plugins/）
`PluginManifest`（name/version/description + skills/MCP servers/apps/hooks 文件路径）声明式分发；marketplace（OpenAI curated/bundled、npm、local）加载/升级/开关管理，materialize 成 skills/MCP/hooks 回流到既有系统。

### 11.3 Skills & Hooks & Prompts & Memories
- **skills/**：SKILL.md 发现/快照缓存/frontmatter 解析；`@tool` mention 隐式调用检测。
- **hooks/**：11 个生命周期事件（PreToolUse、PermissionRequest、PostToolUse、Pre/PostCompact、SessionStart/End、UserPromptSubmit、SubagentStart/Stop、Stop），TOML 规则映射，插件可声明 hooks，schema fixtures 生成。
- **prompts/**：编译进的系统提示模板（压缩、review rubric、goals、按审批/沙箱模式的权限说明）。
- **memories/**：读写分离；write 侧启动期两阶段流水线（extract→consolidate）+ 剪枝；read 侧注入/引用解析；以 extension 暴露 memories 工具命名空间。

### 11.4 Feature Flags（features/）
集中注册表：`Feature` 枚举 + Stage（UnderDevelopment/Experimental/Stable/Deprecated/Removed）+ `[features]` TOML 与 legacy 开关解析 + 类型化子配置（CodeMode、GuardianV2、NetworkProxy、MultiAgentV2、token budget…）。

### 11.5 Code Mode（实验性）
模型写 JavaScript 调用"函数形式的工具"替代逐个 tool call：工具 JSON schema 渲染为 TypeScript 类型（code-mode-protocol）；**嵌入式 V8**（rusty_v8）运行 cell actor（code-mode-runtime）；独立 host 进程经 framed websocket/gRPC 通信（code-mode-host，`just code-mode-host`）；客户端侧 gRPC/WS/进程持有/disabled 多种 session provider（code-mode/）。v8-poc 为 Bazel 链接验证 crate。

### 11.6 Cloud Tasks（cloud-tasks*）
ChatGPT 云任务集成：CloudBackend trait 客户端（创建/列表/watch attempt/应用 diff 到本地分支）+ ratatui TUI 浏览器 + mock 后端。

---

## 12. 可观测性与支撑设施

- **otel/**：OTLP exporter 配置、provider 生命周期、W3C trace-context 解析、会话遥测事件。
- **analytics/**：类型化产品分析事件（Guardian 结果、RPC transport 等）→ reducer 管线 → 上报。
- **diagnostics/**：进程级静态 gauge 注册表（const atomics 计数器）。
- **feedback/**：ring-buffer tracing 事件捕获 + 日志管理 + 上传。
- **utils 亮点**：`pty`（跨平台 PTY/ConPTY、进程组、输出上限 1MiB——shell 工具执行骨干）、`stream-parser`（模型输出流的增量解析：citation 剥离、隐藏 tag、plan 提取、UTF-8 安全分块）、`fuzzy-match`（Unicode 正确的子序列匹配，TUI 过滤高亮）、`output-truncation`（token 预算数学的中部截断）。

---

## 13. 构建系统与发布

- **justfile**：`just fmt`（scripts/format.py 统一 Rust/Bazel/Python 格式）、`just fix`（clippy --fix）、`just test`（nextest、no-fail-fast）、`just build-for-release`（Bazel `//codex-rs/cli:release_binaries`）、三个 schema 生成 target（config/app-server/hooks）、Bazel lock 校验。
- **Bazel（Bzlmod）**：MODULE.bazel 声明依赖；每 crate 有 BUILD.bazel，CI 校验 MODULE.bazel.lock 漂移；hermetic LLVM patch 工具链支撑 rusty_v8；RBE 远程执行配置。Cargo 与 Bazel 双轨并存，锁文件一致性由 CI 强制。
- **CI**：rust-ci(-full/-full-nextest-platform)、bazel、cargo-deny、codespell、blob-size-policy、release workflows（含 zsh patch 构建、V8 canary）。
- **SDK**：`sdk/typescript`（TS SDK：codex.ts/thread.ts/turnOptions.ts）、`sdk/python` + `python-runtime`。
- **npm 包装**：`codex-cli/bin/codex.js` ESM launcher 按平台映射 optional-dependency 包（`@openai/codex-linux-x64` 等），各包内含原生二进制，JS 层零逻辑纯 spawn。
- **Clippy 收紧**：workspace 级 deny unwrap_used/expect_used/uninlined_format_args 等 50+ lint；sqlx SQLite 构造函数 deny 列表在 clippy.toml。

---

## 附：值得注意的设计取舍

1. **core 不再膨胀**：AGENTS.md 明确"resist adding code to codex-core"，新功能优先独立 crate（ext/* 即产物）；但历史包袱仍在（`Session::new` ~30 参数、`run_turn` 巨型函数、"conversation→thread"术语迁移以 deprecated 别名过渡）。
2. **协议即契约**：in-process Op/Event 与 wire JSON-RPC v2 严格分层；v1 冻结保兼容，`rawResponseItem/*` 视为 breaking 敏感面；TS 绑定/schema fixture 由代码生成锁定。
3. **安全默认失败关闭**：托管网络无可用代理时空网络策略、DangerFullAccess 下仍装 seccomp、项目配置层 denylist、requirements 层凌驾用户配置。
4. **测试基建投入大**：insta 快照（UI 变更必须配快照评审）、wiremock SSE mock（mount_sse_once/ResponseMock 断言请求体）、test_codex 集成框架、mock 云后端、Bazel/Cargo 双跑。
