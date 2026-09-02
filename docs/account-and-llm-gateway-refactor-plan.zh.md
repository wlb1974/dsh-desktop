# DSH Desktop 账号、设备绑定与 LLM 网关改造方案

> 状态：提案，待跨项目评审
> 日期：2026-09-01
> 涉及项目：`dsh-desktop`、`yunzhi-idaas`、`xiaoyuanhao-core-gateway`

## 1. 方案结论

本项目不再新建一套账号中心，也不以 PAT 为前提。现有基础设施可以覆盖大部分需求，缺少的部分分别补在 IDaaS 和 Core Gateway：

- IDaaS 继续管理手机号账号。复用现有短信验证码、按手机号查用户、用户不存在时自动注册等能力。
- IDaaS 新增 DSH Desktop 专用会话，负责设备首次绑定、短期 Access Token、Refresh Token 轮换、设备列表和远程撤销。
- Core Gateway 新增独立 LLM Gateway 入口和个人 LLM Key，负责协议转发、身份认证、模型权限、余额拦截、限流和调用事件编排。
- DSH Desktop 生成设备密钥对。私钥、Refresh Token 和个人 LLM Key 只保存在 Electron 主进程控制的系统安全存储中。
- 模型供应商主密钥只存在于服务端。Core 验证个人 LLM Key 后，使用内部服务凭据访问下游，不能把用户凭据继续转发给下游。

最终链路如下：

```text
手机号 + 验证码
      │
      ▼
IDaaS 查找或创建 C 端用户
      │
      ▼
绑定设备公钥，签发桌面会话
      │
      ├── 短期 Access Token：访问账号和密钥管理接口
      └── 轮换 Refresh Token：恢复桌面会话
      │
      ▼
Core Gateway 创建 llm_pk_ 个人密钥
      │
      ▼
DSH Desktop 携带 llm_pk_ + 设备签名调用 LLM
      │
      ▼
Core 校验身份、设备、Scope、模型和余额
      │
      ▼
按 OpenAI Chat Completions / OpenAI Responses / Anthropic Messages 协议调用模型服务
```

手机号登录态、桌面会话和个人 LLM Key 是三种不同对象。它们不能合并为一个长期 Bearer Token。

LLM Gateway 不复用 `/app-api/open/aiproxy/**`。新入口拥有独立的 URL、鉴权 Filter、Scope、协议适配和灰度开关，避免个人模型流量继续进入历史兼容代理。

## 2. 现有能力与复用边界

### 2.1 IDaaS

IDaaS 已经具备手机号注册登录的主链路。`digital` Client 对应：

```text
VerificationCodeLoginHandler
  → ToCMobileUserProviderHandler
  → ToCMobileSignUpHandler
  → BaseCommonTokenIssuanceHandler
```

可直接复用：

- `/send-code` 发送短信验证码；
- `VerificationCodeService` 对接现有短信服务；
- `VerificationCodeLoginHandler` 校验手机号和验证码；
- `ToCMobileUserProviderHandler` 按手机号、用户池和用户类型查找用户；
- `ToCMobileSignUpHandler` 在用户不存在时创建账号；
- 现有用户、账号绑定、用户池和登录成功事件；
- `JwtIdaasTokenStore`、RSA 签名和 OAuth Client 配置能力；
- `/verify`、`/logout` 和用户 Token 索引的部分实现。

现有 `yunzhi_token` 不适合直接作为桌面会话，原因是：

- Token 没有绑定设备；
- Redis Token 的自动续期代码未启用，也没有可靠的 Refresh Token 轮换；
- Cookie 的 HttpOnly 改造仍停留在 Proposed ADR；
- 登录处理器会记录完整请求 JSON，可能包含手机号和验证码；
- `code.test` 代码默认值为 `true`，线上虽然配置为 `false`，但不符合 fail-closed；
- `User-Pool` 和 `User-Type` 来自请求头，桌面登录不能信任客户端提交的租户边界；
- 缺少设备列表、单设备撤销、Refresh Token 重用检测和会话版本。

### 2.2 `idaas-personal-jwt` 工作树

该工作树没有任何领先于 `master` 的提交，功能文件仍是未提交或未跟踪状态，也没有 Core/Auth Center 消费方。它只能作为代码样例，不能整体合并。

可以吸收的部分：

- IDaaS 签发短期个人 JWT；
- `token_use` Claim；
- JWT 日志脱敏；
- Token TTL 上限测试；
- 公开验签公钥的基础配置。

需要舍弃或重做的部分：

- `PAT-derived` 前提；
- `PersonalTokenApi.issue(userId, orgId, deviceId, requestId)` 的可信输入假设；
- 固定 `userType=B`；
- 空 Scope 和无 Refresh Token 的 OAuth Client；
- 只校验 `deviceId` 非空、却不写入 JWT 和不验证设备签名的实现；
- 无法立即撤销的纯 JWT 会话。

### 2.3 Core Gateway

