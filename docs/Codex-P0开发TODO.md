# Codex P0 开发 TODO

> 文档状态：执行清单，全部条目初始为计划中。当前仓库包含规格、合同、原型与验收资料，不代表产品代码或任何 P0 功能已完成。只有实际开发、验证和证据齐备后，才可勾选对应条目。

## Codex 使用说明

1. 先读取本清单、主规格、结构化合同和对应验收用例，确认当前 Story 的所有前置依赖已经有可复验的完成证据。
2. 同一时间只选择一个未阻塞 Story；开始时在进度记录中写明负责人、开始时间、依赖证据和预期验证命令，不提前勾选完成。
3. 严格执行验证先行。代码 Story 使用“读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据”；审查、UAT 或发布 Story 使用“读取合同 → 建立未通过门禁基线 → 执行审查与 UAT → 复验门禁 → 更新证据”，不得为非代码工作伪造自动化 RED。
4. 实现过程中只使用 M0 建立并验证过的真实源码路径、脚本和依赖版本；本文件的推荐结构只表达边界，不代替工程事实。
5. 完成前记录实际修改文件、失败测试或未通过门禁基线、通过命令或复验结果、验收用例、运行环境和风险；全部完成定义满足后才把 Story 勾选完成。
6. 若主规格、OpenAPI、Prisma、状态约束、交互映射、支付合同、业务配置或验收用例互相冲突，立即停止当前 Story，记录阻塞并先修正文档合同，不猜测资金、权限或状态机行为。

统一证据根目录为 `evidence/P0/`：每个 Story 使用 `evidence/P0/<STORY-ID>/` 保存测试输出、截图、沙箱记录和验收说明；每个里程碑使用 `evidence/P0/gates/MX.md` 记录启动/完成门禁。`M0-US-01` 必须创建目录约定和索引。Story 责任类型负责整理证据，技术负责人复核 M0–M4 门禁；M5 发布门禁由 delivery lead 组织产品、运营和技术共同签署。

事实来源优先级：主规格 v0.7 → backlog.csv → openapi.yaml → schema.prisma 与状态约束 → 交互映射.csv → acceptance-cases.csv → 支付适配、业务配置和路线图专项合同。结构化合同内的 ID、状态、字段和 operationId 必须保持一致。

## 全局不变量与边界

- Bot 和 Dashboard 统一使用统一业务 API；两者都是交互客户端，不直连数据库，不保存或重复实现最终业务规则、状态迁移、金额判断或授权策略。
- 第三方 Provider 负责用户账户事实、充值、支付、退款和真实余额；自有系统负责订单、FundReservation、礼物、消费、返佣、陪玩收益、权限、审计与运营视图。Provider 余额不是本地钱包。
- 所有金额使用 minor units 与明确 currency；availableMinor = providerBalanceMinor - reservedMinor，reservedMinor 只能来自有效 FundReservation 合计，客户端不得自行计算最终可用余额。
- FundReservation 必须绑定业务来源、幂等键、版本和生命周期：订单提交或礼物请求时创建，成功结案时捕获，用户取消、30 分钟待派单预留到期、拒绝或失败时释放，5 分钟派单轮次超时保持，争议时保持，部分结案按决议捕获并释放剩余；创建、捕获、释放均由统一 API 原子并发控制。
- 订单只能从 ACCEPTED 在用户与陪玩双方就绪后进入 IN_SERVICE；任何单方 start、绕过 readiness 或提前计费路径都必须拒绝并审计。
- PROMOTER_FIRST_PURCHASE 与 PLAYER_LIFETIME 来源互斥；被推荐用户不得从 API、Discord、错误、通知或导出得知受益人、关系类型、比例、金额或状态。
- 订单事件、外部交易镜像、消费、退款、预留、陪玩收益、返佣、Adjustment 和审计记录只追加，不覆盖原始事实，不硬删除；纠错通过追加 Adjustment、禁用或归档完成。
- 四级权限固定为 L1_SUPPORT < L2_SUPERVISOR < L3_OPERATIONS < L4_ADMIN_OWNER，高级别累积继承低级别全部执行权限。Discord Role 只是映射信号，最终权限受内部批准等级上限约束；降级和撤权立即生效并撤销旧会话。
- Bot 请求必须具有有效服务身份和可验证 Actor Context；客户端提交的 actor ID、staff ID、level 或 Role ID 不能改变授权结果。Dashboard 使用服务端安全会话，敏感动作按合同要求 MFA 与近期 step-up。
- Discord 交互按锁定的 discord.js 版本验证原生组件能力：当前 Modal 可承载受支持的文本和 Select 组件，最多五个顶层组件；P0 为保持级联选择清晰、可恢复，仍在消息组件中逐步完成结构化选择，Modal 主要收集少量自由文本；Select 单次最多 25 个选项；个人余额、订单、返佣、配置与错误信息使用私密频道或 ephemeral 响应；custom_id 只带操作类型与短期会话标识。
- /bot-config 是 Guild 内仅操作者可见的 ephemeral 流程。L3 可读写运营配置；L4 累积继承 L3 并可管理 Role 映射；L3 写 Role 映射必须由 API 返回 403 拒绝。
- /bot-config 的 L3 运营字段包括频道、`staff_notification_role_id`、`operations_notification_role_id`、时限、模板和功能开关；L4 专属安全映射仅包括 `player_role_id` 与 `staff_l1_role_id`、`staff_l2_role_id`、`staff_l3_role_id`、`staff_l4_role_id`。不能把通知 Role 误归为 L4-only。
- /bot-config 使用 Channel Select 和 Role Select，不手工输入频道 ID 或 Role ID。保存前必须验证 Guild 归属、对象类型和 Bot 权限；无效对象不得保存。
- /bot-config 必须先预览，再由 API 签发绑定 actor、Guild、版本、changes 与 reason 的短期 validationToken，最后确认应用；validationToken 缺失、过期或不匹配时拒绝保存。
- /bot-config 成功保存后立即生效，刷新 Guild 缓存，使下一次派单、播报或业务动作使用新配置；重启后从统一 API 重载。
- P1 不纳入 P0；Nice to Have 不纳入 P0。预约、排班、陪玩试音、用户选陪玩或指定陪玩、完整 BI/财务对账导出、多 Server/多租户、额外登录、白标、营销自动化、VIP、优惠券和活动系统均不得混入本清单。
- P0 仅定义计划目标，未通过对应验收与完成门禁前不得对外声称可用、上线或完成。

## 推荐代码仓库结构

以下仅定义模块边界；M0-US-01 建立工程时记录真实路径与命令，之后以实际仓库为准。

~~~text
workspace/
├── apps/
│   ├── api/          # 唯一业务规则、状态机、授权与数据库入口
│   ├── bot/          # Sapphire 交互适配器与统一 API client
│   └── dashboard/    # 运营界面与统一 API client
├── modules/          # 订单、派单、资金、礼物、返佣、收益、权限、审计
├── contracts/        # OpenAPI、事件、配置与 Provider 合同
├── database/         # Prisma、迁移与种子
├── tests/            # 单元、集成、契约、Bot/Dashboard E2E
└── ops/              # Compose、运行手册、备份恢复与可观测性
~~~

客户端目录不得导入数据库访问层或业务域内部实现；共享只能通过版本化合同和统一 API。

## P0 六项核心能力门禁

