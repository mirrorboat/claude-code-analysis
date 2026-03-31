# Swarm 是怎么实现的

Claude Code 里的 swarm 不是一个单独的“超级模型”，也不是简单地开几个子进程。它本质上是一套**多会话协作系统**：主会话负责决策、调度和管理，swarm 成员负责并行执行任务，二者通过文件化消息总线、团队状态文件和权限同步机制协同工作。

如果只用一句话概括：

**主模型发起 swarm，运行时的成员是独立的 Claude 会话，但它们共享一套团队协议。**

下面按“从启动到运行再到收尾”的顺序，把这套机制讲清楚。

---

## 1. 先区分两个概念：subagent 和 swarm teammate

理解 swarm 的第一步，是先不要把它和普通 subagent 混为一谈。

### 普通 subagent
普通 subagent 更像“在同一个任务链里派出去的一个局部执行者”。它通常是主会话在一次工具调用中临时启动的，执行完就返回结果。它的生命周期、上下文和主会话更紧密。

### swarm teammate
swarm teammate 则不同。它是一个**真正的团队成员**：

- 有自己的身份
- 有自己的名字和团队名
- 有自己的任务状态
- 有自己的 mailbox
- 可以被单独发消息
- 可以独立申请权限
- 可以被单独终止或恢复

也就是说，swarm teammate 更接近“协作中的另一台 Claude”，而不是“主模型的一次内部子调用”。

---

## 2. 谁决定启动 swarm？

答案是：**主会话里的主模型决定，但真正执行的是代码层的调度器。**

当主模型在处理用户请求时，如果它选择了“把任务交给一个团队成员”，系统会进入 teammate spawn 路径。这个决策不是模型自己神奇地完成的，而是通过 `Agent` 工具的输入结构来表达的：

- 如果输入里有 `team_name`
- 并且同时指定了 `name`
- 就会被识别为“要启动 swarm 成员”

这时，不再走普通 subagent 路径，而是走 teammate 的 spawn 流程。

换句话说：

- **主模型负责发起**
- **系统负责落地**
- **swarm 成员负责执行**

---

## 3. 启动一个 swarm 成员时，系统会做什么？

启动过程分成几步。

### 第一步：整理成员身份
系统会为新成员生成稳定身份信息，主要包括：

- `agent_id`
- `agent_name`
- `team_name`
- `agent_color`
- `parent_session_id`

其中 `agent_id` 不是随机的，而是由名字和团队组合成一个稳定标识。这样做的好处是：

- 便于跨进程识别成员
- 便于消息路由
- 便于恢复和重连
- 便于 UI 和团队文件关联

### 第二步：选择运行后端
swarm 成员并不总是以同一种方式运行。Claude Code 支持三类后端：

- `tmux`
- `iTerm2`
- `in-process`

系统会根据当前环境自动检测并选择最佳后端：

- 如果你在 tmux 里，优先用 tmux
- 如果你在 iTerm2 且可用 it2 CLI，就用 iTerm2
- 如果外部环境没有 pane backend，但允许 in-process，就退回到 in-process
- 如果没有可用后端，就直接报错或提示安装/配置

这个设计很关键，因为它说明 swarm 的“协作层”与“执行层”是分离的。  
**swarm 不是某一个终端工具，它只是借助终端工具承载协作。**

---

## 4. 三种后端分别怎么工作？

### 4.1 tmux 后端
tmux 是最典型的 swarm 载体。

它有两种模式：

#### 在 tmux 里面运行
如果主会话本身就在 tmux 中，swarm 成员会被切分到当前窗口里：

- 主会话通常保留在左侧
- 新成员出现在右侧
- 多个成员继续分裂排列

这种模式更像“共享一个大终端工作区”。

#### 在 tmux 外运行
如果主会话不在 tmux 中，系统会创建一个独立的 `claude-swarm` session 和专用窗口。  
这相当于自动搭建一个 swarm 工作台。

### 4.2 iTerm2 后端
iTerm2 后端利用 it2 CLI 做原生分屏。