可参考复用：

- `CapabilityProxyService` 的请求、响应、超时和流式透传实现经验；
- `@RequireScope`、API Key 前缀分派和统一错误响应的实现方式；
- 调用者上下文、模型服务地址、超时配置和内部服务 Token 配置；
- 现有 Dubbo/HTTP 客户端、配置中心、指标和限流基础设施。

必须改造：

- 新建 `/llm-gateway/**`，不把新入口映射到 `CapabilityGatewayController`；
- 新建 `LlmGatewayAuthFilter`，不让 `llm_pk_` 落入历史 `ApiKeyAuthInterceptor` 的兼容分支；
- 分别支持 OpenAI Chat Completions、OpenAI Responses 和 Anthropic Messages，协议原样透传，不在网关内部互相转换；
- 新增统一拦截管线：身份、设备证明、可用模型、余额预检、限流、转发、用量结算和调用事件；
- Core 必须删除调用方传入的认证头和内部上下文头，再注入供应商凭据和可信上下文；
- Core 不承载业务数据库。个人 Key、会话、模型目录、余额和调用记录通过 Provider/SPI 获取或写出；
- LLM 流取消必须中止下游连接，避免用户取消后仍继续产生费用。

### 2.4 DSH Desktop

保持现有仓库边界：

- `deepseek-harness/` 是只读上游 submodule，不修改其中代码；
- `dsh-plugin-desktop/` 继续管理 Electron 启动、打包和发布；
- 新增桌面自有包 `dsh-product-cloud/`，通过 Cordis Host/Client 插件装配；
- 继续使用公开的 credentials、authorization、LLM adapter 和 settings 扩展点；
- 上游兼容模式不改路由、不覆盖官方 Client。

桌面端需要自有 LLM adapter。现有 DeepSeek adapter 无法为每次请求增加设备证明头，直接复用只能得到普通 Bearer Key，无法实现密钥与设备的强绑定。

## 3. 凭证设计

### 3.1 IDaaS Access Token

Access Token 使用 IDaaS RSA 私钥签名的 JWT，只用于短期访问 Core 的账号和密钥管理接口。

建议 Claim：

```json
{
  "iss": "https://idaas.xiaoyuanhao.com",
  "aud": ["core-gateway", "dsh-desktop"],
  "sub": "<userId>",
  "sid": "<desktopSessionId>",
  "did": "<deviceId>",
  "client_id": "dsh-desktop",
  "user_pool": "xyh",
  "user_type": "C",
  "tenant_kind": "PERSONAL",
  "scope": ["profile:read", "device:manage", "llm-key:manage"],
  "token_use": "desktop_access",
  "session_version": 1,
  "iat": 0,
  "exp": 0,
  "jti": "<random>"
}
```

规则：

- TTL 10 分钟，最多 15 分钟；
- 不持久化到磁盘；
- Core 校验签名、Issuer、Audience、`token_use`、过期时间和 Scope；
- IDaaS 提供版本化 JWKS，支持双密钥轮换；
- JWT 只解决短期验签，不承担远程撤销。撤销由 Session 状态完成。

### 3.2 Refresh Token

Refresh Token 使用至少 256 bit 随机 opaque 值，建议前缀 `dsh_rt_`。

- 服务端只保存带 Pepper 的哈希；
- 每次刷新都轮换，旧 Token 立即失效；
- 发现旧 Token 再次使用时，撤销整个 Token Family；
- 绑定 `userId + sessionId + deviceId + publicKey`；
- 每次刷新必须提交设备签名；
- 绝对有效期 30 天，空闲有效期 7 天；
- 仅保存于 Electron 主进程安全存储。

### 3.3 个人 LLM Key

个人 LLM Key 是 Core Gateway 签发的随机 opaque Key，建议格式：

```text
llm_pk_<keyId>_<randomSecret>
```

规则：

- 创建时只返回一次明文；
- Core 生成 Key，并通过 `LlmCredentialStore` 保存 Key ID、可识别前缀和 `HMAC-SHA-256(pepper, plaintext)`；Core 本身不落库；
- 绑定 `userId + sessionId + deviceId`；
- 默认授予 `llm:invoke`，模型列表和余额查询分别使用 `llm:models:read`、`llm:balance:read`；
- 保存模型白名单、有效期、状态、创建时间和最后使用时间；
- 每次 LLM 请求同时提交设备签名；
- Core 校验 IDaaS Session 仍有效；设备或 Session 撤销后，Key 最迟在 30 秒状态缓存窗口结束时不可用；
- 默认有效期 30 天，桌面端在到期前自动轮换；
- 仅保存于 Electron 主进程安全存储，不写入上游 `.credentials.yaml`。

这类 Key 是桌面产品凭证，不等同于可复制到任意脚本使用的开放平台 API Key。后续若需要用户手工创建通用 API Key，应增加独立 Token 类型、MFA 和 IP/来源策略，不能复用本期的设备绑定 Key。

