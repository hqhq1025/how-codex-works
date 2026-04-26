# 源码索引与命令速查

这一页只做源码阅读和命令核对：核心术语、关键路径、常用命令、验证命令、按问题定位源码。资料来源和外部参考集中放在 [参考资料与来源](./reference.md)。

本版以 `openai/codex@87bc724` 为核对快照，提交日期 2026-04-25，提交信息为 `[codex] remove responses command (#19640)`。

## 快速入口

| 入口 | 用途 |
|------|------|
| [openai/codex](https://github.com/openai/codex) | 官方仓库 |
| [核对源码快照：87bc724](https://github.com/openai/codex/tree/87bc72408c5ef08f8d21f2cdd00c55451c3be33f) | 本版文档对应的源码状态 |
| [Codex 官方文档](https://developers.openai.com/codex) | 产品与使用文档 |
| [README.md](https://github.com/openai/codex/blob/main/README.md) | 安装、入口和产品边界 |
| [codex-rs/README.md](https://github.com/openai/codex/blob/main/codex-rs/README.md) | Rust CLI 能力和 workspace 总览 |
| [docs/config.md](https://github.com/openai/codex/blob/main/docs/config.md) | 配置文档入口 |
| [docs/install.md](https://github.com/openai/codex/blob/main/docs/install.md) | 构建和安装 |
| [codex-rs/docs/protocol_v1.md](https://github.com/openai/codex/blob/main/codex-rs/docs/protocol_v1.md) | 协议术语和 queue-pair 心智模型 |

## 核心概念速查

| 概念 | 简短解释 | 主要源码 |
|------|----------|----------|
| `Op` | 外部提交给 Codex core 的操作，比如用户输入、中断、审批、压缩 | `codex-rs/protocol/src/protocol.rs` |
| `Submission` | UI 或客户端写入 Submission Queue 的消息包装 | `codex-rs/protocol/src/protocol.rs` |
| `Event` / `EventMsg` | Codex core 发回 UI 的事件，承载文本流、工具状态、审批请求、完成状态 | `codex-rs/protocol/src/protocol.rs` |
| `Session` | 一次 Codex 运行中的配置、状态和服务集合 | `codex-rs/core/src/session/mod.rs` |
| `Task` | 对用户输入的一次执行任务，内部可能包含多轮 turn | `codex-rs/core/src/tasks/` |
| `Turn` | 一次模型请求、工具执行和 follow-up 的循环单元 | `codex-rs/core/src/session/turn.rs` |
| `Thread` | app-server 视角下的可恢复对话和工作单元 | `codex-rs/core/src/thread_manager.rs` |
| `ToolRouter` | 把模型返回的 tool call 路由到内部工具 payload 和 handler | `codex-rs/core/src/tools/router.rs` |
| `ToolRegistry` | 统一管理工具 spec、handler 和执行结果转换 | `codex-rs/core/src/tools/registry.rs` |
| `ToolOrchestrator` | 工具执行边界，连接审批、hook、sandbox、升级执行 | `codex-rs/core/src/tools/orchestrator.rs` |
| `apply_patch` | Codex 的结构化代码编辑工具，支持 patch 解析、预览、审批和执行 | `codex-rs/core/src/tools/handlers/apply_patch.rs` |
| `TurnDiffTracker` | 记录 turn 内文件基线和最终 unified diff | `codex-rs/core/src/turn_diff_tracker.rs` |
| `ContextFragment` | 模型上下文片段，承载环境、权限、skills、plugins、hooks 等输入 | `codex-rs/core/src/context/fragment.rs` |
| `HookRuntime` | 运行 hook、处理 block/approval/additional context | `codex-rs/core/src/hook_runtime.rs` |
| `SubAgent` | 受父 session 管理的子 Codex thread | `codex-rs/core/src/codex_delegate.rs` |
| `SessionTask` | Codex 工作流抽象，包住普通 turn、review、compact 等任务 | `codex-rs/core/src/tasks/mod.rs` |
| `GoalRuntime` | 管理线程目标、预算、自动继续和外部目标状态 | `codex-rs/core/src/goals.rs` |
| `Rollout` | session/thread 的持久化记录，用于恢复、列表、压缩和记忆 | `codex-rs/core/src/rollout.rs` |
| `Memory` | 跨 thread 的长期信息提取和合并流水线 | `codex-rs/core/src/memories/README.md` |
| `App Server` | 面向 IDE、desktop、自动化客户端的 JSON-RPC 层 | `codex-rs/app-server/README.md` |
| `MCP server` | 实验性能力，把 Codex 暴露给其他 MCP client 调用 | `codex-rs/mcp-server/src/message_processor.rs` |

## 源码路径索引

### Core 和协议

| 路径 | 读它是为了什么 |
|------|----------------|
| `codex-rs/Cargo.toml` | workspace 全局地图 |
| `codex-rs/core/README.md` | core crate 的平台假设 |
| `codex-rs/core/src/lib.rs` | core 对外暴露模块 |
| `codex-rs/docs/protocol_v1.md` | queue-pair、Session、Task、Turn 的术语 |
| `codex-rs/protocol/src/protocol.rs` | `Op`、`Submission`、`Event`、`SandboxPolicy` |
| `codex-rs/core/src/session/mod.rs` | `Codex`、`Session` 和审批相关逻辑 |
| `codex-rs/core/src/session/handlers.rs` | `submission_loop` 和 `Op` 分发 |
| `codex-rs/core/src/session/turn.rs` | `run_turn`、模型请求和 agent loop |
| `codex-rs/core/src/codex_thread.rs` | `CodexThread` 异步接口 |
| `codex-rs/core/src/thread_manager.rs` | 创建、恢复、fork 和管理 thread |

### 工具和执行

| 路径 | 读它是为了什么 |
|------|----------------|
| `codex-rs/core/src/tools/spec.rs` | 多来源工具规格构建 |
| `codex-rs/core/src/tools/router.rs` | 工具调用路由 |
| `codex-rs/core/src/tools/registry.rs` | `ToolHandler` 和 registry |
| `codex-rs/core/src/tools/orchestrator.rs` | 审批、sandbox、升级执行 |
| `codex-rs/core/src/tools/parallel.rs` | 工具并发运行时 |
| `codex-rs/core/src/tools/tool_search_entry.rs` | discoverable tools 的搜索入口 |
| `codex-rs/core/src/tools/handlers/apply_patch.rs` | apply_patch handler、流式 diff、patch 审批输入 |
| `codex-rs/core/src/tools/runtimes/apply_patch.rs` | apply_patch runtime 请求和执行 |
| `codex-rs/core/src/turn_diff_tracker.rs` | turn 级最终 diff 聚合 |
| `codex-rs/core/src/exec.rs` | 命令执行、超时、输出收集 |
| `codex-rs/tools/` | 可复用工具 schema 和工具规格 |
| `codex-rs/tools/src/agent_tool.rs` | `spawn_agent` 工具说明和使用约束 |
| `codex-rs/core/src/codex_delegate.rs` | 子 Codex thread 的启动和事件桥接 |

### 入口与前端

| 路径 | 读它是为了什么 |
|------|----------------|
| `codex-rs/cli/src/main.rs` | CLI 子命令分发 |
| `codex-rs/tui/` | 交互式终端 UI |
| `codex-rs/exec/src/lib.rs` | `codex exec` headless 主流程 |
| `codex-rs/exec/src/exec_events.rs` | exec JSONL 事件结构 |
| `codex-rs/app-server/README.md` | app-server 说明 |
| `codex-rs/app-server/src/codex_message_processor.rs` | JSON-RPC 请求处理 |
| `codex-rs/app-server-protocol/` | app-server 协议类型 |
| `codex-rs/app-server-client/` | app-server 客户端 |

### 安全、沙箱和网络

| 路径 | 读它是为了什么 |
|------|----------------|
| `codex-rs/core/src/guardian/` | Guardian 自动 reviewer |
| `codex-rs/sandboxing/` | 跨平台 sandbox 抽象 |
| `codex-rs/sandboxing/src/manager.rs` | sandbox 类型选择和转换 |
| `codex-rs/sandboxing/src/seatbelt.rs` | macOS Seatbelt |
| `codex-rs/linux-sandbox/` | Linux sandbox helper |
| `codex-rs/windows-sandbox-rs/` | Windows sandbox |
| `codex-rs/network-proxy/README.md` | 网络代理设计 |
| `codex-rs/network-proxy/src/` | HTTP/SOCKS 代理实现 |
| `codex-rs/execpolicy/` | exec policy |
| `codex-rs/shell-command/` | shell 命令解析和安全判断 |
| `codex-rs/core/src/network_policy_decision.rs` | 网络策略判断 |
| `codex-rs/core/src/tools/network_approval.rs` | 工具执行中的网络审批 |

### 上下文、压缩和记忆

| 路径 | 读它是为了什么 |
|------|----------------|
| `codex-rs/core/src/context_manager/` | prompt history 管理 |
| `codex-rs/core/src/context/` | 上下文 fragment 类型 |
| `codex-rs/core/src/context/fragment.rs` | `ContextualUserFragment` 和 fragment registration |
| `codex-rs/core/src/compact.rs` | inline compaction |
| `codex-rs/core/src/compact_remote.rs` | remote compaction |
| `codex-rs/core/templates/compact/prompt.md` | inline compact handoff prompt |
| `codex-rs/core/templates/compact/summary_prefix.md` | compaction summary 前缀 |
| `codex-rs/core/src/message_history.rs` | 用户输入历史 |
| `codex-rs/core/src/rollout.rs` | session rollout 读写 |
| `codex-rs/rollout/` | rollout crate |
| `codex-rs/state/` | SQLite 状态 |
| `codex-rs/core/src/memories/` | memory 生成和管理 |
| `codex-rs/core/src/memories/README.md` | memory pipeline 设计 |
| `codex-rs/core/src/goals.rs` | persisted thread goals |

### MCP、插件和定制

| 路径 | 读它是为了什么 |
|------|----------------|
| `codex-rs/codex-mcp/` | Codex 作为 MCP client |
| `codex-rs/rmcp-client/` | MCP 低层客户端 |
| `codex-rs/mcp-server/src/message_processor.rs` | Codex 作为实验性 MCP server |
| `codex-rs/mcp-server/src/codex_tool_runner.rs` | MCP tool 调用 Codex |
| `codex-rs/core/src/mcp.rs` | core 内 MCP 管理入口 |
| `codex-rs/core/src/skills.rs` | skill 加载和注入 |
| `codex-rs/core/src/skills_watcher.rs` | skill 变更监听和缓存失效 |
| `codex-rs/skills/` | skill 管理 |
| `codex-rs/core-skills/` | core skills |
| `codex-rs/hooks/` | hook runtime |
| `codex-rs/core/src/hook_runtime.rs` | core 侧 hook 生命周期 |
| `codex-rs/hooks/src/schema.rs` | hook 输入输出 schema |
| `codex-rs/hooks/src/engine/output_parser.rs` | hook stdout 解析和 fail-closed 逻辑 |
| `codex-rs/core/src/plugins/` | plugin 管理 |
| `codex-rs/core/src/plugins/manager.rs` | plugin marketplace、skills、apps、MCP 聚合 |
| `codex-rs/plugin/` | plugin crate |
| `codex-rs/core/src/agents_md.rs` | AGENTS.md |
| `codex-rs/config/src/config_requirements.rs` | 组织级配置约束 |
| `codex-rs/core/src/config/` | core 配置结构和加载结果 |
| `codex-rs/core/src/config_loader/` | 配置层合并 |
| `codex-rs/core/config.schema.json` | `config.toml` JSON Schema |

### Task、Goal 和多 agent

| 路径 | 读它是为了什么 |
|------|----------------|
| `codex-rs/core/src/tasks/mod.rs` | `SessionTask`、任务启动、取消和完成处理 |
| `codex-rs/core/src/tasks/regular.rs` | 普通用户输入如何进入 `run_turn` |
| `codex-rs/core/src/tasks/review.rs` | review task |
| `codex-rs/core/src/tasks/compact.rs` | compact task |
| `codex-rs/core/src/tasks/undo.rs` | undo task |
| `codex-rs/core/src/tasks/user_shell.rs` | 用户 shell task |
| `codex-rs/core/src/tasks/ghost_snapshot.rs` | ghost snapshot task |
| `codex-rs/core/src/goals.rs` | goal runtime 状态机 |
| `codex-rs/core/templates/goals/continuation.md` | 自动继续 prompt 模板 |
| `codex-rs/core/templates/goals/budget_limit.md` | 预算限制 prompt 模板 |
| `codex-rs/core/src/codex_delegate.rs` | 子 Codex thread 和父 session 审批桥接 |
| `codex-rs/core/src/tools/handlers/multi_agents.rs` | 多 agent v1 工具 handler |
| `codex-rs/core/src/tools/handlers/multi_agents_v2.rs` | 多 agent v2 工具 handler |

## 命令速查

这些命令来自官方 README、Rust README 和源码入口。不同版本可能会有变化，细节以当前安装的 `codex --help` 为准。

```bash
# 交互式 TUI
codex

# 非交互执行任务
codex exec "summarize this repository"

# 非交互 JSONL 输出
codex exec --json "run the tests and explain failures"

# 运行实验性 MCP server
codex mcp-server

# 管理 MCP server 配置
codex mcp --help

# 调试平台 sandbox
codex sandbox --help

# 启动 app-server
codex app-server --help
```

## 验证命令

维护这份文档时，可以用下面几类检查确认内容没有漂移。

```bash
# 查协议核心类型
rg -n "pub enum Op|pub struct Submission|pub struct Event" codex-rs/protocol/src/protocol.rs

# 查 agent loop
rg -n "submission_loop|run_turn|run_sampling_request" codex-rs/core/src

# 查工具系统
rg -n "ToolRouter|ToolOrchestrator|ToolCallRuntime" codex-rs/core/src/tools

# 查 app-server thread/turn API
rg -n "thread/start|turn/start|thread/fork|thread/compact/start" codex-rs/app-server/README.md

# 查多 agent 工具
rg -n "spawn_agent|wait_agent|send_input" codex-rs/tools/src codex-rs/core/src

# 查 apply_patch 编辑链路
rg -n "ApplyPatchHandler|ApplyPatchArgumentDiffConsumer|TurnDiffTracker" codex-rs/core/src

# 查 hook 生命周期
rg -n "run_pre_tool_use_hooks|run_permission_request_hooks|run_post_tool_use_hooks" codex-rs/core/src/hook_runtime.rs

# 查 task 和 goals
rg -n "trait SessionTask|GoalRuntimeEvent|RegularTask" codex-rs/core/src/tasks codex-rs/core/src/goals.rs
```

## 按问题定位源码

| 问题 | 先看 | 再看 |
|------|------|------|
| 一次用户输入怎么进入 core | `codex-rs/protocol/src/protocol.rs` 的 `Op` / `Submission` | `codex-rs/core/src/session/handlers.rs` 的 `submission_loop` |
| 模型为什么会多次请求 | `codex-rs/core/src/session/turn.rs` 的 `run_turn` | `run_sampling_request`、`try_run_sampling_request`、`SamplingRequestResult` |
| 工具列表从哪里来 | `codex-rs/core/src/session/turn.rs` 的 `built_tools` | `codex-rs/core/src/tools/spec.rs`、`codex-rs/tools/` |
| 模型 tool call 如何执行 | `codex-rs/core/src/tools/router.rs` | `codex-rs/core/src/tools/registry.rs`、`parallel.rs` |
| shell / patch 为什么会被审批 | `codex-rs/core/src/tools/orchestrator.rs` | `codex-rs/core/src/tools/sandboxing.rs`、`core/src/guardian/` |
| sandbox 怎么选平台 | `codex-rs/sandboxing/src/manager.rs` | `seatbelt.rs`、`linux-sandbox/`、`windows-sandbox-rs/` |
| 网络为什么会单独审批 | `codex-rs/core/src/tools/network_approval.rs` | `codex-rs/core/src/network_policy_decision.rs`、`network-proxy/` |
| history 和 rollout 有什么区别 | `codex-rs/core/src/context_manager/history.rs` | `codex-rs/core/src/rollout.rs`、`codex-rs/rollout/` |
| 压缩怎么不破坏当前任务 | `codex-rs/core/src/compact.rs` | `codex-rs/core/src/session/turn.rs` 的 auto compact 分支 |
| mid-turn compact 为什么要插入 initial context | `codex-rs/core/src/compact.rs` 的 `InitialContextInjection` | `codex-rs/core/src/session/turn.rs` 的 mid-turn compact 分支 |
| remote compact 输出为什么要过滤 | `codex-rs/core/src/compact_remote.rs` | `process_compacted_history`、`should_keep_compacted_history_item` |
| memory 如何跨线程生成 | `codex-rs/core/src/memories/README.md` | `phase1.rs`、`phase2.rs`、`start.rs` |
| app-server 如何复用 core | `codex-rs/app-server/README.md` | `codex-rs/core/src/thread_manager.rs`、`codex-rs/core/src/codex_thread.rs` |
| 子 agent 如何接回父 session | `codex-rs/core/src/codex_delegate.rs` | `codex-rs/tools/src/agent_tool.rs`、`core/src/tools/handlers/multi_agents*.rs` |
| patch 为什么能提前预览 | `codex-rs/core/src/tools/handlers/apply_patch.rs` 的 `ApplyPatchArgumentDiffConsumer` | `codex-rs/protocol/src/protocol.rs` 的 `PatchApplyUpdatedEvent` |
| 最终 diff 从哪里来 | `codex-rs/core/src/turn_diff_tracker.rs` | `codex-rs/core/src/tools/handlers/apply_patch.rs` |
| skills/plugins 如何进入 prompt | `codex-rs/core/src/session/turn.rs` | `codex-rs/core/src/skills.rs`、`codex-rs/core/src/plugins/injection.rs` |
| hook 能阻断哪些工具 | `codex-rs/core/src/hook_runtime.rs` | `codex-rs/hooks/src/engine/output_parser.rs` |
| task 和 turn 有什么区别 | `codex-rs/core/src/tasks/mod.rs` | `codex-rs/core/src/tasks/regular.rs`、`codex-rs/core/src/session/turn.rs` |
| goal 自动继续怎么避免并发 | `codex-rs/core/src/goals.rs` | `codex-rs/core/templates/goals/continuation.md` |
| 公开资料里称道的功能如何回源码 | `codex-rs/README.md`、`codex-rs/exec/src/lib.rs`、`codex-rs/app-server/README.md` | `codex-rs/core/src/session/review.rs`、`codex-rs/core/src/compact.rs`、`codex-rs/core/src/codex_delegate.rs` |
| 每个工具从哪里生成、由谁执行 | `codex-rs/tools/src/tool_registry_plan.rs`、`codex-rs/tools/src/tool_spec.rs` | `codex-rs/core/src/tools/spec.rs`、`codex-rs/core/src/tools/handlers/` |

## 深读检查清单

读某个模块时，可以用这 6 个问题检查是否真的读懂：

1. 这个模块处理的是协议、状态、工具、副作用，还是 UI？
2. 它的输入类型和输出类型分别是什么？
3. 它会修改哪些持久状态：history、rollout、state DB、文件系统，还是都不修改？
4. 它失败时向模型返回错误、向 UI 发事件，还是直接中断 turn？
5. 它有没有跨平台、跨前端或跨工具来源的抽象？
6. 如果自己实现最小版，哪些逻辑可以删掉，哪些边界必须保留？

## 阅读顺序建议

第一轮只看路径，不看细节：

1. `codex-rs/README.md`
2. `codex-rs/Cargo.toml`
3. `codex-rs/docs/protocol_v1.md`
4. `codex-rs/protocol/src/protocol.rs`
5. `codex-rs/core/src/session/handlers.rs`
6. `codex-rs/core/src/session/turn.rs`
7. `codex-rs/core/src/tools/router.rs`
8. `codex-rs/core/src/tools/orchestrator.rs`
9. `codex-rs/core/src/exec.rs`

第二轮按主题深入。想看 UI 读 `tui`，想看自动化读 `exec`，想看多前端读 `app-server`，想看扩展读 MCP、skills、plugins、hooks。