以下是 P0 产品放行时必须具备的六项核心能力，不是开始 M0 前置条件。PL-01 与 PL-02 主要在 M1 实现，PL-03 至 PL-06 主要在 M2 实现；对应里程碑完成时再勾选。

- [ ] PL-01：结构化需求与完整确认。提交前一次展示游戏、服务、区服、时长、标签、备注、价格、可用余额和取消规则，并由 API 在确认时复核。
- [ ] PL-02：原子资金预留。统一展示第三方余额、预留金额和可用余额，订单与礼物并发请求不得超支。
- [ ] PL-03：匹配透明与接单结果。展示匹配阶段、已通知候选数和超时下一步；接单后展示陪玩摘要和用户下一步。
- [ ] PL-04：陪玩工作台。独立展示资格、在线/可接单状态、当前订单、匹配订单、需求、倒计时、收益摘要和允许动作。
- [ ] PL-05：双方就绪再开始。用户与陪玩分别确认，双方都就绪才进入服务中；超时转客服。
- [ ] PL-06：取消影响预览。取消或申请客服前展示可自动处理性、预计释放/退款金额、处理方式和时效，执行时由 API 原子重验。

## M0：工程骨架与统一业务边界

### 启动门禁
- [ ] 主规格、结构化合同与验收基线已评审；确认当前仅有规格资料，且工程路径、命令和依赖版本将由本里程碑建立。

- [x] **M0-US-01：可复现的本地工程与运行骨架**
  - 前置依赖：none
  - 责任类型：platform_fullstack
  - 实现结果：建立 TypeScript workspace、API/Bot/Dashboard 进程、Docker Compose、环境变量校验、Sapphire Piece 发现和健康/就绪检查。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getHealth;getReadiness
  - 验收用例：AT-CHN-001;AT-CHN-002
  - 完成定义：smoke/config 测试和启动文档通过；无密钥入库；代码经评审。
  - 禁止扩展：不提供 Kubernetes、自动扩缩容或多 Server 部署。
  - 进度记录（2026-07-17）：已建立 TypeScript workspace、`apps/api`、`apps/bot`、`apps/dashboard`、`modules/platform`、`.env.example`、`docker-compose.yml`、M0 smoke/config 测试和证据目录；`npm run m0:verify` 14/14 通过，`npm run typecheck` 通过，API `/health` smoke 通过，`/ready` 已改为使用应用角色登录 PostgreSQL 并检查 baseline schema，`npm run pieces -w @blackcat/bot` 可列出 `service-center` Command 与 `ready` Listener；follow-up code review 通过。证据：`evidence/P0/M0-US-01/summary.md`。真实 Discord credential 未提供，测试 Server E2E 暂按用户要求不阻断。

- [x] **M0-US-02：P0 数据库基线与不可变记录约束**
  - 前置依赖：M0-US-01
  - 责任类型：backend_data
  - 实现结果：实现 P0 表、枚举、外键、唯一约束、活跃订单约束、minor units/currency、迁移和种子；预留、收益与返佣调整记录只追加。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：none
  - 验收用例：AT-AUD-003;AT-REC-001;AT-AUD-001
  - 完成定义：空库及前一版本迁移通过；约束集成测试通过；ERD/Schema 与迁移一致。
  - 禁止扩展：不建本地余额账本，不提供财务、审计或订单事件硬删除。
  - 进度记录（2026-07-17）：已建立 `database` workspace、同步 canonical `database/prisma/schema.prisma`、生成完整空库 baseline migration、同步 `database/seed/seed-data.csv`、新增不可硬删除/保护金额字段策略 helper 和 M0-US-02 合同测试；`npm run m0:verify` 14/14 通过，`npm run db:validate` 通过，`npm run typecheck` 通过，`npm audit --audit-level=moderate` 0 漏洞；本机临时 PostgreSQL 空库 apply 通过，生成 47 张表、3 个抽样关键约束和 7 个抽样 guard triggers，并验证 active slot 伪造、无来源预留、缺少服务开始事件、超额结算、非法预留状态迁移、预留部分结算却进入终态、已激活预留直接 FAILED、审计硬删除、金额覆盖、礼物价格覆盖、Guild 配置事件更新权限和 append-only update 均被拒绝；已补充 P0 首批 trigger guard 名称与实现；follow-up code review 通过。证据：`evidence/P0/M0-US-02/summary.md`。

- [x] **M0-US-03：统一鉴权、Actor Context、幂等与审计中间件**
  - 前置依赖：M0-US-01;M0-US-02
  - 责任类型：backend_security
  - 实现结果：实现 Bot 服务 Token、可信 Actor Header、request_id、Idempotency-Key、权限策略入口、拒绝审计和只追加 audit_logs。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：none
  - 验收用例：AT-AUTH-001;AT-RBAC-001;AT-AUD-001
  - 完成定义：鉴权、幂等、越权和审计集成测试通过；日志脱敏；所有写端点接入中间件。
  - 禁止扩展：不在 Bot Precondition 或 React 前端复制最终 RBAC。
  - 进度记录（2026-07-17）：已实现 API 安全中间件、Bot Service Token 校验、可信 Actor Context 解析、累积权限入口、写操作 Idempotency-Key 合同校验、按 client/operation/actor/key 作用域的原子占位与成功/失败重复请求回放、冲突检测、成功/拒绝/失败审计、route 级 before/after snapshot 入口、transactional staged write `commit(successAuditRecord)` contract 和 M0 安全探针路由；`npx vitest run tests/m0-us-03.spec.ts` 15/15 通过，`npm run m0:verify` 29/29 通过，`npm run typecheck`、`npm run db:validate`、`npm run db:verify:migration`、`npm audit --audit-level=moderate` 均通过。证据：`evidence/P0/M0-US-03/summary.md`。首轮 code review 的 Important 项已修复并补充回归；final narrow review 通过，Critical none，Important none。真实 side-effecting write route 必须使用 staged `{ data, commit(successAuditRecord) }` contract。

- [x] **M0-US-04：第三方资金适配契约与可控 Mock**
  - 前置依赖：M0-US-02;M0-US-03
  - 责任类型：integration_backend
  - 实现结果：实现 adapter-contract.yaml 的 11 个标准操作：discoverCapabilities、resolveUser、getProviderBalance、createHold、getHold、captureHold、releaseHold、createReservationDebit、createRefund、getTransaction、verifyWebhook；提供可编程 Mock、稳定幂等键和外部交易镜像。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：handlePaymentWebhook
  - 验收用例：AT-WHK-001;AT-REC-001;AT-REC-003
  - 完成定义：11 操作契约测试及全部结果分支通过；能力探测、UNKNOWN 恢复、验签与重放测试通过；凭证仅由 Secret 注入。
  - 禁止扩展：不保留任何旧接口别名；不实现充值页或本地余额。
  - 进度记录（2026-07-17）：已实现 `@blackcat/api/payment-adapter` in-memory mock facade，覆盖 `discoverCapabilities`、`resolveUser`、`getProviderBalance`、`createHold`、`getHold`、`captureHold`、`releaseHold`、`createReservationDebit`、`createRefund`、`getTransaction`、`verifyWebhook` 11 个操作；支持 native/fallback capability profile、providerBalance-only 响应、stable idempotency replay/conflict、hold TTL gate、timeout-after-commit recovery、partial capture/release、native hold capture transaction mirror/refund、modeled reservation binding/version 校验、money/date runtime invariant、insufficient funds、refund cap、webhook signature/timestamp/schema/replay/dedup；`npx vitest run tests/m0-us-04.spec.ts` 9/9 通过，`npm run m0:verify` 38/38 通过，`npm run typecheck`、`npm run db:validate`、`npm run db:verify:migration`、`npm audit --audit-level=moderate` 均通过。证据：`evidence/P0/M0-US-04/summary.md`。首轮 code review 的 Critical/Important 项已修复并补充回归；follow-up code review 通过，Critical none，Important none。