### 3.4 设备证明

首版沿用现有基础设施已经使用的 RSA-SHA256：

- 桌面主进程生成 RSA 2048 或 3072 位密钥对；
- 公钥使用 X.509 DER Base64 格式上传；
- 私钥由 Electron `safeStorage` 加密；
- `deviceId` 是随机稳定标识，不读取硬件序列号，也不做不可撤销的机器指纹；
- 服务端允许 5 分钟时钟偏差，Nonce 在 10 分钟窗口内只能使用一次。

通用签名串固定为：

```text
DSH1
<HTTP method>
<normalized path>
<SHA-256 body hex>
<sessionId>
<deviceId>
<unix timestamp>
<nonce>
```

设备首次绑定时，签名串改为绑定 Challenge、设备 ID 和公钥指纹。IDaaS 和桌面仓各保存同一组测试向量，避免 Java 与 TypeScript 的换行、路径和编码差异。

## 4. 完整业务流程

### 4.1 首次注册或登录

1. Electron 主进程检查设备密钥，不存在则创建密钥对和随机 `deviceId`。
2. 用户输入手机号；Renderer 通过受限 IPC 请求主进程发送短信。
3. IDaaS 校验图形验证码或风控票据，对手机号、设备和 IP 分别限流。
4. IDaaS 返回 `challengeId`、过期时间和再次发送等待时间，不暴露手机号是否已经注册。
5. 用户输入验证码，主进程构造设备绑定证明并调用创建 Session 接口。
6. IDaaS 一次性消费 Challenge，按 `xyh/C` 查找用户。不存在则静默创建，存在则复用原用户。
7. IDaaS 创建或更新设备绑定，签发 Access Token 和 Refresh Token。
8. 主进程把 Refresh Token 和设备私钥写入安全存储，Access Token 只保存在内存。
9. Renderer 只收到用户昵称、脱敏手机号、设备名称和登录状态，不收到任何 Token。

手机号注册和登录使用同一个接口，避免先探测“手机号是否存在”。并发请求必须依赖账号唯一约束和幂等键，保证同一手机号不会创建两个用户。

### 4.2 启动恢复和刷新

1. 主进程读取 Refresh Token 和设备私钥。
2. 生成 Timestamp、Nonce 和签名，调用 IDaaS Refresh 接口。
3. IDaaS 校验 Refresh Token 哈希、Token Family、Session、设备状态、签名和 Nonce。
4. IDaaS 原子轮换 Refresh Token，返回新 Access Token 和 Refresh Token。
5. 主进程先安全落盘新 Refresh Token，再切换内存 Access Token。
6. 刷新失败时清理本地会话，进入未登录状态；不能无限重试。

并发刷新由主进程共享一个 Promise，所有调用等待同一次刷新结果。

### 4.3 创建个人 LLM Key

1. 首次调用模型或 Key 临近过期时，主进程调用 Core `POST /api/v1/llm/personal-keys`。
2. 请求携带 IDaaS Access Token 和设备证明。
3. Core 验证 JWT，并向 IDaaS 校验 `sessionId + deviceId + sessionVersion`。
4. Core 只允许服务端定义的 Scope 和模型策略，不能直接接受客户端任意 Scope。
5. Core 生成 Key 明文，通过 `LlmCredentialStore` 把哈希和绑定信息写入当前有状态 Provider；首期由 IDaaS 内部接口承载。
6. Core 返回一次明文和到期时间。
7. 主进程写入安全存储，并向 DSH credentials 写一个不含 Secret 的 Grant 引用。

### 4.4 调用 LLM

1. 自有 adapter 读取内存或安全存储中的个人 LLM Key。
2. adapter 对请求体求 SHA-256，并通过主进程设备私钥签名。
3. adapter 按协议选择新入口：OpenAI Chat Completions、OpenAI Responses 或 Anthropic Messages。
4. Core 根据 `llm_pk_` 前缀进入独立 LLM Gateway 鉴权链。
5. Core 校验 Key 哈希、状态、有效期、Session、设备签名、Nonce 和 `llm:invoke` Scope。
6. `LlmModelPolicyProvider` 校验协议、模型和用户策略；`LlmBalanceProvider` 查询余额并按请求上限预占。
7. Core 删除外部认证头和所有内部上下文头，再注入供应商凭据和服务端签名的 Caller Context。
8. 协议适配器把请求原样转发给对应上游，并按各自协议透传流式响应。
9. 完成、失败或取消时结算实际 Usage，并通过 `LlmInvocationRecorder` 发出调用事件。
10. 客户端取消时，Core 关闭下游连接，不能继续消耗供应商额度。

### 4.5 注销和远程撤销

本机注销顺序：

1. Core 撤销当前设备的个人 LLM Key；
2. IDaaS 撤销当前 Session 和 Refresh Token Family；
3. 主进程清除内存 Token 和安全存储中的 Refresh Token、LLM Key；
4. 保留设备私钥和 `deviceId`，下次登录可再次确认同一设备。用户选择“移除此设备”时才删除私钥。

