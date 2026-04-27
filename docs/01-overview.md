# 1. 概述：Codex 是一个 agent runtime

## 核心问题

为什么 Codex CLI 不是一个简单的终端聊天程序？因为它实现的是一个可以被多种前端复用的 agent runtime。终端只是入口之一，`codex exec`、app-server、实验性的 MCP server、桌面应用和 IDE 都在不同程度上复用同一套核心。

## 源码入口

- `README.md`：项目定位和安装方式
- `codex-rs/README.md`：Rust CLI 和关键 crate 总览
- `codex-rs/Cargo.toml`：workspace 成员
- `codex-rs/core/README.md`：核心 crate 的平台假设
- `codex-rs/core/src/lib.rs`：`codex-core` 对外暴露的模块

## 四层结构

```mermaid
graph TB
    Entrypoints["入口层: cli / tui / exec / app-server / mcp-server experimental"]
    Protocol["协议层: codex-protocol / app-server-protocol"]
    Core["核心层: codex-core"]
    System["系统层: exec / sandboxing / network-proxy / rollout / state"]

    Entrypoints --> Protocol
    Protocol --> Core
    Core --> System
```

入口层负责把外部世界接进来。`cli` 解析命令行，`tui` 负责终端交互，`exec` 负责非交互自动化，`app-server` 提供 JSON-RPC，实验性的 `mcp-server` 把 Codex 暴露给其他 MCP client。

协议层负责把这些入口统一成类型。`codex-protocol` 定义核心 agent 的 `Op` 和 `Event`，`app-server-protocol` 定义前端和 app-server 的 JSON-RPC 消息。

核心层负责 agent 真正的行为。`codex-core` 里有 `ThreadManager`、`CodexThread`、`Session`、模型客户端、工具路由、上下文、记忆、插件、技能和安全策略。

系统层负责和操作系统打交道。命令执行、沙箱、网络代理、会话记录、SQLite 状态库都在这里。

## 关键 crate 速读

| crate | 先读什么 | 解决的问题 |
|-------|----------|------------|
| `protocol` | `src/protocol.rs` | 核心输入输出协议 |
| `core` | `src/session/turn.rs` | agent loop 和会话状态 |
| `tools` | `README.md` | 工具定义和 schema |
| `cli` | `src/main.rs` | 命令行入口和子命令分发 |
| `tui` | `src/` | 终端 UI 和事件渲染 |
| `exec` | `src/lib.rs` | headless 模式和 JSONL 输出 |
| `app-server` | `src/codex_message_processor.rs` | 多前端 JSON-RPC 服务 |
| `mcp-server` | `src/message_processor.rs` | Codex 作为实验性 MCP server |
| `sandboxing` | `src/manager.rs` | 平台沙箱选择和命令转换 |
| `network-proxy` | `README.md` | 网络访问控制 |

## workspace 为什么这么大

本版核对快照里，`codex-rs/` 下可以找到 78 个 `Cargo.toml`。这不是因为核心 agent loop 需要 78 个 crate，而是因为 Codex 把本地 runtime 的边界拆得很细：

| 分组 | 代表 crate | 为什么要拆出来 |
|------|------------|----------------|
| 入口 | `cli`、`tui`、`exec`、`app-server`、`mcp-server` | 不同前端复用 core |
| 协议 | `protocol`、`app-server-protocol`、`exec-events` 相关模块 | 类型生成、JSONL、JSON-RPC 边界 |
| 核心状态 | `core`、`rollout`、`state`、`thread-store` | turn、thread、持久化和索引 |
| 工具 | `tools`、`apply-patch`、`exec-server`、`file-search` | 工具 schema、执行和专用能力 |
| 安全 | `sandboxing`、`linux-sandbox`、`windows-sandbox-rs`、`network-proxy`、`execpolicy` | 平台隔离、网络控制、命令策略 |
| 扩展 | `codex-mcp`、`rmcp-client`、`skills`、`plugin`、`hooks` | MCP、skills、plugins、hooks 生命周期 |
| 观测与产品支撑 | `analytics`、`otel`、`models-manager`、`login` | 登录、模型目录、埋点和诊断 |

