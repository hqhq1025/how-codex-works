# 参考资料与来源

这一页记录这份导读参考过的资料、每类资料适合支撑什么判断，以及哪些资料只作为背景阅读。源码路径、关键 crate 和验证命令已经单独放到 [源码索引与命令速查](./source-index.md)。

资料使用原则很简单：产品定位和公开功能以官方文档为准；实现机制以 `openai/codex` 公开源码为准；社区文章用于补充使用场景、体验评价和问题意识；闭源工具的内部机制不写成事实。

本版源码核对快照是 `openai/codex@87bc724`，提交日期 2026-04-25，提交信息为 `[codex] remove responses command (#19640)`。

## 项目形态参考

| 资料 | 参考内容 | 边界 |
|------|----------|------|
| [Windy3f3f3f3f/how-claude-code-works](https://github.com/Windy3f3f3f3f/how-claude-code-works) | 中文源码教程的组织方式、章节密度、图文结构、资料来源意识和学习路线设计 | 只作为教程形态参考；不照搬内容，不把 Claude Code 的闭源内部机制当作 Codex 源码事实 |

## 资料可信度分层

| 层级 | 资料类型 | 用来支撑什么 | 不用来支撑什么 |
|------|----------|--------------|----------------|
| A | 官方源码与官方文档 | Codex CLI/runtime 的实现事实、产品边界、配置和命令 | 其他闭源 coding agent 的内部实现 |
| B | 官方发布资料和技术媒体报道 | 首次发布时的定位、开源属性、风险提醒 | 具体源码机制 |
| C | 社区源码导读和专题文章 | 阅读路线、使用经验、失败模式、对比视角 | 未经源码核对的内部细节 |
| D | 讨论区、评论、个人测评 | 哪些功能被讨论、被称道或被吐槽 | 普遍结论、性能数字、内部原因 |

## 官方源码与文档

| 资料 | 用在导读里的位置 | 作用 |
|------|------------------|------|
| [openai/codex](https://github.com/openai/codex) | 全文 | 官方仓库，确认开源许可证、目录结构、README、Rust workspace 和源码快照 |
| [openai/codex@87bc724](https://github.com/openai/codex/tree/87bc72408c5ef08f8d21f2cdd00c55451c3be33f) | 全文 | 固定源码核对版本，避免主分支变化导致章节漂移 |
| [README.md](https://github.com/openai/codex/blob/main/README.md) | 首页、01、08、11、18 | 确认 Codex CLI 本地运行、IDE、desktop app、Codex Web 的边界 |
| [codex-rs/README.md](https://github.com/openai/codex/blob/main/codex-rs/README.md) | 01、08、18 | 确认 Rust CLI 是维护主线，以及 MCP、`codex exec`、sandbox 等 Rust CLI 能力 |
| [codex-rs/docs/protocol_v1.md](https://github.com/openai/codex/blob/main/codex-rs/docs/protocol_v1.md) | 02、03、07、08、11 | 确认 queue-pair、Session、Task、Turn 的官方术语 |
| [docs/config.md](https://github.com/openai/codex/blob/main/docs/config.md) | 05、07、09、14、18 | 确认 MCP 配置、approval、notify、state DB、config schema 等入口 |
| [docs/exec.md](https://github.com/openai/codex/blob/main/docs/exec.md) | 08、18 | 核对 `codex exec` 使用方式和 headless 语义 |
| [docs/sandbox.md](https://github.com/openai/codex/blob/main/docs/sandbox.md) | 05、18 | 辅助理解 sandbox 模式和平台边界 |
| [docs/agents_md.md](https://github.com/openai/codex/blob/main/docs/agents_md.md) | 09、13 | 确认 `AGENTS.md` 的规则注入路径和范围 |
| [docs/skills.md](https://github.com/openai/codex/blob/main/docs/skills.md) | 09、13、18 | 辅助核对 skills 的定位和本地发现方式 |
| [codex-rs/app-server/README.md](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md) | 07、08、11、18 | 确认 app-server JSON-RPC、Thread/Turn/Item、review、compact、goals、plugin/app API |

## 官方开发者文档

| 资料 | 用在导读里的位置 | 作用 |
|------|------------------|------|
| [Codex 文档首页](https://developers.openai.com/codex) | 参考入口 | 确认官方文档目录和功能范围 |
| [CLI overview](https://developers.openai.com/codex/cli) | 08、18 | 确认 CLI 本地运行、Rust、计划包含关系和基本定位 |
| [CLI features](https://developers.openai.com/codex/cli/features) | 08、11、18 | 参考 interactive mode、review、web search、approval modes、`exec`、MCP、slash commands |
| [Command line options](https://developers.openai.com/codex/cli/reference) | 08、09 | 核对 CLI flags、model、sandbox、approval 等命令行入口 |
| [Sandbox](https://developers.openai.com/codex/concepts/sandboxing) | 05、18 | 区分 sandbox 技术边界和 approval 决策边界 |
| [Agent approvals & security](https://developers.openai.com/codex/administration/agent-approvals-security) | 05、14、18 | 补充审批、安全和管理策略背景 |
| [Non-interactive mode](https://developers.openai.com/codex/noninteractive) | 08、18 | 参考 `codex exec`、stdin piping、CI 自动修复示例 |
| [App Server](https://developers.openai.com/codex/app-server) | 07、08、18 | 补充 app-server 自动化接口、事件和集成语义 |
| [Model Context Protocol](https://developers.openai.com/codex/mcp) | 07、09、18 | 确认 CLI/IDE 中 MCP server 配置、OAuth、常见服务器和共享配置 |
| [MCP Server guide](https://developers.openai.com/codex/guides/agents-sdk) | 07、18 | 辅助理解 Codex 与 Agents SDK/MCP 方向的关系 |
| [Agent Skills](https://developers.openai.com/codex/skills) | 09、13、18 | 确认 skills 的目录层级、触发方式、metadata 和最佳实践 |
| [Plugins](https://developers.openai.com/codex/plugins) | 09、13、18 | 确认 plugins 打包 skills、apps、MCP servers 的分发模型 |
| [Build plugins](https://developers.openai.com/codex/plugins/build) | 09、18 | 参考插件构建和 marketplace 语义 |
| [Hooks](https://developers.openai.com/codex/hooks) | 09、14、18 | 确认 `PreToolUse`、`PostToolUse`、`Stop`、`UserPromptSubmit` 等 hook 事件和输出协议 |
| [Subagents 概念](https://developers.openai.com/codex/concepts/subagents) | 15、18 | 确认 subagents 需要显式触发、适合并行噪声工作、会增加 token 成本 |
| [Subagents 配置](https://developers.openai.com/codex/subagents) | 15、18 | 参考 agent role、CSV fan-out、并发和配置样例 |
| [Codex app features](https://developers.openai.com/codex/app/features) | 07、11、18 | 参考 app 的 worktree、automation、review pane、browser、computer use、image 支持 |
| [Worktrees](https://developers.openai.com/codex/app/worktrees) | 11、18 | 参考 Codex app 的 worktree、handoff、detached HEAD 和清理策略 |
| [Automations](https://developers.openai.com/codex/app/automations) | 11、16、18 | 参考后台自动化、thread automation、sandbox 风险和示例 |
| [IDE extension features](https://developers.openai.com/codex/ide/features) | 07、08、18 | 参考 IDE 中 open files、selected code、image input、cloud handoff 等体验 |

## 社区源码导读和专题文章

| 资料 | 用在导读里的位置 | 使用方式 |
|------|------------------|----------|
| [nothankyouzzz/codex-notes](https://github.com/nothankyouzzz/codex-notes) | 02、03、参考资料页 | 中文学习笔记，适合辅助建立 agent loop 的阅读路线 |
| [NeuZhou/awesome-ai-anatomy codex-cli](https://github.com/NeuZhou/awesome-ai-anatomy/tree/main/codex-cli) | 01、11、参考资料页 | 英文架构拆解，适合补充高层架构视角 |
| [AlexKenbo/codex-harness-internals](https://github.com/AlexKenbo/codex-harness-internals) | 参考资料页 | 反向整理的 harness 资料，适合查概念；关键数字需要回源码核对 |
| [Codex CLI Context Compaction](https://codex.danielvaughan.com/2026/03/31/codex-cli-context-compaction-architecture/) | 17、18 | 提供 compaction 使用侧解释、触发路径和失败模式线索；实现结论回 `compact.rs` 核对 |
| [Codex CLI Review Workflows](https://codex.danielvaughan.com/2026/03/30/codex-cli-review-command-code-review-workflows/) | 16、18 | 提供 `/review` 工作流、review model 和 MCP review 延伸用法；实现结论回 `ReviewTask` 核对 |
| [Codex CLI + Claude Code MCP workflow](https://www.jdhodges.com/blog/codex-cli-claude-code-mcp-speeds-command-line/) | 07、15、18 | 作为社区把 Codex 当 MCP 工具接入其他 agent 的案例，不把个体测速泛化 |

## 社区讨论与媒体报道

| 资料 | 用在导读里的位置 | 使用方式 |
|------|------------------|----------|
| [Hacker News 发布讨论](https://news.ycombinator.com/item?id=43708025) | 18 | 观察社区对开源、本地终端、sandbox、MCP、上下文、成本和竞品的讨论点 |
| [HN Algolia item 43708025](https://hn.algolia.com/api/v1/items/43708025) | 18 | 便于抓取和核对 HN 讨论文本 |
| [TechCrunch 发布报道](https://techcrunch.com/2025/04/16/openai-debuts-codex-cli-an-open-source-coding-tool-for-terminals/) | 18 | 参考首次发布时的公开定位、开源属性、多模态命令行和安全风险提醒 |

## 章节来源映射

| 章节 | 主要资料来源 |
|------|--------------|
| 首页、00、01 | 官方 README、`codex-rs/README.md`、`protocol_v1.md`、源码快照 |
| 02 Agent Loop | `session/handlers.rs`、`session/turn.rs`、`protocol_v1.md`、社区 agent loop 笔记 |
| 03 协议层 | `protocol.rs`、`protocol_v1.md`、app-server README |
| 04、19 工具系统与工具图鉴 | `tools/spec.rs`、`tool_registry_plan.rs`、`router.rs`、`registry.rs`、`orchestrator.rs`、MCP 官方文档、公开可见竞品工具体验 |
| 05 Sandbox 与安全 | Sandbox 官方文档、`sandboxing`、`guardian`、`network-proxy`、exec policy |
| 06、17 上下文压缩 | `compact.rs`、`compact_remote.rs`、`turn.rs`、compaction 社区文章 |
| 07 MCP 与 App Server | app-server README、MCP 官方文档、`mcp-server`、`codex-mcp` |
| 08 CLI、TUI 与 Exec | CLI features、Non-interactive mode、`exec/src/lib.rs`、TUI 源码 |
| 09、13、14 定制与注入 | config 文档、Skills、Plugins、Hooks、`AGENTS.md`、context fragments |
| 10 最小必要组件 | 前面章节的源码结论综合 |
| 11、18 产品体验与称道特性 | 官方 features/app/IDE 文档、HN 讨论、TechCrunch、社区专题文章、源码交叉核对 |
| 12 apply_patch | `apply_patch.rs`、`turn_diff_tracker.rs`、协议事件 |
| 15 多 Agent | Subagents 官方文档、`codex_delegate.rs`、`agent_tool.rs` |
| 16 Task、Review 与 Goals | `tasks/`、`goals.rs`、review 官方与社区资料 |

## 未采用或谨慎采用的资料

| 类型 | 处理方式 |
|------|----------|
| SEO 型竞品对比文章 | 只用于发现可能的讨论主题，不引用未经核对的 benchmark、价格或内部机制 |
| 社交媒体短帖 | 稳定性和可追溯性较弱，除非能回到官方资料或源码，否则不作为主要来源 |
| 闭源工具内部实现推断 | 不写成事实，只能写成公开可见行为或使用体验 |
| 未标明版本的教程 | 版本漂移风险高，内容需要回当前源码快照核对 |
| 单次个人测速 | 可以作为案例，不作为一般性能结论 |

## 后续维护规则

新增章节时，资料来源页需要同步更新三件事：

1. 增加新增资料的链接、用途和可信度。
2. 在章节来源映射里标出该章主要依赖哪些资料。
3. 如果引用社区判断，正文里要能看出它是使用经验或问题线索，不是源码事实。

源码路径、类型名、验证命令不要堆在这一页，统一放到 [源码索引与命令速查](./source-index.md)。这一页只回答资料从哪里来、可信度怎样、被用来支撑什么。