它会：

- 首先从领导会话分出第一个成员
- 后续成员再从已有成员会话继续分裂
- 维护一个 session ID 列表，避免对已经关闭的 pane 继续分裂

这套逻辑的核心是：  
**成员的视觉布局和执行逻辑绑定，但不强绑定于主进程。**

### 4.3 in-process 后端
这是最特殊的一种。

所谓 in-process，不是开新的终端窗口，而是：

- 仍然在同一个 Node.js 进程里
- 通过 AsyncLocalStorage 做上下文隔离
- 每个成员有自己的 TeammateContext
- 每个成员有自己独立的 AbortController
- AppState 里也会注册成独立任务

这意味着它在“物理上”共享进程，但在“逻辑上”仍然是独立 teammate。

---

## 5. swarm 成员启动后会收到什么输入？

这部分是 swarm 实现里最容易误解的地方。

很多人会以为新成员启动后，主会话会把完整上下文直接塞给它。实际上不是。

swarm 成员启动时，拿到的是三类输入：

### 5.1 身份参数
包括：

- agent id
- agent name
- team name
- color
- parent session id
- plan mode 标志
- agent type

这让新成员知道自己是谁、属于哪个团队、和谁有关联。

### 5.2 环境和权限继承
系统会继承一部分主会话的运行环境，例如：

- permission mode
- model 选择
- settings
- plugin 目录
- 某些浏览器或执行相关开关

但这不是无脑复制。  
如果成员要求 plan mode，或者定义里有特殊权限策略，系统会按规则覆盖或限制继承结果。

### 5.3 初始任务 prompt
真正的任务内容并不是直接通过一个“长参数”硬塞进去的。  
对 tmux / iTerm2 这种外部后端，系统会：

1. 先把 Claude 进程启动起来
2. 再把初始 prompt 写入成员的 mailbox
3. 新成员启动后自己读取 mailbox，开始第一轮对话

这点非常重要，因为它意味着 swarm 的输入不是“进程启动参数 + 一次性文本”，而是一个**消息驱动的启动协议**。

---

## 6. swarm 的通信方式是什么？

swarm 的核心不是共享内存，而是**文件化消息系统**。

### 6.1 每个成员都有 inbox
每个 teammate 都有自己的 inbox 文件，位于团队目录下。  
消息内容包括：

- from
- text
- timestamp
- read 标记
- color
- summary

### 6.2 消息写入带锁
为了避免并发写冲突，写 mailbox 时会使用锁文件和重试机制。  
这保证了多个成员同时写消息时，文件不会损坏。

### 6.3 读消息是轮询式的
成员不是靠事件推送收消息，而是定期轮询 mailbox：

- 先读所有消息
- 扫描未读消息
- 优先处理 shutdown request
- 再优先处理来自 team lead 的消息
- 最后才处理普通 peer 消息

这个优先级设计很关键，避免团队协作中“闲聊消息淹没领导指令”。

---

## 7. 主会话如何干预 swarm 运行？

swarm 运行过程中，主会话不是旁观者，而是持续管理者。

### 7.1 可以发送普通消息
主会话可以往成员 inbox 写消息，这些消息会作为对话输入被成员读取。

### 7.2 可以发送权限决策
当成员在工具调用时遇到权限判断，可能会向主会话请求审批。  
主会话会接收到这个请求，然后给出：

- allow
- reject
- 修改后的输入
- 权限更新

### 7.3 可以终止成员
终止方式因后端而异：

- tmux / iTerm2：杀掉对应 pane 或 session
- in-process：通过 AbortController 中断，同时走 shutdown 协议

### 7.4 可以看到成员状态
成员会被注册到 AppState 和 team file 中，因此 UI 可以显示：

- 谁在运行
- 谁已空闲
- 谁已完成
- 谁被阻塞
- 谁在等待权限

所以 swarm 不是“黑盒并发”，而是一个**有状态、可视化、可控**的协作系统。

---

## 8. swarm 成员有哪些权限？

