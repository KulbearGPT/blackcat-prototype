# API 使用说明

## 1. 定位与范围

`openapi.yaml` 是计划开发的 Prototype P0 统一业务 API 合同，采用 OpenAPI 3.1.0；它描述交付目标，不表示产品已实现。Sapphire Bot 与 React Dashboard 复用同一套 endpoint、DTO 和应用用例；`/admin` 仅表示需要工作人员权限，不表示只能由 Dashboard 调用。API 不提供 `/bot`、`/dashboard` 两套重复业务路由。

只有 API 可以访问 PostgreSQL。第三方系统承担用户解析、真实余额、扣款、退款与 webhook 验证；本地保存用户映射、订单、`FundReservation`、派单、礼物、消费、两类推荐归因与返佣、陪玩收益、客服任务、审批、Guild Bot 配置、外部交易镜像、Outbox 和审计事实。所有订单和礼物消费入口必须经过统一 API。

P0 仅支持即时单。预约/排班、陪玩试音或审核材料展示、玩家 audition、用户选择/指定陪玩、多人订单、多级返佣、自动提现、自建钱包和硬删除财务/审计记录均不在合同中。

## 2. 基础约定

- 业务前缀：`/api/v1`；健康检查为 `/health`、`/ready`。
- 金额：`amountMinor` 等整数最小货币单位字段与 `currency` 同时出现；P0 配置币种为 `CNY`。
- 时间：RFC 3339 UTC `date-time`。
- 标识：本地业务实体使用 UUID；Discord User/Guild/Channel/Message 使用字符串 Snowflake。
- 分页：`cursor` + `limit`，响应返回不透明 `nextCursor`。
- 响应：成功统一包含 `requestId` 与 `data`；失败统一包含 `requestId` 与 `error`。
- 余额：统一返回 `providerBalanceMinor`、`reservedMinor`、`availableMinor`、`currency`、`fetchedAt`；`availableMinor = providerBalanceMinor - reservedMinor`。
- 并发：可变对象写请求携带 `expectedVersion`；订单/礼物状态与预留创建、捕获或释放在同一并发控制边界内，陈旧普通对象版本返回 `409 CONFLICT`，陈旧 Bot 配置版本返回 `409 CONFIG_VERSION_CONFLICT`。
- 删除：订单、结案、交易、消费、礼物、返佣、陪玩收益、退款、Bot 配置事件和审计没有 DELETE 路由；返佣与陪玩收益纠错只追加 Adjustment，不修改原始金额或创建带 `reversalOf` 的新主记录。

## 3. 鉴权与 Actor Context

### Bot 调用

Bot 使用 `BotServiceToken`，并传入真实点击者：

```http
Authorization: Bearer <bot-service-token>
X-Client-Source: DISCORD_BOT
X-Actor-Discord-User-Id: 123456789012345678
X-Actor-Guild-Id: 987654321098765432
X-Discord-Interaction-Id: 112233445566778899
Idempotency-Key: discord:112233445566778899
```

Actor Header 只有在 Bot 服务 Token 校验成功后才可信。Bot 不提交 `actor_id`、工作人员级别、权限码或可直接授权的 Role ID；API 按 Discord 身份读取本地用户/员工账号和对象归属。

### Dashboard 调用

Dashboard 使用服务端 `p0_session` Cookie，并传 `X-Client-Source: DASHBOARD`。操作者、有效级别、scope、`permissions_version` 和 step-up 状态均从服务端会话与数据库解析；浏览器自报 Actor Header、Role 或级别不参与授权。

### 内部与 Webhook

- 自动派单 Worker 使用 `InternalServiceToken`，Actor 来源为 `SYSTEM_JOB`。
- Presence 与 Role 同步只接受受限 Bot 服务身份。
- 支付 webhook Actor 来源为 `PROVIDER`，不是人类操作者。公网入口接收供应商发送的原始 bytes 和实际媒体类型，由服务器捕获完整 HTTP headers 与接收时间；原始 bytes 在任何解析或反序列化之前交给中立 adapter 验签。

