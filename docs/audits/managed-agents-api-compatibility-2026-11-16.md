# OMA 与 Claude Managed Agents API 兼容性调研

研究日期：2026-11-16。范围：Claude Managed Agents 公开端点与官方 SDK 的实际请求约定，对照本仓库 `/v1` 路由、Handler、OpenAPI 与文档。本文是静态代码与公开规范核对，没有启动本地服务，也没有运行 `verify-be` 或官方 SDK E2E。

## 结论摘要

OMA 的 `/v1` 在路径覆盖、资源形状、事件模型和 SSE 流式协议上与 Claude Managed Agents 高度一致，多数官方 SDK 请求可以直接发过来。但存在一个会直接阻断最新 TypeScript/Python SDK 的硬差异，以及若干“形状兼容、语义未实现”的功能缺口。

| 维度 | 结论 |
| --- | --- |
| 端点覆盖 | Claude Managed Agents 端点参考中的公开路径基本全部实现，另有 OMA 扩展端点 |
| 请求/响应形状 | 与官方 SDK 生成类型高度一致；`openapi/oma.en.json` 含 658 个 schema，大量沿用 `BetaManagedAgents*` 命名 |
| 最大硬不兼容 | `environments`、`memory_stores`、`vaults` 要求 `?beta=true`，最新 TypeScript/Python SDK 只发送 `anthropic-beta` 头 |
| 主要语义缺口 | `auto` 权限策略不自动评估；`workflows`/`advisor` 接受但不执行；outcome 不自动评分；Dreams API 缺失 |
| 错误信封 | 基本一致，但部分 409 返回 `conflict_error`，Claude 使用 `invalid_request_error` |
| 分页 | 大部分一致；`GET /v1/sessions` 缺少 `prev_page` 反向分页 |
| 模型参数 | `effort`、`speed`、`inference_geo` 可被接受和存储，但未完整应用到实际模型请求 |

## 调研基准

对照了四类材料：

1. Anthropic 官方 Managed Agents 端点参考：`anthropics/skills` 的 `managed-agents-api-reference.md`。
2. 官方 SDK 实际请求：Go SDK、TypeScript SDK、Python SDK 的 beta 资源实现，重点检查是否发送 `?beta=true`。
3. OMA 公开 OpenAPI：`openapi/oma.en.json`。
4. OMA 实际 Handler 与文档：`internal/api/server.go` 路由注册、各资源 handler、错误映射、分页、Beta 门禁，以及 `docs/en/**` 与 `docs/design/be/**`。

## 端点覆盖

Claude Managed Agents 端点参考中的资源，OMA 均已注册：

| 资源 | Claude 路径 | OMA 状态 |
| --- | --- | --- |
| Agents | `/v1/agents`、`/{id}`、`/{id}/archive`、`/{id}/versions` | 完全实现 |
| Sessions | `/v1/sessions`、`/{id}`、`/{id}/archive` | 完全实现 |
| Events | `/{session_id}/events`、`/{session_id}/events/stream` | 完全实现 |
| Threads | `/{session_id}/threads`、`/{thread_id}`、`/archive`、`/events`、`/stream` | 完全实现 |
| Session Resources | `/{session_id}/resources`、`/{resource_id}` | 完全实现 |
| Environments | `/v1/environments`、`/{id}`、`/archive`、`/work/stats`、`/work/{id}/stop` | 完全实现 |
| Deployments | `/v1/deployments`、`/{id}`、`pause`、`unpause`、`archive`、`run` | 完全实现 |
| Deployment Runs | `/v1/deployment_runs`、`/{id}` | 完全实现 |
| Vaults | `/v1/vaults`、`/{id}`、`/archive` | 完全实现 |
| Credentials | `/{vault_id}/credentials`、`/{credential_id}`、`/archive`、`/mcp_oauth_validate` | 完全实现 |
| Memory Stores | `/v1/memory_stores`、`/{id}`、`/archive` | 完全实现 |
| Memories | `/{store_id}/memories`、`/{memory_id}` | 完全实现 |
| Memory Versions | `/{store_id}/memory_versions`、`/{version_id}`、`/redact` | 完全实现 |
| Files | `/v1/files`、`/{id}`、`/{id}/content` | 完全实现 |
| Skills | `/v1/skills`、`/{id}`、`/{id}/versions`、`/{version}`、`content` | 完全实现 |