swarm 成员的权限不是无限制复制的，而是按规则构造出来的。

### 8.1 继承主会话的基础权限
成员会继承一部分主会话的运行环境和权限偏好，例如：

- 是否 bypass permissions
- 是否 accept edits
- 是否 auto mode
- 使用哪个 model
- 使用哪些 settings / plugins

### 8.2 可以被 plan mode 约束
如果团队或成员被标记为需要 plan mode，那么成员必须先经过计划审批，再进入实际实现。

这是一种安全边界：  
**成员可以并行，但不能无约束地写。**

### 8.3 可以被工具白名单约束
成员的工具权限可以通过允许列表进行收敛。  
如果某些工具不在许可范围内，它们就无法直接使用，或者必须走审批流程。

### 8.4 还有前端和策略层的限制
即使工具可用，frontmatter、plugin policy、hooks policy、MCP 连接状态等，也可能进一步影响成员能否使用某类能力。

换句话说，swarm 的权限是一个多层叠加结果：

- 运行时模式
- 成员定义
- 团队策略
- 工具权限
- MCP 可用性
- 计划模式

---

## 9. swarm 如何和主模型沟通？

这是 swarm 最核心的协作机制。

### 9.1 通过 mailbox
这是最普遍的方式：

- 主会话给成员发任务
- 成员给主会话发状态更新
- 成员之间也可以互发消息

消息是普通文本，但会被系统包装成结构化消息处理。

### 9.2 通过权限桥
对于 in-process teammate，权限请求可以直接桥接到主 UI。  
也就是说，主会话可以用统一的权限弹窗处理 teammate 的工具请求，而不是让它自己卡在后台等待。

### 9.3 通过 task state
成员会在 AppState 里注册为任务，因此主 UI 能实时看到它的：

- 运行状态
- 进度
- 产出
- 空闲状态
- 终止状态

### 9.4 通过团队文件
team file 是团队持久化的结构化真相源。  
成员加入、退出、颜色、角色、模式等，都会更新到团队文件中，便于恢复和再连接。

---

## 10. swarm 和普通 subagent 的区别，为什么重要？

这个问题很重要，因为很多实现细节都来自这一区分。

### 普通 subagent
- 生命周期短
- 更像一次函数调用
- 通常跟主会话处于同一个执行链

### swarm teammate
- 生命周期长
- 是可通信、可管理、可恢复的独立成员
- 有自己的身份、状态和消息机制
- 可以跨终端后端运行

所以，swarm 不是“多次调用 Agent 工具”，而是**把 Agent 工具升级成一个团队协作基础设施**。

---

## 11. 主模型什么时候会启动 swarm 成员？

总结成最实用的一句：

**当主模型判断某个任务适合并行协作，并且调用 `Agent` 工具时，如果输入包含 team 语义，系统就会启动 swarm 成员。**

常见触发场景包括：

- 需要并行探索多个方向
- 需要一个成员专门调查、另一个成员专门实现
- 需要在一个团队里进行分工协作
- 需要长期存在、可以通信的执行者

也就是说，swarm 的启动是**模型驱动的策略决策 + 系统驱动的工程落地**。

---

## 12. 你可以把整个 swarm 生命周期理解成这样

### 启动
主会话决定分配任务 -> 生成 teammate 身份 -> 选择 backend -> 创建 pane/session/进程

### 初始化
写入初始 prompt 到 mailbox -> teammate 读取 prompt -> 进入自己的对话循环

### 运行
teammate 执行工具 -> 遇到权限时请求审批 -> 通过 mailbox 或权限桥和主会话交互

### 协作
成员之间、成员与主会话之间持续交换消息和状态

### 收尾
任务完成或被终止 -> 更新 task state、team file、日志和通知 -> 必要时回收 pane/进程

---

## 13. 最后的结论

Claude Code 的 swarm 不是“一个模型变成多个模型”，而是：

- **主模型负责决策**
- **系统负责创建独立 teammate**
- **teammate 通过 mailbox 和权限桥与主会话协作**
- **运行状态由 team file 和 AppState 统一管理**
- **后端可以是 tmux、iTerm2 或 in-process**
- **成员可以被管理、终止、恢复、通知、授权**