## 4. 幂等规则

所有 POST、PUT、PATCH 业务写操作均要求 `Idempotency-Key`；支付 webhook 使用 `providerKey + providerEventId` 去重。相同 key 与相同请求指纹返回首次结果，相同 key 与不同指纹返回 `409 IDEMPOTENCY_CONFLICT`。

业务 intent 与第三方写都使用稳定键；重试不得换 key：

- 订单预留：`reservation:order:{orderId}:v1`，在 `submitOrder` 创建。
- 礼物预留：`reservation:gift:{giftRequestId}:v1`，在 `createOrderGiftRequest` 创建。
- 订单捕获/扣款：`debit:order:{orderId}:v1`，在 `confirmOrder` 完成订单时执行。
- 礼物捕获/扣款：`debit:gift:{giftRequestId}:v1`，在 `approveGiftRequest` 批准时执行。
- 退款：`refund:{externalTransactionId}:{resolutionId}:v1`

第三方写超时或状态为 `UNKNOWN/PENDING` 时，不得换 key 盲重试。系统先按原 key 查询交易；礼物播报重试仅重跑 Outbox，不再次扣款。

## 5. 角色、权限与审批

固定工作人员角色为 `L1_SUPPORT`、`L2_SUPERVISOR`、`L3_OPERATIONS`、`L4_ADMIN_OWNER`。四级角色采用累积继承：高等级自动继承全部低等级执行权限，顺序固定为 `L1 < L2 < L3 < L4`。每个 operation 均声明 `x-required-permissions`、`security` 和 `x-actor-context`；API 以 `actor rank >= required rank` 判断最低级别，再检查累积权限码、scope、对象状态、对象归属、金额与近期 step-up。客户端不得按“角色名称必须完全相等”授权。

| 规则 | P0 行为 |
|---|---|
| L1 | 可查询、认领、核对、备注、提交审批；不能扣款、退款、最终批准/拒绝或执行破坏性操作 |
| L2 礼物 | `amountMinor <= 200000 CNY` 可批准 |
| L2 退款 | `amountMinor <= 50000 CNY` 可执行 |
| 超 L2 阈值 | 创建审批并升级 L3，不产生半成功资金操作 |
| `amountMinor >= 500000 CNY` | 最低要求 L4；L4 可直接发起并执行，不要求另一人复核 |
| L3/L4 敏感操作 | 需要 15 分钟内 step-up；否则返回 `428 STEP_UP_REQUIRED` |

业务审批采用 `MINIMUM_LEVEL_DIRECT`：达到所需最低级别的同一名工作人员可以发起、批准或执行，L3/L4 不因自己是申请人而被拒绝。审批请求主要用于低级别向高级别移交，不是强制双人控制。所有资金、高额和配置动作仍必须填写原因、通过近期 step-up、重新校验对象版本与预留，并写入不可删除审计。账号首次提升到 L3/L4 仍需现有 L4 确认，因为目标账号在升级生效前尚未拥有高级权限；这不属于业务审批继承。

Role 同步中，L1/L2 可自动生效；首次 L3/L4 仅产生 `ROLE_ELEVATION_PENDING`，待现有 L4 确认。Role 移除或降级立即降低内部权限、增加 `permissionsVersion` 并撤销旧会话。

新增权限边界：本人能力使用 `player.workspace.read`、`order.readiness.confirm`、`order.cancellation.preview`、`gift.request`；客服按 scope 使用 `order.pause` 与 `order.resume`；L2+ 仅可用 `referral.read` 读取脱敏列表；归因机密详情与修改统一要求 `referral.manage`，至少 L3 且每次读取均写审计事件。Bot 配置读取要求 `bot_config.read`，日常字段修改要求 `bot_config.operational.manage`（L3+），Role 映射字段修改还要求 `bot_config.security.manage`（L4）；最终判断始终由 API 基于有效 `staff_account` 和累积权限完成。

与 03 数据模型统一的 canonical 状态如下：

