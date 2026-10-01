# REALM Cleanup Phase 4 执行计划

状态：执行中（Batch 0 可行性审计为 CONDITIONAL，安全默认值已采用）

本计划用于记录模型调用效率、结构化契约、取消恢复和异步尾流的分批实现边界。它不替代 `STATUS.md` 的完成台账，也不把静态推导当成真实 provider benchmark。

## 1. 基线与证据纪律

- 初始审计基线：`0d2aa88b9010de38ce0ce5e99073675fa2781ee1`
- 当前实现基线：`16bde138c619fdebc5a020e8c924c0298220c4da`
- 私有开发仓库是当前 canonical source；本计划不包含凭据、连接串、真实玩家内容或生产数据。
- Kimi Code 每批使用绝对 `cwd=/home/lyle/works/REALM`，同一工作树只允许一个 writer。
- 每批先由 Kimi 复核当前代码和本计划，输出 GO/CONDITIONAL/NO-GO；可行后在本批直接实现。
- Kimi 不负责最终提交；Iris 在 callback、实际 diff、测试和边界检查闭合后按主题提交。
- `accepted`、`busy`、卡片状态、`prompt.completed`、进程存在和单次测试绿灯都不能单独证明完成。
- 逻辑模型调用数、fake gateway 次数和真实 provider HTTP 请求数必须分开记录。

## 2. 不可破坏的边界

以下边界适用于全部后续批次：

1. 不删除 v37 write gate、权限校验或 runtime transaction boundary 来换查询数。
2. 不给完整 `EVENTS_SQL` 直接加 `LIMIT`；长 Record SSE 必须另做授权感知 cursor/keyset 设计。
3. 不在 provider 没有 tool-stream pause 能力时实现 tool-pause。
4. `record_confirmed` growth 不自动升格为 Story/World Canon。
5. 模型不能直接维护未经授权的长期状态；长期状态必须经 schema、服务层、授权和幂等键。
6. 不为减少 token 绕过 recipient、membership、visibility、worldline 或 Record scope 校验。
7. 不修改生产 migration、开发/生产数据库、propagation worker、systemd 或公共服务器，除非另有明确任务授权。
8. 不把 provider secret、API key、连接信息、完整 prompt、玩家原文或生产数据写入代码、报告、日志或提交。

## 3. 已采用的安全默认值

为避免决策闸门无限阻塞，执行流采用以下保守默认：

- Preview/cancel 后续修复目标：Record scope + world membership + viewer 可见性校验。
- ordinary react 暂不注入其他角色的 public outcomes，继续由 Narrator 统一陈述。
- DM 允许真正的零激活回合；是否改变 prefetch 范围先测量，当前不贸然改为 lazy-only。
- SSE 窗口化、viewer cursor、默认 propagation topology、Canon Article 起草、Exposure 进入叙事、causal merge 硬门和 Story thread 不混入模型清理批，分别设计审批。
- 真实 provider 性能与成本结论必须由隔离环境实测；fake gateway 只证明行为和调用阶梯。

## 4. 已完成批次

### Batch 0：计划可行性审计

- 结果：CONDITIONAL。
- 产物：`/tmp/realm-kimi-phase4-feasibility.md`。
- 结论：10 项当前可安全实现，若干项目已完成但文档落后；Preview 授权、零激活补人和长期大件需要明确边界。
- Iris 已核对报告、源码引用、session 事件流和工作树；报告中一个 Exposure 文件路径已校正为 `database/postgres/propagation-exposure-read.ts`。

### Batch 1：结构化契约

- commit：`ebb0900874ff1431a6d77e41fbb488ac1648e3aa`
- 完成：`object[]`、nullable、创世/初夜/晶化 schema 对齐；删除 `respondOnly` 与 tools 混用；新增契约和 fail-closed 测试。
- Iris 验收：focused `15/15`，相关 `107/107`，完整 `npm test` `140/140`，lint 通过，allowlist 与敏感扫描通过。

### Batch 2A：模型 transport 与 stream deadline

- commit：`16bde138c619fdebc5a020e8c924c0298220c4da`
- 完成：classifier/planner/NLG 阶段 policy；JSON 默认 chat；Preview NLG 显式 stream；stream 单次 timeout override 与底层 abort。
- Iris 验收：focused `10/10`，完整 `npm test` `140/140`，lint 通过，事件流包含原回合和后台续跑回合的最终 `busy=false`，未产生越界文件。

## 5. 当前执行批次

### Batch 2B：gateway/settings snapshot、调用观测

目标：减少同一回合重复加载设置和构造 gateway，建立不泄露内容的模型调用观测出口。

允许范围：

- `modules/application/model-settings-service.ts`
- `modules/orchestration/model-powered.ts`
- `modules/inference/types.ts`、必要的 gateway public metadata
- `modules/inference/structured-output.ts` 的现有脱敏日志接线
- 对应 inference/orchestration/model-settings 测试和本计划策略表

必须满足：

- gateway cache 有明确 key、失效规则和并发 single-flight；save/discover/activate 或设置变化不会长期使用旧配置。
- 同一回合可以复用同一 settings/gateway snapshot，但不得以缓存牺牲设置更新、provider 隔离或权限边界。
- 调用观测只允许 stage、provider/model 标识、transport、elapsed、usage 数值、finish reason、provider retry、structured repair 等元数据；禁止 prompt、玩家原文、连接串、密钥和 token 文本。
- 观测失败不能改变回合语义，不能阻塞模型调用。
- retry/repair 差异表要能从代码和测试对应上，不增加重试次数。