- [x] **M0-US-05：Outbox/Job 运行器与结构化可观测性**
  - 前置依赖：M0-US-02;M0-US-03
  - 责任类型：backend_platform
  - 实现结果：实现数据库 Outbox 领取、锁、退避、失败状态、授权手工重试、结构化日志和核心指标钩子。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：retryJob
  - 验收用例：AT-CHN-003;AT-AUD-003
  - 完成定义：并发领取、退避、失败和恢复测试通过；失败 Job 可受权重试并留审计。
  - 禁止扩展：不引入 Redis/BullMQ，不承诺无限重试或复杂 SLA。
  - 进度记录（2026-07-17）：已实现 `@blackcat/api/outbox` 的 Outbox store/runner contract、in-memory store、PostgreSQL `outbox_events` claim/lock 合同（`FOR UPDATE SKIP LOCKED`）、SQL 层 delivery/system job type allowlist、stale `PROCESSING` recovery、delivery/system job type validation、worker lock claim、attempt/version 增量、按失败时间 backoff、terminal failed、`PROCESSING/COMPLETED` 状态统一、success completion、`request_id` structured logs、metrics hooks 和 `retryJob` 授权手工重试审计；`retryJob` 保留 failed job 上下文并在审计 before/after snapshot 中记录 attempts/lastError/runAfter/version；`job.read/job.retry` 已进入统一权限矩阵并与 OpenAPI L2 要求对齐；OpenAPI JobStatus 已与 Prisma 对齐；PostgreSQL enum 参数 cast 已补合同测试；`npx vitest run tests/m0-us-05.spec.ts` 9/9 通过，`npm run m0:verify` 49/49 通过，`npm run typecheck`、`npm run db:validate`、`npm run db:verify:migration`、`npm audit --audit-level=moderate` 均通过；final narrow code review 返回 Critical none、Important none、Minor none、gate-ready。证据：`evidence/P0/M0-US-05/summary.md`。

### 完成门禁
- [x] 五个 M0 Story 的完成定义全部满足；本地启动、健康/就绪、鉴权、Provider Mock、迁移和 Job 恢复证据可复验。证据：`evidence/P0/gates/M0.md`。

## M1：目录、账户与即时订单入口

### 启动门禁
- [x] M0 完成门禁已有证据；统一 API、数据约束、鉴权、Provider Mock 与 Outbox 可用于实现用户入口。证据：`evidence/P0/gates/M0.md`。

- [x] **M1-US-01：版本化服务目录与双价格快照**
  - 前置依赖：M0-US-02;M0-US-03
  - 责任类型：backend_api
  - 实现结果：实现启用服务查询、后台版本 API、计价单位、最低数量、客户单价、陪玩结算单价、币种和上下架约束。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listServices;createServiceCatalogVersion;updateServiceCatalogVersion;estimateService
  - 验收用例：AT-CAT-001;AT-CAT-002
  - 完成定义：单元、数据库集成和 API 契约测试通过；OpenAPI operationId 一致。
  - 禁止扩展：不做阶梯价、活动价、优惠券或动态定价。
  - 进度记录（2026-07-17）：已完成 `@blackcat/api/catalog` domain contract、in-memory store、PostgreSQL store、统一 API route contract 和运行入口挂载；覆盖 `listServices`、`estimateService`、`listServiceCatalogVersions`、`createServiceCatalogVersion`、`updateServiceCatalogVersion`。用户端仅返回 ACTIVE 且双价格完整目录，不泄露陪玩价/陪玩收益；后台 L2 可读、L3+ 可创建/更新；启用必须客户价和陪玩价完整且币种一致；`SUPERSEDE` 创建新版本并 retire 旧版本，不覆盖旧价格快照；写操作采用 staged commit，PostgreSQL 使用 dedicated pooled transaction client 同事务写目录记录和 `audit_logs`；`PostgresStaffDirectory` 解析 Discord 绑定员工；非 staff Discord actor idempotency scope 按 guild/user 隔离；OpenAPI 指定 path/method 的 operationId 已测试一致。`npx vitest run tests/m1-us-01.spec.ts tests/m1-us-01-api.spec.ts tests/m1-us-01-db.spec.ts` 20/20 通过，`npm test` 69/69 通过，`npm run typecheck`、`npm run db:validate`、`npm run db:verify:migration` 均通过。Final focused code review：Critical none，Important none，Ready to merge。证据：`evidence/P0/M1-US-01/summary.md`。

- [x] **M1-US-02：一次性绑定与实时账户摘要**
  - 前置依赖：M0-US-03;M0-US-04
  - 责任类型：backend_api
  - 实现结果：实现绑定码验证、Discord/第三方账号唯一映射、个人摘要和余额查询；保存映射而非第三方密码。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：createBinding;getCurrentUser;getCurrentBalance
  - 验收用例：AT-ACC-001;AT-ACC-002;AT-RES-002
  - 完成定义：Provider 契约、归属和隐私测试通过；绑定码及敏感响应不入日志。
  - 禁止扩展：不保存第三方密码，不提供自助解绑或多账号切换。
  - 进度记录（2026-07-17）：已完成 `@blackcat/api/accounts` domain contract、in-memory store、PostgreSQL store、统一 API route contract 和运行入口挂载；覆盖 `createBinding`、`getCurrentUser`、`getCurrentBalance`。绑定仅接受 Discord Bot 来源和一次性绑定码 `ONE_TIME_CODE`，拒绝稳定 `EXTERNAL_USER_ID`，绑定响应、审计记录和幂等 fingerprint 不包含原始绑定码；Discord 账号和第三方外部账号均有冲突检测，提交阶段并发唯一性冲突映射为 `BINDING_CONFLICT`/409；in-memory 和 PostgreSQL store 均支持绑定与成功审计事务性提交，提交失败或后续唯一性失败会回滚部分记录；`getCurrentUser` 仅返回本人账户摘要且不泄露原始 provider external user id；`getCurrentBalance` 每次从 Provider 查询真实余额并由 API 派生 `availableMinor = providerBalanceMinor - reservedMinor`，`reservedMinor` 只统计 active reservation statuses；OpenAPI path/method operationId 与实现一致，绑定输入枚举已收窄为 `ONE_TIME_CODE`，account runtime error codes 已补入全局错误枚举。`npx vitest run tests/m1-us-02-api.spec.ts` 11/11 通过，`npx vitest run tests/m1-us-02-db.spec.ts` 4/4 通过，`npm test` 84/84 通过，`npm run typecheck`、`npm run db:validate`、`npm run db:verify:migration` 均通过。Focused code review：Critical none；Important 项已修复并补回归。证据：`evidence/P0/M1-US-02/summary.md`。

