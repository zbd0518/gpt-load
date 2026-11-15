# Octopus 与 GPT-Load 的功能差异

> **读者：** 要在两者间做选型、或判断"gpt-load 缺什么"的人。
> **自检：** 一句话结论 · 触发场景 · 细节/证据 · 相关位置。

---

**结论：** 两者只约 40% 功能重叠。octopus **宽**（站点同步、号池、告警通知、语义缓存、WASM 插件、RBAC、全局模型映射），gpt-load **深**（38 个渠道模块、11 个协议方言、WebRTC 实时、凭据级加密与请求脱敏、不可变快照发布）。octopus 已停更（`v2.7.3-Fix2` 为最终版本），所以"octopus 有而 gpt-load 没有"的能力只能作为**短期借用或中期自建**对象，不能作为长期依赖。

**何时用：** 评估选型、决定是否给 gpt-load 补告警/通知/报表、讨论多用户权限或全局模型映射时。

## 事实基线（2026-11 实测）

| | gpt-load（本仓） | octopus |
|---|---|---|
| 路径 | `E:\workspace\source\gpt-load` | `E:\workspace\goland\octopus`（module `github.com/lingyuins/octopus`） |
| 状态 | `main` @ `v2.0.0-rc.64`，活跃迭代 | README 顶部声明 `v2.7.3-Fix2` 为最终版本、停止维护；HEAD 在 `fix/site-model-case-pk-conflict`，有 2 个未提交文件 |
| 规模 | internal 1331 个 Go 文件（742 测试）/ 13.05 万行非测试 + 760 前端文件 | internal 902 个 Go 文件（391 测试）/ 11.7 万行非测试 + 399 前端文件 |
| 栈 | Gin + dig 依赖容器 + logrus；Vue3 + Vite + Tailwind4 | Gin + Cobra/Viper + zap；Next.js SSG 嵌入 |

> 统计陷阱：octopus 的 `build/rebuild-20261102-001/` 含一份被 `.gitignore` 忽略的**源码快照**（906 个 `.go`），不要计入规模。

## 功能对照（只列有差异的）

| 能力 | gpt-load | octopus |
|---|---|---|
| 内置渠道模块 | ✅ 38 个（`internal/channel/modules/`） | 🔶 少量类型 + Hub 站点适配（aihubmix/axonhub/claudecodehub/ldoh/octopus/sapi/sub2api） |
| 数据面协议方言 | ✅ 11 个（Chat/Responses/Images/Embeddings/Rerank/Decisions/Anthropic/Gemini/GeminiEmbeddings/AudioSpeech/Mistral），端点目录 35 条 | 🔶 4 个（Chat/Responses/Anthropic/Embeddings）+ 媒体中继 |
| 实时/长连接 | ✅ WebRTC（`gateway/live_media.go`，pion）、WebSocket、Mistral realtime | ❌ 无（`grep realtime` 只命中价格表） |
| 订阅账号 OAuth | ✅ 分期授权 + 授权态机（ready/refreshing/需重新授权/结果未知）+ 配额观测 | 🔶 号池 OAuth + token 定时刷新 |
| 上游平台管理（New-API 等） | ❌ 当普通渠道接入 | ✅ 站点：多账号、同步、自动签到、套餐监控 |
| 调度 | ✅ 组优先级/权重 + Native→Converted 路由分层 + 会话/prompt-cache 亲和 + 冷却/黑名单 | ✅ 5 策略（轮询/随机/故障转移/加权/Auto）+ 按延迟选端点 + 熔断 + 条件分组 + AI 生成路由表 |
| 全局模型映射（正则/通配） | ❌ 仅分组 alias + 上游改写 | ✅ `ModelMapping`（exact/wildcard/regex + 优先级 + 分组作用域） |
| Key 治理 | ✅ 组/协议/模型过滤、RPM、并发、成本上限、IP/CIDR | ✅ 白名单、过期、费用上限、RPM/TPM、按模型配额、IP 白名单 |
| 告警与通知 | ❌ 无 | ✅ 错误率/费用/额度/渠道下线 → 8 渠道（Webhook/Gotify/Email/Telegram/飞书/钉钉/企微/ntfy） |
| 计划报表 | ❌ | ✅ 日/周/月调度送达 |
| 多用户与权限 | ❌ 单 `AUTH_KEY`，无用户无角色 | ✅ admin/editor/viewer + JWT + WebAuthn/Passkey |
| 请求隐私 | ✅ 脱敏 + 恢复令牌（`requestredact`，20+ 测试文件） | 🔶 请求过滤 + 隐私 issue |
| 缓存 | ❌ 仅 `prompt_cache_key` 亲和与响应所有权绑定 | ✅ 语义缓存（流式 SSE 重放）+ 失败重放缓存 |
| 扩展性 | ❌ | ✅ WASM 插件（extism/wazero + SDK） |
| 备份 | ❌ | ✅ WebDAV 定时备份/一键恢复 |
| 配置生效 | ✅ 不可变快照：事务内预编译 → 整体 Publish → 启动 drain 恢复 | 🔶 写库 + 刷新缓存，运行时状态可持久化（Redis 可选） |

## 选型判断

| 你的情况 | 选谁 |
|---|---|
| 多份订阅账号统一治理、按凭据配额调度 | gpt-load（octopus 只有静态 Key 号池） |
| 要语音/实时通话、Images/Embeddings/Rerank/Mistral 端点 | gpt-load |
| 要被动告警、套餐到期提醒、计划报表 | octopus |
| 要分享给他人、需要分权限 | octopus |
| 想靠缓存省钱 | octopus |
| 不愿改源码就扩展网关 | octopus（WASM 插件） |
| 重视凭据不外泄、请求体可审计 | gpt-load |

## 三个容易记错的判断

- **调度单位不同**：gpt-load 是 `Group + Credential`（凭据带身份代际、冷却、配额观测）；octopus 是 `Channel + Key`，另有独立的号池层。这决定了"订阅账号治理"只有 gpt-load 能做。
- **gpt-load 没有任何自建缓存**：`prompt_cache_key` 亲和只是"把同一会话压到同一凭据以命中上游缓存"；真正省钱的缓存层只有 octopus 有。
- **"AI 路由"不是同一件事**：gpt-load 的 `automodel` 是按模型选目标/补全档位；octopus 的是用 LLM 生成整张路由表 + JSON 条件命中。

## 坑

- **不要复制 octopus 的 `codestable/` 结构**：它是旧形态（`audits/brainstorms/features/feedback/gates/goals/refactors/reference/requirements/roadmap`），与当前 cs 新形态（`vision/spec/epics/issues/notes/talks/tools`）不兼容。只借用其 spec/vision 的**写法密度**。
- **octopus 的 spec 已漂移**：文中写 Go 1.24，`go.mod` 已是 1.27.3。制度记忆必须按代码校准，不能当权威——这也是本仓 spec 的写法要求。
- octopus 的 `build/` 是构建产物 + 源码快照，不属于源码规模。

**相关：** `codestable/talks/001-observability-gaps.md`。

证据位置：gpt-load `internal/control/http_routes.go`（管理面路由全集）、`internal/gateway/router.go`（数据面端点目录）、`internal/channel/modules/`、`internal/gateway/live_media.go`、`internal/automodel/`、`internal/requestredact/`；octopus `internal/server/handlers/`、`internal/model/model_mapping.go`、`internal/relay/semantic_cache.go`、`internal/hub/`、`README_zh.md`。