读源码时不需要一开始理解所有 crate。可以先抓住 `protocol -> core -> tools -> sandboxing -> app-server/exec` 这条主线。其他 crate 多数是在给这条主线补平台、协议、扩展或产品能力。

## 五条主设计原则

Codex 的架构可以用五条原则概括：

1. 协议先行：外部世界只通过 `Submission` 进来，通过 `Event` 出去。
2. 工具集中管控：模型提出工具调用，执行前必须经过 router、registry、hook、approval、sandbox 等边界。
3. 状态分层：prompt history、rollout、SQLite state、memory 不混成一份聊天记录。
4. 多入口复用：CLI、TUI、exec、app-server、实验性 MCP server 都围绕 core runtime 构建。
5. 扩展按生命周期分层：config、AGENTS.md、skills、plugins、hooks、MCP、apps 各有进入 runtime 的位置。

这些原则也解释了 Codex 为什么比 demo 难读。它不是为了展示一次模型调用，而是在处理“一个本地 agent 长期运行在用户工作区里”这件事。

## 一次请求穿过哪些层

```mermaid
flowchart TB
    Input["用户输入 / app-server request / exec prompt"] --> Protocol["Submission + Op"]
    Protocol --> Session["Session / TurnContext"]
    Session --> Context["ContextManager + injections"]
    Context --> Prompt["Prompt + ToolSpec"]
    Prompt --> Model["ModelClient stream"]
    Model --> Stream["ResponseEvent"]
    Stream --> ToolCall{"tool call?"}
    ToolCall -->|no| Assistant["assistant message"]
    ToolCall -->|yes| Router["ToolRouter"]
    Router --> Registry["ToolRegistry"]
    Registry --> Safety["approval / hooks / sandbox"]
    Safety --> Output["ToolOutput"]
    Output --> History["history + rollout"]
    History --> Prompt
```

这条链路里，模型只占中间一段。Codex 真正的工程量在模型前后的系统：输入如何排队、上下文如何构造、工具如何安全执行、输出如何流式通知、状态如何落盘、失败后如何恢复。

## 设计取舍

Codex 选择 Rust 不是表面风格问题。单二进制分发、平台沙箱、异步 I/O、跨平台命令执行都更适合放在一个 native runtime 里做。代价是模块数量和编译复杂度都变高，读源码时要先建立地图，否则很容易在近 80 个 crate 之间迷路。

另一个明显取舍是协议先行。很多 agent 会把 UI 和 agent loop 直接写在一起，Codex 则把 `Submission` 和 `Event` 作为硬边界。这个边界增加了类型和事件数量，但换来的是多前端复用、可测试性和可恢复性。

## 如果自己做 Agent，可以学什么

不要一开始就堆功能。先把系统切成三层：外部入口、核心协议、工具运行时。只要这个边界稳，后面加 TUI、HTTP API、MCP、桌面应用都不会逼你重写 agent loop。

Codex 还提醒了一件事：coding agent 的难点不只是模型推理，而是模型推理旁边的工程边界。工具执行、安全、上下文、持久化和用户交互，任何一块没做好，都会让 agent 看起来不可靠。

## 源码规模背后的分层

当前快照下，`codex-rs/` 里有 100 多个 `Cargo.toml`，这很容易让人误判为“架构太碎”。更合理的读法是把 crate 看成边界声明：哪些代码必须跟操作系统贴近，哪些代码属于协议，哪些代码只是产品入口。