如果从工程视角看，swarm 的本质是：

**一个以文件消息为通信底座、以终端 pane/进程为执行载体、以团队文件为状态源的多代理协作框架。**

# Claude Code 里的 Subagent 是怎么实现的

Claude Code 里的 subagent 不是“主模型自己随便想一想的中间步骤”，而是一套完整的**独立 agent 执行机制**。它的目标是把复杂任务拆开，让主会话能把局部工作交给专门化的执行单元去做，再把结果收回来继续推进主任务。

如果把 swarm teammate 理解成“团队成员”，那么 subagent 更像是“主会话内部派出的专业执行者”。

这篇文章只讲 subagent 的实现，不讲 swarm teammate。

---

## 1. Subagent 不是普通函数调用

Claude Code 里的 subagent 不是一次简单的工具函数执行，而是一个完整的 agent 生命周期：

- 有自己的系统提示
- 有自己的工具集
- 有自己的消息上下文
- 有自己的权限策略
- 会记录自己的 transcript
- 会触发自己的 hook
- 会单独产出结果，再回流给主会话

所以，subagent 更像是“一个被主会话临时派生出来的 Claude 任务”，而不是内部函数。

---

## 2. subagent 的入口：Agent 工具

Claude Code 把 subagent 的启动入口统一放在 `Agent` 工具里。

主模型在处理用户请求时，如果决定把某个局部任务交给一个子执行者，就会通过 `Agent` 工具发起这个任务。这里最关键的参数是：

- `prompt`：subagent 要做什么
- `subagent_type`：要用哪种专门化 agent
- `model`：可选的模型覆盖
- `run_in_background`：是否后台运行
- `description`：任务摘要

也就是说，subagent 的启动是显式的、结构化的，不是靠 prompt 里偷偷暗示出来的。

---

## 3. subagent 的类型系统

Claude Code 并不是只有一种 subagent。它把 subagent 做成了“类型化 agent”。

这意味着同一个 `Agent` 工具，可能启动不同职责的执行者，比如：

- 通用执行型
- 探索型
- 验证型
- 计划型
- 代码引导型
- 状态栏设置型

这种设计的本质是：  
**主会话不只是在“调用一个子 agent”，而是在“挑选一种子 agent 角色”。**

这让 subagent 的行为更稳定，也更容易被系统优化。

---

## 4. subagent 的核心实现：runAgent

subagent 真正的执行核心是 `runAgent()`。

它做的不是“跑一个 prompt”，而是完整地搭建一个子 agent 的运行环境。这个环境包括：

### 4.1 独立的上下文
subagent 会构造自己的 `ToolUseContext`，不直接沿用主会话的所有状态。  
这样做的目的是避免：

- 主会话和子会话互相污染
- 运行中状态被覆盖
- 工具使用结果混在一起
- 自动压缩或重写消息导致上下文错乱

### 4.2 独立的工具集
subagent 会基于自己的角色、权限和当前可用工具，生成一套专属工具池。  
不是主会话能用什么，它就一定能用什么。

### 4.3 独立的系统提示
subagent 会拿到自己的系统提示。  
这个系统提示通常会包含：

- 角色定义
- 任务指令
- 运行环境信息
- 工具可用性信息
- 某些额外约束

这意味着 subagent 是“按角色构造”的，而不是“复制主模型”。

### 4.4 独立的消息历史
subagent 不只是接收一条 prompt，而是会构造自己的初始消息序列，然后进入自己的 `query()` 循环。

它的消息来源可能包括：

- 主会话传入的任务 prompt
- 上下文 fork 出来的消息
- 之前的 tool use / tool result
- skills 预加载内容
- hooks 插入的额外上下文

---

## 5. subagent 为什么需要自己的上下文？

这是 subagent 设计的关键。

如果没有独立上下文，subagent 就只是“主会话里的一次工具调用”。  
一旦有了独立上下文，subagent 就可以具备这些能力：