对照结果：端点参考中列出的公开 Managed Agents 路径，OMA 没有缺项。

OMA 额外提供、Claude 端点参考未列出的路径包括：

- `POST /v1/agents:search`
- `/v1/channels`、`/v1/tunnels`、`/v1/mcp`
- `/v1/organizations` 管理接口
- `/v1/chat/completions`
- Environment prebuild、work list/poll/ack/heartbeat 等更多 worker 端点
- `/v1/webhooks` 管理接口
- evaluation、analytics、platform console 等 `/api` 接口

这些属于加法扩展，不破坏已有 SDK 调用。

## 最大的硬不兼容：Beta 激活方式

### OMA 的实际门禁

| 资源 | OMA 要求 | 证据 |
| --- | --- | --- |
| Agents | 不要求 beta | `internal/agents/handler.go` 无门禁 |
| Sessions 普通 CRUD | 不要求 beta | `internal/sessions/service.go` |
| Session threads | `anthropic-beta: managed-agents-2026-04-01` | `internal/sessions/thread_beta.go:10` |
| Environments | `?beta=true` | `internal/environments/handler.go:153` |
| Memory Stores | `?beta=true` | `internal/memory/handler.go:168` |
| Vaults | `?beta=true` | `internal/vaults/handler.go:174` |
| Deployments / Runs | `anthropic-version: 2023-06-01` + `anthropic-beta: managed-agents-2026-04-01` | `internal/deployments/handler.go:1532` |
| Files | 不要求 | `docs/design/be/files-api-compatibility.md` |
| Skills | 不要求；`skills-2025-10-02` 仅选择旧版响应形状 | `internal/skills/contract.go` |
| Webhooks 管理 | `anthropic-beta: webhooks-2026-03-01` | `internal/webhooks/handler.go:30` |

### 官方 SDK 是否发送 `?beta=true`

这是关键差异点。不同 SDK、不同版本的实现并不一致：

| SDK | 发送 `?beta=true` 的版本 | 不发送的版本 |
| --- | --- | --- |
| Go | v1.68.0（本仓库 `go.mod` 固定）、v1.84.0（当前最新 release） | `main` 分支已移除，未来 release 可能不再发送 |
| TypeScript | 1.0.0 | 1.4.0、1.8.0（当前 latest）只发送 `anthropic-beta` 头 |
| Python | 2.0.0 | 2.8.0（当前 latest）只发送 `anthropic-beta` 头 |

因此：

- Go SDK 用户当前可正常工作，因为 Go SDK 仍带 `?beta=true`。
- 最新 TypeScript/Python SDK 用户创建 environment、memory store、vault 时会收到 400，错误大意是 `Environments API requires beta=true`、`Memory Stores API requires beta=true` 或 vault 对应错误。
- 由于 session 创建依赖 environment，TypeScript/Python 用户会在第一步 environment 创建就被卡住。
- 仓库的 `tests/e2e/typescript/memory-stores-e2e.ts` 与 `tests/e2e/python/memory_stores_e2e.py` 未固定 SDK 版本；按当前 latest SDK 推断，第一个 memory 请求会得到 400 而不是期望的 404。该推断未实跑。

### 影响与绕过

影响范围：`POST/GET/POST/DELETE /v1/environments*`、`/v1/memory_stores*`、`/v1/vaults*`。

不受影响：agents、sessions、events、threads、resources、deployments、files、skills。

可行绕过：

1. 使用 Go SDK（当前版本）。
2. 在 SDK 中配置默认 query 参数 `beta=true`。
3. 在反向代理层为这三类路径补 `beta=true`。
4. 固定使用仍会发送 `?beta=true` 的旧版 TypeScript/Python SDK。

OMA 文档在 `docs/en/api/versioning-beta.mdx` 中明确写了这三类资源要求 `?beta=true`，因此这是有意的契约差异，不是遗漏。

## 请求/响应形状兼容性