- `UserStatus`：`ACTIVE`、`PAUSED`、`SUSPENDED`、`DISABLED`。普通 `setUserOperationalStatus` 只接受前三项；`DISABLED` 保留给 L4 系统级终止流程。
- `StaffTaskType`：`ORDER_ASSIST`、`CANCELLATION_ASSIST`、`GIFT_REVIEW`、`PLAYER_START_LATE`、`PLAYER_NO_SHOW`、`CUSTOMER_NO_SHOW`、`SERVICE_INTERRUPTED`、`COMPLETION_REVIEW`、`DISPUTE`、`AUTOMATION_FAILURE`。细分失败原因写入 `reasonCode`，不另造任务类型。

## 6. 主要端点组

| 领域 | 代表 operationId | 说明 |
|---|---|---|
| 健康 | `getHealth`、`getReadiness` | 区分进程存活与依赖就绪 |
| 绑定/个人中心 | `createBinding`、`getCurrentUser`、`getCurrentBalance`、`listCurrentUserStaffTasks` | 第三方账号映射、派生余额与本人客服进度 |
| 目录/估价 | `listServices`、`estimateService`、`estimateOrder` | 用户响应不含陪玩结算价 |
| 订单 | `createOrder`、`submitOrder`、`previewOrderCancellation`、`cancelOrder` | 提交时预留；取消前预览；捕获前释放、捕获后退款 |
| 派单/接单 | `dispatchOrder`、`acceptOrder`、`declineOrderOffer` | 五项资格过滤、事务性唯一接单与本人拒单 |
| 陪玩工作台 | `getPlayerWorkbench`、`setPlayerAvailability`、`listCurrentPlayerEarnings` | 派生资格、匹配单、倒计时、当前订单、本人收益与下一步 |
| 服务闭环 | `setOrderReadiness`、`requestOrderCompletion`、`confirmOrder` | 双方 READY 才进入服务中；完成时捕获订单预留 |
| 礼物 | `createOrderGiftRequest`、`cancelGiftRequest`、`approveGiftRequest`、`rejectGiftRequest` | 请求时预留；批准捕获；取消/拒绝释放 |
| 礼物结果/播报 | `getAdminGiftRequest`、`handlePaymentWebhook`、`retryJob` | 规范化扣款结果；播报失败不重复扣款 |
| 返佣归因 | `listReferralAttributions`、`createReferralAttribution`、`getReferralAttributionConfidential` | `PROMOTER_FIRST_PURCHASE` 与 `PLAYER_LIFETIME` 互斥；费率由服务端配置 |
| 消费/返佣/收益 | `listCurrentUserConsumptions`、`listCurrentUserCommissions`、`listCommissions`、`listPlayerEarnings` | 本人返佣只限 beneficiary 且来源客户脱敏；原记录不可覆盖 |
| 管理用户 | `listAdminUserConsumptions`、`createUserRiskFlag` | staff scope 消费镜像查询；追加风险事件和审计但不自动暂停用户 |
| 自动化接管 | `pauseOrderAutomation`、`resumeOrderAutomation` | L1+ 可按订单暂停/恢复，版本化并审计 |
| Dashboard | `getDashboardSummary`、`listStaffTasks`、`listApprovalRequests`、`listAuditLogs` | 八项最小指标；Bot 员工按钮调用同一 endpoint |
| 权限 | `syncDiscordRoles`、`approveStaffRoleElevation`、`updateStaffRole` | Role 仅是输入信号，内部账号才是授权事实 |
| Bot 配置 | `getBotConfig`、`validateBotConfigChange`、`updateBotConfig`、`testBotConfigDelivery` | Bot/Dashboard 共享；API 负责字段授权、Discord 对象校验、版本控制和事件追加 |

### 6.1 Guild Bot 配置合同

