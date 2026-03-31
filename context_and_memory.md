# Memory管理

可以把 Claude Code 的 memory 系统理解成一套“文件型知识库 + 自动整理 + 按需召回”的机制，而不是传统数据库。

**整体结构**
- 记忆不是存到某个内部数据库里，而是落在磁盘上的 Markdown 文件中。
- 每条记忆通常是一个 topic 文件，外加一个索引文件 `MEMORY.md`。
- 模型每次开始工作时，会先读这些文件，把它们当作长期上下文使用。
- 系统还会在后台周期性整理、去重、更新这些记忆。

**1. 记忆是怎么存的**
- 每条记忆都有类型，主要分四类：
- `user`：用户画像和能力背景。
- `feedback`：用户对 Claude 工作方式的偏好。
- `project`：当前项目里的动态信息、目标、截止时间、协作状态。
- `reference`：外部系统的指引，比如某个 Slack、Linear、Dashboard 是干什么的。
- 这些记忆都要求保存“当前代码和 git 历史推不出来的东西”，避免把可直接推导的信息重复存进去。

**2. 增加记忆**
- 增加记忆主要有两条路。
- 第一条是 turn 结束后的自动抽取：Claude 在完成一轮对话后，会启动一个后台抽取器，扫本轮对话里“值得长期保留”的信息，然后写入记忆文件。
- 第二条是用户明确要求“记住这个”：系统会把它当成高优先级信号，立即写入合适的记忆类型。
- 还有一条辅助路径是定期整理：当积累了足够多的历史信息后，后台会把零散日志和旧记忆重新归纳成更干净的 topic 文件。

**3. 记忆怎么写入**
- 写入时不是直接把所有内容堆进 `MEMORY.md`。
- 正确流程是：
  1. 先创建或更新某个 topic 文件。
  2. 再在 `MEMORY.md` 里加一条指向它的索引。
- 这样做的好处是：索引很短，正文可以比较长；检索时先看索引，再进正文；后续更新也更容易。
- 如果是团队记忆，则还会区分“个人目录”和“团队目录”，分别维护各自的索引。

**4. 记忆怎么更新**
- 更新的核心策略是“不要重复建新文件，优先改旧文件”。
- 自动抽取器和整理器都会先扫描现有记忆，判断已有文件是否已经覆盖相同主题。
- 如果覆盖，就更新旧文件的内容、摘要或索引，而不是再创建一份重复记忆。
- 对于项目型信息，系统还会主动把相对时间改成绝对日期，避免以后看不懂。

**5. 记忆怎么删除**
- 删除没有单独的“删除 API”，本质上还是文件操作。
- 如果某条记忆过时、错误，模型会被要求直接删掉对应 topic 文件，或者从 `MEMORY.md` 中移除索引项。
- 如果用户说“不要记这个”或“忽略这条记忆”，系统会把它当成删除/失效处理，而不是继续保留然后附加一条反向说明。
- 团队记忆比较特殊：本地删掉不等于远端同步删除，所以它更像一套有同步语义的共享文件系统。

**6. 记忆怎么检索**
- 检索分两层。
- 第一层是“预加载”：系统启动时会把记忆目录中的相关内容直接塞进 system prompt，让 Claude 一开始就带着长期背景工作。
- 第二层是“按需召回”：当用户的问题明显和某些记忆有关时，系统会先扫描所有记忆标题和摘要，再挑最相关的少量文件拿出来。
- 这个召回过程不是全量暴读，而是先轻量筛选，再让模型判断相关性，最后只把最有用的几条加入上下文。
- 另外还有 freshness 处理：太旧的记忆会带上“可能过时”的提醒，防止模型把历史快照当成当前事实。

**7. 团队记忆是怎么协作同步的**
- 团队记忆是 repo 级共享的，适合放“对整个项目协作都成立”的知识。
- 它有本地缓存，也有远端服务端副本。
- 启动时会先从远端拉取最新内容，再监听本地目录变化。
- 一旦本地有改动，watcher 会延迟几秒后自动 push 到远端。
- push 不是整包覆盖，而是按文件内容 hash 做增量同步，所以只有变更文件会上传。
- 如果多人同时改同一条记忆，系统会用冲突检测和重试机制处理，尽量避免把别人的更新误删。
- 但它不做复杂的内容级合并，通常是“本地编辑优先、冲突后重算再推”。

