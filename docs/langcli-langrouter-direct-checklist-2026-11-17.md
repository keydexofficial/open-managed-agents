# Langcli 直连 LangRouter 方案核对清单

> 前提：Langcli 的 `ANTHROPIC_BASE_URL=https://api.langrouter.ai`，模型请求与 WebSearch/WebFetch 都直接走 LangRouter，使用用户独享的 API key，不经过 OMA `/v1/messages`。
>
> 这份清单用于逐项核对。勾选前需要确认对应代码路径、配置和验证场景。

---

## 1. `ANTHROPIC_BASE_URL` 仍被 OMA 自己的端点复用

Langcli 里这个变量不只给模型用：

- upstream proxy：`{ANTHROPIC_BASE_URL}/v1/code/upstreamproxy/ca-cert` 和 `/ws`
- OMA Files 下载：`src/services/api/filesApi.ts:30`

如果它指向 LangRouter：

- OMA 的 upstream proxy 会去请求 LangRouter 不存在的路径，relay fail-open 关闭；
- Bash/MCP/hook 子进程失去 OMA 的 `HTTPS_PROXY`、MITM、Vault 凭据注入和出站策略；
- 依赖 OMA Files API 的下载/文件路径会指向 LangRouter。

如果你们不使用 OMA upstream proxy/Vault/Tunnel MCP/Files 下载，可以接受；否则需要在 Langcli 里拆一个 `OMA_BASE_URL`，让 upstream proxy 和 Files 继续指向 OMA，而模型/Web 指向 LangRouter。这是很小的补丁，但必须做。

另外 OMA 当前会设置 `CCR_UPSTREAM_PROXY_ENABLED=1`。直连模式下应显式关闭，否则每次启动都会尝试 LangRouter 的 CA/WS 并打警告。

- [ ] 已确认是否使用 OMA upstream proxy / Vault / Tunnel MCP / Files 下载
- [ ] 若使用：Langcli 已拆分 `OMA_BASE_URL` 与 `ANTHROPIC_BASE_URL`
- [ ] 若不使用：已显式关闭 `CCR_UPSTREAM_PROXY_ENABLED`
- [ ] 已复核 `GIT_SSL_CAINFO=/root/.ccr/ca-bundle.crt` 在直连模式下的行为
- [ ] 已验证 Bash/MCP/hook 子进程的出站行为

---

## 2. 用户 key 必须注入为 `ANTHROPIC_AUTH_TOKEN`，不是 `ANTHROPIC_API_KEY`

Langcli 的 WebSearch/WebFetch 只读：

```text
ANTHROPIC_AUTH_TOKEN || apiKeyHelper
```

（`src/tools/WebSearchTool/adapters/apiAdapter.ts`、`src/tools/WebFetchTool/WebFetchTool.ts`）

如果只注入 `ANTHROPIC_API_KEY`，模型可能能用，但 WebSearch 仍然报 `No API token available`。直连模式下应：

- 注入 `ANTHROPIC_AUTH_TOKEN=<用户 LangRouter key>`；
- 移除或旁路 OMA 的 `anthropic_oauth` FD，避免 Langcli 把它当成 Claude.ai 订阅认证而覆盖 API key；
- 实测 `isClaudeAISubscriber` / `getAuthTokenSource` 的实际优先级。

- [ ] 已注入 `ANTHROPIC_AUTH_TOKEN`
- [ ] 已移除/旁路 `anthropic_oauth` FD
- [ ] 已验证模型请求使用用户 key
- [ ] 已验证 WebSearch/WebFetch 使用用户 key
- [ ] 已验证 OAuth FD 不会覆盖用户 key

---

## 3. 附件/文件引用仍然会断

OMA 投递 worker 时会把用户图片/文档改写成：

```json
{"type":"file","file_id":"file_..."}
```

正常路径是 OMA `/v1/messages` 在转发前把 `source.type=file` 展开为 base64/text。Langcli 直连 LangRouter 后，LangRouter 不认识 OMA 的 `file_...`，图片/PDF/文档输入会失败。

需要二选一：

- OMA 在投递 worker 前预展开成 base64/text；
- 或 Langcli 在发模型请求前用 `OMA_BASE_URL` 调 OMA Files API 解析 `file_id`；
- 或保留一个很薄的 OMA 模型代理专门做附件解析。

- [ ] 已确认用户消息中的图片/PDF/文档来源
- [ ] 已选定附件解析方案
- [ ] 已验证 `source.type=file` 在模型请求前被正确展开
- [ ] 已验证图片和 PDF 输入
- [ ] 已验证超过大小限制/不可用附件降级行为

---

## 4. 模型协议和模型名要逐个确认

Langcli 按模型名选择协议：

- `claude-*` → Anthropic `/v1/messages`
- `kimi-k2.5`、`deepseek-flash`、`glm-5.2` 等 canonical 非 Claude 名 → OpenAI `/v1/chat/completions`
- 其他名字 → 默认 Anthropic 协议