总体结论：形状兼容度很高，官方 SDK 的 JSON 解析基本不会失败。

### 一致的资源形状

已核对以下响应/请求字段与官方 schema 基本一致：

- Agent：`id`、`version`、`name`、`description`、`model`、`system`、`tools`、`mcp_servers`、`skills`、`multiagent`、`metadata`、时间戳。
- Session：`agent` 快照、`environment_id`、`status`、`resources`、`vault_ids`、`budget`、`usage`、`stats`、`outcome_evaluations`。
- Session Resource：`file`、`github_repository`、`memory_store` 三种类型及 `mount_path`、`created_at`、`updated_at`。
- Thread：`id`、`agent`、`status`、`parent_thread_id`、`stats`、`usage`。
- Memory：`content_sha256`、`content_size_bytes`、`path`、`view`、`precondition`。
- Vault/Credential：`mcp_oauth`、`static_bearer`、`environment_variable` 三种 auth。
- Files：`filename`、`mime_type`、`size_bytes`、`downloadable`、`scope`。
- Skills：稳定版与 `skills-2025-10-02` 旧版两套响应形状。

### ID 前缀

主要前缀与 Claude 示例一致：

`agent_`、`sesn_`、`sesrsc_`、`sthr_`、`sevt_`、`env_`、`file_`、`skill_`、`vlt_`、`vcrd_`、`memstore_`、`mem_`、`memver_`、`depl_`、`drun_`、`outc_`

少量差异：

- Webhook 事件 ID 使用 `wevt_`，Claude 示例为 `whe_`。
- Skill version 内部 ID 使用 `skver_`，但对外通常暴露版本号字符串。

这些前缀差异通常不影响 SDK，因为 SDK 不强制校验前缀。

### 分页

- 大多数资源使用 `page` / `next_page`，与 Claude 一致。
- Files 同时支持 `page` / `next_page`、`after_id` / `before_id`、`has_more` / `first_id` / `last_id`。
- Skills 稳定版使用 `page` / `next_page`。
- 默认 `limit=20`，与常见 Anthropic 行为一致；部分资源允许更高上限（sessions 最大 1000）属于更宽松。

具体缺口：

- `GET /v1/sessions` 的响应只有 `data` 和 `next_page`，见 `internal/sessions/handler.go:49`。
- 但 `openapi/oma.en.json` 的 `BetaManagedAgentsListSessions` 声明了 `prev_page`。
- 因此 OMA 实际不支持 sessions 的反向分页；官方 SDK 可能把 `prev_page` 当作可选字段，不会崩，但该功能缺失。

### 错误信封

OMA 使用标准 Anthropic 信封：

```json
{
  "type": "error",
  "request_id": "req_...",
  "error": {
    "type": "invalid_request_error",
    "message": "..."
  }
}
```

`internal/httpapi/errors.go` 与 `internal/httpapi/app_error.go` 基本对齐。但存在类型映射差异：

| 状态码 | Claude 错误 `type` | OMA 错误 `type` | 场景 |
| --- | --- | --- | --- |
| 409 | `invalid_request_error` | `conflict_error` | Agent 版本冲突、Environment 重名、Session 状态冲突、Vault 并发修改 |
| 413 | `request_too_large` | `invalid_request_error`（消息为 `request_too_large`） | 请求体过大 |
| 402 | 无 | `billing_error` | OMA 扩展 |
| 504 | 无 | `timeout_error` | OMA 扩展 |

证据：

- `internal/httpapi/app_error.go:87`
- `internal/agents/errors.go:72`
- `internal/sessions/errors.go:127`
- `internal/environments/errors.go:91`

影响：HTTP 状态码仍是 409，官方 SDK 通常按状态码抛出通用错误；如果业务代码按 `error.type` 分支，会与 Claude 不一致。

### 429 与 `Retry-After`

Claude 的 `rate_limit_error` 会带 `retry-after`，SDK 会据此自动重试。OMA 的 API 层没有统一设置 `Retry-After` 头，因此 SDK 的自动退避可能退化为默认策略。

## 事件与流式协议兼容性

这部分兼容度很高，基本逐项对齐：