远程撤销设备时：

- IDaaS 先把设备和全部 Session 标记为撤销；
- 事务 Outbox 通知 Core 撤销关联 Key；
- Core 每次使用 Key 时仍校验 Session 状态，不能仅依赖异步消息；
- Core 可缓存有效 Session 30 秒，撤销事件到达时主动清缓存；
- Outbox 消费幂等并持续重试，避免 Core 短暂不可用造成永久漏撤销。

## 5. 服务端接口

### 5.1 IDaaS 公共接口

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `POST` | `/api/v1/desktop/sms/challenges` | 发送短信，返回 Challenge |
| `POST` | `/api/v1/desktop/sessions` | 验证短信、注册或登录、绑定设备 |
| `POST` | `/api/v1/desktop/sessions/refresh` | 设备签名刷新并轮换 Refresh Token |
| `GET` | `/api/v1/desktop/sessions/current` | 当前 Session 和用户投影 |
| `DELETE` | `/api/v1/desktop/sessions/current` | 注销当前 Session |
| `GET` | `/api/v1/desktop/devices` | 当前用户设备列表 |
| `DELETE` | `/api/v1/desktop/devices/{deviceId}` | 远程撤销设备 |
| `GET` | `/.well-known/jwks.json` | Core 获取 IDaaS 验签公钥 |

所有写接口只接受 HTTPS。Token、验证码和密钥不能放入 URL、Query String 或错误信息。

### 5.2 IDaaS 内部接口

| 接口 | 用途 |
| --- | --- |
| `checkDesktopSession(sid, deviceId, version)` | Core 检查 Session 是否有效 |
| `getDevicePublicKey(sid, deviceId)` | Core 创建 Key 时获取已绑定公钥 |
| `revokeDesktopSession(...)` | 管理端或风控撤销 Session |
| `createPersonalLlmKey(hash, binding, policy)` | 保存 Core 生成的个人 Key 哈希和绑定信息 |
| `resolvePersonalLlmKey(keyId)` | Core 鉴权时读取 Key 元数据和哈希 |
| `list/rotate/revokePersonalLlmKey(...)` | 管理个人 Key 生命周期 |

内部接口需要服务身份认证，不能只因为请求来自内网就信任 `userId`。返回值不包含手机号明文和 Refresh Token。

### 5.3 Core Gateway 接口

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `POST` | `/api/v1/llm/personal-keys` | 创建当前设备的个人 LLM Key |
| `GET` | `/api/v1/llm/personal-keys` | 返回 Key 元数据，不返回明文 |
| `POST` | `/api/v1/llm/personal-keys/{keyId}/rotate` | 轮换 Key，旧 Key 短暂 Grace 后失效 |
| `DELETE` | `/api/v1/llm/personal-keys/{keyId}` | 撤销 Key |
| `POST` | `/llm-gateway/openai/v1/chat/completions` | OpenAI Chat Completions，支持 SSE |
| `POST` | `/llm-gateway/openai/v1/responses` | OpenAI Responses API，支持事件流 |
| `POST` | `/llm-gateway/anthropic/v1/messages` | Anthropic Messages API，支持 SSE |
| `GET` | `/llm-gateway/v1/models` | 返回当前用户和 Key 实际可用的模型 |
| `GET` | `/llm-gateway/openai/v1/models` | OpenAI SDK 兼容的模型列表投影 |
| `GET` | `/llm-gateway/anthropic/v1/models` | Anthropic SDK 兼容的模型列表投影 |
| `GET` | `/llm-gateway/v1/balance` | 查询余额、授信、冻结额和可用额投影 |
| `GET` | `/llm-gateway/v1/capabilities` | 返回已启用协议、版本和能力开关 |

管理接口使用 IDaaS Access Token 和设备证明；模型接口使用个人 LLM Key 和设备证明。

OpenAI SDK 的 Base URL 配置为 `/llm-gateway/openai/v1`，Anthropic SDK 的 Base URL 配置为 `/llm-gateway/anthropic`。三个协议只共享治理管线，不共享请求或响应 DTO，避免字段丢失和流式事件被错误转换。

这里的“SDK 兼容”是协议兼容，不等于个人 Key 可以脱离受信设备使用。DSH adapter 是首期标准客户端；其他 SDK 只有在支持自定义 Base URL，并能为每次请求注入设备签名头时才能接入。拿到 `llm_pk_` 后在任意脚本直接调用不属于本期目标。

Core 中的调用顺序固定为：

```text
LlmGatewayAuthFilter
  → DeviceProofInterceptor
  → ModelPolicyInterceptor
  → BalanceReservationInterceptor
  → RateLimitInterceptor
  → ProtocolProxy
  → UsageSettlement
  → InvocationEventPublisher
```

### 5.4 协议转发要求