- 独立调用工具
- 独立申请权限
- 独立记录结果
- 独立产出中间消息
- 独立做长期任务
- 独立触发 hook

这也是 Claude Code 能把“探索、验证、实现、整理”拆成多个专门 agent 的基础。

---

## 6. subagent 的权限模型

subagent 的权限不是随便继承的，而是经过重新构造的。

系统会基于主会话当前的权限状态，结合 subagent 自身的角色定义，决定它能做什么。

### 6.1 继承主会话的基础权限
它通常会继承一部分主会话已有的能力，例如：

- 当前 permission mode
- 工具白名单
- 某些自动允许规则
- 当前模型和运行参数

### 6.2 角色定义可进一步收紧或覆盖
如果某个 subagent 定义了自己的 permissionMode、effort、可用工具范围，那么系统会据此改写它的上下文。

### 6.3 async subagent 和 sync subagent 不一样
如果 subagent 是异步运行的，系统会更倾向于：

- 避免它阻塞主 UI
- 避免它弹出不合适的交互式 prompt
- 必要时把权限请求回流给主会话

这说明 subagent 的权限不是“给了就完事”，而是和它的运行方式绑定的。

---

## 7. subagent 的消息生命周期

subagent 的消息生命周期分三段：

### 7.1 启动前
主会话决定要启动 subagent，并把任务信息、角色类型、模型等参数组织好。

### 7.2 运行中
subagent 进入自己的 query loop，按轮次处理：

- 用户消息
- 工具调用
- 工具结果
- 进度消息
- hook 插入消息

同时，它会把自己的状态、tool use 结果、转录内容记录下来。

### 7.3 结束后
subagent 完成任务后，会把结果回流给主会话，主会话继续执行自己的推理链。

这和“直接在主模型内部继续想”很不一样。  
这里的结果是由一个独立 agent 完成后回传的。

---

## 8. transcript 和 sidechain：为什么 subagent 要单独记日志？

Claude Code 会给 subagent 单独记录 transcript，通常称为 sidechain transcript。

这有几个目的：

- 方便恢复
- 方便审计
- 方便调试
- 方便展示子 agent 的过程
- 方便后续 resume 或继续接着跑

这也是 subagent 和普通内部函数调用最大的差别之一。  
**它的执行过程是可追踪的。**

---

## 9. hooks：subagent 生命周期的可插拔扩展点

Claude Code 对 subagent 有专门的生命周期 hook。

常见的有：

- SubagentStart
- SubagentStop

这些 hook 允许系统在 subagent 启动前后执行额外逻辑，比如：

- 注入上下文
- 记录审计信息
- 做权限检查
- 做自定义通知
- 做文件或状态同步

这使 subagent 的生命周期不仅“可运行”，而且“可编排”。

---

## 10. skills：subagent 的行为增强层

subagent 还可以预加载 skills。

skills 的作用不是简单增加提示词，而是给 subagent 附加“专门能力包”：

- 预定义工作流程
- 任务提示
- 专门的操作指南
- 特定领域的行为约束

系统会把这些 skills 的内容加载进 subagent 的初始消息里，让它从一开始就带着特定任务知识运行。

这意味着 subagent 不只是“一个模型”，而是“一个角色 + 一组能力 + 一套上下文”的组合体。

---

## 11. fork subagent：一种更特殊的子代理形态

Claude Code 里还有一种特殊的 subagent 路径，可以理解为“fork subagent”。

它的目标不是简单地做一个新任务，而是尽可能继承主会话的上下文结构，做到：

- prompt 前缀尽量一致
- 工具集尽量一致
- 消息结构尽量一致
- 以便获得更好的 cache 命中和行为一致性

这种 fork 模式和普通 subagent 的区别在于：

- 普通 subagent：更像一个角色化任务执行者
- fork subagent：更像对当前对话状态的受控分叉

这说明 Claude Code 对 subagent 的设计已经不只是“派工”，而是在考虑**上下文可复用性**和**推理成本优化**。