LangRouter 同时支持 web search/fetch；还需要确认它对每个模型支持哪种协议，以及工具调用、流式 usage、thinking 的格式是否和 Langcli 的客户端一致。OMA 的 `api_backend` 在直连模式下不再决定协议，不能作为依据。

如果 LangRouter 对非 Claude 模型只支持 OpenAI 协议，那 Langcli 的 canonical 路由正好匹配；如果它只支持 Anthropic 协议，就需要强制 firstParty 或调整模型名。

- [ ] 已列出所有 LangRouter 模型名
- [ ] 已确认每个模型使用 Anthropic 还是 OpenAI 协议
- [ ] 已验证工具调用格式
- [ ] 已验证流式 usage / thinking
- [ ] 已固定 Langcli 版本并记录模型表
- [ ] 已确认是否使用 `langrouter/auto`（建议 OMA Agent 不使用）

---

## 5. OMA 的模型计费/预算/观测会消失

直连后：

- `/v1/messages` 的 usage、session budget、计费、模型请求 span 不再产生；
- 公开 `model_request_start/end` 事件消失；
- per-agent 模型约束变成建议值，用户可用 `/model` 改；
- 模型费用由 LangRouter 直接向用户收，OMA 无法可信计量。

如果 OMA 仍要展示/限制模型用量，需要重新定义；如果费用完全由 LangRouter 承担，就把 OMA 的预算/计费改成平台/工具用量或直接关闭模型预算。

- [ ] 已确认模型费用由谁收取
- [ ] 已确认 OMA 是否还需要模型 usage 展示
- [ ] 已确认 session budget 是否仍需执行
- [ ] 已定义模型请求 span/公开事件缺失后的替代方案
- [ ] 已确认 per-agent 模型约束是否仍可接受为建议值

---

## 6. Web 策略/SSRF/审计

即使用户 key 是自己的，WebSearch/WebFetch 仍绕过 OMA 的：

- 环境/工具域名策略；
- 内容预算；
- SSRF/重定向校验；
- 搜索计量和审计；
- 租户数据边界。

WebFetch 还有本地直连 fallback，在放开出站后可以访问内网地址。如果接受 BYO 模式，需要在产品上明确这些限制由 LangRouter/用户自行承担。

- [ ] 已确认是否接受 OMA 域名策略旁路
- [ ] 已确认是否接受 OMA 内容预算/SSRF 校验旁路
- [ ] 已确认 WebFetch 本地直连 fallback 的 SSRF 风险
- [ ] 已确认 Web 工具用量如何计量/审计
- [ ] 已确认租户数据边界说明

---

## 7. 模型改名

如果所有 OMA 模型名都改成 LangRouter 名：

- 注意 Langcli 的 canonical 名会触发 OpenAI 协议，确认 LangRouter 的 OpenAI 端点支持工具调用和流式；
- OMA 公共 API/DB 里的模型 ID 变更是一次破坏性迁移（agents、sessions、snapshots、batches、deployments、usage）；
- Langcli 的别名会随版本变化，建议 pin Langcli 版本并维护显式模型表；
- `langrouter/auto` 会让 per-agent 模型约束和复现性变差，建议不要用于 OMA Agent 的固定模型。

- [ ] 已列出需要迁移的 DB/API 字段
- [ ] 已决定是否保留 OMA 公共 ID + LangRouter 映射
- [ ] 已处理 agents / sessions / snapshots / batches / deployments
- [ ] 已处理 usage/billing 历史
- [ ] 已更新 SDK 示例、文档和测试
- [ ] 已决定 `langrouter/auto` 是否允许

---

## 最小落地清单

- [ ] Langcli 拆 `OMA_BASE_URL` 与 `ANTHROPIC_BASE_URL`，保证 upstream proxy/Files 仍指向 OMA；或明确关闭这些 OMA 功能
- [ ] 注入 `ANTHROPIC_AUTH_TOKEN=<用户 LangRouter key>`，移除 `anthropic_oauth` FD，实测模型和 Web 工具都使用该 key
- [ ] 显式关闭 `CCR_UPSTREAM_PROXY_ENABLED`，并验证 `GIT_SSL_CAINFO`、Bash/MCP 出站行为
- [ ] 解决 `source.type=file` 附件解析
- [ ] 明确每个 LangRouter 模型的协议、工具调用、流式 usage
- [ ] 重定义模型 usage/budget/billing/observability
- [ ] 明确 Web 工具策略/SSRF/审计由谁承担
- [ ] E2E 覆盖：图片/PDF、工具调用、流式、resume、subagents、WebSearch/WebFetch、key 轮换

---

一句话：WebSearch/WebFetch 走 LangRouter 在这个架构里可以成立；真正剩下的是 base URL 双用、key 注入方式、附件解析、模型协议和 OMA 计费/观测合同的重新定义。