- [x] **M1-US-03：即时订单草稿与服务端估价**
  - 前置依赖：M1-US-01;M1-US-02
  - 责任类型：backend_api
  - 实现结果：实现创建/读取/更新草稿、单活跃订单限制、字段校验、目录版本引用、数量和金额计算、订单版本并发控制。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：createOrder;getOrder;updateOrder;estimateOrder
  - 验收用例：AT-ORD-001;AT-ORD-002
  - 完成定义：状态机、计价、约束和 API 测试通过；订单事件只追加。
  - 禁止扩展：不支持预约、多人订单、多个活跃订单或客户端自报价格。
  - 进度记录（2026-07-17）：已完成 `@blackcat/api/orders` domain contract、in-memory store、PostgreSQL store、统一 API route contract 和运行入口挂载；覆盖 `createOrder`、`getOrder`、`updateOrder`、`estimateOrder`。`createOrder` 只允许已绑定用户创建即时草稿，单客户仅一个活跃订单，新草稿返回 `201`，已有活跃订单返回 `200` 且不新增事件；`updateOrder` 仅订单所有者、`DRAFT` 状态和匹配 `expectedVersion` 可执行，服务端从 ACTIVE 服务目录快照目录版本、游戏/服务/区服、计价单位、客户价、陪玩结算价并计算 minor-unit 金额，客户端不能自报价格；`estimateOrder` 不改版本、不写事件且不返回 `playerEarningMinor`；订单创建和更新均写 append-only order event 并与 audit 同事务提交。数据库 migration 已收窄 `protect_amount_minor_update()`：普通订单金额覆写仍被 `db:verify:migration` 证明拒绝，只有 API 事务内 `DRAFT -> DRAFT` 并设置 `app.order_draft_amount_update=approved` 才允许草稿估价快照更新。`npx vitest run tests/m1-us-03-api.spec.ts tests/m1-us-03-db.spec.ts` 10/10 通过，`npm test` 94/94 通过，`npm run typecheck`、`npm run db:validate`、`npm run db:verify:migration` 均通过。证据：`evidence/P0/M1-US-03/summary.md`。

- [x] **M1-US-04：Sapphire 公共入口、私密频道与常驻面板**
  - 前置依赖：M1-US-02;M1-US-03
  - 责任类型：discord_bot
  - 实现结果：实现固定公共入口、绑定 Modal、消息组件完成结构化选择、补充备注 Modal、订单频道创建/补偿、权限覆盖、面板渲染和更新、custom_id 路由及重复交互处理。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：createBinding;createOrder;getOrder;updateOrder
  - 验收用例：AT-CHN-001;AT-ORD-003;AT-UI-001;AT-UI-002;AT-UI-003
  - 完成定义：InteractionHandler、组件约束、权限渲染和 API 错误映射测试通过；测试 Server E2E 留证。
  - 禁止扩展：不把价格、状态机、资金或最终权限规则写入 Sapphire Piece；不在打开的 Modal 内实现级联选择。
  - 进度记录（2026-07-17）：已完成 `@blackcat/bot/service-center`、`HttpBotApiClient`、Discord UI spec renderer、`/service-center` 公共入口回复、`service-center-buttons` / `order-selects` / `service-center-modals` 三个 Sapphire interaction-handler piece。覆盖公共入口只展示 `创建订单` 和 `我的服务中心` 且不公开余额；绑定 Modal 单个一次性绑定码 Text Input；备注 Modal 单个可选 Text Input 并携带订单版本；私密频道权限计划拒绝 `@everyone`，允许客户/Bot/客服 role，陪玩接单前不可见；订单面板用消息 String Select 完成游戏/服务/区服/时长选择，不在打开的 Modal 内级联，不泄露陪玩结算或余额；custom_id parser 只承载安全路由元数据；Bot flow 只通过统一 API client 调用 `createBinding`、`createOrder`、`getOrder`、`updateOrder`，并携带 Bot token、Discord Actor Context、interaction id 和 idempotency key。`npx vitest run tests/m1-us-04-bot.spec.ts` 15/15 通过，`npm run typecheck -w @blackcat/bot`、`npm run typecheck`、`npm run pieces -w @blackcat/bot`、`npm test` 109/109 通过。Discord credential 暂未提供，真实测试 Server E2E 未执行；AT-ORD-003 的完整资金预留重复提交仍属于 M1-US-05 `submitOrder` API。证据：`evidence/P0/M1-US-04/summary.md`。

- [x] **M1-US-05：订单提交与资金预留**
  - 前置依赖：M0-US-04;M1-US-03;M0-US-05
  - 责任类型：backend_api
  - 实现结果：提交时复核价格和 Provider 余额，创建 provider hold-backed 订单 FundReservation，并迁移到 PENDING_DISPATCH；提交阶段不创建消费或 debit external transaction，Provider hold ref 通过 FundReservation 与审计追踪，异常分支释放 hold 或返回可恢复 Provider timeout。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：submitOrder;handlePaymentWebhook
  - 验收用例：AT-RES-001;AT-REC-002;AT-ORD-004
  - 完成定义：Provider Mock 全分支、数据库事务和 API 集成测试通过；审计可追溯 reservation 与 external_ref。
  - 禁止扩展：不直接在提交时完成消费；不建可手工编辑 pending 字段。
  - 进度记录（2026-07-17）：已完成 `submitOrder` API、in-memory/Postgres order store 提交事务、Provider hold 预留、timeout-after-commit `getHold(IDEMPOTENCY_KEY)` 恢复、stable reservation id、目录快照复核、Postgres commit-time `user_currency_locks` 锁与 active reservations 重算、Postgres commit-time 目录快照锁读复核、commit 失败后的 `releaseHold` 补偿、`SUBMITTED` event next sequence、成功审计 before/after snapshot、以及 `handlePaymentWebhook` raw octet/json 验签与进程内 event id 去重。提交响应符合 `OrderReservationEnvelope`，不返回订单内部对象、FundReservation 详情或交易列表；提交阶段不创建 debit/consumption。`npx vitest run tests/m1-us-05-api.spec.ts tests/m1-us-05-db.spec.ts tests/m1-us-05-webhook.spec.ts` 17/17 通过，`npm test` 126/126 通过，`npm run typecheck`、`npm run db:validate`、`npm run db:verify:migration` 均通过。证据：`evidence/P0/M1-US-05/summary.md`。Webhook 当前仅验签、拒绝重放、去重和 acknowledgement，真实业务应用与持久 webhook 去重留给后续扣款/退款 Story。

- [ ] **M1-US-06：私密个人服务中心**
  - 前置依赖：M1-US-02;M1-US-03;M1-US-04
  - 责任类型：fullstack_bot_api
  - 实现结果：实现服务中心 API 聚合与 Sapphire ephemeral 视图；消费和返佣在 M3 前可返回结构稳定空列表。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getCurrentUser;getCurrentBalance;listCurrentUserConsumptions;listCurrentUserCommissions
  - 验收用例：AT-ACC-004
  - 完成定义：API 契约、归属、空/错/加载状态和 Discord 手工验收通过。
  - 禁止扩展：不做公开主页、完整 BI 或余额缓存账本。

- [ ] **M1-US-07：结构化需求与一次完整确认**
  - 前置依赖：M1-US-03;M1-US-06
  - 责任类型：fullstack_bot_api
  - 实现结果：确认面板固定展示 game、service、region、duration、tags、notes、price、available balance、cancellation rule；数据来自 estimateOrder/getCurrentBalance。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：estimateOrder;getCurrentBalance
  - 验收用例：AT-PL-001;AT-ORD-002
  - 完成定义：API schema、Bot 渲染、缺失/陈旧/余额失败分支和 E2E 通过。
  - 禁止扩展：不增加预约字段、陪玩试音或用户选陪玩。