OpenAI Chat Completions：

- 保留 `messages`、`tools`、`tool_choice`、`reasoning`、`response_format`、`stream_options` 和 Usage；
- 流式响应保持 OpenAI SSE Data 和 `[DONE]` 结束语义；
- 鉴权接受 `Authorization: Bearer llm_pk_...`，转发前必须删除。

OpenAI Responses：

- 保留 `input`、`instructions`、`tools`、`previous_response_id`、Reasoning 和输出项；
- 原样透传 `response.created`、文本增量、工具参数增量、完成和错误事件；
- 网关不把 Responses 请求降级或转换为 Chat Completions。

Anthropic Messages：

- 保留 `system`、内容块、`tools`、`tool_choice`、Thinking 和 Usage；
- 校验并允许转发受支持的 `anthropic-version`、`anthropic-beta`，拒绝未知强制版本；
- 鉴权接受 `x-api-key: llm_pk_...`，转发前删除并注入真正的上游凭据；
- 保持 `message_start`、`content_block_*`、`message_delta`、`message_stop` 事件顺序。

公共要求：

- 三种协议都要求设备证明头，DSH adapter 负责注入；
- 治理层只读取模型、流式开关、最大 Token 等必要字段，不修改原始业务 Payload；
- 拦截阶段产生的错误由 `ProtocolErrorMapper` 转为对应 SDK 能识别的错误结构；
- 请求和响应 Header 使用明确 Allowlist，不能透传 Cookie、用户认证头或内部服务头；
- 支持背压、SSE 及时 Flush、客户端断连传播、请求体上限和统一超时；
- Core 为每次进入网关的请求生成自己的 `requestId`，同时保存供应商 Request ID；不能把供应商 ID 当作内部幂等键；
- 认证失败、模型拒绝、余额不足等未转发请求也生成结果事件，便于后续审计系统还原决策。

## 6. 状态、数据与扩展接口

### 6.1 IDaaS

继续使用现有用户和账号表，新增：

`idaas_device_binding`

| 字段 | 说明 |
| --- | --- |
| `id`、`device_id`、`user_id` | 稳定主键和归属 |
| `public_key`、`public_key_thumbprint`、`algorithm` | 设备公钥 |
| `display_name`、`platform` | 用户可识别的设备信息 |
| `status`、`bound_at`、`last_seen_at`、`revoked_at` | 生命周期 |

`idaas_desktop_session`

| 字段 | 说明 |
| --- | --- |
| `session_id`、`user_id`、`device_id` | 会话归属 |
| `refresh_hash`、`previous_refresh_hash`、`token_family_id` | Refresh 轮换和重用检测 |
| `version`、`status` | 撤销和版本控制 |
| `absolute_expires_at`、`idle_expires_at`、`last_used_at` | 有效期 |
| `created_ip_hash`、`last_ip_hash` | 风控摘要，不存无必要的原始 IP |

短信 Challenge、Nonce 和限流计数放 Redis；审计事件和撤销 Outbox 持久化到数据库。

### 6.2 个人 Key 存储

Core Gateway 不新增数据库表。首期由 IDaaS 实现 `LlmCredentialStore`，保存 `idaas_llm_personal_key`：

| 字段 | 说明 |
| --- | --- |
| `key_id`、`key_prefix`、`key_hash`、`hash_version` | Key 标识和不可逆存储 |
| `user_id`、`session_id`、`device_id` | 身份和设备归属 |
| `device_public_key`、`public_key_thumbprint` | Key 使用时验证设备证明 |
| `scopes`、`model_policy_ref` | 权限和外部策略引用 |
| `status`、`expires_at`、`revoked_at` | 生命周期 |
| `created_at`、`last_used_at` | 审计元数据 |

Core 只允许短时内存缓存，不把个人 Key、余额、模型策略和调用记录写入本地数据库。后续迁移到专用凭证系统时，替换 `LlmCredentialStore` Provider，不改变桌面端和公共 API。

### 6.3 Core Gateway 状态接口

Core 定义以下稳定接口，当前实现可以是 IDaaS、静态配置或空实现，后续再接入 Billing Center、模型管理和调用审计系统：

| 接口 | 职责 | 无 Provider 时的行为 |
| --- | --- | --- |
| `LlmCredentialStore` | 创建、解析、轮换和撤销个人 Key | 生产环境拒绝启动 |
| `LlmModelPolicyProvider` | 返回模型目录并判断用户是否可用 | 可用只读静态配置；缺配置时拒绝调用 |
| `LlmBalanceProvider` | 查询余额、预占和结算 | 返回 `BALANCE_PROVIDER_UNAVAILABLE`，不放行付费调用 |
| `LlmInvocationRecorder` | 接收开始、拒绝、完成、失败和取消事件 | 开发环境可 Noop；生产可配置为强制可用 |
| `LlmProviderRouter` | 选择上游和内部服务凭据引用 | 缺少路由时返回 `MODEL_UNAVAILABLE` |