放行条件：缓存命中/失效/并发测试、观测脱敏测试、相关套件、完整 `npm test`、lint 和 diff 检查均通过。

**实现与策略表（已独立验收）**：

- gateway snapshot cache（`model-settings-service.ts`）：key = 无 secret fingerprint（providerId/selectedModel/thinking/timeoutMs/maxTokens/updatedAt）；缓存同一 Promise 实现并发 single-flight；save/discover 成功后显式失效；构造失败不缓存；候选（discover/test）直连工厂不进 active cache。边界：绕过服务直接改设置文件可能保留旧 updatedAt，不承诺热更新。
- 调用观测（`modules/inference/model-call-observer.ts` + model-powered 接线）：默认 no-op 的全局注入点 + 有界内存 ledger（ring buffer，默认 256）；事件只含 requestId(opaque)/stage/providerId(unknown)/model/transport/elapsedMs/usage 数值/finishReason/providerAttempts/structuredRepairs/脱敏 failure kinds/outcome/errorCode；observer 抛错被吞掉不影响业务。
- 阶段表（观测 stage 取值）：classifier（visibility/presence gate/DM review/discovery）、planner（DM plan）、previewNlg（narrator/三类 react）、actor（propose）；probe（settings 探针）未接线（settings 服务直连 `requestStructuredObject`，不经 model-powered）——列为已知未覆盖。
- retry/repair 差异表注释落在 `model-powered.ts` 的 MODEL_STAGE_POLICIES 上方，与 `model-call-observer.test.ts` 的计数断言一一对应。

Batch 2B-P0 已实现并独立验收（scene crystallization 观测接线，经 gateway 观测预检 P0）：

- `structured-output.ts` 增加 additive 可选 `observation`（stage/providerId?/transport?）：提供时 requestStructuredObject 在逻辑调用结束 emit 恰好一条 ModelCallObservation（providerAttempts=真实 call 次数、structuredRepairs/failureKinds 与 model-powered 同口径、成功/双失败/provider 错误三终态、脱敏 errorCode 经 `sanitizeObservationErrorCode` allowlist）；`call` 闭包可附加返回 usage/finishReason（可选，缺省记 null）。缺省 observation 时行为完全不变。
- `scene-crystallization.ts`：extract 接 `stage=crystallization-extract`、adjudicate 接 `stage=crystallization-adjudicate`，providerId 固定 `unknown`（provider/profile 归因需 settings 接口暴露，仍是后续 CONDITIONAL）；prompt/schema/messages/温度/调用次数（初次 + 恰好一次 repair）零变化。
- `model-call-observer.ts`：观测错误码只接受既有模型/取消枚举，任意运行时字符串或超长值归一为 `MODEL_PROVIDER_STEP_FAILED`；防止错误字段原样进入观测账本。
- focused：`tests/scene-crystallization.test.ts` 新增 5 用例（TDD RED→GREEN）：错误码净化、正常 success、repair success、双失败 error、adjudicate 独立 stage/provider error 脱敏码/observer 抛错不影响业务 + chat 入参形状不变 + 调用计数不变。
- 验证：独立相关回归 `46/46`；完整 `npm test` exit 0（core `318/318`、contracts 各套件全绿、render `3/3`）；`npx tsc --noEmit` / 完整 eslint / `git diff --check` 均 exit 0。未跑 scratch PG（本批无 SQL 变更），无 migration/DB/服务改动。独立验收期间发现并修复了新 sanitize helper 原样透传任意 runtime `error.code` 的安全契约偏差，新增红绿回归。

Batch 2B-P1 已实现并独立验收（first-night / genesis-chat 观测接线）：

- `structured-output.ts`：`call` 闭包增加可选第二参 `onProviderRequest`——每次 call 调用按至少一次真实请求计（**失败路径也计数**），阶梯/就地重试的额外请求由回调或返回字段 `providerAttempts` 补记；额外请求在 `call` 抛错时也会在 catch 路径补入；对未使用方零变化。
- `first-night.ts` 接 `stage=first-night`（单次初试 + 恰好一次 repair 语义不变，未按旧注释改 3 次）；`genesis-chat.ts` 接 `stage=genesis-chat`，退化阶梯 ②③ 的真实请求经 `onProviderRequest` 补记（阶梯 3 次 + repair 1 次 = attempts 4 的口径有测试锁定）。providerId 固定 `unknown`（provider/profile 归因仍是后续 CONDITIONAL）。
- focused：`tests/model-call-observation-p1.test.ts`（8 用例，TDD RED→GREEN；已注册 `test:core`）：first-night 正常/repair/双失败/provider error，genesis-chat 正常/阶梯 attempts=3/全退化 attempts=4/阶梯中途 provider error attempts=2/provider error 不发起后续请求/observer 抛错不影响业务 + chat 入参形状与内容脱敏断言。
- 验证：独立相关回归 `73/73`；完整 `npm test` exit 0（core `326/326`、contracts 各套件全绿、render `3/3`）；`npx tsc --noEmit` / 完整 eslint / `git diff --check` 均 exit 0。未跑 scratch PG（本批无 SQL 变更），无 migration/DB/服务改动。独立验收期间发现并修复了阶梯额外请求在 provider error 路径漏计的真实缺陷，并新增红绿回归。

