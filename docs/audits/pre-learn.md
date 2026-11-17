⏺ 结论：看你替换的是哪一层。

  - 用 langcli 替换沙箱外的 DSH Brain：是，工作量巨大，属于重构级，不是“换个依赖”级别。
  - 用 langcli 替换沙箱内的 Claude Code（OMA 的 sandbox 模式）：工作量中等，可行得多。
  - 把 langcli 的 QueryEngine 抽成库嵌入 OMA：也不小，而且 langcli 不少 server/runner 入口目前还是 stub。

  下面分开说。

  1. 为什么“替换 DSH Brain”工作量大

  DSH 不是普通 agent CLI，它是 OMA 专门适配出来的 Brain 服务。oma-brain-dsh/src 约有 99 个文件、9.4k 行 TypeScript，覆盖：

  - /v1/code/brain/claims 领取、lease、epoch fence、release
  - worker/register、heartbeat、SSE 输入流、received/processing/processed ACK
  - DSH 原生 JSONL 持久化 + CCRv2 私有事件复制/恢复
  - 模型代理（/v1/messages / /v1/chat/completions）、usage、span、图片 offload
  - 沙箱 MCP 工具、工具审批、evaluated_permission
  - subagent、thread、skill、memory catalog、time context
  - shutdown drain、崩溃恢复、工具结果未知态处理

  langcli 是一个完整的 Claude Code 系 CLI，不是为 OMA 的 Brain 协议设计的。它的核心循环和工具系统默认在本地文件系统与本地进程上执行，而 OMA brain 模式要求工具全部走远端沙箱 MCP。

  2. 具体不匹配点

  ┌────────────────────────────┬──────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────┬──────────┐
  │           子系统           │                             DSH 已有                             │                                     langcli 现状                                     │ 适配难度 │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ Brain 领取 / lease / epoch │ /v1/code/brain/claims、release、30s lease、fence                 │ 有 /worker/register、/bridge、worker_epoch，但没有 OMA 的 claim 层                   │ 中       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ 模型代理                   │ OMA 模型 token、session_context.models、Anthropic/OpenAI adapter │ 有自己的 Anthropic/OpenAI/LangRouter client，可指向 OMA 代理                         │ 中       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ 工具执行边界               │ 只连 OMA 沙箱 MCP                                                │ BashTool、FileReadTool、FileWriteTool、FileEditTool、GlobTool、GrepTool 全部本地执行 │ 大       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ 会话持久化 / 恢复          │ DSH 原生 JSONL + CCRv2 复制/恢复                                 │ 自己的 transcript/session/history 体系                                               │ 大       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ 事件映射 / 流式            │ DSH 事件 → OMA 公开事件、preview、span                           │ stream-json / ACP 事件模型                                                           │ 大       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ 权限 / 审批                │ tool_confirmation、evaluated_permission、auto/always_ask         │ Claude Code 的 permission modes / hooks                                              │ 中       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ 子 Agent / thread          │ DSH 原生 subagent、child thread、委派权限                        │ coordinator/workerAgent 目录存在，但部分是 stub                                      │ 大       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ Skill / Memory / 图片      │ OMA skill provider、memory catalog、图片物化/offload             │ 有 skills/MCP，但 OMA 语义需要重接                                                   │ 中       │
  ├────────────────────────────┼──────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────┼──────────┤
  │ 关闭 / 恢复 / 安全         │ drain、release、fence、租户隔离                                  │ 完整 CLI 的 telemetry、LangRouter、自动更新、OAuth 需要剥离                          │ 中—大    │
  └────────────────────────────┴──────────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────┴──────────┘

  最核心的冲突是第一条工具边界：

  ▎ langcli 的 Agent 认为自己在操作本地文件系统和本地 shell；OMA brain 模式要求它认为自己在操作远端沙箱。
  ▎ 这不是改一个 baseURL 能解决的，而是要替换整个工具执行层。

  3. langcli 有哪些可复用资产

  langcli 也不是从零开始，它有不少可复用部分：

  - 已经是 Claude Code 系架构，有 QueryEngine.ts、query.ts、tools/ 等完整 agent loop。
  - 有 headless --print + --input-format stream-json + --output-format stream-json。
  - 有 CCRv2 bridge：src/bridge/remoteBridgeCore.ts、replBridge.ts、workSecret.ts，已经处理 /v1/code/sessions、/bridge、worker/register、worker_epoch、SSE。
  - 有 MCP、skills、hooks、subagents、ACP 等能力。
  - 有多模型 Provider 适配（Anthropic、OpenAI-compatible、LangRouter）。

  但注意：

  - src/server/server.ts、src/environment-runner/main.ts、src/self-hosted-runner/main.ts 目前是 3 行 stub，不能直接当服务入口用。
  - QueryEngine / query 与 CLI 状态、权限、analytics、telemetry、session 文件耦合较深，抽成“纯 Brain 库”本身是一次重构。
  - langcli 默认带 LangRouter、自动更新、Anthropic telemetry、OAuth 等，接入 OMA 前必须全部禁用/替换，否则会有凭据和租户隔离风险。

  4. 两条路线的对比

  路线 A：langcli 作为沙箱内 harness（替换 Claude Code）

  对应 OMA 的 environment_runner.agent_mode: sandbox：

  - OMA 已经不要求 Brain 在沙箱外，sandbox 模式仍然会启动 /opt/claude-code/bin/claude。
  - 你只需要把沙箱镜像里的 Claude Code 换成 langcli，并保持相同的 CLI/环境变量/CCRv2 行为。
  - langcli 自称 100% Claude Code 兼容，且已有 CCRv2 bridge，这条路线的契合度明显更高。
  - 主要工作：启动参数、模型代理地址、session token、MCP 配置、skills/memory、事件格式、禁用 LangRouter/telemetry/auto-update。
  - 代价：放弃 DSH 带来的“Brain 在沙箱外”的收益（首 token 延迟、沙箱故障隔离、Brain 独立恢复）。

  这条路是“适配级/中等”，不是“重构级”。

  路线 B：langcli 作为沙箱外 Brain（替换 DSH）

  需要把 langcli 的 agent loop 从 CLI 里剥出来，改造成 OMA Brain 服务：

  - 领取代理由 langcli 自建 session 改为 OMA claim。
  - 工具层从本地执行改为 OMA 沙箱 MCP。
  - 会话层从 langcli transcript 改为 OMA/CCRv2 持久化。
  - 事件层映射到 OMA 公开事件。
  - 权限、子 Agent、skill、memory、图片、shutdown、恢复全部重接。
  - 还要保证不读取宿主机文件、不泄露本机凭据、不把 LangRouter/telemetry 带进多租户环境。

  这条路是“重构级/大”，需要把 langcli 的 agent loop 与执行环境解耦，接近重写一个 Brain adapter。

  5. 建议

  如果目标是“利用 langcli 的多模型能力和 Claude Code 兼容性”，优先级建议是：

  1. 优先考虑路线 A：把 langcli 作为沙箱内 harness，替换 Claude Code。这是最贴合 langcli 现有设计的用法。
  2. 如果必须保留 DSH Brain 架构：不要把 langcli 直接当 Brain；可以先做一个协议 adapter 的 spike：
    - 接入 OMA claim + worker/register
    - 指向 OMA 模型代理
    - 只接一个远端沙箱工具（例如 bash）
    - 跑通一次事件持久化和恢复
  如果这一步就很别扭，说明后续全量替换的成本会非常高。
  3. 如果只是想给 DSH 增加“coding CLI 能力”：更合理的是把 langcli 当作一个 MCP/custom tool 或外部 worker，而不是替换整个 Brain。

  最后，除了技术工作量，langcli 基于泄露代码的授权与合规风险需要单独评估；这会影响它能否进入生产链路，和工程量是两件独立的事。