---

## 12. subagent 会被主会话干预吗？

会，但方式和 swarm 不一样。

subagent 的干预更多发生在“编排层”，而不是“协作层”。

### 12.1 主会话可以决定它是否后台运行
subagent 可以同步运行，也可以异步后台运行。  
如果是后台运行，它会独立推进，但主会话仍然可以收到它的结果。

### 12.2 主会话可以控制它的工具范围
subagent 的工具并不是完全自由的。  
主会话和系统策略会共同决定它能用哪些工具。

### 12.3 主会话可以通过 context / hook / system prompt 影响它
subagent 的行为会受到：

- 角色定义
- 系统提示
- hook 注入上下文
- skills 内容
- permission mode

这些都属于“编排式干预”，不是运行时强行插手。

---

## 13. subagent 和主模型的关系

可以把它理解成下面这个关系：

- **主模型**：负责整体目标、分解任务、决定是否调用 subagent
- **subagent**：负责局部目标、独立执行、返回结果
- **系统**：负责把两者连接起来，并保证上下文、权限和日志不乱

所以 subagent 不是“主模型的附属思考”，而是一个**被主模型调度的独立执行体**。

---

## 14. 为什么 Claude Code 要这么设计 subagent？

主要有四个原因：

### 14.1 分工更清晰
不同 subagent 可以专门做不同类型的工作，比如搜索、验证、实现、总结。

### 14.2 上下文更干净
每个 subagent 拿到的是经过筛选的上下文，不会把主会话的信息全部塞进去。

### 14.3 更容易并行
多个 subagent 可以并行跑，主会话把它们的结果再整合起来。

### 14.4 更容易控制成本
通过角色化、工具裁剪、thinking 关闭、skills 预加载等机制，可以控制 token 和执行开销。

---

## 15. 一句话总结 subagent 的实现

Claude Code 里的 subagent，本质上是一个**从主会话派生出来的、具有独立上下文和权限边界的 agent 执行单元**。  
它通过 `Agent` 工具被触发，通过 `runAgent()` 被执行，通过 transcript、hooks、skills 和权限系统被管理，最终把结果回流给主会话继续推进任务。

# subagent 和 swarm teammate 的对比

- `subagent` 是“主会话内部派生出来的独立 agent 执行单元”。
- `swarm teammate` 是“带团队身份、可通信、可持久管理的独立协作者”。
- 两者都不是普通函数调用，但它们处在不同系统层：`subagent` 偏“对话/推理编排”，`swarm teammate` 偏“多会话协作基础设施”。

下面按系统维度对比。

| 维度 | subagent | swarm teammate |
|---|---|---|
| 触发方式 | 由 `Agent` 工具的 `subagent_type` / fork 路径触发 | 由 `team_name + name` 进入 teammate spawn 路径触发 |
| 入口 | `AgentTool.call()` 里选择具体 agent 类型 | `spawnTeammate()` 统一分发到 tmux / iTerm2 / in-process |
| 核心执行 | `runAgent()` | 后端创建会话/窗格/进程后，再由 `runAgent()` 或独立 Claude 实例执行 |
| 身份模型 | 主要是 agentId、agentType、querySource、上下文隔离 | agentId、agentName、teamName、color、parentSessionId、team file 成员记录 |
| 通信方式 | 主会话内部消息流、transcript、hooks | 文件 mailbox、team file、permissionSync、task state |
| 运行后端 | 通常在同一执行链中运行，可能是同步或异步 subagent | tmux、iTerm2、in-process 三类后端 |
| 是否有自己的终端存在感 | 一般没有独立 pane/window | 通常有独立 pane/window，in-process 例外 |
| 是否可被团队级管理 | 主要由主会话编排 | 可以被团队文件、UI、mailbox、kill/terminate 持续管理 |
| 权限模型 | 继承主会话并按 subagent 角色重新裁剪 | 继承主会话权限，但还叠加 team 级规则、plan mode、backend 约束 |
| 持久化 | transcript / sidechain transcript | team file + mailbox + task state + transcript |
| 典型用途 | 探索、验证、局部实现、fork 对话 | 并行协作、长期成员、互发消息、团队分工 |