Batch 2B-P2 已实现并独立验收（低频管理面观测接线）：

- 四个 stage 独立固定：`genesis-suggestions`（各 step schema/null 语义不变）、`world-genesis`（draft/异常传播不变）、`semantic-conflict`（显式 timeoutMs 与确定性降级 evidence 不变）、`settings-probe`（候选 maxTokens 32/2048 分支、合法 `{"ok":false}` 规则、错误传播、active cache 隔离均不变）。providerId 固定 `unknown`；全部复用 P0 的 `observation` 选项，无新增模型调用。
- focused：`tests/model-call-observation-p2.test.ts`（5 用例，TDD RED→GREEN；已注册 `test:core`）：四 stage 正常 success（调用次数=1）、repair/双失败、provider error 脱敏码（world-genesis 传播、semantic-conflict 降级、settings probe 传播）、请求形状断言（含 semantic-conflict 的 timeoutMs 键与 probe 的 maxTokens=32）、observer 抛错/未配置不影响业务。
- 验证：独立相关回归 `49/49`；完整 `npm test` exit 0（core `331/331`、contracts 各套件全绿、render `3/3`）；`npx tsc --noEmit` / 完整 eslint / `git diff --check` 均 exit 0。未跑 scratch PG（本批无 SQL 变更），无 migration/DB/服务改动。
- 至此预检 U1–U8 全部完成观测接线；provider/profile 归因已由后续方案 A 批次完成；剩余遗留为晶化 abort 透传（留待 self-play 取消语义评估）和 per-record/per-request 归因（需另立隐私决策）。

provider/profile 观测身份线程化已实现（预检方案 A）：

- `types.ts`：`ModelGateway.providerId?: ModelProviderId`（只读可选，catalog 封闭枚举；注释明确无 baseUrl/apiKey/settings 对象、仅用于观测归因、不参与请求构造）。`openai-compatible-gateway.ts` 构造时盖章 `settings.providerId`（active 与 candidate 同源盖章）。
- `model-powered.ts`：callModelJson 与 actor 的 modelCall 闭包捕获当前 gateway 实例的盖章进 4 个 emit 点（不另读 settings、不动 getGateway/cache snapshot）；U1–U8 直连点 observation 传 `gateway.providerId ?? "unknown"`；settings probe 归因候选 gateway/provider。genesis fallbackModel 只换 model，providerId 恒定（测试锁定），providerAttempts 口径不变。
- 兼容：fake/外部实现不盖章 → 观测保持 `unknown`（既有 P0/P1/P2/observer 测试零变化）；观测事件结构不变，仅 providerId 字段从"恒 unknown"变为真实枚举。
- focused：`tests/model-provider-identity.test.ts`（5 用例，TDD RED→GREEN；已注册 `test:core`）：构造盖章（不暴露 settings）、整回合各 stage 身份流过、直连抽样、fallback 阶梯身份恒定 attempts 不变、probe 候选归因 + activate 后 cache 重建身份跟随且旧实例不重标记。
- 验证：identity 及相关观测/缓存/取消回归 `40/40`；完整 `npm test` exit 0（`test:core` `336/336`；`test:contracts` 子命令分别为 `78/78`、`17/17`、`34/34`、`46/46`、`1/1`、`140/140`；render `3/3`）；`npx tsc --noEmit` / 完整 eslint / `git diff --check` 均 exit 0。未跑 scratch PG（本批无 SQL 变更），无 migration/DB/服务改动。真实 provider latency/cost 仍 unknown。

### Provider/profile 观测身份预检（独立只读）

- `providerId` 来自封闭的 `MODEL_PROVIDER_CATALOG`（当前为 `lmstudio` / `openrouter`），不是用户自由文本，不含 URL、API key 或其他 secret；active/candidate gateway 均由对应 settings 构造，cache key 已包含 providerId，save/discover 会失效，不存在 gateway 与身份错位证据。
- genesis-chat 的 fallback 只替换 model，仍使用同一个 gateway/provider；providerAttempts 与 providerId 是独立维度，逻辑调用维持单一 provider 身份即可。settings `test()` / `discover()` 应归因候选 provider，而非 active provider。
- **方案 A：GO**——给 `ModelGateway` 增加只读可选 `providerId?: ModelProviderId`，OpenAI-compatible gateway 用 settings 盖章；model-powered 与 U1–U8 observation 点读取，未盖章 fake gateway 回退 `unknown`。不把 workspace/record、settings 对象、baseUrl 或凭据带进 observer。
- 方案 B（改变 `gateway()` 返回签名）和 C（各调用点重新读取 settings）分别带来 breaking churn 与快照竞态，排除。provider/profile 身份线程化是 additive 小批，不改变 prompt、模型调用次数、retry/repair、缓存或取消语义；真实 provider latency/cost 仍 unknown。


### Scene crystallization abort / async tail 预检（独立只读）

