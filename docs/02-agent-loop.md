# 2. Agent Loop：从一次输入到多轮工具调用

## 核心问题

Codex 如何把一句用户请求变成多轮模型调用和工具执行？答案在 `codex-rs/core/src/session/turn.rs`。这里的 `run_turn` 是普通任务的核心循环，模型每次返回工具调用，Codex 就执行工具、记录结果，再继续下一次模型请求。

## 源码入口

- `codex-rs/core/src/session/handlers.rs`：`submission_loop`
- `codex-rs/core/src/session/turn.rs`：`run_turn`、`run_sampling_request`、`try_run_sampling_request`
- `codex-rs/core/src/stream_events_utils.rs`：处理模型流式事件和输出项
- `codex-rs/core/src/tools/parallel.rs`：工具调用运行时
- `codex-rs/core/src/tasks/`：不同类型的 session task

## 总流程

```mermaid
sequenceDiagram
    participant Client as TUI / App Server / Exec
    participant Loop as submission_loop
    participant Task as SessionTask
    participant Turn as run_turn
    participant Model as ModelClient
    participant Tools as Tool runtime

    Client->>Loop: Submission { Op::UserTurn }
    Loop->>Task: spawn regular task
    Task->>Turn: run_turn(input)
    Turn->>Turn: context updates / hooks / user message
    Turn->>Model: stream prompt + tools
    Model-->>Turn: response events
    Turn->>Tools: tool call futures
    Tools-->>Turn: tool outputs
    Turn->>Turn: append outputs to history
    Turn->>Model: follow-up request
    Model-->>Turn: final assistant message
    Turn-->>Client: TurnCompleted event
```

这张图只画主路径。源码里还要处理几类旁路：中断、审批响应、pending input、hook 阻断、stream 重试、上下文压缩、stop hook 继续执行。读 `turn.rs` 时可以把它拆成三层循环：

| 层级 | 关键函数 | 主要职责 | 不该承担的事 |
|------|----------|----------|--------------|
| Submission loop | `submission_loop` | 接收 `Submission`，按 `Op` 分发 | 不直接调用模型 |
| Turn loop | `run_turn` | 管理一次用户回合里的多次采样、pending input、压缩和 stop hook | 不关心每个工具怎么实现 |
| Sampling loop | `try_run_sampling_request` | 消费模型流事件，把输出项转成 UI 事件和工具 future | 不决定全局 session 生命周期 |

这个分层是 Codex 比 demo loop 深的地方。demo 常见写法是一个 `while` 里同时读用户输入、请求模型、执行工具、打印结果。Codex 把控制消息、模型采样和工具执行分开后，才能支持“模型还在跑时用户发中断”“工具等审批时 UI 继续活着”“stream 断线后重连但不丢 turn 状态”这些真实产品场景。

## 一次 UserTurn 的真实调用链

`Op::UserTurn` 进入 `submission_loop` 后，会被送到 `user_input_or_turn`。这个函数先把 turn-scoped 配置整理成 `SessionSettingsUpdate`，再调用 `new_turn_with_sub_id` 建立新的 turn context。之后才会把用户输入交给 `steer_input`，让它进入当前任务。

```mermaid
flowchart TB
    Sub["Submission { id, op }"] --> Loop["submission_loop"]
    Loop --> Match["match Op"]
    Match --> UserTurn["UserTurn / UserInputWithTurnContext / UserInput"]
    UserTurn --> Update["SessionSettingsUpdate"]
    Update --> NewTurn["new_turn_with_sub_id"]
    NewTurn --> Warn["model/config warnings"]
    Warn --> Steer["steer_input"]
    Steer --> Task["active task"]
    Task --> RunTurn["run_turn"]
```

这里有一个容易忽略的细节：`UserTurn` 带完整的 `cwd`、`approval_policy`、`sandbox_policy`、`model`、`effort`、`service_tier`、`collaboration_mode` 等字段。`UserInput` 是更旧、更轻的入口，依赖 session 里的持久上下文。app-server 或桌面前端如果希望每一轮都能精确控制环境，应该优先理解 `UserTurn` 和 `UserInputWithTurnContext`。

