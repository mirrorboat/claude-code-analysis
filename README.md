# claude code analysis

by CodeX

这份仓库是从 `@anthropic-ai/claude-code` 的 source map 还原出来的，所以目录结构更像“运行时模块地图”而不是原始仓库的源码分层。

先给一个总判断：最大的几块是 `utils/`、`components/`、`commands/`、`tools/`、`services/`，它们分别对应“底层能力”“终端 UI”“用户命令”“模型可调用工具”“业务/集成服务”。

## 整体运行链路
- `src/main.tsx` 是真正的总入口，负责启动前的预热、配置、鉴权、远程模式、MCP、插件、模型、任务、会话、遥测等全局流程；它不是简单的“启动 UI”，而是整个 Claude Code runtime 的编排中心。[main.tsx](src/main.tsx#L1)
- `src/commands.ts` 是命令注册中心，统一把 `commands/*` 下的各个子命令装配成可执行命令，并且大量使用动态导入和 feature flag 做懒加载和 dead-code elimination。[commands.ts](src/commands.ts#L1)
- `src/entrypoints/cli.tsx` 是 CLI 启动壳，专门处理 `--version`、`--dump-system-prompt`、bridge、daemon、bg sessions 等“快路径”。[entrypoints/cli.tsx](src/entrypoints/cli.tsx#L1)
- `src/entrypoints/init.ts` 是全局初始化流程，负责安全环境变量、证书、遥测、代理、LSP、scratchpad、远程设置、清理钩子等。[entrypoints/init.ts](src/entrypoints/init.ts#L1)
- `src/entrypoints/mcp.ts` 把内部工具暴露成 MCP server，对外提供 `ListTools` / `CallTool`。[entrypoints/mcp.ts](src/entrypoints/mcp.ts#L1)
- `src/bootstrap/state.ts` 是全局 session 状态仓库，存放 cwd、模型、权限、遥测、插件、会话 ID、远程控制状态等跨模块共享状态。[bootstrap/state.ts](src/bootstrap/state.ts#L1)
- `src/state/AppState.tsx` 是 React 侧的主状态 provider 和 hooks，负责把 `AppStateStore`、设置变更、mailbox、voice context 等挂到组件树上。[state/AppState.tsx](src/state/AppState.tsx#L1)

## 逐个目录分析

- `assistant/`：只有 `sessionHistory.ts`，说明这是一个非常聚焦的助手模式历史管理目录，主要用于保存/回放 assistant 相关会话轨迹，属于 Kairos/assistant 线上的配套状态模块。
- `bootstrap/`：全局启动态与运行时共享状态的“底座”，核心是 `state.ts`。这里存的不是 UI 状态，而是 session 生命周期级别的全局变量。
- `bridge/`：远程控制/桥接模式的完整实现层，包含配置、启用逻辑、消息通道、权限回调、会话创建、poll/config、trusted device、JWT、UI 与主流程。它负责把本地 CLI 变成可被远程操控的 bridge 环境。[bridge/bridgeMain.ts](src/bridge/bridgeMain.ts#L1)
- `buddy/`：AI 伴侣/宠物式 UI 模块，文件很少，但职责很明确：角色形象、提示文案、通知，以及 buddy 相关类型。它更像一个“增强陪伴感”的可选前台功能。
- `cli/`：低层 CLI 辅助层，不是用户命令本身，而是命令执行的基础设施。`handlers/` 里放的是懒加载的子命令处理器，`transports/` 里是 NDJSON、SSE、WebSocket、Worker uploader 等不同 I/O 传输方式。[cli/handlers/util.tsx](src/cli/handlers/util.tsx#L1) [cli/handlers/agents.ts](src/cli/handlers/agents.ts#L1)
- `commands/`：最大的用户命令面，207 个文件，基本就是所有 `/xxx` 或 `claude xxx` 子命令。这里按功能拆成很多子目录，比如：
  - `commands/login`, `logout`, `config`, `permissions`, `model`, `theme`, `output-style`, `privacy-settings`, `rate-limit-options`, `sandbox-toggle` 这些偏配置/设置；
  - `commands/session`, `resume`, `rewind`, `summary`, `status`, `clear`, `compact`, `copy`, `diff`, `export`, `usage` 这些偏会话与内容管理；
  - `commands/plugin`, `mcp`, `skills`, `reload-plugins`, `install-github-app`, `install-slack-app`, `remote-setup`, `remote-env` 这些偏集成与扩展；
  - `commands/review`, `plan`, `tasks`, `agents`, `branch`, `commit`, `commit-push-pr`, `thinkback`, `thinkback-play` 这些偏工作流；
  - `commands/doctor`, `debug-tool-call`, `heapdump`, `perf-issue`, `bughunter`, `ant-trace`, `mock-limits` 这些偏诊断/实验；
  - 大量 `*.tsx` 说明很多命令带交互式 TUI；`index.ts` 往往是对外注册点。[commands/install-github-app/index.ts](src/commands/install-github-app/index.ts#L1)
- `components/`：终端 UI 组件库，389 个文件，是整个交互界面的外壳和控件集合。重点子目录职责很清晰：
  - `components/messages/` 负责把 assistant/user/tool/system 消息渲染成时间线；
  - `components/permissions/` 负责各种权限申请弹窗；
  - `components/PromptInput/` 负责输入框、历史、提示建议、快捷键和队列命令；
  - `components/agents/` 是 agent 列表、编辑器、向导；
  - `components/mcp/` 是 MCP 服务列表、工具详情、连接恢复；
  - `components/tasks/` 是后台任务、shell/agent/dream 任务状态展示；
  - `components/design-system/`、`components/ui/`、`components/shell/`、`components/wizard/` 是通用视觉与交互基础；
  - `components/LogoV2/`、`components/DesktopUpsell/`、`components/HelpV2/`、`components/FeedbackSurvey/` 则是品牌、帮助和运营引导界面。[components/App.tsx](src/components/App.tsx#L1)
- `constants/`：静态常量集合，像 API 限额、beta 标志、错误 ID、文件常量、GitHub App 常量、系统提示片段等，属于“不会频繁变但又要全局共享”的数据。
- `context/`：React context 集合，包含 mailbox、modal、overlay、prompt overlay、notifications、stats、fps metrics、queued message 等，作用是把横切状态传到 UI 树深处。
- `coordinator/`：协调者模式的开关与逻辑，名字只有 `coordinatorMode.ts`，说明它是一个非常小但很核心的多 agent 协同入口。
- `entrypoints/`：进程级入口层，主要是 `cli.tsx`、`init.ts`、`mcp.ts` 和 SDK/类型定义。这里不是“业务目录”，而是“从进程开始跑起来”的接口层。[entrypoints/mcp.ts](src/entrypoints/mcp.ts#L1)
- `hooks/`：React hooks 集合，既有通用 hooks，也有 `notifs/` 里的通知 hook 和 `toolPermission/` 里的权限相关 hook。它更像是 UI 层的行为逻辑库，而不是普通工具函数。
- `ink/`：终端渲染底层，提供类似 React DOM 的终端版基础设施。这里有 ANSI、双向文本、focus、layout、terminal I/O、组件原语等，是整个 TUI 的“渲染引擎”层。
- `keybindings/`：快捷键系统，包含默认绑定、用户绑定加载、解析器、匹配器、保留快捷键和 provider setup。它负责把物理键盘映射成 Claude Code 的交互动作。
- `memdir/`：记忆目录管理，负责 memory 文件发现、扫描、年龄排序、路径管理、团队记忆路径和提示模板。它属于记忆系统的文件层。
- `migrations/`：持久化设置/模型/功能的迁移脚本，比如把旧配置迁到新配置、把旧模型名迁到新模型名。这一层本质上是升级兼容的“历史债务处理器”。
- `moreright/`：只有 `useMoreRight.tsx`，很像一个很小的布局辅助 hook，作用通常是处理终端布局里“向右再偏一点”的特殊排版。
- `native-ts/`：TypeScript 对 native addon 的封装，包含 `color-diff`、`file-index`、`yoga-layout`。这说明部分性能敏感能力由原生实现承接，TS 只做接口包装。
- `outputStyles/`：输出风格加载器，负责从目录加载不同输出样式配置，给命令输出和 UI 文本做主题化/格式化。
- `plugins/`：插件系统的高层入口，`builtinPlugins.ts` 管内建插件注册与启用状态，`bundled/` 管随产品发布的 bundled skills/plugins。它是“可扩展能力”的装配层。[plugins/builtinPlugins.ts](src/plugins/builtinPlugins.ts#L1)
- `query/`：查询执行时的只读配置快照，`config.ts` 里把 session ID 和运行时 gate 固化成 QueryConfig，便于后续把 query 逻辑重构成纯 reducer。
- `remote/`：远程会话侧的 manager 和 WebSocket/permission adapter，偏“远程会话控制面”，比 `bridge/` 更接近 session 传输与同步本身。
- `schemas/`：模式定义目录，当前看主要是 hooks 相关 schema，说明这里承担的是结构校验/类型契约，不是业务逻辑。
- `screens/`：全屏页面级 UI，当前有 `Doctor`、`REPL`、`ResumeConversation`。它比 `components/` 更高一层，是整页/整屏级别的交互屏幕。
- `server/`：direct connect 相关服务，负责直接连接会话、manager 和类型。它更像是远程/直连场景下的服务器端支撑。
- `services/`：业务和集成服务层，130 个文件，覆盖面最广。按子目录看：
  - `services/api/` 是 HTTP/API 客户端与数据拉取层，像 bootstrap、filesApi、referral、usage、session ingress、overage credit、prompt cache、grove、claude 等；
  - `services/mcp/` 是 MCP 协议栈和官方 registry、OAuth、config、client、vscode SDK 适配；
  - `services/analytics/` 是遥测、GrowthBook、Datadog、1P event logger、sink/exporter；
  - `services/lsp/` 是语言服务器管理与诊断；
  - `services/compact/` 是会话压缩与 micro-compact 相关逻辑；
  - `services/oauth/`、`services/policyLimits/`、`services/remoteManagedSettings/`、`services/settingsSync/`、`services/teamMemorySync/` 是鉴权、策略、远程设置和同步；
  - `services/PromptSuggestion/`、`services/SessionMemory/`、`services/autoDream/`、`services/extractMemories/`、`services/toolUseSummary/`、`services/AgentSummary/` 则是围绕对话、记忆、建议和摘要的智能服务；[services/tips/tipRegistry.ts](src/services/tips/tipRegistry.ts#L1)
- `skills/`：技能系统的装配层，`bundledSkills.ts` 管内置技能，`loadSkillsDir.ts` 管从目录加载技能，`mcpSkillBuilders.ts` 负责把技能和 MCP/插件生态结合起来。
- `state/`：应用状态仓库，`AppStateStore.ts`、`AppState.tsx`、`store.ts`、`selectors.ts`、`onChangeAppState.ts` 共同负责状态建模、订阅和更新。
- `tasks/`：任务执行模型层，`LocalShellTask`、`LocalAgentTask`、`InProcessTeammateTask`、`DreamTask`、`LocalMainSessionTask` 等把“正在做的事情”抽象成不同 task 类型。`LocalShellTask` 尤其重要，它负责后台 shell 命令、输出监控、卡住检测和任务完成通知。[tasks/LocalShellTask/LocalShellTask.tsx](src/tasks/LocalShellTask/LocalShellTask.tsx#L1)
- `tools/`：模型最核心的执行能力层，184 个文件，是真正“LLM 可以调用什么”的实现。子目录功能非常明确：
  - `AgentTool/` 负责创建、运行、恢复、展示 agent；
  - `BashTool/` 和 `PowerShellTool/` 负责 shell 命令执行、权限、沙箱、静默/破坏性命令识别；
  - `FileReadTool/`、`FileWriteTool/`、`FileEditTool/`、`GlobTool/`、`GrepTool/`、`NotebookEditTool/` 负责文件与代码操作；
  - `LSPTool/` 负责语言服务器辅助；
  - `MCPTool/`、`ListMcpResourcesTool/`、`ReadMcpResourceTool/`、`McpAuthTool/` 负责 MCP 协议工具；
  - `WebSearchTool/`、`WebFetchTool/` 负责联网获取信息；
  - `SkillTool/`、`ToolSearchTool/`、`Task*`、`Team*`、`SendMessageTool/`、`RemoteTriggerTool/`、`AskUserQuestionTool/`、`SleepTool/`、`SyntheticOutputTool/` 等是更高层的协作/控制/生成工具；
  - `shared/` 放公共工具逻辑，`testing/` 放测试辅助。
  `BashTool` 本身展示了工具层的典型职责：解析命令、做安全审查、决定是否进沙箱、生成 UI 文本、与后台 task 联动。[tools/BashTool/BashTool.tsx](src/tools/BashTool/BashTool.tsx#L1)
- `types/`：跨模块共享类型定义，比如 command、hooks、ids、logs、permissions、plugin、textInputTypes 等，是类型层的公共合同。
- `upstreamproxy/`：CCR/远程环境里的上游代理转发层，负责 relay 和 proxy 本体，属于网络基础设施。
- `utils/`：最大的支撑库，564 个文件，几乎所有“非 UI、非命令、非工具”的能力都在这里。按子目录看最关键的是：
  - `utils/bash/`：bash 解析、AST、命令切分、引用、specs；
  - `utils/model/`：模型配置、能力、provider、模型字符串、别名、deprecated/deprecation、allowlist；
  - `utils/permissions/`：权限模式、规则、分类器、自动模式、危险命令、路径校验、权限申请和变更；
  - `utils/plugins/`：插件发现、加载、缓存、市场、MCP/LSP 集成、安装管理；
  - `utils/settings/`：设置读取、校验、变更检测、MDM、迁移、内部写入；
  - `utils/swarm/`：多 agent 编排、tmux/iterm/in-process backend、teammate、重连、权限同步；
  - `utils/telemetry/`：session tracing、Perfetto/BigQuery 导出、plugin/skill telemetry；
  - `utils/shell/`：shell provider 选择、read-only 验证、PowerShell/bash 兼容；
  - `utils/secureStorage/`：keychain、明文 fallback、预热；
  - `utils/claudeInChrome/`、`utils/computerUse/`、`utils/deepLink/`、`utils/hooks/`、`utils/processUserInput/`、`utils/sandbox/`、`utils/task/`、`utils/teleport/`、`utils/git/`、`utils/github/`、`utils/memory/` 等分别覆盖浏览器集成、电脑控制、深链、hook、输入分发、沙箱、任务输出、远程 teleport、git、GitHub 和记忆。
  这一层可以理解为“系统运行的工具箱和规则引擎”。
- `vim/`：Vim 模式的状态机、动作、对象和过渡逻辑，负责把编辑体验做成 modal editing。
- `voice/`：语音模式开关和相关逻辑，文件很少，说明这是一个独立但相对轻量的功能模块。

**你可以把这个仓库理解成 5 层**
- 入口层：`main.tsx`、`entrypoints/`
- 状态层：`bootstrap/`、`state/`、`context/`
- 能力层：`tools/`、`tasks/`、`services/`、`utils/`
- 交互层：`components/`、`screens/`、`hooks/`、`ink/`
- 扩展层：`commands/`、`plugins/`、`skills/`、`bridge/`、`remote/`、`assistant/`、`buddy/`

## 重要的二级目录
我先挑 10 个最影响产品行为的二级目录来讲，这些目录基本决定了 Claude Code 的“怎么交互、怎么执行、怎么扩展、怎么控风险”。

**1. `commands/plugin`**
- 这是插件管理命令的入口层，只有一个 `index.tsx`，说明它本质上是一个“命令壳”，负责把 `/plugin`、`/plugins`、`/marketplace` 这些别名导到真正的交互页。[commands/plugin/index.tsx](src/commands/plugin/index.tsx#L1)
- 它不直接做重逻辑，而是把插件管理 UI 延迟加载出去，避免启动时把整个插件界面都拉进来。
- 这一层对应的是“用户如何进入插件管理”，不是“插件怎么加载”。

**2. `commands/review`**
- 这是 PR review 的命令层，`/review` 走本地 code review prompt，`/ultrareview` 则是远程 web 侧的 bughunter 路径。[commands/review.ts](src/commands/review.ts#L1)
- 这个目录的职责非常明确：把“我想 review 一个 PR”转成可执行的提示词或远程 review 流程。
- 这里能看出产品把“普通 review”和“超长/更重的 remote review”分成了两条入口。

**3. `components/messages`**
- 这是聊天时间线里所有消息气泡的渲染中心，负责 assistant 文本、tool use、system 消息、折叠内容、进度提示等。[components/messages/AssistantToolUseMessage.tsx](src/components/messages/AssistantToolUseMessage.tsx#L1)
- `AssistantToolUseMessage` 里能看到它要做的事很多：根据 tool name 找工具、解析输入、区分 queued/resolved/error 状态、渲染进度条和工具名、处理透明包装工具。
- 简单说，这个目录负责把“模型在做什么”翻译成用户可读的终端消息。

**4. `components/permissions`**
- 这是权限申请弹窗和审批工作流目录，像 Bash、FileEdit、FileWrite、ComputerUse、Plan Mode 之类的确认界面都在这里。[components/permissions/BashPermissionRequest/BashPermissionRequest.tsx](src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx#L1)
- 这里不只是“弹窗 UI”，而是把权限决策、分类器建议、破坏性命令警告、沙箱提示、快捷键反馈整合成一条审批链。
- 从 `BashPermissionRequest` 能看出它会处理 compound command、sed edit、自动批准提示、权限解释器，这就是整个安全执行面的前台。

**5. `components/PromptInput`**
- 这是底部输入框的大脑，几乎所有用户输入相关的状态都从这里汇聚：历史记录、命令建议、模型切换、fast mode、vim mode、agent/team 视图、MCP、voice、fast icon、stash prompt、队列命令等。[components/PromptInput/PromptInput.tsx](src/components/PromptInput/PromptInput.tsx#L1)
- 它不是一个普通输入框，而是一个“输入调度中枢”：输入方式、提交方式、快捷键、上下文建议、队列、提示和侧问答都在这里串起来。
- 如果你想理解“Claude Code 为什么能在一个终端里做这么多事”，这个目录最值得看。

**6. `tools/AgentTool`**
- 这是多 Agent 能力的核心实现目录，负责 spawn agent、恢复 agent、background agent、worktree/remote isolation、teammate 协作、agent 进度、系统 prompt 拼装、权限模式和任务生命周期。[tools/AgentTool/AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx#L1)
- 它的输入/输出 schema 很重，说明它不是“一个简单工具”，而是一个完整的子执行系统。
- 你可以把它理解成：主模型如何把子任务分配给别的 agent，以及如何收回结果。

**7. `tools/BashTool`**
- 这是最关键的执行工具之一，负责 shell 命令解析、语义识别、安全校验、沙箱决策、读写/搜索命令折叠、破坏性命令警告和输出消息渲染。[tools/BashTool/BashTool.tsx](src/tools/BashTool/BashTool.tsx#L1)
- 它会把命令分成“搜索/读取/列表/静默/危险”等不同语义，再决定怎么展示、怎么审批、是否自动后台执行。
- 从这个目录能直接看出产品对 shell 执行是“强管控”的，不是简单 `exec` 一把梭。

**8. `services/mcp`**
- 这是 MCP 协议与服务器/客户端的业务层，负责连接、认证、资源/工具枚举、tool call、elicitation、流式传输、WebSocket/stdio/streamable HTTP transport、输出持久化等。[services/mcp/client.ts](src/services/mcp/client.ts#L1)
- 这里同时管“怎么连 MCP 服务”和“怎么把 MCP tool 安全地映射成 Claude Code 的 tool”。
- 和 `components/mcp` 相比，`services/mcp` 是协议和数据层，`components/mcp` 是 UI 层。

**9. `utils/permissions`**
- 这是权限系统的规则引擎目录，包含权限模式、规则解析、自动模式、危险命令识别、路径校验、deny tracking、permission update、classifier 决策等。[utils/permissions/permissionSetup.ts](src/utils/permissions/permissionSetup.ts#L1)
- 这个目录决定“什么命令能自动放行、什么必须审批、什么会被判定为危险”，是安全策略的核心。
- 它不仅处理 Bash，也处理 PowerShell、文件系统、auto mode 和 settings 绑定，所以它是权限逻辑的总入口。

**10. `utils/plugins`**
- 这是插件加载、命令/技能提取、市场管理、安装、缓存、变量替换、依赖解析和 telemetry 的底层目录。[utils/plugins/loadPluginCommands.ts](src/utils/plugins/loadPluginCommands.ts#L1)
- 它负责把插件目录里的 markdown/skill 文件变成可执行命令，还要处理 frontmatter、命名空间、用户配置替换和重复路径去重。
- 如果说 `commands/plugin` 是“入口 UI”，那 `utils/plugins` 就是“插件系统真正的发动机”。

**11. `utils/swarm`**
- 这是多 Agent / teammate 编排层，负责 backend 选择、tmux/iTerm/in-process backend 探测、spawn、重连、权限同步、布局管理和 teammate 初始化。[utils/swarm/backends/registry.ts](src/utils/swarm/backends/registry.ts#L1)
- 它最关键的职责是决定“多 Agent 跑在哪里”：tmux、iTerm2、in-process，还是 fallback。
- 这个目录说明 Claude Code 不只是单 agent 推理器，而是有完整的 swarm/teammate 运行模型。

**我建议你下一步继续看这 3 组**
1. `components/messages` + `components/permissions` + `components/PromptInput`，这三块能把终端交互链路看透。
2. `tools/AgentTool` + `tools/BashTool` + `utils/permissions`，这三块能把执行、安全和多 agent 分配看透。
3. `services/mcp` + `utils/plugins` + `utils/swarm`，这三块能把扩展生态和协作机制看透。