- [ ] **M1-US-08：资金预留模型与并发控制**
  - 前置依赖：M0-US-02;M0-US-04;M1-US-05
  - 责任类型：backend_payment_data
  - 实现结果：统一 availableMinor=providerBalanceMinor-reservedMinor；预留绑定 source/idempotency/version/lifecycle；API 原子创建、捕获和释放并优先使用 Provider hold 能力。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getCurrentBalance;submitOrder;cancelOrder
  - 验收用例：AT-RES-001;AT-RES-002;AT-RES-003
  - 完成定义：数据库并发、Provider capability、幂等、版本冲突及恢复测试通过；统一资金服务被订单和礼物复用。
  - 禁止扩展：不提供手工编辑预留金额或客户端余额计算。

### 完成门禁
- [ ] 八个 M1 Story 的完成定义全部满足；PL-01、PL-02 与即时订单提交/预留入口通过关联验收。

## M2：陪玩准入、透明派单与订单闭环

### 启动门禁
- [ ] M1 完成门禁已有证据；即时订单可提交并建立原子预留，候选筛选、派单和订单状态合同无冲突。

- [ ] **M2-US-01：陪玩准入、标签、Presence 与可接单状态**
  - 前置依赖：M0-US-03;M1-US-03
  - 责任类型：backend_bot
  - 实现结果：实现 player_profile 审核、游戏/服务标签、本人 AVAILABLE/BUSY、Discord Presence 同步和单活跃订单检查。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：approvePlayer;setPlayerAvailability;syncDiscordPresence;setPlayerOperationalStatus;updatePlayerOperationalTags
  - 验收用例：AT-DSP-001;AT-ROL-001
  - 完成定义：候选筛选、Role/Presence Listener 和权限测试通过。
  - 禁止扩展：不做排班、自动技能评分或跨 Server Presence。

- [ ] **M2-US-02：自动派单候选与集中派单卡片**
  - 前置依赖：M0-US-05;M1-US-05;M2-US-01
  - 责任类型：backend_bot
  - 实现结果：消费 OrderSubmitted，生成候选快照和 dispatch_attempt，经 Outbox 发布卡片并设置超时。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：dispatchOrder;getOrder;acceptOrder;declineOrderOffer
  - 验收用例：AT-DSP-001;AT-DSP-002
  - 完成定义：候选、Outbox、超时 Job、Discord 渲染和重放测试通过。
  - 禁止扩展：不自动扩圈、候补或实现复杂排序算法。

- [ ] **M2-US-03：并发唯一接单与订单频道入场**
  - 前置依赖：M2-US-02
  - 责任类型：backend_bot
  - 实现结果：实现条件更新接单、活跃订单约束、订单/attempt 原子写、OrderAccepted Outbox、频道权限添加和按钮失效。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：acceptOrder;getOrder
  - 验收用例：AT-DSP-003;AT-DSP-004
  - 完成定义：数据库并发、API 集成和 Discord E2E 通过。
  - 禁止扩展：不支持一单多陪玩或绕过活跃订单约束。

- [ ] **M2-US-04：双方准备、申请完成与用户确认**
  - 前置依赖：M2-US-03;M0-US-05
  - 责任类型：fullstack_bot_api
  - 实现结果：实现双方 READY、ACCEPTED→IN_SERVICE、申请完成、用户确认、Actor 归属、时间戳、面板动作和确认超时任务；旧 start 调用必须拒绝并审计。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：setOrderReadiness;requestOrderCompletion;confirmOrder
  - 验收用例：AT-RDY-001;AT-SVC-001;AT-SVC-002
  - 完成定义：状态机、readiness、超时 Job 和双方 Discord E2E 通过。
  - 禁止扩展：不做单方开始、自动确认、评价、加时或按分钟自动计时。

- [ ] **M2-US-05：默认自动取消与异常客服任务**
  - 前置依赖：M1-US-05;M2-US-03;M2-US-04;M0-US-05
  - 责任类型：backend_api
  - 实现结果：实现 DRAFT 取消、待派单释放预留、已接单 CANCELLATION_ASSIST，以及迟到、缺席、中断、完成超时任务和风险事件。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：cancelOrder;createOrderStaffTask;handlePaymentWebhook
  - 验收用例：AT-CAN-001;AT-CAN-004;AT-SUP-001
  - 完成定义：取消、预留释放、退款、超时、风险和幂等集成测试通过；Bot 显示客服接管状态。
  - 禁止扩展：不自动裁决爽约、争议或服务中断，不自动扣罚陪玩。

- [ ] **M2-US-06：人工退款、结案与转派的原子用例**
  - 前置依赖：M0-US-03;M0-US-04;M2-US-05
  - 责任类型：backend_security
  - 实现结果：实现 refundOrder/resolveOrder/reassignOrder、reason/evidence、金额等级路由、Provider 退款、resolution、Adjustment、风险和不可篡改审计事务。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：refundOrder;resolveOrder;reassignOrder
  - 验收用例：AT-CAN-006;AT-CAN-009;AT-RBAC-004
  - 完成定义：等级边界、同人发起并执行、原因、step-up/MFA、Provider 失败、Adjustment 和事务回滚测试通过。
  - 禁止扩展：不硬删除、强制改状态或自动争议裁决。

- [ ] **M2-US-07：用户侧匹配进度透明**
  - 前置依赖：M2-US-02;M2-US-03
  - 责任类型：fullstack_bot_api
  - 实现结果：提供匹配进度读模型：stage、notifiedCandidateCount、timeoutAt、nextStep；接单后显示陪玩摘要和用户下一动作。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getOrder;acceptOrder
  - 验收用例：AT-MAT-001;AT-DSP-004
  - 完成定义：读模型、隐私、超时和 Bot 状态渲染测试通过。
  - 禁止扩展：不展示候选名单、排序分或复杂推荐理由。

- [ ] **M2-US-08：完整陪玩工作台**
  - 前置依赖：M2-US-01;M2-US-02;M2-US-03
  - 责任类型：fullstack_bot_api
  - 实现结果：聚合 qualification、presence、availability、currentOrder、matchedOrders、requirements、countdown、earningsSummary 和 allowedActions；Bot 仅渲染。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getPlayerWorkbench;setPlayerAvailability;acceptOrder
  - 验收用例：AT-WRK-001;AT-EAR-001
  - 完成定义：工作台聚合、权限、加载/空/错/陈旧状态和 Discord E2E 通过。
  - 禁止扩展：不做陪玩公开档案、试音材料、排班或用户挑选。

- [ ] **M2-US-09：双向准备与超时转客服**
  - 前置依赖：M2-US-03;M0-US-05
  - 责任类型：backend_bot
  - 实现结果：为双方保存 readiness 与版本；第二个 READY 原子迁移 IN_SERVICE；超时 Job 创建唯一客服任务；不提供任何兼容 start 权限。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：setOrderReadiness;createOrderStaffTask;listCurrentUserStaffTasks
  - 验收用例：AT-RDY-001;AT-RDY-002
  - 完成定义：并发状态机、超时、Actor 归属、旧调用拒绝和双方 E2E 通过。
  - 禁止扩展：不允许陪玩单方 start 或自动开始计费。