- static + in-memory measured：玩家提交和 self-play beat 完成后均 fire-and-forget 进入 per-record single-flight；执行顺序为 extract → growth 写入 → adjudicate → applyDelta/rejection。提交后 Turn controller 已在 finally 清理，`cancelMessage` 对在途 crystallization 返回 `false`，不能触及尾流；extract/adjudicate 当前没有 AbortSignal，self-play stop 也不触及已提交拍的尾流。
- 真实一致性缺口：single-flight 只串行化，不抑制 stale run。run A 被 run B 挂入 pending 后，A 仍会写 correction/growth/rejection，B 随后可能再写一遍；growth 在 adjudicate 之前，不能只保护最后一个 applyDelta 写点。此次 measured 使用 in-memory/fake，无 DB/真实 provider，逻辑调用计数不等于 HTTP benchmark。
- **P0 方案 B'：GO**——按 `(workspaceId, recordId)` 派生 generation token；pending 覆盖和新调度使旧 generation 失效；在 growth 写入前、adjudicate 前、applyDelta/rejection 前全部检查，过期 run 静默跳过写入，re-arm 只执行最新 generation。provider 请求照常完成，零新增取消面、API、数据库 schema、provider 行为或跨 Record 影响。
- **P1 方案 A：CONDITIONAL**——将 AbortSignal 透传到尾流 gateway 可节省 provider 消耗，但会改变当前 post-commit `cancelMessage=false` 契约，需先决定谁能取消已提交尾流；P2 record-scoped tail controller 当前 NO-GO，依赖该决策。
- 当前按安全默认暂不实现 P1：提交后不取消晶化尾流，`cancelMessage` 保持现有 `false` 语义；generation stale-write suppression 作为现阶段收口。

依赖 Batch 2B 验收。目标是把 memory budget 接入生产 prefetch，按优先级裁剪上下文，并去掉 scene crystallization 的重复事件表示。禁止改变 recall 授权和写入语义。必须增加字符/token 预算、优先级裁剪、超预算和现有场景回归测试。

**实现记录（已独立验收）**：

- prefetch 预算（`modules/memory/public.ts`）：`MEMORY_PREFETCH_TOKEN_BUDGET = 320`——依据 recall limit=6 行、单行常规 ≤160 字符（CJK 约 1 token/字符），6 行常规规模 ≈ 240–320 token；`formatMemoryPrefetchLines` 统一接入生产装配（`local-record-service.ts` 的 `createMemoryPrefetchHub` recall），recall 顺序与 `- ...` 行形保持，超预算按 `fitMemoryLinesWithinBudget` 行内截断语义裁尾部。recall 的数量/关键词/embedding/授权/写入路径零改动。
- extraction 去重（`scene-crystallization.ts` + `local-record-service.ts`）：结构化 `recentDialogue`（speaker/participant/recipient 保留）是 extraction 的 canonical 事件表示；`turnSummary` 降级为 legacy fallback（仅 recentDialogue 缺席/为空时渲染），生产装配不再把 turnSummary 传给 `extract()`；adjudication/rejection 的 turnSummary 语义保持（不同请求，不重复）。
- focused：`tests/context-budget-dedup.test.ts`（6），已注册 `test:core`。
- Iris 独立验收：focused `6/6`，完整 `npm test` `140/140`，build/render 通过，lint 与 `git diff --check` 通过；allowlist 精确匹配且敏感模式 0 命中。

### Batch 2D：true cancellation

单独语义批。目标是把 Turn abort signal 贯通 gateway chat/stream 的底层 fetch，确保取消后没有正式 Event、迟到 Preview 或继续消耗的 provider 请求。必须保持 presence/插话独立 key 的边界；不能把 cancel 变成数据库写入。需要覆盖 provider abort、阶段边界、重试停止和 Preview end aborted。

**实现记录（已独立验收）**：

- gateway（`types.ts` + `openai-compatible-gateway.ts`）：`ModelChatRequest.signal`（仅进程内，body 由 chatBody 逐字段构造不会泄漏）；chat/stream/native 三处把外部 signal 并入 per-call timeout controller（attach + 已 aborted 不发请求 + timer/listener 全清理）。
- runtime（`runtime/public.ts`）：`TurnRuntimeDependencies.signal` 与四类 stage context 的可选 signal；continueTurn 阶段开始检查（accepted 态保留 durable 行、内存态取消终态；其余态统一 markTurnFailure）；阶段失败 catch 归一为 TURN_CANCELLED；retryTurn 在 signal aborted 时拒绝重启。
- model-powered：modelCall/chatWithStream/callModelJson 全链接入（abort 后不 retry、不 fallback、不 repair）；visibility/planner/actor/narrator/reviewer/presence 调用点全部透传。
- local-record-service：Turn controller 提前到 prefetch/可见性裁决之前注册 + finally 统一清理；resolveTurnVisibility 透传 signal；玩家/presence/插话/self-play 各自独立 deps clone（独立 key 互不影响）；Preview hub 的 end 对 aborted 会话强制 aborted（迟到 committed 降级）；translateRuntimeError 的 TURN_CANCELLED 直达分支与 currentFailure 路径同一诊断形态。
- focused：`tests/true-cancellation.test.ts`（12）+ `tests/m4-record-service.test.ts` 两个服务级用例（可见性/规划阶段取消），已注册 `test:core`。
- Iris 独立验收：focused `12/12`，相关回归 `91/91`，完整 `npm test` `140/140`，build/render 通过，lint 与 `git diff --check` 通过；allowlist 精确匹配且敏感模式 0 命中。另以 hanging JSON body 探针确认 response body 读取阶段的 abort 已修复。

## 6. 后续安全实现批次

### Batch 3：回合热路径与异步尾流