| 方法与路径 | operationId | 行为 |
|---|---|---|
| `GET /api/v1/admin/bot-config?guildId=...` | `getBotConfig` | 返回当前版本、经 Schema 校验的配置和当前操作者可管理字段 |
| `POST /api/v1/admin/bot-config/validate` | `validateBotConfigChange` | 不落配置；返回 `normalizedChanges`、`warnings`、`errors`、`mayApply`、所需权限，以及可应用时的短期 `validationToken` |
| `PATCH /api/v1/admin/bot-config` | `updateBotConfig` | 要求预览签发的 `validationToken`，再以 `guildId + expectedVersion` 原子更新当前行并追加事件，返回新版本和 `auditEventId` |
| `POST /api/v1/admin/bot-config/test-delivery` | `testBotConfigDelivery` | 校验 Guild、频道用途和 Bot 权限后发送带测试标记的消息，不创建订单、礼物、消费或配置事件 |

校验请求使用 `{guildId, expectedVersion, changes, reason}`；应用请求还必须提交 `validationToken`。Token 由 API 签名并绑定真实 staff actor、Guild、`expectedVersion`、规范化后的 `changes` 摘要和 `reason`，默认 5 分钟失效；缺失、过期、操作者不符或内容不符均拒绝，PATCH 仍需重新执行字段授权和业务校验。`changes` 只能包含批准的配置键；API 先按字段计算授权，再校验 Channel/Category 类型、Guild 归属、Bot 权限、Role 是否可管理、数值范围和模板占位符。L3 可管理频道、通知 Role、时间规则、模板和功能开关；`player_role_id` 与 `staff_l1_role_id` 至 `staff_l4_role_id` 属于 L4 Role 映射。L3 提交这些字段时，即使 Discord 中拥有同名 Role，也返回 `403 PERMISSION_DENIED`。

应用时先验证 `validationToken`，再在一个事务内执行 `UPDATE guild_bot_configs SET version = version + 1 ... WHERE guild_id = :guildId AND version = :expectedVersion`，然后插入同一新版本的 `guild_bot_config_events`；更新零行返回 `409 CONFIG_VERSION_CONFLICT`，客户端必须重新读取、重新预览并取得新 Token 后再次确认，不能覆盖较新配置。事件记录变更前值、变更值、原因、真实工作人员、来源与时间，并且只允许追加。写入成功后立即失效对应 Guild 缓存并加载新版本；下一次派单、礼物播报、客服提醒或功能开关判断必须使用新配置。

`BotConfigFieldSet` 不接受 Token、数据库连接、支付密钥、Webhook/OAuth Secret，也不接受服务、价格、礼物目录或返佣规则。Bot 启动或重启时按 Guild 调用 `getBotConfig` 重建缓存；写入成功后立即刷新对应 Guild 缓存，缓存不能成为授权或版本事实来源。

`FundReservation` 是生命周期记录，不是可手工修改的余额字段。订单在 submit 时 reserve、completion 时 capture；礼物在 request 时 reserve、approval 时 capture。取消、拒绝、过期、无人接单或失败在捕获前执行 release；捕获后的资金返回必须追加 refund/Adjustment。争议保持预留，部分结案按决议 capture 并 release 余量。

返佣归因只有 `PROMOTER_FIRST_PURCHASE`（一次性固定额或首笔符合条件净消费比例）和 `PLAYER_LIFETIME`（长期符合条件净订单和/或净礼物比例）。同一被推荐客户只能有一个有效且互斥的归因。客户 API、错误、Discord、通知和导出不得向被推荐客户暴露 source、beneficiary、program type、rate、amount 或 status。`/me/commissions` 仅返回当前 actor 是 beneficiary 的记录，来源客户只给无稳定 ID 的脱敏展示。

返佣与陪玩收益列表返回主记录、`adjustments` 和 `netAmountMinor`。`netAmountMinor = amountMinor - sum(adjustments.amountMinor)`；部分冲正不改原主记录金额，只有累计全额冲正才可把主记录状态标为 `REVERSED`。`updateCommission` 或 `updatePlayerEarning` 使用 `CREATE_REVERSAL` 时，响应 `resultType` 为 `ADJUSTMENT_CREATED`，并返回新建的 `CommissionAdjustment` 或 `PlayerEarningAdjustment`。