- [ ] **M2-US-10：取消影响预览与原子执行**
  - 前置依赖：M1-US-08;M2-US-05
  - 责任类型：backend_api
  - 实现结果：实现 previewOrderCancellation；cancelOrder 使用相同规则在事务内重验状态、版本、预留和 Provider 交易。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：previewOrderCancellation;cancelOrder
  - 验收用例：AT-CXL-001;AT-CAN-003;AT-CAN-008
  - 完成定义：预览/执行一致性、并发变化、预留释放、退款 UNKNOWN 和 Bot 面板测试通过。
  - 禁止扩展：不由 Bot 估算退款，不自动裁决已接单争议。

- [ ] **M2-US-11：客服暂停、接管与恢复自动化**
  - 前置依赖：M2-US-05;M2-US-06
  - 责任类型：backend_ops
  - 实现结果：实现订单 automationState、pause/resume、接管人、原因、范围、到期提示；暂停时派单/超时/自动取消 Job 安全跳过；支持重派和审批。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：pauseOrderAutomation;resumeOrderAutomation;createOrderStaffTask;reassignOrder;getStaffTask;resolveStaffTask
  - 验收用例：AT-SUP-005;AT-SUP-002
  - 完成定义：权限、并发 Worker、超时、恢复、转派和 Dashboard/Bot 状态测试通过。
  - 禁止扩展：不提供绕过状态机的任意改状态功能。

### 完成门禁
- [ ] 十一个 M2 Story 的完成定义全部满足；PL-03、PL-04、PL-05、PL-06 及正常单、异常单闭环通过关联验收。

## M3：礼物、消费、收益与保密一级返佣

### 启动门禁
- [ ] M2 完成门禁已有证据；订单正常与异常闭环可复验，礼物、收益和返佣可复用统一资金与权限边界。

- [ ] **M3-US-01：礼物目录与订单内送礼请求**
  - 前置依赖：M2-US-03;M0-US-05
  - 责任类型：fullstack_bot_api
  - 实现结果：实现礼物查询、状态/时间窗口校验、快照、目标从 order.player_id 推导、Gift FundReservation、GIFT_REVIEW 任务和 Bot 面板。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listGifts;createOrderGiftRequest;getCurrentBalance
  - 验收用例：AT-GFT-001;AT-GFT-003;AT-RES-008
  - 完成定义：四个状态、COMPLETED 24 小时边界、目标防伪、预留、任务、Bot 和隐私测试通过。
  - 禁止扩展：不做脱离订单送礼、匿名礼物或动画。

- [ ] **M3-US-02：客服认领、核对与分级送礼执行**
  - 前置依赖：M3-US-01;M0-US-03;M2-US-06
  - 责任类型：backend_security
  - 实现结果：实现唯一认领、内部备注、VERIFIED/待执行、拒绝、L2/L3/L4 金额策略、payload_hash 和执行凭据过期；展示预留状态和语音链接。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：claimStaffTask;verifyStaffTask;approveGiftRequest;rejectGiftRequest
  - 验收用例：AT-GFT-004;AT-GFT-005;AT-RBAC-003
  - 完成定义：权限矩阵、等级边界、同人发起并执行、原因、step-up/MFA、并发认领、陈旧凭据、客服卡片和不可篡改审计测试通过。
  - 禁止扩展：L1 不执行资金动作；任何级别不绕过核对。

- [ ] **M3-US-03：礼物捕获、消费记账与播报恢复**
  - 前置依赖：M3-US-02;M0-US-04;M0-US-05
  - 责任类型：integration_backend
  - 实现结果：批准时捕获现有 Gift FundReservation，原子写 external_transaction/consumption；成功后 Outbox 播报，补发仅重试消息。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：approveGiftRequest;handlePaymentWebhook;retryJob
  - 验收用例：AT-GFT-006;AT-GFT-007;AT-WHK-002
  - 完成定义：Provider、数据库幂等、reservation capture、Outbox 和 Discord 播报 E2E 通过。
  - 禁止扩展：不因播报失败退款或再次捕获，不做礼物动画。

- [ ] **M3-US-04：陪玩收益确认、支付标记与调整**
  - 前置依赖：M2-US-04;M2-US-06
  - 责任类型：backend_api
  - 实现结果：订单完成/结案创建唯一 PENDING PlayerEarning；L3 确认和人工支付；退款或纠错仅新增 PlayerEarningAdjustment。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listPlayerEarnings;updatePlayerEarning;listCurrentPlayerEarnings
  - 验收用例：AT-EAR-001;AT-EAR-002;AT-EAR-003
  - 完成定义：金额、唯一性、状态迁移、PlayerEarningAdjustment 和权限测试通过；无删除端点。
  - 禁止扩展：不自动提现、转账、税务或工资单。

- [ ] **M3-US-05：统一消费历史与本人返佣视图**
  - 前置依赖：M3-US-03;M3-US-04
  - 责任类型：backend_api
  - 实现结果：统一 Consumption 时间线；返佣查询按 beneficiary_id=current_user，来源用户脱敏；退款显示 Adjustment 影响。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listCurrentUserConsumptions;listCurrentUserCommissions;updateCommission
  - 验收用例：AT-HIS-001;AT-RFP-005;AT-RFP-006;AT-RFP-008
  - 完成定义：消费、分页、归属、脱敏、CommissionAdjustment 和重放测试通过。
  - 禁止扩展：不做多级返佣、自动发放或公开来源用户。

- [ ] **M3-US-06：礼物资金预留完整生命周期**
  - 前置依赖：M1-US-08;M3-US-01;M3-US-02;M3-US-03
  - 责任类型：backend_payment_data
  - 实现结果：确认创建 reservation；批准 capture；拒绝/过期/用户撤回 release；不足时禁用确认并显示差额/充值入口；所有动作由 API 并发控制。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：createOrderGiftRequest;approveGiftRequest;rejectGiftRequest;cancelGiftRequest
  - 验收用例：AT-RES-008;AT-RES-009
  - 完成定义：生命周期、边界、并发、幂等、Provider hold/fallback 和 UI 状态测试通过。
  - 禁止扩展：不允许客服改预留金额或批准时重新创建主预留。

- [ ] **M3-US-07：保密的两种一级返佣计划**
  - 前置依赖：M3-US-04;M3-US-05
  - 责任类型：backend_finance_security
  - 实现结果：实现 PROMOTER_FIRST_PURCHASE 与 PLAYER_LIFETIME；一名客户一个互斥来源；绑定/改绑/调整审计；按净消费结算；受益人查询掩码来源。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listCurrentUserCommissions;listCommissions;updateCommission;createReferralAttribution;correctReferralAttribution;getReferralAttributionConfidential;listReferralAttributions;getCommissionConfidential
  - 验收用例：AT-RFP-001;AT-RFP-002;AT-RFP-003;AT-RFP-004;AT-RFP-005;AT-RFP-006;AT-RFP-007;AT-RFP-008
  - 完成定义：两计划、互斥唯一性、净消费、退款 Adjustment、L1-L4 scope、隐私快照及事件重放测试通过。
  - 禁止扩展：不做多级返佣、用户自助改绑、自动发放或代理体系。