Batch 3A 只读预检已完成：fake orchestrator 3 次测量为 `20–23ms`，各阶段 `1L/1P`、无 retry/repair；roster=4 的 prefetch 每回合发起 4 次 recall，`begin=0ms` 且慢 recall 不阻塞 consume；既有 scratch PG 日志记录 `scopeResolve=1`、`projectionEvents=2`、`projectionMeta=3`、`total=214`。真实 provider 延迟未测量。基于测量结果，先做 C1/C3 envelope 四项读取并行化；prefetch 激活角色裁剪与 Narrator/react 并行暂缓，晶化 single-flight 另立批次。

Batch 3C 已实现并独立验收（envelope 四项读取并行化）：

- `local-record-service.ts` 的 `loadConsistentEnvelope`：scope（preResolvedScope 或 resolveRuntimeScope）仍先行——归档/不存在 Record 的 fail-closed（NOT_FOUND/404）语义不变；scope 就绪后 `loadConsistentSnapshot` / `listAuthorizedAffordances` / `loadFirstNightState` / `loadSelfPlayState` 四项以 `Promise.all` 并行（互相无数据依赖）。提交后快照仍单独重读（writeToken 依赖新 canonicalVersion），返回对象字段、firstNight 懒调度副作用、selfPlay failStale→findLatest 与 warn→null、affordances 错误传播全部保持。
- focused：`tests/envelope-parallelism.test.ts`（5 用例，barrier/latch 证明四读取全部开始后才放行 + scope 先序 + 返回契约 + 懒调度/warn→null/错误传播/scope 失败不启动读取），已注册 `test:core`。
- 验证：loopback scratch PostgreSQL 全套 exit 0（140/140），`postgres-turn-efficiency` query counts 与 3A 基线逐项一致（`total=214`，零新增 SQL）；独立 focused `5/5`、相关回归 `76/76`、完整 `npm test` 核心 `140/140` + render `3/3`；`tsc --noEmit` / 完整 eslint / `git diff --check` 通过。
- 已知边界：四项并行后，若 snapshot 抛错（如 WRITE_CONFLICT），firstNight 的 pending 懒调度可能仍被触发一次——该调度本身是幂等自愈重试，语义安全，特此记录。

Batch 3D 已实现并独立验收（per-record scene crystallization single-flight + re-arm）：

- `local-record-service.ts`：原 `scheduleSceneCrystallization` 拆为同步调度器 + `runSceneCrystallization`（原 try/catch 管线主体逐行保留）。调度器沿用 memorySync 的 `inFlight + pending + finally re-arm` 模式：public gate 在入队前（restricted/private 不读 projection、不进 flight/pending）；key 为 `(workspaceId, recordId)`；在途时只保留最新一个 pending（覆盖旧 pending，不改写在途 run 的 input）；run 成功/返回 null/抛错都在 finally 清理并恰好 re-arm 一次；re-arm 重新走 bounded `loadRecentAuthorizedEvents(..., 6)` + public-only filter，不复用旧 run 的 recent events/current/delta/verdict。不同 Record 各自 single-flight，无全局队列；self-play 与玩家回合共用同一 key 空间（来源语义不变）。
- focused：`tests/crystallization-singleflight.test.ts`（5 用例：barrier 证明无重叠 + 恰好一次 re-arm + 最新上下文；extract 抛错仍 re-arm 且 map 不卡死；跨 workspace/Record 独立；restricted 零接触且不留 pending；rejected 语义不变），已注册 `test:core`。
- 验证：loopback scratch PostgreSQL 全套 exit 0（140/140，含 `postgres-scene-crystallization`），`postgres-turn-efficiency` query counts 与 3A 基线逐项一致（`total=214`）；独立 focused `5/5`、相关应用/晶化回归 `61/61`、完整 `npm test` core `297/297` + contracts `140/140` + render `3/3`；`tsc --noEmit` / 完整 eslint / `git diff --check` 通过。零新增 SQL、零新增模型调用类型。

P0 已实现（stale-generation write suppression，经 crystallization abort 预检 P0）：

- `local-record-service.ts`：每个 public schedule 递增 `(workspaceId, recordId)` key 的 generation——在途 run 随之 stale。stale run 的模型调用允许完成（本批不取消 provider），但五个写点前 fail-closed 静默返回：① growth（world entity/claim + profile notes）② adjudicate（不再浪费第二个模型逻辑调用）③ recordRejection ④ applyDelta ⑤ graph inflow。stale 之前已完成的写入不回滚（growth 在 adjudicate 前，粒度如实记录）。pending 保存 input+generation，re-arm 沿用不重新分配；finally 只释放自己的 flight，无 pending 且仍最新时清理 token（map 有界）。token 仅进程内写入资格，不进 prompt/请求/观测/数据库；restricted/private 入队前跳过不变；不同 Record/workspace 互不影响。
- focused：`tests/crystallization-singleflight.test.ts`（8 用例，TDD RED→GREEN）：stale 写点全零 + 恰好一次 re-arm 只写最新；growth 后/adjudicate 前 stale（growth 不回滚、adjudicate/applyDelta 零调用）；applyDelta 前（approved）与 recordRejection 前（rejected）分支；A 在途 B/C 排队只跑 C；跨 Record 互不失效；extract 抛错仍 re-arm 不卡死；restricted 零接触；单次 rejected 语义不变。
- 验证：独立 focused `8/8`；相关 singleflight/scene/growth/local-record/self-play/取消/并行回归 `95/95`；完整 `npm test` exit 0（`test:core` `336/336`；`test:contracts` 子命令分别为 `78/78`、`17/17`、`34/34`、`46/46`、`1/1`、`140/140`；render `3/3`）；`npx tsc --noEmit` / 完整 eslint / `git diff --check` 均 exit 0。未跑 scratch PG（本批无 SQL 变更），无 migration/DB/服务改动。