| 能力 | OMA 状态 |
| --- | --- |
| `user.message`、`user.interrupt`、`user.custom_tool_result`、`user.tool_confirmation`、`user.tool_result` | 支持 |
| `user.define_outcome` | 接受并持久化，但不自动评分 |
| `system.message` | 支持 |
| `agent.message`、`agent.thinking`、`agent.tool_use/result`、`agent.mcp_tool_use/result`、`agent.custom_tool_use` | 支持 |
| `agent.thread_context_compacted`、`agent.thread_message_received/sent` | 支持 |
| `session.status_running/idle/rescheduled/terminated`、`session.error`、`session.usage`、`session.updated`、`session.deleted` | 支持 |
| `session.thread_created`、`session.thread_status_*` | 支持 |
| `span.*` 观测事件 | 支持 |
| SSE 流式 | 支持 |
| `event_deltas[]` 预览 | 支持 `agent.message` / `agent.thinking` |
| `event_start` / `event_delta` 线格式 | 与 Claude 文档一致 |
| Thread 独立流 | 支持 |
| Thread 需要 `managed-agents-2026-04-01` 头 | 与 Claude 一致 |

证据：

- `internal/sessions/stream_hub.go:331`
- `internal/sessions/worker_stream_preview.go`
- `docs/en/events-and-streaming.mdx`
- `docs/design/be/session-thread-beta-gating.md`

结论：事件/SSE 是 OMA 兼容度最高的部分。

## 功能语义缺口

这些是“能接受请求，但行为与 Claude 不完全一致”的部分，官方 SDK 不会报错，但结果会不同。

### `auto` 权限策略不自动评估

Claude 的 `permission_policy: {"type":"auto"}` 会让服务端评估每次工具调用，自动执行、拒绝或暂停。

OMA 接受 `auto`，但实际按 `always_ask` 处理：

- `internal/codesessions/tool_permissions.go:338`
- `internal/codesessions/tool_permissions.go:677`
- 文档也明确承认：`docs/en/permission-policies.mdx:19`

影响：使用 `auto` 的 Agent 会频繁暂停等待人工确认，而不是自动放行安全调用。

### Multiagent 的 `workflows` / `advisor` 不执行

OMA 支持 `coordinator` 模式和 `subagents` 委托，但 `workflows` 和 `advisor` 只做配置兼容，不执行实际 workflow run 或 advisor 咨询。

证据：

- `docs/en/multiagent-orchestration.mdx:44`
- `internal/agents/cma_multiagent.go`

影响：Claude 的 dynamic workflows / advisor 场景在 OMA 中不会运行。

### Outcome 不自动评分

Claude 的 `user.define_outcome` 会启动评分循环（`max_iterations`、rubric、结果迭代）。

OMA 只接受并记录 outcome 定义，不运行自动评估或反馈迭代：

- `docs/en/outcomes.mdx:6`
- `docs/en/define-outcomes.mdx:6`
- `internal/sessions/service_helpers.go:299`

影响：`outcome_evaluations` 字段存在，`session.outcome_evaluation_ended` Webhook 也预留，但不会产生真实评分结果。

### Dreams API 缺失

Claude SDK 有 `client.beta.dreams` 资源，OMA 没有对应路由：

- `docs/en/dreams.mdx:7`

如果用户依赖 Dreams API，OMA 不兼容。

### 模型参数只做形状兼容

OMA 接受并存储：

- `model.speed: "standard" | "fast"`
- `model.effort: low/medium/high/xhigh/max`
- `model.inference_geo: "us" | "global"`

但在 Go 运行时代码中未发现将这些字段应用到实际模型请求；`inference_geo` 文档也明确说“不保证地理路由”：

- `docs/en/agent-setup.mdx:197`
- `internal/agents/handler.go:761` 负责解析和校验
- 后续 `codesessions` / brain 侧未发现 effort/speed/geo 的使用

结论：字段可通过 SDK 校验，但行为不等价于 Claude。

### Webhook 生命周期缺少 `session.status_scheduled`

Claude 的 Webhook 事件表包含 `session.status_scheduled`（Session 创建完成、可接收事件）、`session.status_run_started`、`session.status_idled` 等。