----
❯ "用 langcli 替换沙箱内的 Claude Code（OMA 的 sandbox 模式）：工作量中等，可行得多。" OMA的sandbox中运行的agent loop用的也是claude code ?                                                                

⏺ 是的，OMA 的 sandbox 模式默认就是让 Claude Code 在沙箱内跑 agent loop。更准确地说：

  ▎ sandbox 模式会启动 environment_runner.claude_path 指向的 Claude Code 兼容 harness，默认路径是 /opt/claude-code/bin/claude。

  代码依据

  - 默认路径：
    - internal/config/defaults.go:68：ClaudePath: "/opt/claude-code/bin/claude"
  - sandbox 模式启动 environment-manager 时要求该二进制存在，并传入 --claude-path：
    - internal/environments/environment_manager.go:238 解析 claude_path
    - :259 只有 agent_mode == "brain" 才设置 ENVRUNNER_SKIP_CLAUDE_CODE=true
    - :277-279 sandbox 模式执行：
    environment-manager task-run \
    --session <code-session-id> \
    --session-mode resume-cached \
    --claude-agent-version current \
    --claude-path /opt/claude-code/bin/claude
  - sandbox 模式发布运行时标识为 claude_code_local：
    - internal/environments/runner.go:887："runtime": "claude_code_local"
  - 沙箱运行时契约也写明：
    - docs/en/sandbox-runtime-reference.mdx:9：Agent runtime = /opt/claude-code/bin/claude，可配置。

  两种模式的区别

  ┌────────────────────────────┬──────────────────────────────────────────────┬─────────────────────────────────────────────────────────────┐
  │                            │             agent_mode: sandbox              │                      agent_mode: brain                      │
  ├────────────────────────────┼──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Agent loop / harness       │ Claude Code，在沙箱内                        │ DSH，在沙箱外                                               │
  ├────────────────────────────┼──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ 沙箱里有什么               │ environment-manager + Claude Code + MCP/工具 │ environment-manager + 沙箱 MCP/工具执行器，跳过 Claude Code │
  ├────────────────────────────┼──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ ENVRUNNER_SKIP_CLAUDE_CODE │ 不设置                                       │ 设置为 true                                                 │
  ├────────────────────────────┼──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ 运行时标识                 │ claude_code_local                            │ Brain/DSH                                                   │
  ├────────────────────────────┼──────────────────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ 本地开发默认               │ Docker Compose 快速启动走这条                │ just dev-init 设置 agent_mode: brain                        │
  └────────────────────────────┴──────────────────────────────────────────────┴─────────────────────────────────────────────────────────────┘

  所以：

  - sandbox 模式：Claude Code 自己就是 agent loop，运行在沙箱里，通过 CCRv2 和 OMA 后端通信。OMA 后端负责会话、事件、权限、模型代理和沙箱编排，不是 agent loop。
  - brain 模式：DSH 在沙箱外做 agent loop；沙箱里不再需要 Claude Code，environment-manager 只负责工具执行环境。

  这也解释了为什么“用 langcli 替换 sandbox 里的 Claude Code”相对可行：你替换的是沙箱内那个 harness 二进制；而“替换 DSH”替换的是沙箱外整个 Brain 协议与持久化体系，完全是两个量级的工作。