Batch 3E 预检已独立核对，结论 **NO-GO（本清理线不改 checkpoint 生产写路径）**：

- `run_checkpoint` 是 runtime adapter 的唯一恢复 source of truth；`lockedRun`、`loadTurn`、幂等读取和 `commitRelease` 初读均只解码该列。
- 除 `state` 外，`plan_payload` / `candidate_payload` / `validation_summary` / `release_receipt` / `release_fingerprint` / `completed_record_version` 等重复列暂无运行时 SQL 读取消费者；`state` 仍被删除守卫和 recovery partial index 使用。运维手工查询无文档契约，记为 unknown，不据此擅自停写。
- loopback scratch PG 合成回合实测为 `turn_runs INSERT×1 + UPDATE×9`；重复列存在约 KB 级写放大，但不是当前热路径瓶颈。真实 provider 负载未测量。
- 本批只完成字段消费矩阵与写放大记录，不改 adapter、schema、migration、GRANT 或 ops 契约。可选的“停写三列”降为 **CONDITIONAL-GO**，必须另立批次并取得书面 ops contract；删列/迁移方案不纳入 Phase 4。

### 独立安全批：Preview/cancel Record scope + membership + viewer 授权

已实现并独立验收：

- `LocalRecordService.authorizeRecordViewer(recordId, principalId?)`：只读三步窄授权——① `resolveRuntimeScope`（未知 Record/无席位 → NOT_FOUND）；② delivery projection 的 membership JOIN + viewer 可见性（非成员/无 viewer projection → 同形 NOT_FOUND 安全 404，不泄漏存在性）；③ record head 缺失 → LOCAL_RUNTIME_NOT_INITIALIZED（既有 503 形态）。零副作用：不写 account last-opened、不签发 writeToken、不订阅、不返回 envelope。
- `app/api/record/preview/route.ts` / `app/api/record/messages/cancel/route.ts`：导出 `handlePreviewGet` / `handleCancelPost`（注入式测试同既有 route 模式）；session principal 先过授权再 subscribe/cancel——未通过不创建 SSE body、不发送 preview 事件、不调用 cancelMessage（在途 Turn 不被 abort）。principal 只来自 session；body/query 的 principal/workspace/worldline 伪造字段被结构性忽略。gate 关闭的本地 fallback 保留，但 scope 检查不绕过。
- product 决策：凡该 Record 的已授权 viewer 均可取消其上的在途回合（与 Preview 同一授权面）；本批不做更严格的发起人绑定，记录在案。
- focused：`tests/preview-cancel-auth.test.ts`（13 用例：401/404/503 矩阵、伪造身份、SSE chunk 交付与 abort 清理、cancel 命中/未知 key 404-false、fallback、service 级零副作用证明，以及真实 cancel `POST` wrapper 的 503 回归），已注册 `test:core`；`tests/local-record-application.test.ts` 4 处 service 字面量补接口方法。独立验收期间发现并修复了 cancel wrapper 的错误映射回归：`getLocalRecordService()` 初始化失败仍保持 503，不会被外层 catch 改成 500。
- 验证：独立 focused `13/13`、相关 route/SSE/service/cancellation 回归 `60/60`、完整 `npm test` exit 0（core `310/310`、contracts 各套件全绿、PostgreSQL scratch `140/140`、render `3/3`）；`tsc --noEmit` / 完整 eslint / `git diff --check` 通过。零新增 SQL、零新增模型调用类型。

在 2B–2D 通过后，继续评估尚未闭合项：

- Narrator/react 并行求值，但必须先证明不改变 react 的信息集、recipient/authority、durable candidate 和 Preview 顺序；
- memory prefetch 全 roster 与激活角色范围的隔离实测；
- checkpoint 停写三列：3E 预检结论为 NO-GO；若未来接受失去运维排障列，需书面 ops contract 后另立批次；
- extraction 写入批量化与确定性 Claim 幂等；
- 只在有真实隔离测量时采纳 SQL/事务优化。

### Batch 4：增长时序与长期叙事

Batch 4 预检已独立核对：

- growth Claim/entity 当前生产写入把 `validFromTick` 固定为 `0`；scratch 实测来源 Event 的 `world_tick` 已为正值，但落库仍是 `valid_from_tick=0`。读侧 `recordKnowledge` 的 Record cursor、`valid_from/valid_to` 与 supersede 过滤已经就位，缺陷集中在 producer。
- `profileNotes` 当前是 JSONB 字符串数组，仅做去重、限长和上限淘汰；不覆盖基底 profile 的语义成立，但没有 source event/cursor、创建/生效时间、撤销或生命周期。
- growth 的 producer → durable → authorized read → Character Runner/Narrator prompt 链已闭合；restricted/private 入口结构性跳过，record-local knowledge 与 Canon 分离。真实 provider 行为未测量。
- Story thread、commitment、open clue 全仓没有 schema、service、route 或模型输入消费；按产品设计缺口处理，不在本线顺手实现。
- **4A：GO**——只修 growth Claim/entity 的来源 Event Record cursor 绑定，现有 schema/grants 足够，零 migration；必须覆盖 cursor/branch/幂等/未来不可见回归。
- **4B：CONDITIONAL-GO**——profile note 可先做无 migration 的 JSONB metadata 双读，但长期可审计/撤销需要新表、索引和独立审批。
- **4C：NO-GO**——Story thread/commitment/open clue 另立产品与数据模型设计批。