OMA 的 Webhook 允许列表中没有 `session.status_scheduled`：

- `internal/webhooks/handler.go:49`
- 创建无 initial events 的 Session 时直接发 `session.status_idled`：`internal/sessions/service.go` 创建流程

影响：订阅 `session.status_scheduled` 的 Claude 客户端在 OMA 中不会收到该事件。

### Webhook 管理接口是 OMA 自有契约

Claude 文档说明目前没有编程式 Webhook 端点管理 API，SDK 的 `client.beta.organization.webhooks` 使用 `/v1/organizations/webhooks`。

OMA 提供的是：

- `POST/GET /v1/webhooks`
- `GET/POST/DELETE /v1/webhooks/{id}`
- `POST /v1/webhooks/{id}/regenerate_signing_secret`

并且 OMA 文档自己标注字段级兼容性尚未与 Claude 独立确认：

- `docs/en/api/resources/webhooks.mdx:6`

OMA 没有实现 `/v1/organizations/webhooks`。因此使用组织级 Webhook 管理 SDK 的代码在 OMA 上会 404。

Webhook 投递本身是兼容的：使用 Standard Webhooks 的 `webhook-id`、`webhook-timestamp`、`webhook-signature` 头，官方 SDK 的 `unwrap()` 可验证。

### 模型目录与别名

Claude 直接使用 Claude 模型 ID；OMA 使用组织配置的 Provider 模型目录：

- 模型 ID 原样发送，不提供 Claude 名称别名、映射或默认上游。
- 证据：`docs/en/compatibility.mdx:29`

影响：`claude-opus-5` 等 ID 必须在 OMA 的 LLM Models 中配置为某个 Provider 的精确模型 ID，否则请求失败。

### 多工作区 header

Claude SDK 支持可选的 `anthropic-workspace-id` 头，用于可跨 Workspace 的凭证。

OMA 的 `/v1` API Key 路由没有读取该头；API Key 绑定到单一 Workspace。对普通 workspace API Key 无影响，但不支持“一个凭证选择多个 Workspace”的 Claude 语义。

## 资源级兼容矩阵

| 资源 | 路径覆盖 | 形状 | 语义 | 主要差异 |
| --- | --- | --- | --- | --- |
| Agents | 完全 | 完全 | 完全 | 409 错误类型为 `conflict_error` |
| Agent Versions | 完全 | 完全 | 完全 | 无明显差异 |
| Sessions | 完全 | 完全 | 完全 | 缺 `prev_page` |
| Events | 完全 | 完全 | 完全 | 高度一致 |
| Threads | 完全 | 完全 | 完全 | 需要 managed beta 头，与 Claude 一致 |
| Session Resources | 完全 | 完全 | 完全 | 自托管仅支持 memory_store，与 Claude 一致 |
| Environments | 完全 | 完全 | 部分 | 必须 `?beta=true`；额外 `state`/`scope` 字段 |
| Deployments | 完全 | 完全 | 完全 | 需要精确 beta 头；无 jitter |
| Deployment Runs | 完全 | 完全 | 完全 | 无明显差异 |
| Vaults / Credentials | 完全 | 完全 | 部分 | 必须 `?beta=true`；409 错误类型 |
| Memory Stores / Memories / Versions | 完全 | 完全 | 部分 | 必须 `?beta=true`；更新用 POST（与官方 SDK 一致，与端点文档的 PATCH 不同） |
| Files | 完全 | 完全 | 完全 | 无 beta 门禁；双分页方案 |
| Skills | 完全 | 完全 | 完全 | 稳定版 + 旧版 beta 形状；额外 content 端点 |
| Webhooks 投递 | 完全 | 完全 | 完全 | Standard Webhooks 兼容 |
| Webhooks 管理 | 部分 | 部分 | 部分 | OMA 自有路径；无 `/v1/organizations/webhooks` |
| Dreams | 缺失 | 缺失 | 缺失 | 未实现 |
| Outcome Evaluation | 接受 | 完全 | 缺失 | 不自动评分 |
| Multiagent workflows/advisor | 接受 | 完全 | 缺失 | 不执行 |
| Permission `auto` | 接受 | 完全 | 部分 | 按 `always_ask` 处理 |