支付状态按对象语义分离：单笔退款 `RefundTransactionStatus` 仅有 `UNKNOWN/PENDING/SUCCEEDED/FAILED`；原扣款镜像使用 `DebitAggregateStatus`，额外包含 `PARTIALLY_REFUNDED/REFUNDED`。Webhook 规范事件同时保留 `eventType` 与四态交易 `status`，API 再根据累计成功退款更新原扣款聚合状态。

## 7. 调用示例

更新订单草稿时，每个业务意图使用独立且稳定的 key：

```bash
curl -X PATCH 'https://api.example.invalid/api/v1/orders/8f0de2a5-a8d5-4c24-b32c-4b6673acb510' \
  -H 'Authorization: Bearer <bot-service-token>' \
  -H 'Content-Type: application/json' \
  -H 'X-Client-Source: DISCORD_BOT' \
  -H 'X-Actor-Discord-User-Id: 123456789012345678' \
  -H 'X-Actor-Guild-Id: 987654321098765432' \
  -H 'Idempotency-Key: discord:112233445566778899' \
  -d '{"expectedVersion":1,"serviceCatalogId":"4b4aeef1-e4f0-43cf-9716-87a26a1d7299","unitCount":2,"region":"NA","notes":"Chinese voice communication preferred.","voiceChannelId":null}'
```

Dashboard 的同一工作人员用例使用会话 Cookie；无需也不得在 body 中提交 `staffId` 或 `level`：

```bash
curl -X POST 'https://api.example.invalid/api/v1/admin/staff-tasks/1b354942-391a-4c81-bf17-403e228d1ce9/claim' \
  -H 'Cookie: p0_session=<server-session>' \
  -H 'Content-Type: application/json' \
  -H 'X-Client-Source: DASHBOARD' \
  -H 'Idempotency-Key: dashboard:task-claim:1b354942-391a-4c81-bf17-403e228d1ce9:v1' \
  -d '{"expectedVersion":1}'
```

## 8. 错误与客户端处理

- `401 AUTH_REQUIRED`：Bot Token、Dashboard 会话或必需 Actor Context 无效。
- `403 STAFF_ACCOUNT_REQUIRED/PERMISSION_DENIED`：内部员工账号、权限码、级别、scope 或对象归属不满足。
- `409 CONFLICT/IDEMPOTENCY_CONFLICT`：刷新对象；同一业务意图仍复用原 key。
- `409 CONFIG_VERSION_CONFLICT`：Guild Bot 配置已被他人更新；重新读取、预览并确认，不得以旧 `expectedVersion` 覆盖。
- `409 RESERVATION_CONFLICT/CANCELLATION_PREVIEW_STALE`：刷新预留、对象与取消预览，不得绕过 preview 直接执行。
- `409 REFERRAL_ATTRIBUTION_CONFLICT`：归因版本、唯一来源或互斥计划冲突；仅向有权限的工作人员返回。
- `422 INSUFFICIENT_AVAILABLE_BALANCE`：使用响应中的差额提示充值，不创建订单/礼物预留或部分业务事实。
- `422 SERVICE_NOT_AVAILABLE/PLAYER_NOT_ELIGIBLE/REFERRAL_INELIGIBLE`：不产生部分写入。
- `202 APPROVAL_PENDING`：显示审批编号、所需级别与到期时间，不重复执行原动作。
- `202 ROLE_ELEVATION_PENDING`：保持原有效级别，等待 L4。
- `428 STEP_UP_REQUIRED`：完成近期验证后使用原 Idempotency-Key 重放一次。
- `503/504`：第三方不可用或结果待对账；不得将未知状态显示为成功。

## 9. 第三方适配边界