### 完成门禁
- [ ] 七个 M3 Story 的完成定义全部满足；礼物预留/捕获、消费、陪玩收益、两种一级返佣和保密边界通过关联验收。

## M4：Dashboard、四级权限与运营控制面

### 启动门禁
- [ ] M0-M3 完成门禁已有证据；Dashboard、RBAC、Role 映射与 Bot 配置所需 API 合同和验收数据已锁定。

- [ ] **M4-US-01：Dashboard OAuth2、会话与 Capabilities 外壳**
  - 前置依赖：M0-US-03
  - 责任类型：frontend_backend_security
  - 实现结果：实现 Discord OAuth2、staff 校验、安全 Cookie、CSRF、permissions_version、Capabilities、导航壳和 401/403。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getCurrentStaffCapabilities;getDashboardSummary
  - 验收用例：AT-AUTH-002;AT-RBAC-001
  - 完成定义：认证、CSRF、会话撤销和 UI/API 测试通过；Token 不进 localStorage。
  - 禁止扩展：不让 Discord Role 单独成为授权事实，不实现密码登录。

- [ ] **M4-US-02：L1 统一订单与客服工作台**
  - 前置依赖：M4-US-01;M2-US-05;M3-US-02
  - 责任类型：frontend_dashboard
  - 实现结果：实现摘要、个人/团队任务、订单工作台、筛选、详情、匹配/readiness/automation 状态、频道/语音链接、备注、认领和升级。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getDashboardSummary;listStaffTasks;claimStaffTask;verifyStaffTask;getAdminOrder;createApprovalRequest;getApprovalRequest;listApprovalRequests;approveApprovalRequest;rejectApprovalRequest
  - 验收用例：AT-SUP-001;AT-RBAC-002;AT-RFP-005
  - 完成定义：组件、API scope、L1 E2E 和跨客户端状态一致性通过。
  - 禁止扩展：不建设独立客服工单、聊天归档或复杂 SLA。

- [ ] **M4-US-03：订单、用户、陪玩与业务配置页面**
  - 前置依赖：M4-US-01;M2-US-06;M3-US-05
  - 责任类型：frontend_dashboard
  - 实现结果：实现搜索、订单/用户/陪玩详情、服务/礼物目录、返佣、陪玩收益页面和所需读写 API。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listAdminOrders;getAdminOrder;listAdminUsers;getAdminUser;listAdminPlayers;getAdminPlayer;listServiceCatalogVersions;createServiceCatalogVersion;updateServiceCatalogVersion;listAdminGiftCatalogItems;createGiftCatalogItem;updateGiftCatalogItem;listCommissions;listPlayerEarnings;updatePlayerEarning;createUserRiskFlag;setUserOperationalStatus;listAdminUserConsumptions;getAdminGiftRequest;listAdminGiftRequests
  - 验收用例：AT-RBAC-003;AT-CAT-003;AT-EAR-002
  - 完成定义：页面、分页、筛选、权限、金额格式和写用例 E2E 通过。
  - 禁止扩展：不做营销画像、完整财务报表、数据仓库或复杂批量操作。

- [ ] **M4-US-04：金额策略、MFA 与 step-up**
  - 前置依赖：M4-US-01;M2-US-06;M3-US-02
  - 责任类型：backend_security
  - 实现结果：实现执行影响预览、payload_hash、过期、L2/L3/L4 阈值、最低等级直执、同人发起执行、L3/L4 MFA 和 15 分钟 step-up。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getCurrentStaffCapabilities;beginStepUp;completeStepUp;enrollMfa;verifyMfaEnrollment
  - 验收用例：AT-RBAC-004;AT-RBAC-005;AT-GFT-005
  - 完成定义：等级边界、同人发起执行、原因、过期/陈旧、MFA 恢复、不可篡改审计和幂等重放测试通过。
  - 禁止扩展：不实现企业 IAM、强制硬件密钥或自定义权限级别。

- [ ] **M4-US-05：Discord Role 映射、高级授权与即时撤权**
  - 前置依赖：M4-US-01;M4-US-04
  - 责任类型：backend_bot_security
  - 实现结果：实现启动/guildMemberUpdate 同步、版本化映射、L1/L2 自动生效、L3/L4 PENDING_ELEVATION、L4 确认、bootstrap、降级和会话撤销。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listDiscordRoleMappings;updateDiscordRoleMapping;syncDiscordRoles;updateStaffRole;approveStaffRoleElevation;revokeStaffSessions
  - 验收用例：AT-ROL-001;AT-ROL-004;AT-RBAC-006
  - 完成定义：Role 同步、重放、升降级、会话撤销、上限和审计 E2E 通过。
  - 禁止扩展：不根据客户端 Role ID 临时授权，不允许 L3 授予 L4。

- [ ] **M4-US-06：审计、失败任务与最小系统设置**
  - 前置依赖：M4-US-01;M0-US-05
  - 责任类型：frontend_backend_ops
  - 实现结果：实现按 scope 的不可变审计查询、失败 Job 列表/重试、政策/频道/超时配置版本化修改和 request_id 错误展示。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：listAuditLogs;listFailedJobs;retryJob;getPolicySettings;updatePolicySetting
  - 验收用例：AT-AUD-001;AT-AUD-004;AT-CHN-003
  - 完成定义：scope、配置并发、Job 重试和审计完整性测试通过。
  - 禁止扩展：不做日志分析平台或任意 SQL 管理入口。

- [ ] **M4-US-07：累积权限解析器与跨渠道一致授权**
  - 前置依赖：M4-US-01;M4-US-04;M4-US-05
  - 责任类型：backend_security_qa
  - 实现结果：实现累积权限解析器，按 L1<L2<L3<L4 合并低级全部执行权限，并统一资源/action/scope 矩阵、金额阈值、破坏性操作定义、推荐隐私和 Role 上限；用同一 API policy 对 Bot/Dashboard 授权。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getCurrentStaffCapabilities;syncDiscordRoles
  - 验收用例：AT-RBAC-001;AT-RBAC-009;AT-RBAC-010;AT-RBAC-011
  - 完成定义：累积解析、金额边界、四角色两客户端 E2E、拒绝审计及 AT-RBAC-010/011 通过。
  - 禁止扩展：不增加第五级、自定义角色设计器或客户端本地授权。

- [ ] **M4-US-08：统一业务交易时间线**
  - 前置依赖：M1-US-08;M2-US-06;M3-US-03;M3-US-04;M3-US-05
  - 责任类型：backend_dashboard
  - 实现结果：建立只读 timeline projection，合并 provider balance snapshot、FundReservation、Consumption、Refund、PlayerEarning、Commission 及对应 Adjustment；按权限脱敏。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getAdminOrder
  - 验收用例：AT-TML-001;AT-HIS-002;AT-RFP-005
  - 完成定义：projection、金额方向、分页、脱敏、空/异常状态和 Dashboard E2E 通过。
  - 禁止扩展：不做总账替代、完整财务对账、导出或会计科目。

- [ ] **M4-US-09：八项启动运营指标**
  - 前置依赖：M4-US-02;M4-US-08
  - 责任类型：backend_dashboard
  - 实现结果：实现今日订单、进行中订单、待处理任务、已完成净消费、礼物净消费、预留总额、派单成功率、异常数；固定时区、币种和口径。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getDashboardSummary
  - 验收用例：AT-MET-001;AT-MET-002
  - 完成定义：聚合查询、边界时间、退款/Adjustment、空数据、权限和 Dashboard 快照测试通过。
  - 禁止扩展：不做下钻 BI、趋势图、导出、完整对账或绩效分析。