调用事件至少包含：`requestId`、协议、用户、Key、设备、模型、开始/结束时间、状态、输入/输出 Token、余额预占 ID、供应商 Request ID 和错误分类。默认不包含 Prompt、工具参数和完整响应。

`LlmInvocationRecorder` 只负责输出事件，不保证本地持久化。Core 可使用有界内存队列和批量发送，但不能用本地数据库、文件或日志冒充可靠记录。未来承载系统负责幂等、落库、查询、对账和保留周期；幂等键统一使用 Core 生成的 `requestId`。

Recorder 的故障策略由 `llm.gateway.recorder.required` 控制。开发和首期联调可以关闭并上报监控；生产若开启强制模式，Sink 不健康时应在建立上游连接前拒绝新调用。对于已经开始的流式调用，只记录发送失败指标并进入外部补偿流程，不能截断已返回给用户的响应来伪造回滚。

## 7. 各项目改造清单

### 7.1 `yunzhi-idaas`

1. 新增 `dsh-desktop` OAuth Client，服务端固定 `xyh/C`，不读取调用方的用户池头。
2. 复用手机号认证、用户查找和自动注册 Handler，增加 `DesktopSessionIssuanceHandler`。
3. 新增设备、Session、Refresh Token、Nonce 和 Outbox 服务。
4. 从 `idaas-personal-jwt` 选择性迁移 JWT Claim、TTL 和日志脱敏测试。
5. 新增 JWKS 和密钥轮换，Core 只接受指定 Issuer、Audience 和 `token_use`。
6. `code.test` 默认改为 `false`；生产环境禁止启动测试验证码模式。
7. 删除完整登录 JSON、验证码、Token 和手机号明文日志。
8. 为会话 Cookie 补 HttpOnly；桌面 API 本身不依赖 Cookie。
9. 短信接口增加图形验证码/风控、手机号/IP/设备限流和错误统一化。
10. 为手机号并发注册、Refresh 重用、Nonce 重放和设备撤销增加测试。
11. 首期实现 `LlmCredentialStore` 内部接口和 `idaas_llm_personal_key`，只保存个人 Key 哈希和绑定元数据。

### 7.2 `xiaoyuanhao-core-gateway`

1. 新增独立 `core-llm-gateway` 模块，拥有 `/llm-gateway/**` 路由、鉴权 Filter、协议代理和治理管线。
2. 不修改 `aiproxy` 的对外语义；新入口不经过 `CapabilityGatewayController` 和历史 `ApiKeyAuthInterceptor`。
3. 增加 `LlmGatewayAuthFilter` 和 `llm_pk_` 解析，Scope 使用 `llm:invoke`、`llm:models:read`、`llm:balance:read`。
4. 分别实现 OpenAI Chat Completions、OpenAI Responses、Anthropic Messages 的请求、流式响应、错误和 Usage 处理。
5. 新增 IDaaS JWT、Desktop Session 和 `LlmCredentialStore` 客户端。
6. 定义 `LlmModelPolicyProvider`、`LlmBalanceProvider`、`LlmInvocationRecorder`、`LlmProviderRouter` SPI 和稳定 DTO。
7. 构造统一 `CallerContext`；删除外部认证头和内部上下文头，再注入供应商凭据和签名上下文。
8. 完成余额预占/结算、模型拦截、限流、SSE flush、断连传播和协议错误映射。
9. 调用记录只发事件，不在 Core 落库；生产环境是否强制 Recorder 可用由配置决定。
10. 设备/Session 撤销采用同步状态检查、短缓存和 Outbox 事件双保险。

### 7.3 `dsh-desktop`

新增 `dsh-product-cloud/`：

```text
dsh-product-cloud/
├─ src/account/          # 手机号登录、Session 状态机、刷新
├─ src/device/           # 密钥生成、签名、设备信息
├─ src/vault/            # safeStorage、原子写入、迁移
├─ src/gateway/          # 个人 Key 管理和自有 LLM adapter
├─ src/host/             # 受限 IPC/HTTP Host API
├─ src/client/           # 登录、设备、Key 和用量页面
└─ tests/
```

具体工作：

- Electron 主进程生成设备密钥，Renderer 不接触私钥和 Token；
- 手机号和验证码通过严格 IPC Schema 传给主进程，请求结束后清空 UI 状态；
- Refresh Token、设备私钥和个人 LLM Key 使用 `safeStorage` 加密并原子写入；
- Access Token 只保存在内存；
- 自有 adapter 添加设备证明头，分别适配 OpenAI Chat Completions、OpenAI Responses 和 Anthropic Messages；
- 三种协议保持各自的流式事件和错误结构，不通过客户端或 Core 做协议互转；
- credentials 中只写无密 Grant 引用，不写明文 Key；
- 设置页提供登录状态、脱敏手机号、设备列表、远程撤销、Key 状态、可用模型和余额投影；
- Linux `safeStorage` 返回 `basic_text` 时禁止“保持登录”，只允许内存会话；
- 保留 BYOK 和上游兼容模式，不在请求失败时自动切换，避免改变隐私和计费边界。