Batch 4A 已实现并独立验收（growth Claim/entity 绑定来源 Event 的 Record cursor，repo SQL 派生）：

- 设计选择：**repo 内 SQL 派生**（而非调用方瘦读）——`WorldKnowledgeRepository.appendDialogueGrowth` 在写事务内按 `(workspace_id, world_id, worldline_id, record_id, source_event_id)` 读 `events.world_tick` 并校验同 scope；来源不存在/跨 scope 返回 `false` 且零写入（fail-closed，绝不回退 0），entity upsert 与幂等 claims 用同一个派生 cursor 统一盖章。调用方（`applyDialogueGrowth`）只传草稿（draft 类型结构性不含 valid_from/valid_to/source 字段），无法注入假时序；无额外 round-trip（同事务），无 migration/GRANT/RLS 变更。
- `applyDialogueGrowth`：`sourceEventId` 缺失（self-play 空 release 等）→ warn + 跳过世界 growth（原行为是写 tick=0 的假时序 claim）；`appendDialogueGrowth` 返回 false → warn + 零写入；玩家回合来源改为只从 `run.release.eventIds` 选择真实已提交事件（优先 narration/player，绝不拼接猜测 ID）。其余 best-effort/restricted/private/public-only/确定性 ID/ON CONFLICT 幂等语义不变；profile note 路径不动。旧 `valid_from_tick=0` 数据不迁移、不回写。
- 测试：内存 stub 同契约更新 + 主 growth 用例补 entity/claim 同 cursor 断言 + 新增 fail-closed 用例（unknown source 零写入 + 可诊断 warn + 回合不受阻）；scratch PG `postgres-world-growth` 补 tick 绑定（claim/entity == 来源事件 world_tick>0）、future claim cursor 前不可见、repo 级 unknown/cross-record/cross-worldline/cross-workspace fail-closed 零写入、repo 级幂等重放单行同 cursor。独立验收另补了 release eventIds 取值偏差修正。
- 验证：独立 focused 相关测试 `12/12`（本地无 DATABASE_URL 的 PostgreSQL 单测 skip 1 项）；loopback scratch 全套 `140/140`；完整 `npm test` exit 0（core `311/311`、contracts 各套件全绿、render `3/3`）；`tsc --noEmit` / 完整 eslint / `git diff --check` 通过。零 migration/GRANT/RLS 变更。

Batch 4B 已实现并独立验收（profile note provenance/lifecycle，无 migration JSONB 双读）：

- entry 形状固定为 `{note, sourceRecordId, sourceEventId, validFromTick, validToTick, createdAt, revokedAt}`。provenance 不信任调用方：`appendProfileNotes(scope, notes, {sourceEventId})` 在同一写事务内按完整 scope 五元组校验已提交事件并用 DB 侧 `world_tick`/`recorded_at` 盖章；来源缺失/跨 scope 返回 `false` 零写入（fail-closed，与世界 growth 同一边界）。零 migration/新表/GRANT/RLS 变更；写入仍走 `gateWorldWrite + FOR UPDATE` + active roster。
- 双读共存：`mergeProfileNotes`/`normalizeProfileNoteEntry`/`profileNoteTextAt` 纯函数——旧字符串原样保留（不迁移形状），malformed object fail-closed 丢弃，按 note 文本跨形状精确去重，160 字限长与 8 条上限丢最旧不变。
- 读取侧（`record-scope.ts`）：`combineProfileSummary` 接收本 Record head 的 `effectiveTick`（绝不用 global worldline head），object entry 只渲染当前有效的 note 文本——revoked/过期（validToTick <= cursor）/未来（validFromTick > cursor）不渲染；metadata（source IDs/cursor/timestamps/revocation）绝不进入 prompt。cursor 不可用时保守只过滤 revokedAt。
- `applyDialogueGrowth`：profile note 与世界 growth 同一 fail-closed 边界（sourceEventId 缺失 warn 跳过；false warn 零写入）。旧字符串数据不回写。
- 测试：内存新增双读/生命周期/去重/畸形单元用例与 note fail-closed 断言；scratch PG 补 object entry 落库 metadata（validFromTick == 来源 Event world_tick、createdAt 来自 DB recorded_at）与混合数组渲染边界（legacy/当前有效可读，revoked/过期/未来/畸形不可读，metadata 不泄漏进 profileSummary）。本批不新增 revoke API（`revokedAt`/`validToTick` 读取过滤已实现并测试）；temporal 过滤已用 Record cursor 实现，无遗留 TODO。

### Supersede / 共存预检（独立只读，阻塞产品决策）