## 文档内部不一致

调研中发现几处 OMA 文档与代码不一致，建议以代码/最新设计为准：

1. `docs/en/limits.mdx:6` 说“OMA does not currently implement `budget` fields or priced-cost metering”，但代码已实现 session budget：
   - `internal/sessions/budget.go`
   - `internal/sessions/budget_test.go`
   - `internal/deployments/handler_budget_test.go`
2. `docs/en/api/resources/webhooks.mdx:49` 说“OMA has not implemented budget configuration...”，同样与代码不符。
   - 更准确的描述应是：budget 已实现，`session.budget_reached` Webhook 目前是保留订阅、无生产者。
3. `openapi/oma.en.json` 声明了 `prev_page`，但 handler 未返回。

## 对 SDK 使用者的建议

### Go SDK

当前可直接使用：

- 仓库固定 `anthropic-sdk-go v1.68.0`。
- 当前最新 release v1.84.0 仍发送 `?beta=true`。
- 注意：Go SDK `main` 分支已移除 `?beta=true`，升级到未来 release 前需复测 environments/memory/vaults。

### TypeScript / Python SDK

需要满足以下之一：

1. 固定到仍发送 `?beta=true` 的版本：
   - TypeScript 1.0.0。
   - Python 2.0.0（或确认仍发送该 query 的版本）。
2. 通过 SDK 默认 query 或反向代理为以下路径补 `beta=true`：
   - `/v1/environments*`
   - `/v1/memory_stores*`
   - `/v1/vaults*`
3. 等待 OMA 改为同时接受 `anthropic-beta` 头。

### 推荐的最小兼容性验证

如果要在真实环境确认，建议跑：

1. Go SDK：agents → environments → sessions → events stream → files → deployments。
2. TypeScript/Python SDK：同一流程，重点观察 environment/memory/vault 是否 400。
3. 错误路径：Agent 版本冲突、Session 状态冲突、Memory path 冲突。
4. Webhook：至少验证 `session.status_idled`、`session.status_run_started`、`session.status_terminated`。
5. 流式：`event_deltas[]`、thread stream、`system.message`。

本次未运行这些真实链路。

## 总体判断

按“官方 SDK 能否直接指向 OMA 使用”评估：

- Go SDK：高（当前版本可直接用）。
- TypeScript/Python 最新 SDK：中低（env/memory/vault 被 beta query 阻断）。
- HTTP/cURL 直接调用：高（只要按 OMA 文档补 `?beta=true`）。
- 事件/SSE 与文件/技能：高。
- 高级语义（auto、outcome、workflows/advisor、Dreams、effort/speed/geo）：中低。

如果目标是“Claude Managed Agents SDK 无感切换到 OMA”，优先需要补齐：

1. environments/memory_stores/vaults 同时接受 `anthropic-beta` 头。
2. 409 错误类型改为 `invalid_request_error`。
3. Session 列表补 `prev_page`。
4. 补 `session.status_scheduled` Webhook 事件。
5. 实现 `auto` 评估、outcome 评分、workflows/advisor，或明确作为“不支持”能力对外声明。

## 参考资料

- [Anthropic Managed Agents endpoint reference (anthropics/skills)](https://github.com/anthropics/skills/blob/main/skills/claude-api/shared/managed-agents-api-reference.md)
- [Claude Managed Agents reference](https://platform.claude.com/docs/en/managed-agents/reference)
- [Claude Managed Agents sessions](https://platform.claude.com/docs/en/managed-agents/sessions)
- [Claude Managed Agents session operations](https://platform.claude.com/docs/en/managed-agents/session-operations)
- [Anthropic TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript)
- [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python)
- [Anthropic Go SDK](https://github.com/anthropics/anthropic-sdk-go)
- 本地证据：`openapi/oma.en.json`、`internal/api/server.go`、`internal/environments/handler.go`、`internal/memory/handler.go`、`internal/vaults/handler.go`、`internal/sessions/*`、`internal/webhooks/*`、`docs/en/**`