根工作区、打包闭包、Loader smoke、类型检查和发布测试要同步加入新包。

## 8. 实施顺序

### 阶段 0：冻结契约和测试向量

- 确定三类凭证、TTL、错误码、签名串和状态机；
- 固定 OpenAPI、Dubbo/SPI DTO、协议样例、数据库迁移和 Java/TypeScript 签名测试向量；
- 明确个人账户没有 `orgId` 时的 Core 策略，使用 `tenant_kind=PERSONAL`，不伪造组织；
- 建立三个项目的独立功能分支。不要直接在现有脏 worktree 上继续开发。

退出条件：接口和威胁模型评审通过，签名测试向量在 Java 与 TypeScript 中一致。

### 阶段 1：IDaaS 手机号登录和设备会话

- 完成短信接口加固、自动注册幂等和 `dsh-desktop` Handler Chain；
- 完成设备绑定、Session、Refresh 轮换、JWT、JWKS 和撤销；
- 完成设备列表和远程撤销；
- 清理敏感日志、测试模式和 Cookie 安全问题。

退出条件：新老手机号都可登录；复制 Refresh Token 到另一台设备无法刷新；旧 Refresh Token 重放会撤销 Token Family。

### 阶段 2：Core 独立 LLM Gateway

- 完成新路由、`llm_pk_` 创建与鉴权，并接入 IDaaS Key Store 和 Session 状态检查；
- 完成 OpenAI Chat Completions、OpenAI Responses、Anthropic Messages 的同步与流式转发；
- 接入静态模型 Provider，完成可用模型查询和调用前拦截；
- 固定余额、调用事件和供应商路由 SPI；未接入余额 Provider 时付费调用失败关闭；
- 启用内部供应商凭据和签名 Caller Context，外部认证信息不得转发。

退出条件：三个协议都有契约测试和流式冒烟；只拿到 `llm_pk_`、没有设备私钥时无法调用；不可用模型和余额不足在转发前被拒绝；下游收不到用户 Key。

### 阶段 3：桌面纵向切片

- 完成登录 UI、主进程会话、Vault、Key 管理和自有 adapter；
- 完成重启恢复、并发刷新、注销和远程撤销反馈；
- 接入设置页和首次登录引导；
- 通过 Windows/macOS 打包和 Headless Loader 测试。

退出条件：全新用户从手机号注册到首次 LLM 响应可完整走通，应用重启后可恢复，Renderer/日志/配置文件中找不到 Token。

### 阶段 4：外部治理系统和发布加固

- 按产品范围扩展 OpenAI Files、Embeddings、Images 等非首期端点；
- 对接 Billing Center 或其他余额系统，实现查询、预占、结算和对账；
- 接入调用记录承载系统，实现幂等落库、查询和保留策略；
- 完成压测、密钥轮换、风控、审计、告警和事故预案；
- 按白名单灰度，保留 BYOK 和兼容模式回滚路径。

## 9. 验收标准

### 9.1 功能

1. 未注册手机号验证成功后只创建一个 IDaaS C 端用户。
2. 已注册手机号登录不会创建重复账号。
3. 同一账号可绑定多台设备，并可单独撤销其中一台。
4. 应用重启后可用 Refresh Token 和设备证明恢复登录。
5. 登录后可自动创建个人 LLM Key，并分别完成 OpenAI Chat Completions、OpenAI Responses 和 Anthropic Messages 调用。
6. `/models` 只返回当前用户实际可用的模型，不可用模型在请求转发前被拒绝。
7. `/balance` 返回 Provider 给出的余额投影；余额不足时不建立上游连接。
8. 用户可以查看 Key 元数据，但任何查询接口都不返回 Key 明文。

### 9.2 安全

1. 短信验证码只能使用一次，发送和验证均有限流和尝试次数限制。
2. 拷贝 Refresh Token 到另一台设备时刷新失败。
3. 拷贝个人 LLM Key、但没有设备私钥时 LLM 调用失败。
4. Timestamp 超窗、Nonce 重放、请求体被修改时签名校验失败。
5. 远程撤销设备后，Session、Refresh Token 和个人 LLM Key 全部失效。
6. Core 不向下游转发用户的 Authorization、X-Api-Key 或伪造的 X-OPC 头。
7. 数据库、Redis、应用日志、诊断包、崩溃上报和 Renderer 存储中没有明文 Secret。
8. IDaaS JWT 的 Issuer、Audience、`token_use` 或 Scope 不正确时 Core 拒绝请求。
9. 余额 Provider 不可用或返回未知状态时，付费模型调用失败关闭，不允许按零成本或无限额度放行。

### 9.3 工程