业务 API 只依赖 `adapter-contract.yaml` 的 11 个标准操作：`discoverCapabilities`、`resolveUser`、`getProviderBalance`、`createHold`、`getHold`、`captureHold`、`releaseHold`、`createReservationDebit`、`createRefund`、`getTransaction`、`verifyWebhook`。优先使用供应商原生 hold/capture/release；供应商不支持时使用本地 `FundReservation`，但 reserve/capture 前必须重新读取第三方余额并原子校验。无论采用哪种后端，任何订单或礼物消费都不得绕过统一 API。业务请求和本地表不出现具体供应商字段，也不保留旧适配器方法别名。

公网 `POST /api/v1/webhooks/payment/{providerKey}` 不要求供应商提交 JSON envelope，也不规定具体签名头名。它接受 `application/octet-stream` 及供应商实际使用的任意媒体类型，request body schema 为 `string/binary`；API 服务器捕获完整 HTTP headers、原始 bytes 与 `receivedAt`，内部编码为 `ProviderWebhookEnvelope` 后调用 04 Adapter 的 `verifyWebhook`。`ProviderWebhookEnvelope` 仅是内部 adapter 入参，不是公网请求 DTO。

验签、时间窗与重放校验必须直接针对未解析、未重新序列化的原始 bytes；只有校验通过后才允许解析业务负载并返回 `DEBIT_UPDATED/REFUND_UPDATED` 规范事件。未知外部引用隔离并告警，不创建无归属财务记录。

## 10. P0 迁移说明

Bot 与 Dashboard 必须共同迁移到 canonical 合同，不在任一客户端保留资金、资格、readiness、推荐或审批规则副本。

| 旧合同或理解 | P0 canonical 合同 |
|---|---|
| `startOrder` / `startOrderService` 由陪玩单方面开始 | 删除普通 start 路由；双方分别调用 `setOrderReadiness`，仅双 READY 自动进入 `IN_SERVICE` |
| 旧 `READINESS_TIMEOUT` | 持久化迁移为 `PLAYER_START_LATE`；超时创建客服任务，不开始计费 |
| `submitOrder` 立即扣款 | submit 创建订单 `FundReservation`；`confirmOrder` 完成时 capture |
| 礼物批准前无资金占用 | `createOrderGiftRequest` 请求时 reserve；批准 capture；取消/拒绝/过期 release |
| 余额只返回 `availableMinor` | 返回 `providerBalanceMinor`、`reservedMinor`、`availableMinor`、`currency`、`fetchedAt` |
| 无 preview 直接取消 | 先 `previewOrderCancellation`，再携带 `previewId` 与 `expectedVersion` 调用 `cancelOrder` |
| `POST /admin/referrals` / `createReferral` 且客户端传 `rateBps` | `POST /admin/referral-attributions` / `createReferralAttribution`；计划经济参数由服务端配置 |
| `/me/commissions` 返回通用 `Commission` | 仅返回当前 actor 是 beneficiary 的 `BeneficiaryCommission`，来源客户脱敏 |
| 任意工作人员读取完整推荐归因 | L2 仅脱敏列表；L3+ confidential detail，读取与修改均审计 |
| 公网 webhook 提交 `ProviderWebhookEnvelope` | 公网提交原始 bytes；服务器捕获 headers，内部构造 envelope 调用 `verifyWebhook` |
| `Commission.reversalOf` / `PlayerEarning.reversalOf` | 主记录不覆盖；冲正追加 `CommissionAdjustment` / `PlayerEarningAdjustment` |

## 11. 校验方式

交付时执行以下检查：

```bash
ruby -ryaml -e 'doc=YAML.safe_load(File.read(ARGV[0]), [], [], true); abort unless doc["openapi"] == "3.1.0"' openapi.yaml
npx --yes @redocly/cli lint openapi.yaml --extends=minimal
```

另执行静态遍历，检查 operationId 唯一、所有 `$ref` 可解析、每个 operation 均有 `security`、`x-required-permissions`、`x-actor-context` 和响应；除 webhook 外所有 POST/PUT/PATCH 均引用 `Idempotency-Key`，并禁止 `/api/v1/bot/*`、`/api/v1/dashboard/*` 平行路径。