## run_turn 之前先解决上下文注入

`run_turn` 一开始并不会立刻请求模型。它先处理几类“模型前置输入”：

- 根据模型窗口和上一次模型设置判断是否要 pre-sampling compact
- 记录 turn context update，并维护 `reference_context_item`
- 从输入里收集显式 plugin、skill、app/connector mentions
- 解析 skill 依赖，必要时提示安装 MCP 依赖
- 运行 session start hook 和 user prompt submit hook
- 把 skill/plugin 注入项写入 history
- 启动 ghost snapshot 或 turn diff 相关状态

这些动作解释了为什么 Codex 的上下文不只是聊天历史。模型真正看到的是当前 history、base instructions、turn context diff、skill/plugin 注入、工具 specs 和可能的额外上下文的组合。`run_turn` 的主循环之前做这么多准备，是为了让每次采样都建立在当前配置、项目规则和扩展状态上。

## 外层：submission_loop 只做分发

`submission_loop` 在 `session/handlers.rs` 里。它不直接跑模型，而是把不同 `Op` 分发给不同处理器。

常见分支包括：

- `Op::UserInput` / `Op::UserTurn`：开始一个普通用户回合
- `Op::Interrupt`：中断当前回合
- `Op::Compact`：触发上下文压缩
- `Op::ExecApproval` / `Op::ApplyPatchApproval`：把用户审批结果送回等待中的工具调用
- `Op::Shutdown`：关闭 session loop

这个分发层让审批、中断、压缩和普通输入都走同一条队列。工具等待审批时，整个 runtime 仍然可以处理新的控制消息。

## 中层：SessionTask 管生命周期

普通用户请求不会直接调用 `run_turn`。它会被包装成 session task。task 做几件 `run_turn` 之外的事：

- 发出 `TurnStarted` 事件
- 准备或复用模型连接
- 处理 pending input
- 在回合结束后整理最终状态

这样 `run_turn` 可以专注于单个回合的模型循环，不需要承担整个 session 的生命周期。

## 内层：run_turn 管模型和工具循环

`run_turn` 的核心可以简化成：

```text
prepare turn context
record user input and context updates

loop:
  input = history.for_prompt()
  result = run_sampling_request(input, tools)
  if result.needs_follow_up:
    continue
  run stop hooks
  if stop hook asks to continue:
    continue
  break
```

`needs_follow_up` 是这条循环最重要的信号。模型返回普通文本时，回合可以结束；模型返回工具调用时，工具结果会写入历史，下一次模型请求继续处理。

`run_turn` 里有几组状态值得重点看：

| 状态 | 作用 | 为什么重要 |
|------|------|------------|
| `client_session` | turn-scoped 模型连接，会复用 sticky routing / websocket 状态 | stream 重试时不必重建所有状态 |
| `can_drain_pending_input` | 控制用户运行中追加的输入何时进入 history | 避免新输入插到本轮采样之前 |
| `turn_diff_tracker` | 跟踪本轮文件差异，用于 UI 和 turn diff event | 让前端看到本轮实际改了什么 |
| `stop_hook_active` | 防止 stop hook 继续执行时丢状态 | hook 可以要求模型继续补一轮 |
| `last_agent_message` | 保存最终 assistant 文本 | stop hook 和 after-agent hook 会用 |

`pending_input` 是这一章最容易漏掉的点。用户在模型运行期间继续输入时，Codex 不会无脑把它插进当前 prompt。它会先通过 `inspect_pending_input` 交给 hook 检查，允许的输入记录到 history，被阻断的输入会带额外上下文或重新排队。这个机制让前端可以支持“运行中继续说话”，同时不破坏当前采样边界。

## run_sampling_request 做两件事

`run_sampling_request` 负责构建请求并处理重试。