**8. 安全和约束**
- 系统会尽量避免把能从代码和历史中推导出来的东西写进 memory，防止记忆库变成重复信息仓库。
- 团队记忆会做 secret 检查，避免把密钥、token 之类的敏感信息同步给所有协作者。
- 路径访问也做了严格约束，防止越权写到 memory 目录外面。
- 记忆系统还有一个重要原则：记忆只代表“曾经成立过”的事实，不能替代当前代码状态。当前代码和当前文件内容永远优先于旧记忆。


# Memory代码分析

**1. 记忆存放在哪里**
- 自动记忆路径由 [paths.ts](/src/memdir/paths.ts#L30) 和 [paths.ts](/src/memdir/paths.ts#L223) 决定，默认落在 `~/.claude/projects/<repo>/memory/`，或由 `CLAUDE_CODE_REMOTE_MEMORY_DIR` / `autoMemoryDirectory` 覆盖。
- 自动记忆的入口文件是 [MEMORY.md](/src/memdir/paths.ts#L253)。
- 团队记忆是自动记忆的子目录 [teamMemPaths.ts](/src/memdir/teamMemPaths.ts#L84)，入口是 [team/MEMORY.md](/src/memdir/teamMemPaths.ts#L92)。
- 记忆类型只有四种：`user`、`feedback`、`project`、`reference`，并且明确排除了能从代码、git history、CLAUDE.md 直接推导出来的内容 [memoryTypes.ts](/src/memdir/memoryTypes.ts#L14) [memoryTypes.ts](/src/memdir/memoryTypes.ts#L183)。

**2. 如何“增加记忆”**
- 主路径是 turn-end 的后台抽取器 `extractMemories`。它在一次完整 query loop 结束后触发 [extractMemories.ts](/src/services/extractMemories/extractMemories.ts#L5) [stopHooks.ts](/src/query/stopHooks.ts#L107)。
- 抽取器会先扫描现有记忆文件清单，再把这份清单塞给 forked subagent，避免它重复造轮子 [extractMemories.ts](/src/services/extractMemories/extractMemories.ts#L395) [memoryScan.ts](/src/memdir/memoryScan.ts#L24)。
- 子 agent 的 prompt 明确要求：先读现有文件，优先更新已有文件而不是创建重复项；只允许在 memory 目录内使用 `Read/Grep/Glob/Bash(只读)/Edit/Write` [extractMemories/prompts.ts](/src/services/extractMemories/prompts.ts#L29)。
- 记忆文件写法是“两步”：先写 topic 文件，再把它挂到 `MEMORY.md` 索引里 [memdir.ts](/src/memdir/memdir.ts#L219) [extractMemories/prompts.ts](/src/services/extractMemories/prompts.ts#L70)。
- 写入内容的分类和格式约束来自 `memoryTypes.ts`，比如 `project` 要把相对时间转成绝对日期，`feedback` 要写出 Why/How to apply [memoryTypes.ts](/src/memdir/memoryTypes.ts#L76) [memoryTypes.ts](/src/memdir/memoryTypes.ts#L79)。
- 手工入口是 `/memory` 命令：它会列出现有记忆文件，允许创建新文件，并直接打开编辑器 [memory.tsx](/src/commands/memory/memory.tsx#L21) [memory.tsx](/src/commands/memory/memory.tsx#L83)。
- 另外还有一个后台整合器 `/dream` / autoDream，会定期把长期积累的日志/上下文整理成 topic 文件和 `MEMORY.md` [autoDream.ts](/src/services/autoDream/autoDream.ts#L2) [consolidationPrompt.ts](/src/services/autoDream/consolidationPrompt.ts#L17)。

**3. 如何“更新记忆”**
- 更新本质上就是编辑已有 topic 文件，而不是新建重复文件。抽取 prompt 和 consolidation prompt 都明确要求“先检查已有文件，优先 update” [extractMemories/prompts.ts](/src/services/extractMemories/prompts.ts#L32) [consolidationPrompt.ts](/src/services/autoDream/consolidationPrompt.ts#L37)。
- `extractMemories` 还会把 `MEMORY.md` 作为机械索引更新的一部分，但它强调“真正的 memory 是 topic 文件” [extractMemories.ts](/src/services/extractMemories/extractMemories.ts#L463)。
- `autoDream` 也会合并新信息、修正旧内容、删掉被证伪的事实，并重新整理索引 [consolidationPrompt.ts](/src/services/autoDream/consolidationPrompt.ts#L46) [consolidationPrompt.ts](/src/services/autoDream/consolidationPrompt.ts#L55)。
- `loadMemoryPrompt()` 会在系统提示中注入记忆目录已经存在的事实，避免模型反复做 `mkdir` 或先检查路径 [memdir.ts](/src/memdir/memdir.ts#L419) [memdir.ts](/src/memdir/memdir.ts#L452)。

**4. 如何“删除记忆”**
- 代码里没有单独的“delete memory API”。删除就是让模型直接删文件，或者把内容从 `MEMORY.md` 索引里移除。
- 这在 prompt 里是显式要求的：用户说 “forget” 时，要“find and remove the relevant entry” [memdir.ts](/src/memdir/memdir.ts#L243) [extractMemories/prompts.ts](/src/services/extractMemories/prompts.ts#L87)。
- `/memory` 命令的 UI 也只是打开文件编辑器，没有特殊删除逻辑；真正删除就是在编辑器里删内容或删文件 [memory.tsx](/src/commands/memory/memory.tsx#L21)。
- 团队记忆有一个重要差异：本地删除不会自动从远端删除。同步层明确写了“File deletions do NOT propagate”，所以下次 pull 还会把远端内容恢复回来 [teamMemorySync/index.ts](/src/services/teamMemorySync/index.ts#L14)。
- 也就是说，团队记忆的删除要靠远端同步语义或后续 pull/push 机制配合，而不是单纯删本地文件就完事。

**5. 如何“检索记忆”**
- 第一层检索是“系统提示词加载”：`loadMemoryPrompt()` 会把 auto/team 记忆目录整段注入到系统 prompt 中；如果启用 team memory，会把两个目录都加载 [memdir.ts](/src/memdir/memdir.ts#L419) [memdir.ts](/src/memdir/memdir.ts#L448)。
- `getMemoryFiles()` 会递归扫描 `CLAUDE.md`、`.claude/rules/*.md`、`CLAUDE.local.md`，以及 `AutoMem` / `TeamMem` 入口文件，用于完整上下文视图 [claudemd.ts](/src/utils/claudemd.ts#L790) [claudemd.ts](/src/utils/claudemd.ts#L979)。
- 第二层检索是“相关性召回”：`findRelevantMemories()` 先扫 memory 目录头信息，再让 Sonnet 从候选里挑最多 5 个相关文件 [findRelevantMemories.ts](/src/memdir/findRelevantMemories.ts#L39) [findRelevantMemories.ts](/src/memdir/findRelevantMemories.ts#L97)。
- 召回出来的文件会通过 `readMemoriesForSurfacing()` 读全文，并在太长时截断，同时加上 freshness note [attachments.ts](/src/utils/attachments.ts#L2279) [attachments.ts](/src/utils/attachments.ts#L2327)。
- 这个相关记忆召回是异步预取，不阻塞主 turn；而且会用 `alreadySurfaced` 去重，避免反复给模型同一批记忆 [attachments.ts](/src/utils/attachments.ts#L2204) [attachments.ts](/src/utils/attachments.ts#L2231)。
- 预取触发点是 `startRelevantMemoryPrefetch()`，它基于最近的真实 user prompt 发起搜索 [attachments.ts](/src/utils/attachments.ts#L2361)。

**6. Team memory 的同步语义**
- 团队记忆是 repo 级共享的，前提是 first-party OAuth 可用 [teamMemorySync/index.ts](/src/services/teamMemorySync/index.ts#L148)。
- `pullTeamMemory()` 会把服务端内容写回本地，服务端优先；`pushTeamMemory()` 则是本地编辑后做 delta upload，遇到 412 会拉 hashes 再重试 [teamMemorySync/index.ts](/src/services/teamMemorySync/index.ts#L767) [teamMemorySync/index.ts](/src/services/teamMemorySync/index.ts#L889)。
- 本地和远端的差异是按文件内容 hash 比较，不是按时间戳 [teamMemorySync/index.ts](/src/services/teamMemorySync/index.ts#L949)。
- watcher 会在目录变更后 debounce 2 秒再 push，并且在启动时先 pull 一次 [watcher.ts](/src/services/teamMemorySync/watcher.ts#L35) [watcher.ts](/src/services/teamMemorySync/watcher.ts#L270)。
- 写团队记忆时还有 secret guard：`FileWriteTool` / `FileEditTool` 会拒绝把潜在 secret 写进 team memory [FileWriteTool.ts](/src/tools/FileWriteTool/FileWriteTool.ts#L112) [FileEditTool.ts](/src/tools/FileEditTool/FileEditTool.ts#L116) [teamMemSecretGuard.ts](/src/services/teamMemorySync/teamMemSecretGuard.ts#L1)。

如果你愿意，我可以下一步继续给你画一张“记忆系统调用链图”，把 `system prompt / extractMemories / relevant_memories / team sync / /memory UI` 这几条线串成一张流程图。

# 节省上下文
根据 `restored-src/src` 里的代码，Claude Code 省上下文不是靠单一技巧，而是一个分层策略。真正减少发给模型的内容，主要有 6 类：

**1. 自动压缩：把旧历史总结成更短的上下文**
- 主入口在 [`query.ts`](src/query.ts#L396-L467)。
- 处理顺序是：先做 `snip`，再做 `microcompact`，再做 `contextCollapse`，最后才进 `autocompact`。
- `autoCompact.ts` 里先算阈值，再决定是否压缩；并且保留一个 `13k` token 左右的 buffer，避免把窗口顶满。
- 当压缩成功后，`compact.ts` 会构建新的消息序列：`boundaryMarker + summaryMessages + messagesToKeep + attachments + hookResults`，也就是把旧历史大面积替换成摘要和少量保留尾巴。
- 关键点：自动压缩不是简单删历史，而是“旧内容总结 + 新内容保留”。

参考：
- [`query.ts`](src/query.ts#L396-L467)
- [`autoCompact.ts`](src/services/compact/autoCompact.ts#L72-L90)
- [`compact.ts`](src/services/compact/compact.ts#L328-L337)

**2. Session Memory：把长期对话压成单独的记忆文件，再拿它替换上下文**
- `SessionMemory` 不是把原始历史一直留着，而是周期性抽取一个 `MEMORY.md` 风格的笔记文件。
- 触发条件在 [`sessionMemory.ts`](src/services/SessionMemory/sessionMemory.ts#L134-L180)：同时看 token 增长、工具调用数、以及最近一轮有没有 tool call。
- 真正抽取时，会 fork 一个子 agent 去更新内存文件，然后把这份文件作为后续压缩的摘要来源。
- 如果 session memory 可用，自动压缩会优先走这条路径：`trySessionMemoryCompaction()` 成功就直接用 session memory 构造压缩结果，失败才回退到传统 summarization。
- 这意味着：历史信息被移出主对话，主上下文只保留“摘要 + 最近尾部”。

参考：
- [`sessionMemory.ts`](src/services/SessionMemory/sessionMemory.ts#L134-L180)
- [`sessionMemory.ts`](src/services/SessionMemory/sessionMemory.ts#L272-L349)
- [`sessionMemoryCompact.ts`](src/services/compact/sessionMemoryCompact.ts#L505-L560)
- [`sessionMemoryCompact.ts`](src/services/compact/sessionMemoryCompact.ts#L437-L503)

**3. Context Collapse：把历史分段折叠成“投影视图”**
- `query.ts` 里明确写了：`contextCollapse.applyCollapsesIfNeeded(...)` 是在自动压缩前跑的。
- 这里的思想不是“再做一次摘要”，而是对历史做一层读时投影，折叠后的视图再送去模型。
- 代码注释里还说明了：折叠信息存在独立 store，不只是 REPL 数组里的内容，所以它能跨 turn 持久化。
- 如果发生 413 / prompt too long，`recoverFromOverflow()` 会先尝试“drain staged collapses”，再考虑 reactive compact。
- 这是一种“先尽量保留粒度，再在真的超限时折叠”的策略。

参考：
- [`query.ts`](src/query.ts#L428-L447)
- [`query.ts`](src/query.ts#L609-L648)
- 另外 `runPostCompactCleanup()` 会在主线程压缩后清理 context-collapse 状态，避免残留状态污染后续轮次：
  - [`postCompactCleanup.ts`](src/services/compact/postCompactCleanup.ts)

**4. Snip：直接删掉中间历史，只保留前后两端**
- `query.ts` 里 `HISTORY_SNIP` 是在 microcompact 之前执行的。
- 它会返回：
  - 新的 `messages`
  - `tokensFreed`
  - 可选的 `boundaryMessage`
- 这里非常关键的一点是，`snip` 删除后，剩余消息的 `tokenCountWithEstimation()` 还是会看到旧 usage，所以代码要把 `snipTokensFreed` 显式减掉，避免误判上下文还超限。
- 从 `sessionStorage.ts` 的注释看，snip 是“删中间段”，会在 transcript 里记录 `removedUuids`，并重连 parent 链，否则 resume 会把被删掉的历史又拼回来。
- 这类机制比摘要更激进，属于“物理裁剪”。

参考：
- [`query.ts`](src/query.ts#L396-L410)
- [`query.ts`](src/query.ts#L592-L640)
- [`sessionStorage.ts`](src/utils/sessionStorage.ts#L1962-L2038)

**5. 相关记忆检索：只注入最相关的 memory 文件**
- `loadMemoryPrompt()` 会把 `MEMORY.md` / `.claude` 记忆系统接进 system prompt，但不是把整个目录暴力塞进去。
- `findRelevantMemories()` 会先扫描 memory 文件头，再用 Sonnet 做一个最多 5 个文件的选择。
- `startRelevantMemoryPrefetch()` 会在用户 turn 开始时异步预取相关记忆，不阻塞主流程。
- 这减少的是“无关长期记忆”进入上下文的概率，尤其适合 memory 文件很多的项目。
- 这里还有明确的截断策略：`MEMORY.md` 只允许 200 行 / 25KB 左右，超了会警告并截断。
- 所以 Claude Code 不是把所有 memory 都塞进 prompt，而是“索引 + 选择 + 截断”。

参考：
- [`memdir.ts`](src/memdir/memdir.ts#L187-L259)
- [`memdir.ts`](src/memdir/memdir.ts#L419-L470)
- [`findRelevantMemories.ts`](src/memdir/findRelevantMemories.ts#L26-L75)
- [`attachments.ts`](src/utils/attachments.ts#L2361-L2423)
- [`prompts.ts`](src/services/SessionMemory/prompts.ts#L1-L40)

**6. 大工具结果外置：把超大 tool result 持久化到磁盘**
- `applyToolResultBudget()` 会在进入模型前检查工具结果大小。
- 太大的结果不会原封不动塞进上下文，而是：
  - 写到 session 目录里的文件
  - 在消息里换成一个预览 + 文件路径
- 这一步对省上下文非常实用，因为 bash / grep / read / web fetch 之类的输出经常是最大块内容。
- 另外它还是确定性的：`reconstructContentReplacementState()` 能在 resume 时重建同样的替换决策，避免 prompt cache 被打乱。

参考：
- [`toolResultStorage.ts`](src/utils/toolResultStorage.ts#L924-L980)
- [`toolResultStorage.ts`](src/utils/toolResultStorage.ts#L1-L120)

**7. 还有一些配套的“减肥”动作**
- `QueryEngine.ts` 里只有在 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 等条件满足时才注入 memory-mechanics prompt，避免无意义地常驻额外指令。
- `SessionMemory/prompts.ts` 会对 session memory 的每个 section 做长度预算，过长时提示压缩。
- `sessionMemoryCompact.ts` 在构造压缩结果时还会把 session memory 自己再截断一次，避免“摘要反而太大”。
- `buildPostCompactMessages()` 通过统一消息顺序减少重复和错位，配合 boundary / preserved segment 元数据，防止重复携带旧上下文。

参考：
- [`QueryEngine.ts`](src/QueryEngine.ts#L310-L325)
- [`prompts.ts`](src/services/SessionMemory/prompts.ts#L226-L246)
- [`sessionMemoryCompact.ts`](src/services/compact/sessionMemoryCompact.ts#L459-L503)

**我对这套设计的判断**
- 它的核心不是“单次压缩”，而是“三层上下文预算”：
  - 短期：删中间段、裁大 tool result、snip
  - 中期：自动压缩 / reactive compact / context collapse
  - 长期：Session Memory + `MEMORY.md` + relevant memories
  
# 节省上下文代码分析
**Claude Code 是怎么省上下文的：一份不看代码也能读懂的解析**

很多人把“省上下文”理解成一句话：对话太长了，就把前面的内容删掉一些。Claude Code 的实现远不止如此。它不是单纯丢历史，而是用一套分层策略，把“原始长对话”变成“可继续工作、可恢复、可追踪”的紧凑上下文。

如果只用一句话概括：

**Claude Code 的上下文管理核心，是把信息分成短期、中期、长期三层处理：短期内容尽量裁剪，中期内容压缩成摘要，长期内容外置到记忆系统里，只在需要时选择性注入。**

下面按机制拆开讲。

---

## 一、先说结论：它不是“删上下文”，而是“重组上下文”

Claude Code 省上下文的目标不是让模型“少看点内容”这么简单，而是确保模型看到的内容满足三个条件：

1. 当前任务足够完整，能继续干活
2. 重要历史不会丢失
3. 上下文长度始终控制在模型窗口内

所以它做的不是一种动作，而是多种动作组合：

- 把旧历史总结成更短的摘要
- 把过大的工具输出移到文件里
- 把对任务不再关键的中间片段裁掉
- 把长期知识写进独立记忆文件
- 只把真正相关的记忆和文件注入当前 turn

这意味着，Claude Code 不是“记忆力差”，而是“主动做压缩和归档”。

---

## 二、第一层：短期裁剪，先减掉最占空间的内容

### 1. 超大的工具结果不会原封不动塞进上下文

在实际使用中，最容易膨胀上下文的内容往往不是用户输入，而是工具输出，比如：

- `read` 出来的大文件
- `grep`、`find`、`bash` 的大量输出
- web 抓取结果
- 日志、测试输出、构建输出

Claude Code 对这些结果不是简单截断，而是先判断是否超过预算。超过后，它会把完整内容保存到磁盘，只把一段简短预览和文件位置放进上下文。

这样做有两个好处：

- 模型还能知道“结果是什么”
- 真要看完整内容时，可以去文件里读，不需要把全部文本一直压在对话里

这属于非常实用的上下文节省手段，因为很多会话真正的“内存杀手”就是工具输出。

### 2. 空结果也会被处理成有意义的短消息

Claude Code 还会处理一种很容易被忽略的情况：某些工具结果本身是空的。

如果一个工具返回空内容，系统不会让模型面对一个“什么都没有”的结果，而是给出一个极短的提示，表示这个工具已经完成但没有输出。这样可以避免模型在转场时误判上下文断裂，也减少无效重试。

这类细节看起来小，但对长会话很重要，因为它减少了很多“无意义的大段空白解释”。

---

## 三、第二层：中期压缩，把历史变成摘要

### 1. 自动压缩是主力

Claude Code 最核心的省上下文机制，就是自动压缩。它会在上下文接近阈值时，把较早的消息压缩成摘要，只保留最近一段仍然重要的尾部内容。

这里不是简单“删前面”，而是：

- 提炼前面的任务目标
- 记录关键技术点
- 保留重要文件、错误和修复
- 尽量保留任务继续所需的信息结构

压缩后，模型看到的不再是完整原始历史，而是一个“摘要 + 尾部”的组合。这样既省 token，又不至于完全失忆。

### 2. 压缩前会先做预处理

自动压缩并不是裸着跑。进入压缩前，系统还会先做几件事：

- 裁剪过大的工具结果
- 处理一些可重建的信息
- 如果开启了历史裁剪机制，还会先删一部分中间历史
- 再把当前消息流交给压缩逻辑

这说明 Claude Code 的压缩不是唯一动作，而是一个总控点，前面还有一串“减脂”步骤。

### 3. 压缩结果不是单一摘要，而是结构化重建

压缩结果通常不是只有一段摘要文本，而是由几部分组成：

- 边界标记
- 摘要消息
- 保留的消息尾巴
- 必要的附件
- 一些 hook 的结果

这种结构很重要，因为它说明 Claude Code 不是把历史压成一坨文字，而是把上下文拆成“摘要层”和“保留层”。保留层里一般是最近对任务最关键的消息，摘要层则承接被压缩掉的历史。

---

## 四、第三层：Session Memory，把长期知识从对话里搬出去

### 1. Session Memory 的作用不是“再记一遍”，而是“抽象成笔记”

Claude Code 里有一套独立的 Session Memory 系统。它会定期把当前会话中重要的信息抽取出来，写进一个单独的记忆文件。

这个文件一般会覆盖这些主题：

- 当前在做什么
- 用户的具体需求是什么
- 关键文件和函数有哪些
- 遇到过什么错误，怎么修的
- 哪些经验值得保留
- 当前工作进度和下一步

这和普通摘要有区别。普通摘要是“让模型继续对话”；Session Memory 更像“给未来的自己留笔记”。

### 2. 它不是一直抽，而是有触发条件

Session Memory 不会每一轮都写。它会看几个信号：

- 上下文已经增长到一定程度
- 距离上次抽取已经积累了足够的新内容
- 最近的对话里有足够的工具调用或信息密度
- 当前不是处在不适合抽取的边界时刻

也就是说，它不是“固定间隔机械写笔记”，而是“在合适的时候把有价值的内容归档”。

### 3. 记忆内容也会被预算控制

Session Memory 文件本身也不是无限膨胀的。系统会：

- 检查每个 section 是否过长
- 检查整份记忆是否超预算
- 提示压缩过大的 section
- 在被重新注入上下文时再次做截断

这很关键。因为如果长期记忆本身也无限增长，那就会从“省上下文”变成“另一个上下文炸弹”。Claude Code 明显意识到了这一点，所以对记忆文件同样做了预算管理。

### 4. 记忆不是强塞进上下文，而是按需注入

Claude Code 不是把所有记忆文件都放进 prompt。它会先扫描记忆目录，再挑出最相关的少量文件。这个筛选过程本身还是用模型做的，但输入是精简后的文件名、描述和更新时间，不是全文。

因此，长期记忆的使用方式是：

- 先建索引
- 再选相关
- 最后只注入少数有帮助的内容

这就是“长期知识外置 + 选择性回流”。

---

## 五、第四层：只把真正相关的记忆注入当前任务

### 1. 记忆检索不是全量加载，而是相关性选择

Claude Code 会先扫描记忆文件的标题、描述和元信息，再让模型判断哪些记忆和当前问题最相关。最终只会选出少量文件。

这意味着：

- 即使你有很多记忆文件，也不会全部进上下文
- 当前任务相关的知识优先
- 纯工具说明、API 文档类内容在不需要时会被排除

这是一种非常节省上下文的策略，因为它把“信息检索”前置了。

### 2. 还会避开已经刚刚讲过的内容

如果某些记忆文件已经在前面的 turn 里注入过，系统会尽量避免重复再注入。原因很简单：

- 重复内容浪费 token
- 重复内容容易让模型产生冗余偏移
- 重复内容不提升任务完成度

所以这个系统不只关心“相关不相关”，也关心“是不是已经讲过”。

### 3. 预取是异步的，不挡主流程

Claude Code 还会在用户发言后提前启动记忆检索，但不阻塞主工作流。换句话说：

- 主模型可以先开始干活
- 记忆检索在后台跑
- 如果准备好了，就在合适时机注入
- 如果来不及，也不会卡住整个 turn

这是一种很成熟的工程设计。它保证了“省上下文”不会变成“省上下文但响应巨慢”。

---

## 六、第五层：历史裁剪，必要时直接删中间段

### 1. Snip 不是摘要，而是物理裁剪

Claude Code 还有一种更激进的机制：snip。它会把历史中的某些中间段直接删掉，只留下前后最关键的部分。

这和压缩不同：

- 压缩是把旧内容变短
- snip 是把旧内容直接移出当前上下文

所以 snip 更像“手术切除”，而不是“压缩整理”。

### 2. 为什么能删中间段？

因为很多对话中间内容虽然存在，但并不影响后续决策。比如：

- 长时间的调试中间态
- 多轮重复搜索输出
- 已经被后续结论覆盖的尝试
- 大段不再相关的日志

这些内容如果继续留在上下文里，既浪费 token，又会干扰模型判断。snip 就是把这些“对当前任务价值低”的中段拿掉。

### 3. 删除后还要修复链路

snip 不只是删消息那么简单。Claude Code 还会在 transcript 层面重连消息链，否则恢复会出现“又把删掉的东西拼回来了”的问题。

这说明它对历史管理不是停留在表层 UI，而是深入到会话持久化结构。

---

## 七、第六层：不同恢复机制按优先级接管

Claude Code 在上下文快撑爆时，不是所有机制一锅上，而是有优先级和兜底顺序。

大体上是：

1. 先尝试短期裁剪和预处理
2. 再尝试 microcompact / context collapse / 自动压缩
3. 如果真的超了，再尝试恢复性压缩
4. 如果仍然失败，再把错误暴露出来

这有个重要意义：**系统尽量避免“直接报错结束”**，而是先尽可能自救。

也就是说，Claude Code 的上下文管理不是一个被动触发的 fail-safe，而是一套主动的 recovery pipeline。

---

## 八、为什么这套设计比“简单摘要”更强

很多产品的上下文管理只有一种：超长了就总结。Claude Code 更强的地方在于它把上下文当成一种可治理的资源，分层处理。

它的优势主要有四个：

### 1. 保留任务连续性
不是所有旧历史都删掉，而是保留最近决策和关键尾部。

### 2. 降低摘要失真
摘要不是一刀切，结构化重建更容易保住任务语义。

### 3. 控制高风险大块内容
工具结果、记忆文件、长日志都被单独治理。

### 4. 可恢复、可追踪
即使压缩或裁剪了，系统仍然能通过 boundary、memory 文件、transcript、记忆索引把很多内容找回来。

---

## 九、把它理解成一个四步流程最容易

如果你想快速记住 Claude Code 的上下文节省逻辑，可以把它想成这四步：

### 第一步：先减肥
把超大工具结果、无用中间块、重复内容先缩掉。

### 第二步：再摘要
把旧对话压成结构化摘要，保留关键尾巴。

### 第三步：长期归档
把持续重要的信息写进 Session Memory 和相关记忆文件。

### 第四步：按需恢复
在当前任务需要时，只注入少量真正相关的记忆和文件。

---

## 十、最后的结论

Claude Code 节省上下文的方法，本质上不是“少记”，而是“分层记、分层压、分层取回”。

它做对了几件事：

- 把高噪声内容外置
- 把长历史摘要化
- 把长期知识文件化
- 把相关内容选择性注入
- 把失败恢复做成自动化流程