| 层 | 代表路径 | 读法 |
|----|----------|------|
| 产品入口 | `cli`、`tui`、`exec`、`app-server`、`mcp-server` | 看它们如何把外部请求转成 core 能懂的协议 |
| 核心运行时 | `core`、`protocol`、`thread-store`、`rollout`、`state` | 看 turn、task、history、事件和持久化 |
| 工具与副作用 | `tools`、`apply-patch`、`exec`、`file-search`、`shell-command` | 看模型能做什么，以及副作用如何被封装 |
| 安全边界 | `sandboxing`、`linux-sandbox`、`windows-sandbox-rs`、`execpolicy`、`network-proxy` | 看命令、文件和网络如何被限制 |
| 扩展系统 | `codex-mcp`、`rmcp-client`、`skills`、`plugin`、`hooks`、`connectors` | 看外部能力如何进入工具或上下文 |

```mermaid
graph TB
    subgraph Entrypoints["入口"]
        CLI["cli"]
        TUI["tui"]
        EXEC["exec"]
        APP["app-server"]
        MCPServer["mcp-server"]
    end

    subgraph Runtime["核心运行时"]
        Protocol["protocol"]
        Core["core"]
        State["state / rollout / thread-store"]
    end

    subgraph Effects["副作用"]
        Tools["tools"]
        Patch["apply-patch"]
        Shell["exec / shell-command"]
    end

    subgraph Guardrails["安全边界"]
        Policy["execpolicy"]
        Sandbox["sandboxing"]
        Net["network-proxy"]
    end

    Entrypoints --> Protocol
    Protocol --> Core
    Core --> State
    Core --> Tools
    Tools --> Patch
    Tools --> Shell
    Shell --> Policy
    Shell --> Sandbox
    Shell --> Net
```

## 一次请求穿过 runtime 的全景

一次用户输入不会直接进入模型。它先被包装成 `Op::UserTurn`，再进入 submission queue。`submission_loop` 负责分发，`SessionTask` 负责生命周期，`run_turn` 负责采样、工具调用和 follow-up。工具执行之后，结果会被写回 history 和 rollout，前端通过 `EventMsg` 看到过程。

```mermaid
sequenceDiagram
    participant Frontend as TUI / exec / app-server
    participant Queue as Submission queue
    participant Session as Session
    participant Task as SessionTask
    participant Turn as run_turn
    participant Tools as Tool runtime
    participant Store as history / rollout / state

    Frontend->>Queue: Submission { Op::UserTurn }
    Queue->>Session: submission_loop dispatch
    Session->>Task: start_task
    Task->>Turn: run_turn
    Turn->>Tools: tool calls
    Tools-->>Turn: tool outputs
    Turn->>Store: record items
    Store-->>Frontend: EventMsg stream
```

这个结构让 Codex 可以同时支持交互式 TUI、非交互 `exec`、桌面 app、IDE 和 MCP server。入口可以变化，核心任务模型不需要重写。

## Codex vs Claude Code 的公开差异

这份导读只比较公开可见部分。Claude Code 的内部实现不能当成事实来源，Codex 的本地 runtime 则能回到公开源码核对。

| 维度 | Codex 可确认事实 | Claude Code 公开可见行为 |
|------|------------------|--------------------------|
| 源码可读性 | `openai/codex` 公开 Rust workspace | 官方 npm 包可观察，内部实现细节不完全等同于开源可验证源码 |
| 工具暴露 | `ToolSpec`、`build_tool_registry_plan` 可追 | 用户能看到 Read/Edit/Bash/MCP 等工具体验 |
| 压缩 | `compact.rs`、`compact_remote.rs`、`ContextManager` 可追 | 公开体验和资料显示有自动压缩，但内部细节不能替代源码事实 |
| 多前端 | app-server、exec、TUI、MCP server 都在源码里 | CLI、IDE 等体验公开可见 |
| 扩展 | MCP、skills、plugins、hooks 公开文档和源码都有 | hooks、MCP、skills 等公开资料可参考 |

真正值得学的不是“谁更像谁”，而是两类产品都在解决同一组问题：长上下文、工具副作用、安全审批、多入口复用、扩展生态和用户可理解的事件反馈。