第一步是构建工具。它会根据当前配置、MCP 状态、动态工具、技能和功能开关生成 `ToolRouter`。模型看到的工具列表不是固定的，而是每个回合根据上下文组装。

第二步是构建 prompt。prompt 包含对话历史、工具定义、base instructions、人格和结构化输出 schema 等。这里的关键不是把所有东西都塞进去，而是把当前回合需要的东西按协议格式交给模型客户端。

`run_sampling_request` 还有一个产品级细节：它在 stream 断开时会按 provider 的重试预算重连。如果 websocket 连续失败，`ModelClientSession` 可以切到 fallback transport，并通过 `StreamError`/`Warning` 把状态告诉前端。这样用户看到的不是静默卡住，而是一次可解释的恢复过程。

## try_run_sampling_request 处理流式事件

模型输出是流式的。`try_run_sampling_request` 需要边收事件边更新 UI，并在输出项完成时判断是否要执行工具。

```mermaid
flowchart TB
    Stream["Response stream"] --> Added["OutputItemAdded"]
    Stream --> Delta["OutputTextDelta / ReasoningDelta"]
    Stream --> Done["OutputItemDone"]
    Stream --> Completed["Completed"]

    Added --> UI1["emit item started"]
    Delta --> UI2["emit item updated"]
    Done --> Handler["handle output item"]
    Handler --> Text["assistant message"]
    Handler --> Call["tool call"]
    Call --> InFlight["push tool future"]
    Completed --> Drain["drain in-flight tools"]
    Drain --> Follow["needs_follow_up?"]
```

`OutputItemDone` 是关键分支。普通消息会被记录为 assistant message；工具调用会交给 `ToolRouter::build_tool_call`，然后放入工具运行时。工具执行结束后，结果会作为新的历史项进入下一轮模型请求。

## ResponseEvent 如何变成前端事件

模型客户端返回的是 `ResponseEvent`，前端看到的是 `EventMsg` 或 app-server 映射后的 item 事件。`try_run_sampling_request` 中几个事件最关键：

| `ResponseEvent` | 处理动作 | 结果 |
|-----------------|----------|------|
| `OutputItemAdded` | 建立 `active_item`，必要时创建 tool argument diff consumer | UI 收到 item started |
| `OutputTextDelta` | 解析 assistant 文本片段或 reasoning delta | UI 实时显示文本 |
| `ToolCallInputDelta` | 把工具参数增量交给 diff consumer | `apply_patch` 等工具可以边流式展示结构化变化 |
| `OutputItemDone` | 调用 `handle_output_item_done` | 文本落 history，工具变成 future |
| `Completed` | 更新 token usage，设置 turn diff 标记 | 判断是否需要 follow-up |

这里能看到 Codex 的事件粒度。工具参数还没完全结束时，UI 就可能获得局部结构化更新；模型完成一个输出项时，工具 future 可以进入 `FuturesOrdered`；整个 response 完成后，再统一判断是否需要下一次采样。

## follow-up 的几种来源

`needs_follow_up` 不只代表工具调用。源码里至少有几类继续点：

- 模型返回工具调用，工具输出写回 history 后需要下一次模型请求
- Responses API 的 `end_turn` 为 `false`
- 本轮采样结束后发现有 pending input
- mailbox 有待处理消息，尤其是多 agent 相关场景
- token 到达 auto compact 阈值，同时还有后续工作要做
- stop hook 要求把额外 prompt 写入 history 并继续

这比简单的“有 tool call 就 loop”更接近真实 agent。Codex 的主循环本质上在问：当前 history 里是否已经包含足够信息，可以结束这个 turn；如果不能，就继续采样，但继续之前可能要先压缩、先记录 hook prompt、或先处理 pending input。

## 中断和失败路径

`try_run_sampling_request` 被取消时返回 `CodexErr::TurnAborted`，`run_turn` 会把它当成已报告的中断处理。图片不合法时，Codex 会尝试把最近工具输出里的图片替换成替代文本再继续，避免坏图片污染后续请求。普通错误会转成 `ErrorEvent` 发给前端，让用户还能继续会话。