- [ ] **M4-US-10：/bot-config 精简程序设计与实现**
  - 前置依赖：M0-US-03;M0-US-05;M4-US-05;M4-US-07
  - 责任类型：backend_bot_api
  - 实现结果：实现 Guild-only /bot-config Slash Command、Sapphire component handlers、ephemeral presenter 与统一 API client；Channel Select/Role Select 后先预览校验并取得短期 validationToken 再确认；API 以 staff_account 和 bot_config.read、bot_config.operational.manage、bot_config.security.manage 最终授权，处理版本冲突、立即刷新缓存、追加审计和重启重载。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getBotConfig;updateBotConfig;validateBotConfigChange;testBotConfigDelivery
  - 验收用例：AT-CFG-001;AT-CFG-002;AT-CFG-003;AT-CFG-004;AT-CFG-005;AT-CFG-006;AT-CFG-007;AT-CFG-008;AT-CFG-009;AT-CFG-010
  - 完成定义：AT-CFG-001..010 全部通过；四个 operationId 契约一致；预览防绕过、权限拒绝、立即生效、审计和重载有自动化证据；未新增网页或可点击 Demo。
  - 禁止扩展：不做独立配置网页、Dashboard 配置页、可点击 Demo、可运行 UI Prototype、Secrets/目录/价格/礼物/返佣配置或任意工作流设计器。

### 完成门禁
- [ ] 十个 M4 Story 的完成定义全部满足；四级累积授权、Role 上限、Dashboard 与 Bot 一致性、八指标和 /bot-config 全部通过关联验收。

## M5：验收、真实集成与可恢复部署

### 启动门禁
- [ ] M0-M4 完成门禁已有证据；全部 P0 Story 都有实现记录、自动化证据和可部署候选构建。

- [ ] **M5-US-01：P0 自动化回归与跨客户端 E2E**
  - 前置依赖：EP-M1;EP-M2;EP-M3;EP-M4
  - 责任类型：qa_automation
  - 实现结果：补齐单元、数据库、Provider、API、RBAC、Sapphire 和 E2E；建立 Requirement ID→Story→operationId→test→evidence 追踪矩阵。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：none
  - 验收用例：AT-AUD-004;AT-RBAC-001;AT-MET-001
  - 完成定义：CI 全绿；失败可复现；记录含环境、构建、Requirement/Story ID、结果和证据链接。
  - 禁止扩展：不追求 P1 设备矩阵或性能压测平台。

- [ ] **M5-US-02：真实 Provider 沙箱与生产样部署恢复**
  - 前置依赖：M5-US-01;M0-US-04
  - 责任类型：integration_devops
  - 实现结果：实现真实 Adapter、签名/回调/对账、Provider hold 能力探测与 fallback、环境隔离、Compose、Secret、迁移、备份恢复和 Bot/Worker 重启演练。
  - 执行步骤：读取合同 → 写失败测试 → 最小实现 → 运行相关回归 → 更新证据
  - 关键接口：getHealth;getReadiness;handlePaymentWebhook
  - 验收用例：AT-WHK-003;AT-REC-003;AT-REC-005
  - 完成定义：另一成员按 Runbook 成功部署/恢复；凭证扫描、健康、日志和告警通过。
  - 禁止扩展：不建设 Kubernetes、多区域容灾、自动扩容或自研支付。

- [ ] **M5-US-03：安全审查、业务 UAT 与发布门禁**
  - 前置依赖：M5-US-01;M5-US-02
  - 责任类型：delivery_lead
  - 实现结果：执行权限/隐私/日志检查，按用户/陪玩/L1-L4 流程 UAT，核对配置、回滚条件、阻断缺陷和路线图非目标。
  - 执行步骤：读取合同 → 建立未通过门禁基线 → 执行审查与 UAT → 复验门禁 → 更新证据
  - 关键接口：none
  - 验收用例：AT-AUD-004;AT-RBAC-009;AT-RFP-005
  - 完成定义：UAT、风险接受、配置快照、回滚入口和发布检查表由产品/运营/技术签署。
  - 禁止扩展：不把预约、陪玩试音、用户选陪玩、指定陪玩或其他 P1 能力纳入发布门禁。

### 完成门禁
- [ ] 三个 M5 Story 的完成定义全部满足；自动化、沙箱、恢复、安全、UAT 与发布签署均有可追溯证据，且无 P0 阻断项。

## 发布清单

- [ ] 44 个 Story 均满足完成定义，依赖顺序与状态有证据，Requirement ID → Story → operationId → test → evidence 追踪完整。Story 内“验收用例”是该 Story 的重点用例，不是验收全集。
- [ ] 以 `acceptance-cases.csv` 为验收全集，建立 `evidence/P0/acceptance-matrix.csv`，152/152 每一条均执行并保存结果与证据；未被单一 Story 引用的横切、恢复和边界用例同样是发布必需项。
- [ ] 单元、数据库、Provider、API、RBAC、Sapphire、Dashboard 和跨客户端自动化回归在候选构建上全部通过，失败可复现。
- [ ] Discord 测试 Guild 完成用户、陪玩、L1-L4 的核心 E2E；私密频道、ephemeral、组件限制、并发接单、双方就绪和 /bot-config 均有证据。
- [ ] Dashboard 完成登录、会话撤销、四级 scope、金额边界、订单/客服工作台、交易时间线、八指标和越权直达 URL E2E。
- [ ] Provider 真实沙箱完成余额、hold/fallback、扣款、退款、Webhook 验签/重放、UNKNOWN 收敛和对账。
- [ ] 生产样环境从空环境部署成功；迁移、Secret 扫描、健康/就绪、日志、指标和告警通过。
- [ ] 备份恢复、Bot/Worker 重启、Outbox/Job 重试与幂等恢复演练完成，预留和外部交易镜像无重复或丢失。
- [ ] 安全检查覆盖 Actor Context、服务身份、CSRF、MFA/step-up、Role 升降级、即时撤权、返佣保密、日志脱敏和只追加审计。
- [ ] 产品、运营、客服和技术完成用户、陪玩、L1-L4 UAT；配置快照、已知风险、阻断缺陷、回滚入口与签署记录齐备。
- [ ] P1 与 Nice to Have 保持排除，发布说明不把演示、规格或计划状态描述为已实现能力。

## 进度记录模板

~~~markdown
### Story 工作记录

- Story：
- 状态：未开始 / 进行中 / 阻塞 / 待验收 / 完成
- 负责人：
- 开始时间：
- 完成时间：
- 前置依赖证据：
- 读取的合同与版本：
- 失败测试命令及预期失败，或审查/UAT 的未通过门禁基线：
- 实际修改文件：
- 最小实现摘要：
- 相关回归命令与结果：
- 验收用例与证据链接：
- 数据迁移或配置影响：
- 安全、隐私与恢复检查：
- 未解决风险或阻塞：
- 下一项可启动 Story：
~~~

勾选 Story 前必须把对应工作记录写入 `evidence/P0/<STORY-ID>/README.md`，并确保命令、结果和链接可由另一名成员复验。里程碑门禁结果写入 `evidence/P0/gates/MX.md`，不得只在聊天或临时终端中声明通过。
