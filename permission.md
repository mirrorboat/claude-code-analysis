# 权限管理机制

这套机制不是单一“允许/拒绝列表”，而是三层叠加：

1. 先构造一个 `ToolPermissionContext`，把来自 CLI、settings、session、policy 的 `allow / deny / ask` 规则合并进去，并记录当前模式 `default / acceptEdits / plan / auto / bypassPermissions / dontAsk` 等。[Tool.ts](src/Tool.ts#L123), [permissionSetup.ts](src/utils/permissions/permissionSetup.ts#L689)
2. 所有工具调用都先经过统一权限管道：`runToolUse -> resolveHookPermissionDecision -> canUseTool -> hasPermissionsToUseTool`，不是直接执行。[toolExecution.ts](src/services/tools/toolExecution.ts#L916), [useCanUseTool.tsx](src/hooks/useCanUseTool.tsx#L32), [permissions.ts](src/utils/permissions/permissions.ts#L473)
3. Bash 这类“命令型工具”再做命令级别匹配：exact / prefix / wildcard，外加路径安全、sed 安全、模式规则、read-only 规则、sandbox 规则。[bashPermissions.ts](src/tools/BashTool/bashPermissions.ts#L778)

**实际判定顺序，按 Bash 命令来说：**
- 先看 exact `deny / ask / allow`。
- 再看 prefix / wildcard 的 `deny / ask`。
- 再做路径约束检查，防止通过绝对路径、重定向等绕过。
- 再看 exact / prefix 的 `allow`。
- 再做 `sed` 的额外限制。
- 再看 mode 相关规则。
- 最后如果是只读命令，直接允许。
- 其余情况就变成 `ask`，等待用户确认。[bashPermissions.ts](src/tools/BashTool/bashPermissions.ts#L1058), [bashPermissions.ts](src/tools/BashTool/bashPermissions.ts#L1124)

**哪些命令会被判成“不能执行”**
- 明确命中 `deny` 规则的命令，直接拒绝。
- 命中 `ask` 规则但当前不是可自动放行场景的命令，会要求确认。
- 命中路径安全检查的敏感写入，比如 `.git/`、`.claude/`、`shell config` 这类，会走 `safetyCheck`，而且是“绕不过去”的。[permissions.ts](src/utils/permissions/permissions.ts#L1252)
- auto mode 下被 classifier 判定危险的命令，会被拒绝。[permissions.ts](src/utils/permissions/permissions.ts#L688)
- headless / `shouldAvoidPermissionPrompts` 场景下，如果没有 hook 或 classifier 能代替用户决策，默认是拒绝。[permissions.ts](src/utils/permissions/permissions.ts#L929)

**哪些命令会被判成“可以执行”**
- 命中 `allow` 规则。
- `bypassPermissions` 模式下，规则检查通过后直接放行，但仍然保留前置的规则/安全检查。[permissions.ts](src/utils/permissions/permissions.ts#L1262)
- `acceptEdits` 模式下，`mkdir / touch / rm / rmdir / mv / cp / sed` 这类文件系统命令可自动允许。[modeValidation.ts](src/tools/BashTool/modeValidation.ts#L7)
- auto mode 下，classifier 判定安全就允许。
- Bash 在 sandbox 启用且 `autoAllowBashIfSandboxed` 打开时，某些本来会问的命令可以被 sandbox 逻辑自动放行，但前提是它确实会在 sandbox 里跑。[permissions.ts](src/utils/permissions/permissions.ts#L1183), [shouldUseSandbox.ts](src/tools/BashTool/shouldUseSandbox.ts#L18)

**Bash 规则匹配的关键点**
- 规则支持 exact、legacy `:*` prefix、以及 `*` wildcard。[shellRuleMatching.ts](src/utils/permissions/shellRuleMatching.ts#L33)
- deny / ask 规则会更激进地剥离前缀 env 和 wrapper，避免像 `FOO=bar denied_cmd` 这种绕过。
- allow 规则只会剥离“安全”的 env / wrapper，故意保守，避免把危险命令误放行。
- compound command 会逐个子命令检查，不能靠 `cmd1 && denied_cmd` 绕过去。
- wildcard 在 prefix 模式下不会匹配 compound command，避免 `foo *` 误吞 `foo ... && curl evil.com`。[bashPermissions.ts](src/tools/BashTool/bashPermissions.ts#L778)

**sandbox 在这里做什么**
- sandbox 不是“权限判定”本身，而是 OS 层的第二道闸：即使命令被允许了，还是会被 sandbox 的文件系统 / 网络配置限制。
- `convertToSandboxRuntimeConfig()` 会把 settings 和 permission rules 转成 `allowWrite / denyWrite / allowRead / denyRead / allowedDomains / deniedDomains` 等配置。[sandbox-adapter.ts](src/utils/sandbox/sandbox-adapter.ts#L172)
- 它还会强制禁止写 settings、`.claude/skills`、裸 git repo 相关文件等，防止权限逃逸。[sandbox-adapter.ts](src/utils/sandbox/sandbox-adapter.ts#L230)
- `shouldUseSandbox()` 决定 Bash 命令到底要不要进 sandbox；如果用户显式禁用 sandbox 且策略允许，也可能直接不走 sandbox。[shouldUseSandbox.ts](src/tools/BashTool/shouldUseSandbox.ts#L13)

**auto mode 还有一个特别处理**
- 进入 auto mode 时，会把“危险 allow 规则”先剥掉，比如 `Bash(*)`、`Bash(python:*)`、`Agent(*)` 这类会绕过 classifier 的规则，退出 auto mode 再恢复。[permissionSetup.ts](src/utils/permissions/permissionSetup.ts#L510), [permissionSetup.ts](src/utils/permissions/permissionSetup.ts#L597)

**auto mode(yolo)**
- 不再默认弹出人工确认，而是先交给 classifier 判定。
- 进入 auto mode 时，会先把那些“会绕过 classifier 的危险 allow 规则”剥掉，比如 `Bash(*)`、`Bash(python:*)`、`Agent(*)` 这类。[permissionSetup.ts](src/utils/permissions/permissionSetup.ts#L510)
- 命令如果本来就是明确 `deny`，还是会直接拒绝，不会因为 yolo 就放行。[permissions.ts](src/utils/permissions/permissions.ts#L1078)
- 命中 `ask` 的时候，auto mode 不会马上弹 UI，而是先跑 classifier；classifier 认为安全就 `allow`，认为危险就 `deny`。[permissions.ts](src/utils/permissions/permissions.ts#L518)
- 有些场景会直接跳过 classifier 的慢路径，比如 `acceptEdits` 能直接放行的文件系统命令、safe allowlist 工具、以及某些安全检查分支。[permissions.ts](src/utils/permissions/permissions.ts#L593), [permissions.ts](src/utils/permissions/permissions.ts#L658)
- 如果 classifier 不可用，代码会根据门控策略选择 fail closed 或 fallback。[permissions.ts](src/utils/permissions/permissions.ts#L843)
- 在不能弹窗的 headless/subagent 场景里，如果 hook 也没给出决策，最后还是会拒绝，不会默默执行。[permissions.ts](src/utils/permissions/permissions.ts#L929)