```mermaid
flowchart TB
    Error["sampling/tool error"] --> Kind{"错误类型"}
    Kind -->|retryable stream| Retry["backoff retry / fallback transport"]
    Kind -->|context window| Compact["auto compact or mark full"]
    Kind -->|turn aborted| Abort["emit aborted path"]
    Kind -->|invalid image| Sanitize["replace tool image and retry"]
    Kind -->|other| Event["send ErrorEvent"]
```

这种失败路径是深读 Codex 时很值得看的部分。生产级 agent 的复杂度往往不在 happy path，而在工具执行一半、stream 掉线、上下文满、用户中断、hook 阻断这些边界上。

## 设计取舍

Codex 没有把工具执行嵌在模型流处理里同步等待。它用 future 管理 in-flight 工具，流结束后再 drain。这让多个工具调用可以并行运行，也让 UI 能持续收到模型文本和工具状态。

另一个重要取舍是所有状态变更都围绕 history 发生。模型只通过历史和工具结果观察世界，工具执行不直接修改模型内部状态。这个约束让回放、压缩、持久化和调试都有统一基础。

## 如果自己做 Agent，可以学什么

先不要急着做复杂 UI。把 agent loop 写成清楚的四段：收输入、构建 prompt、处理模型流、分发工具。只要 `needs_follow_up` 这个信号清晰，复杂工具和长任务都能自然接进去。

同时要把控制消息和普通用户消息放到同一条可排序队列里。中断、审批、压缩都是 agent runtime 的一部分，不应该散落在 UI 回调里。

## 源码级调用链

`run_turn` 不是孤立函数，它只处在调用链中间。完整路径可以按三层看：

| 层 | 关键函数 | 责任 |
|----|----------|------|
| 分发层 | `submission_loop`、`user_input_or_turn` | 从 queue 读取 `Submission`，把 `Op` 分流到任务、审批、配置、中断 |
| 任务层 | `Session::start_task`、`SessionTask`、`RegularTask` | 让一次用户请求拥有可中断、可替换、可记录的生命周期 |
| turn 层 | `run_turn`、`run_sampling_request`、`try_run_sampling_request` | 构建模型请求、消费流式事件、执行工具、判断是否 follow-up |

```mermaid
flowchart TB
    A["Submission"] --> B["submission_loop"]
    B --> C{"Op kind"}
    C -->|UserTurn| D["user_input_or_turn"]
    C -->|Interrupt / approval / answer| E["session control path"]
    D --> F["Session::start_task"]
    F --> G["RegularTask::run"]
    G --> H["run_turn"]
    H --> I["run_pre_sampling_compact"]
    I --> J["record_context_updates_and_set_reference_context_item"]
    J --> K["run_sampling_request"]
    K --> L["try_run_sampling_request"]
    L --> M{"tool calls or pending input?"}
    M -->|yes| N["dispatch tools + record outputs"]
    N --> K
    M -->|no| O["TurnComplete"]
```

`submission_loop` 的重要性在于它把所有外部动作放进同一条顺序流。普通用户输入、审批答复、`request_user_input` 答案、中断、配置变更都不是 UI 回调里的临时状态，而是 core 可以统一处理的 `Op`。

## run_turn 的四段结构

当前源码里，`run_turn` 可以粗略拆成四段。

| 阶段 | 发生什么 | 为什么需要 |
|------|----------|------------|
| turn 准备 | 构建 `TurnContext`，处理 pre-sampling compact | 模型请求前先保证上下文能放进窗口 |
| 上下文注入 | 调 `record_context_updates_and_set_reference_context_item` | 把 AGENTS、skills、plugins、permissions 等变化写进 history |
| 采样循环 | 调 `run_sampling_request`，消费 Responses stream | 让模型输出文本、tool call、reasoning、usage |
| follow-up | 执行工具并把结果作为下一轮输入 | 模型通过工具结果继续判断，直到没有可继续内容 |