- IDaaS 和 Core 的单元测试、集成测试及数据库迁移验证通过；
- `corepack yarn build`、`test`、`typecheck`、`check` 全部通过；
- Headless Loader smoke 不启动图形窗口；
- Windows/macOS 安装包可登录、恢复和注销；
- `deepseek-harness/` 无功能代码修改；
- 个人 Key 改造与 Core 当前 Billing 分支的变更分开提交；
- Core 模块没有新增调用记录、余额账本或模型目录数据库表，也没有直接数据库写入；
- 三类协议分别具有同步、SSE、客户端取消、上游超时和协议错误映射契约测试；
- 被身份、设备、模型、余额和限流拦截的请求都具有唯一 `requestId`，并可生成不含 Prompt 的调用结果事件。

## 10. 灰度和回滚

- IDaaS 用 `dsh.desktop.auth.enabled` 控制新会话入口；
- Core 用 `llm.personal-key.enabled` 和用户白名单控制签发；
- Core 用 `llm.gateway.enabled` 控制新入口总开关，用 `llm.gateway.protocols.openai-chat`、`openai-responses`、`anthropic-messages` 分别控制协议；
- 模型、余额、Recorder 和上游路由 Provider 各自暴露 Readiness；余额和必需 Recorder 未就绪时不进入付费上游；
- Desktop 用 `productCloud.enabled` 控制入口，默认保留现有 Provider；
- 新 Token 和 API 都带协议版本，服务端至少兼容当前与上一桌面版本；
- 网关不可用时不自动切换到 BYOK；由用户确认后切换；
- 回滚桌面版本时不删除 Vault。旧版本不认识的 Grant 作为孤立记录保留；
- 停止签发新 Key 不影响管理员撤销现有 Key；
- 监控短信成功率、登录成功率、Refresh 重用、设备签名失败、Key 签发、401/403/429、首 Token 延迟、流中断和单位 Token 成本。

## 11. 本期不做

- 不引入 PAT、DPAT 或 Auth Center Device Flow 作为桌面登录前提；
- 不支持密码登录；
- 不把 `yunzhi_token` 直接交给 DSH Desktop；
- 不允许桌面持有模型供应商主 Key；
- 不提供可复制到任意第三方脚本的长期开放平台 Key；
- 不复用、改名或改变现有 `/app-api/open/aiproxy/**` 的行为；
- 不在 Core Gateway 建调用记录库、余额账本或模型管理库；
- 不在网关内做 OpenAI Chat Completions、OpenAI Responses 和 Anthropic Messages 之间的协议转换；
- 不修改 `deepseek-harness/`；
- 不在第一期实现企业 SSO、组织切换和复杂 RBAC。接口保留扩展字段即可。

## 12. 需在评审中确认的参数

| 参数 | 默认建议 |
| --- | --- |
| IDaaS Client | `dsh-desktop` |
| 用户池/类型 | `xyh/C`，由服务端固定 |
| Access Token TTL | 10 分钟 |
| Refresh Token | 空闲 7 天、绝对 30 天、每次轮换 |
| 个人 LLM Key TTL | 30 天，到期前自动轮换 |
| 设备算法 | RSA-SHA256，复用现有协议经验 |
| 时钟偏差/Nonce | 5 分钟 / 10 分钟单次使用 |
| Session 状态缓存 | Core 最多 30 秒 |
| 短信验证码 | 5 分钟、一次性、最多 5 次尝试 |
| 个人账户 Tenant | `tenant_kind=PERSONAL`，`tenant_id=userId` |
| 内容留存 | 默认不保存 Prompt 和 Response |
| 新网关公共前缀 | `/llm-gateway`，与 `/app-api/open/aiproxy/**` 完全隔离 |
| 首期协议 | OpenAI Chat Completions、OpenAI Responses、Anthropic Messages |
| 模型目录 | 首期只读静态 Provider；未配置或判定不明时拒绝调用 |
| 余额策略 | 必须预占后调用、按 Usage 结算；Provider 不可用时失败关闭 |
| 调用记录 | Core 仅输出事件；开发可 Noop，生产是否强制 Sink 可用由配置固定 |

以上参数不影响项目分层，可在阶段 0 评审后固定。

## 13. 外部协议基线

阶段 0 以官方协议文档为准建立测试样例：

- [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat/create)
- [OpenAI Responses API 与流式事件](https://platform.openai.com/docs/api-reference/responses-streaming/response/created)
- [OpenAI Models API](https://platform.openai.com/docs/api-reference/models/list)
- [Anthropic Messages API](https://platform.claude.com/docs/en/api/messages/create)
- [Anthropic Messages 流式事件](https://platform.claude.com/docs/en/build-with-claude/streaming)
- [Anthropic Models API](https://platform.claude.com/docs/en/api/models/list)

网关不能运行时自动追随上游协议变化。新增字段应默认透明透传；新增事件类型应允许通过且保序；认证头、计费字段、错误结构或必填字段变化必须经过契约评审、版本化测试和灰度发布。