- 同一 `(workspace, world, worldline, record, subject entity, predicate)` 的不同 growth value 当前按 value 纳入 deterministic claim ID，允许两条有效 claim 共存；scratch 实测两条都进入下一回合 `recordKnowledge`。growth 没有自动 supersede，也没有 functional/single-valued predicate 分类。
- `supersedes_claim_id` 目前只由可信 promotion/canon control path 产生；growth 与模型 extraction 恒为 `null`。append-only 与同 world/worldline 外键保证正常写路径不能跨 scope 改写，但自引用在数据库层可插入并会自我隐藏，属于现有约束缺口。
- 读侧 future/expired/current superseder 的 tick 边界已实测正确；但 `NOT EXISTS` 没有约束 superseder 的 `truth_status` 或 `scope`，低阶梯 `mentioned` superseder 也能隐藏 `record_confirmed` claim。该行为影响 canon/record 两段，不能在本 Phase 默默改变。
- **当前结论：CONDITIONAL，不实现。** 需要产品决定：①哪些 predicate single-valued；②新 value 是否隐藏旧 value；③supersede 只限同 Record 还是允许跨 Record；④superseder 的 truth/scope 是否必须不低于被隐藏 claim。候选方案为 A 保持共存、B 服务层明确白名单 supersede、C 新 schema/审查流；truth/scope 读侧修正是正交的独立行为变更。

### ModelGateway 观测预检（独立只读）

- 全仓生产 ModelGateway 调用静态盘点为 15 处：7 类已通过 model-powered wrapper 观测；8 处仍直连 `gateway.chat` + `requestStructuredObject`，包括 scene crystallization extract/adjudicate、first-night、genesis-chat、genesis-suggestions、world-genesis、semantic-conflict 和 settings `test()`。CLI/scripts 无生产模型调用。真实 provider 延迟、成本和 HTTP 请求数均未测量，保持 unknown。
- 8 处未观测点均不在普通 Turn 同步主链；晶化 extract/adjudicate 是每公共回合 1–2 次的异步尾流主体，其他调用属于初夜、创世或管理面。first-night 注释写“3 次”与实际单次+1 repair 不一致，暂不扩大为行为修改。
- observer 当前失败终态、脱敏错误类别、ring buffer（默认 256）和取消计数语义已核对合格；但 4 个现有 emit 点的 `providerId` 硬编码为 `unknown`，无法按 provider/profile 归因。providerId 线程化依赖 settings 层接口调整，单独标记 CONDITIONAL。
- **最小下一批：GO（纯观测接线）**。优先接入晶化 extract/adjudicate（P0），再接 first-night/genesis-chat（P1）及其余管理面（P2）；使用现有 `requestStructuredObject` additive 观测参数，不增加模型调用、不改变 prompt/schema、retry/repair 上限或取消语义。providerId 线程化和晶化 abort 透传不纳入本批。

4A/4B/4C 之外，继续保留以下边界：

- growth Claim 的 validFrom 绑定来源 Event cursor；
- profile note 增加来源与最小生命周期，不覆盖基底 profile；
- outbox/lease/retry/head 竞争需要单独设计审批，不在本批直接改 migration 或 worker；
- Story thread、commitment、open clue 只允许通过 schema 化服务层落账，模型不能自由维护。

## 7. 单独审批的大件

以下项目不因本计划的安全默认值自动获得实现授权：

- Preview/cancel 完整 Record scope 授权实现（目标已确定，仍需独立安全 focused batch）；
- `turn_runs` 重复列停写或 schema/migration 变更（3E 仅 CONDITIONAL-GO，需 ops contract）；
- SSE keyset/viewer cursor 重设计；
- C16 可恢复 outbox、lease、幂等和 Record head 竞争；
- 普通世界 propagation topology 的业务语义；
- Canon merge Article 的起草和审核流程；
- Exposure 的普通叙事消费者及其 clearance 授权；
- causal conflict 是否成为 merge 硬门；
- 多人邀请、membership、operator、配额/费用治理。

这些大件的设计报告必须给出 producer → durable state → authorization → consumer 的跨边界证据、失败恢复、锁序和回滚条件，再单独派实现批。

## 8. 每批 callback 与验收协议

1. 派单前：记录当前 HEAD、工作树、绝对 cwd、allowlist、禁止面和 callback 状态。
2. Kimi：先复核计划，再实现可行部分；最终回调列出实际文件、diff、测试原始 exit code、未完成事项和潜在回归。
3. Iris：读取 callback 相关报告和实际文件；检查 session 最后主回合、后台任务、`busy=false`、是否有后续续跑。
4. Iris：重新运行 focused、相关回归、完整 `npm test`、lint、typecheck/build（按批次需要），并检查未跟踪文件、秘密和 diff whitespace。
5. 只有独立验收闭合才 stage 明确路径、提交主题 commit；提交后读回 SHA 和工作树。
6. 不 push；公共发布按 `OPEN-SOURCE-RELEASE-PLAN.md` 的独立 staging/allowlist 流程执行。

## 9. 完成定义

本 Cleanup Phase 4 代码流只有在以下条件全部满足时才可称为完成：

- Batch 1、2A、2B、2C、2D、3 的已批准安全子项均有 callback、实际 diff、主机回归和主题 commit；
- 真实取消、上下文预算和模型观测均有结构化测试与失败路径证据；
- 没有未说明的越界文件、敏感配置或后台任务；
- 大件和公网多人治理明确标记为独立计划/阻塞，不伪装成已完成；
- 最终报告区分 implemented、independently verified、structural-only、blocked 和 committed。

未完成的产品决策或公共服务器部署不能被伪称为本地 cleanup 已完成；同样，本地 cleanup 的测试通过不能推出真实 provider benchmark 或公网多人安全已经成立。