```mermaid
stateDiagram-v2
    [*] --> Prepare
    Prepare --> PreCompact: usage too high or model downshift
    Prepare --> InjectContext
    PreCompact --> InjectContext
    InjectContext --> Sampling
    Sampling --> ToolDispatch: tool calls
    ToolDispatch --> RecordOutput
    RecordOutput --> MidTurnCompact: usage high and follow-up needed
    MidTurnCompact --> Sampling
    RecordOutput --> Sampling: follow-up
    Sampling --> Complete: no follow-up
    Sampling --> Error: fatal stream/tool error
    Complete --> [*]
    Error --> [*]
```

## 采样层为什么还要拆成两个函数

`run_sampling_request` 和 `try_run_sampling_request` 的分工容易被忽略。前者偏策略，处理重试、context window exceeded、usage 和上层错误；后者偏事件消费，真正读模型 stream，把 `ResponseEvent` 转成 history item、UI event 和工具调用。

这种拆分的好处是错误恢复不污染流式事件解析。比如 prompt 太长时，上层可以决定是否 compact 或裁剪；流式解析层只负责保证事件顺序、tool call 收集和 output item 合法。

## continue 的来源

Codex 的 turn 不一定因为模型说完文本就结束。只要还有内容要喂回模型，就会继续。

| 来源 | 例子 | 进入方式 |
|------|------|----------|
| 工具调用 | shell、apply_patch、MCP、view_image | 工具 output item 写回 history |
| pending input | 用户在任务中追加 steer | 作为下一轮用户输入合并 |
| API 非终止状态 | Responses API 表示还没 end turn | 保留 response 状态继续采样 |
| compact 后续 | mid-turn compact 之后仍有任务要继续 | initial context 插回合适位置后重采样 |

这个机制解释了 Codex 为什么不是“模型输出一次就结束”。coding agent 的一条用户指令经常对应多轮模型判断，每轮之间由工具结果连接。

## interrupt 和失败路径

中断不是简单 kill 当前 future。任务层要处理几件事：当前 task 要停止，history 里要能记录中断边界，工具进程和 approval 等待要解除，前端还要收到可理解的事件。

| 失败或中断 | 处理目标 |
|------------|----------|
| 用户发 `Op::Interrupt` | 停止当前 task，保留已产生的历史和事件 |
| 新 `UserTurn` 到达 | 旧 task 被打断，新 task 开始，避免两个 task 同时写 session |
| stream 可重试错误 | 由 sampling 层重试，不把临时网络问题直接暴露为任务失败 |
| `ContextWindowExceeded` | 触发 compact 或向上返回，让任务层决定 |
| 工具被拒绝 | 记录拒绝结果，模型可以基于拒绝调整路径 |

如果自己实现 agent，先不要追求复杂恢复策略，但至少要让中断成为协议事件，而不是 UI 层的临时布尔值。

## 逐段源码走读：从 submission 到 turn

这一段适合打开源码对照读。目标不是记住每个分支，而是看清每层只管自己的边界。

### 1. `submission_loop` 只管取消息和分发

`submission_loop` 位于 `codex-rs/core/src/session/handlers.rs`。它从 submission receiver 里拿到 `Submission`，根据 `Op` 类型调用不同 handler。它不负责模型采样，也不直接执行工具。

| 分支 | 处理方式 |
|------|----------|
| `Op::UserTurn` / legacy input | 进入 `user_input_or_turn` |
| approval answer | 唤醒等待中的工具执行 |
| interrupt | 通知当前 task 停止 |
| config/session 类操作 | 更新 session 或返回状态 |

这个分层让审批答复和用户新输入都能按协议顺序进入 core。否则 UI 很容易在模型采样时直接改内部状态。

### 2. `user_input_or_turn` 把输入变成任务

`user_input_or_turn` 会处理用户输入、hook、pending input、任务替换等逻辑。它的重点不是调用模型，而是决定当前输入应该新开任务、打断旧任务，还是作为 pending input 进入已有任务。