---

## 1. 触发层：它们是怎么被启动的

### Subagent
subagent 是在 `Agent` 工具路径里被触发的。系统先看 `subagent_type`，再决定用哪种专门化 agent；如果开启 fork 模式，还会走 fork 路径。也就是说，它是**主会话内部的 agent 派生**。[AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx#L318)

### Swarm teammate
swarm teammate 则要求显式团队语义：`team_name` 和 `name` 一起出现时，系统认为这是团队成员 spawn，而不是普通 subagent。[AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx#L282)

---

## 2. 执行层：它们跑在哪里

### Subagent
subagent 的执行核心是 `runAgent()`。它会为子 agent 构造独立的 `ToolUseContext`、系统提示、工具池、消息序列，然后进入自己的 `query()` 循环。[runAgent.ts](src/tools/AgentTool/runAgent.ts#L697)

这意味着 subagent 是一种**逻辑隔离**，不一定意味着独立进程或独立终端。

### Swarm teammate
swarm teammate 先通过 `spawnTeammate()` 分发到底层 backend，再由 backend 决定是在 tmux、iTerm2 还是 in-process 环境里运行。[spawnMultiAgent.ts](src/tools/shared/spawnMultiAgent.ts#L1040)

所以它是**执行载体隔离**优先：先有独立成员，再决定把这个成员放在哪个运行环境里。

---

## 3. 身份层：它们的“自我认知”不同

### Subagent
subagent 的身份更偏“任务角色”：

- agentType
- querySource
- agentId
- transcript 路径

它强调的是“我是一个什么角色的子执行者”。

### Swarm teammate
swarm teammate 的身份更偏“团队成员”：

- agentId
- agentName
- teamName
- color
- parentSessionId
- team file 成员记录

它强调的是“我是哪个团队里的哪一个人”。[teamHelpers.ts](src/utils/swarm/teamHelpers.ts#L64)

---

## 4. 上下文层：谁拿到的是“完整对话”，谁拿到的是“裁剪过的任务”

### Subagent
subagent 会拿到经过裁剪的上下文：

- 可能包含 fork 出来的历史消息
- 可能包含技能预加载内容
- 可能包含 hook 注入内容
- 工具集按角色和权限重构

它的目标是“让这个角色在局部任务上表现最好”。

### Swarm teammate
swarm teammate 启动时拿到的是：

- CLI 身份参数
- 继承的权限/模型/设置
- 初始 prompt 通过 mailbox 注入

它不是复制主会话全部上下文，而是通过消息协议拿到任务输入。[spawnMultiAgent.ts](src/tools/shared/spawnMultiAgent.ts#L399) [teammateMailbox.ts](src/utils/teammateMailbox.ts#L134)

---

## 5. 通信层：这是两者最本质的差异

### Subagent
subagent 的沟通主要是“编排内通信”：

- 主会话把任务给它
- 它把结果回传给主会话
- 通过 transcript、hook、tool result 形成可追踪链路

它一般不需要面对“别的 teammate”。

### Swarm teammate
swarm teammate 是“协作网络”：

- 主会话给它发任务
- 它给主会话发状态
- teammate 之间也能互发消息
- 权限请求/响应也走协作通道

核心机制是 file mailbox，不是直接函数调用。[teammateMailbox.ts](src/utils/teammateMailbox.ts#L1) [permissionSync.ts](src/utils/swarm/permissionSync.ts#L1)

---

## 6. 权限层：谁能决定做什么

### Subagent
subagent 的权限是“角色化权限”：

- 继承主会话的基础权限
- 按 agent 定义、async/sync、工具白名单重构
- 主要用于防止子任务越权或弹出不合适的交互

权限处理更多发生在 `runAgent()` 里。[runAgent.ts](src/tools/AgentTool/runAgent.ts#L430)

### Swarm teammate
swarm teammate 的权限是“团队化权限”：

- 继承主会话权限
- 叠加 team 级状态
- 可以进入 plan mode
- 可以通过 leader UI 或 mailbox 申请审批
- 可以通过 permissionSync 持久化审批结果

也就是说，swarm teammate 的权限不是单个 agent 自己的，而是**团队协商出来的**。[leaderPermissionBridge.ts](src/utils/swarm/leaderPermissionBridge.ts#L1) [permissionSync.ts](src/utils/swarm/permissionSync.ts#L12)

---

## 7. 生命周期层：谁更“短”，谁更“长”

### Subagent
subagent 通常更短生命周期：

- 任务明确
- 执行完就结束
- 重点是产出结果并回流

### Swarm teammate
swarm teammate 可以更长生命周期：

- 运行中可以持续收消息
- 可以持续被管理
- 可以被终止、恢复、重连
- 可以在 team file 里保留成员状态

这就是为什么 swarm 需要 team file、mailbox 和 task state 这些持久化结构。

---

## 8. 后端层：它们的物理承载不同

### Subagent
subagent 的重点不是物理承载，而是上下文和 agent 角色。它可以在同一进程内运行，也可以在不同执行模式下运行，但系统设计关注点是“它如何被构造”。

### Swarm teammate
swarm teammate 明确依赖后端抽象：

- tmux
- iTerm2
- in-process

这些后端不仅决定“怎么跑”，还决定“怎么显示、怎么 kill、怎么布局”。[backends/types.ts](src/utils/swarm/backends/types.ts#L35) [registry.ts](src/utils/swarm/backends/registry.ts#L128)

---

## 9. 可视化层：UI 关注点也不同

### Subagent
subagent 的 UI 更像“执行过程展示”：

- progress
- transcript
- hook 事件
- 结果回流

### Swarm teammate
swarm teammate 的 UI 更像“团队监控面板”：

- 谁在运行
- 谁空闲
- 谁发了消息
- 谁需要审批
- 哪个 pane 属于哪个成员
- 团队成员列表和颜色

它是一个更完整的“team dashboard”。

---

## 10. 运行时管理：谁会被干预得更深

### Subagent
subagent 会被主会话从编排层管理：

- 起什么角色
- 用什么工具
- 用什么模型
- 是否后台运行

但它更像“受控任务”。

### Swarm teammate
swarm teammate 则会被更深地管理：

- 可以收到普通消息
- 可以收到 shutdown 请求
- 可以通过 mailbox 被远程协调
- 可以被 team file 更新
- 可以被 pane/backend kill

它更像“被组织起来的协作者”。

---

## 11. 一个容易误解的点：in-process teammate 和 subagent 不是同一回事

虽然 in-process teammate 也用 `AsyncLocalStorage` 做隔离，和 subagent 在“共享主进程”这一点上看起来相似，但它仍然属于 swarm teammate 系统，不是普通 subagent。

区别在于：

- subagent 的目标是“对话内局部执行”
- in-process teammate 的目标是“团队成员式协作，只是执行载体恰好在同一进程”

所以不能因为“都在同一进程”就把它们当成同类。

---

## 12. 最终判断

如果从系统设计上看：

- **subagent 是认知执行单元**
- **swarm teammate 是协作执行单元**

更直白一点：

- subagent 解决“这件事该交给哪个专业角色做”
- swarm teammate 解决“多个 Claude 如何像团队一样长期协作”

---

## 13. 实用对照

如果你在读源码，可以用这个判断标准：

- 看见 `subagent_type`、`runAgent()`、`querySource`、`SubagentStart`，你在看 **subagent 系统**
- 看见 `team_name`、`spawnTeammate()`、mailbox、team file、tmux/iTerm2 backend，你在看 **swarm teammate 系统**

---

如果你要，我下一步可以把这篇内容整理成一张更适合博客发布的“对比总表 + 架构图说明”，或者继续写一篇“Claude Code 里的 agent 总体系图”，把 `main agent / subagent / swarm teammate` 三者放在同一张图里讲清楚。