```mermaid
flowchart TB
    A["Op::UserTurn"] --> B["user_input_or_turn"]
    B --> C["UserPromptSubmit hook"]
    C --> D{"blocked?"}
    D -->|yes| E["emit blocked feedback"]
    D -->|no| F{"active task?"}
    F -->|yes| G["interrupt or queue pending input"]
    F -->|no| H["start RegularTask"]
```

这解释了一个产品体验：用户在 agent 工作中追加指令，不只是往 stdin 写一行，而是进入 task/turn 的控制逻辑。

### 3. `RegularTask` 让普通对话成为任务

`RegularTask` 在 `codex-rs/core/src/tasks/regular.rs`。普通用户输入并不是直接调 `run_turn`，而是由 task 包住。task 层负责发 `TurnStarted`、处理 pending input、把结果落进 history 和事件流。

| task 层关心 | turn 层不该关心 |
|-------------|-----------------|
| 当前任务是否被 abort | UI 怎么展示中断 |
| pending input 是否要合并 | 某个前端的快捷键 |
| task 完成后如何发事件 | 工具内部怎么跑 shell |
| 是否触发 stop hook 或 goal continuation | 模型 stream 的低层解析 |

### 4. `run_turn` 是模型和工具的闭环

`run_turn` 才是最接近 agent loop 的函数。它先处理压缩和上下文注入，再进入采样。采样产生的 tool call 会被路由、执行、写回 history；只要有 follow-up，就继续采样。

```text
run_turn
  run_pre_sampling_compact
  record_context_updates_and_set_reference_context_item
  loop:
    run_sampling_request
    dispatch tool calls
    record outputs
    maybe mid-turn compact
    break if no follow-up
```

这个伪代码比真实源码少很多分支，但能保留主线。读源码时可以先按这五个节点找，再展开每个节点的细节。

## 逐段源码走读：采样层

### 1. `run_sampling_request` 管策略

`run_sampling_request` 会准备模型请求、处理重试、处理 `ContextWindowExceeded`，并把 token usage 和 response 状态交回 turn 层。它的职责更接近“跑一次采样请求直到有明确结果”。

| 它负责 | 它不负责 |
|--------|----------|
| 调用 `try_run_sampling_request` | 解析每个 SSE 事件的具体内容 |
| 判断是否重试 | 决定 tool handler 的内部逻辑 |
| 把 context window 错误上抛 | 直接修改 UI |
| 汇总 usage 和输出 | 直接执行 shell |

### 2. `try_run_sampling_request` 管事件消费

`try_run_sampling_request` 处理模型返回的 Responses stream。这里会看到 text delta、reasoning、tool call 参数、completed、error 等事件如何转成内部结构。

```mermaid
flowchart TB
    A["Responses stream"] --> B{"event type"}
    B -->|text delta| C["emit AgentMessageContentDelta"]
    B -->|tool call delta| D["accumulate tool arguments"]
    B -->|completed| E["finalize ResponseItem list"]
    B -->|usage| F["record token usage"]
    B -->|error| G["retryable or fatal error"]
    D --> H["ToolCall list"]
    E --> I["history items"]
```

这层的细节会影响 UI 速度。文本 delta 能尽早显示，tool call 参数能边流式边积累，工具执行就不必等到所有 UI 状态都结束后才开始准备。

## 可复用的实现清单

如果要从零实现一个小 agent，按下面顺序做比直接写 while loop 更稳。

| 顺序 | 要实现的最小能力 | 验收方式 |
|------|------------------|----------|
| 1 | `Submission` / `Event` | UI 和 core 不互相调用内部函数 |
| 2 | `SessionTask` | 同一时间只有一个写 history 的任务 |
| 3 | `run_turn` | 文本回复和工具回复都能回到 history |
| 4 | `needs_follow_up` | 多轮工具调用能自然继续 |
| 5 | interrupt | 中断后 history 仍合法 |
| 6 | context window error | 至少能给出清楚错误或触发简化 compact |
| 7 | event log | 出错后能回放到最后一次状态 |
