# T11-B · propagation / semantic-conflict enablement 前置设计

> 批次：T11-B design/preflight
> 立项：2026-08-21 22:51:19 +08:00
> 硬事实基线：HEAD=15c7837（T11-A2 已收口，工作树 clean，realm-dev.service active）。
> 本批性质：**只完成产品与架构前置设计，不接生产调用，不新增迁移，不启动 Worker，不改变现有冲突预览语义。**

## 一、目标与完成边界

T11-B 的目标是让 M5 中已经存在的 propagation 引擎、离线 Worker 和 semantic-conflict 评估器具备进入真实运行态的明确入口，但必须先把三个前置一次性定稿：

1. 什么产品动作产生 Campaign 与队列任务；
2. Worker 由什么进程启动、如何领取/恢复/重试/幂等；
3. 语义评估如何受模型预算、超时、失败降级和证据落库约束。

本批只交付上述设计和未来实现的验收门槛。**本批不等价于 propagation/semantic-conflict 已接线。**

### 1.1 本批允许修改

- `docs/development/T11-B-PROPAGATION-ENABLEMENT-PREFLIGHT.md`；
- `docs/README.md` 的索引；
- `STATUS.md` 与 `docs/development/EXPERIENCE-ITERATION.md` 的事实型收口记录。

### 1.2 本批禁止修改

- `app/`、`modules/`、`database/`、`scripts/` 活动代码；
- SQL/migrations、package/lock、systemd 服务；
- `/api/worldline/conflict` 既有 legacy/causal 确定性预览语义；
- T11-A2 的 graph-specific SSE、`record/events` 与 `record/preview`；
- 任何「先接一个调用点再补产品语义」的试接线；
- push。

## 二、当前 HEAD 的硬事实审计

以下结论均以 `15c7837` 的源码为依据，不引用 T10-B21-A 的旧扫描替代当前 HEAD 取证。

### 2.1 Canon 合并是现有最接近产品动作的入口

现有 `/api/canon`：

- `POST { action: "propose" }` 创建待审提案；
- `POST { action: "decide", decision: "merge" }` 由成员确认合并；
- `reject` 与 `defer` 只改变提案状态；
- `merge` 由 `createCanonService().decide()` 读取提案、生成 promoted claims、生成不可变 `CanonRevision`，再通过 `CanonRepository.mergeProposal()` 在一个事务中写入提案状态、Claim、Revision；
- `database/postgres/canon-repository.ts` 当前只写 Canon 表和 T11-A2 图谱失效事件，没有 propagation job 入口。

因此，Canon 合并是一个已经存在、用户可理解、可审计的产品动作。它比新增「传播一下」技术按钮更符合世界治理语义。

### 2.2 propagation 合同存在，但队列与运行态仍不完整

当前已有：

- `modules/propagation/public.ts`：确定性 `realm-propagate-v1` 引擎；
- `modules/propagation/worker.ts`：`createPropagationWorker()`、pending/running/done/failed 状态、stale 恢复、失败重试和懒物化接口；
- `database/postgres/propagation-job-queue.ts`：`propagation_jobs` 的 enqueue/claim/mark/recover/retry/stats；
- `database/postgres/propagation-repository.ts`：Campaign、Packet、Exposure 的结果持久化；
- `0010_world_governance.sql` 与 `0011_worldline_merge_semantic_propagation_jobs.sql`：相关表和 RLS/授权契约。

但当前存在四个必须在实现批修正的接线缺口：

1. `propagation_jobs` 外键要求对应的 `information_campaigns` 已存在，而当前 `enqueue()` 只插入 job，不负责原子创建 Campaign；
2. `claimNext(scope)` 的查询只按 `workspace_id` 领取，未按 `world_id/worldline_id` 隔离，返回的 `PropagationJob` 也不携带其世界作用域；一个世界的 Worker 可能误领取另一个世界的 job；
3. Worker 的 `resolveNode` 与 `routes` 由调用方提供，当前数据库没有持久化的 SocialNode/ChannelRoute 拓扑来源；不能把 `world_relations` 直接冒充通信拓扑；
4. 没有生产启动方。`scripts/dev-server.mjs` 只启动 Vinext，`app/` 与 `modules/application/` 没有 `createPropagationWorker()` 调用。

这些缺口不是本批修复项，但已经被纳入未来实现的硬门槛，避免把现有契约包装成完成态。

### 2.3 semantic-conflict 评估器存在，但未被生产装配

当前 `modules/worldline/semantic-conflict.ts`：

- 只导出评估器、`needsSemanticReview()` 和 evidence 类型；
- 模型调用失败或 JSON 无效时可回落到确定性结论；
- `semantic_conflict_evaluations` 是独立 append-only 证据表，不进入正式 Canon；
- 当前构造参数只有 `getGateway` 与可选 `evidenceStore`，没有语义评估专属预算、并发、超时或证据作用域参数；
- `database/postgres/semantic-conflict-repository.ts` 工厂没有生产调用方。

当前模型网关已有全局 `settings.timeoutMs`，但 `ModelChatRequest` 没有每次调用的 timeout/cancellation 覆盖；未来实现不能只在外层 `Promise.race`，然后让底层 fetch 继续占用模型服务。语义评估必须补齐真正可取消或有效截止的调用边界。

### 2.4 既有冲突预览必须保持确定性

`app/api/worldline/conflict/route.ts` 当前有两个既有分支：

- 无 `mode`：legacy 游标分类；
- `mode="causal"`：从成员作用域读取 Claims/CausalEdges，调用 `detectCausalConflicts()`，只读零写。

T11-B 不把模型评估偷偷塞进这两个分支。否则「预览」会从确定性、可重复、零模型成本的契约变成不可预测的模型调用。

## 三、产品决策

### 3.1 Propagation 的唯一自动触发动作

**成功的 Canon merge 产生 Campaign。**

准确语义：

1. 用户通过已有 `/api/canon` 决定某个待审提案为 `merge`；
2. Canon 事务成功写入 promoted Claim 与 `CanonRevision`；
3. 同一个数据库事务中创建一个对应的 `InformationCampaign`、root Packet 和 `propagation_jobs` pending 行；
4. 事务提交后，独立 Worker 领取并计算 Exposure；
5. Worker 失败只改变 job 状态并记录错误，不回滚或改写已经成立的 Canon。

触发范围：

- `story` 与 `worldline` 两种 Canon target level 均可触发；
- 只有本次真正产生了 promoted Claim 的合并才创建传播任务；若没有新的 Claim，仍可写 CanonRevision，但不创建空 Campaign；
- root Packet 的 Claim 集合只包含本次新晋升的 Claim，不把整个世界历史重复广播。

明确不触发：

- `propose`、`reject`、`defer`；
- 普通 Record 事件、场景晶化、图谱普通编辑；
- 世界线 merge；
- 前端挂载、SSE invalidation、手动刷新；
- 新增一个没有业务语义的 `/api/propagate` 技术按钮。

### 3.2 Campaign 身份与幂等

Campaign 必须由不可变 CanonRevision 派生，不能由 HTTP 请求随机生成：

```text
campaignId = campaign_canon_<revisionId>
jobId      = job_<campaignId>
rootPacket = packet_<campaignId>_root
```

未来实现必须同时满足：

- `(workspace_id, world_id, worldline_id, revision_id)` 到 Campaign 的唯一性；
- 重放同一个 Canon merge 请求不会追加第二个 Campaign 或第二个 job；
- job enqueue、Campaign 创建、root Packet 创建必须在 Canon merge 的同一事务内完成；
- Worker 重试只能重复读取同一个 job，不能重新生成新的 Campaign 身份；
- `information_packets` 与 `propagation_exposures` 继续保持 append-only/幂等写入，算法版本固定为 `realm-propagate-v1`。

现有 `propagation_jobs` 只有 job 主键，未来迁移必须补充按 Campaign 的唯一约束或等价的数据库幂等闸门，不能靠应用层先查再插。

### 3.3 首个实现批的安全等级

当前 `WorldClaim` 没有独立的可见性/保密字段，不能凭 `scope` 或 `truthStatus` 猜测 restricted/secret 语义。因此首个实现批采用明确的窄边界：

- 自动传播只接受 `public` Campaign；
- Canon merge 若无法证明本次 Claim 是 public，**不创建传播 job**，并保留 Canon 正史；
- 不把默认 public 写成「所有未来 Claim 永远公开」的隐含承诺；
- restricted/secret 传播必须在后续批次先完成可见性字段、受众裁决和安全测试，再扩大等级。

这条边界保证模型与传播引擎不会把未经审查的秘密事实扩散到社会节点。

### 3.4 Semantic conflict 的唯一模型触发动作

语义评估采用**用户显式请求的语义复审**，不自动污染普通预览：

- 保留现有 `POST /api/worldline/conflict` 的 legacy/causal 语义不变；
- 未来新增清晰独立的 `POST /api/worldline/conflict/semantic`（或等价独立 route），表示用户明确要求对一次确定性报告进行模型复审；
- 服务器重新从成员作用域读取 Claims/CausalEdges 和 changeSet，不信任客户端提交的事实全集；
- 只有 `needsSemanticReview(deterministicSeverity)` 为真时调用模型；当前即 `high-risk`；`none`/`hard` 不调用，已有确定性结论直接返回；
- 模型建议只产生评估结果和 evidence，不自动 merge、branch、reject，不改变 Worldline、Claim、Canon 或 Record；用户若要继续操作，仍须显式执行原有动作。

这样模型承担「解释与辅助裁决」，不夺取世界线的写入权，也不改变 T10-B2 的 deterministic causal preview。

## 四、Propagation 拓扑：必须有真实来源，禁止语义冒充

### 4.1 设计决策

`world_relations` 是世界知识关系（例如人物与势力的关系），不是通信渠道。不能把每条关系自动变成传播 Route，也不能把所有 Entity 做成完全图来制造「世界在传播」的假象。

未来实现必须引入显式的 `PropagationTopologyProvider`：

```ts
interface PropagationTopologyProvider {
  loadSnapshot(scope: WorldScope): Promise<{
    version: string;
    nodes: readonly SocialNode[];
    routes: readonly ChannelRoute[];
    canonOriginNodeKey: string;
  }>;
}
```

首个实现批应使用正式 PostgreSQL 存储（建议新增 migration 0025，不修改 0001–0024）：

- `propagation_nodes`：`node_key`、`clearance`、所属 world/worldline、启用状态；
- `propagation_routes`：from/to、channel、distance、private letter recipient；
- 两表均 workspace RLS + FORCE RLS + append-only/受控更新契约 + `realm_runtime` 最小授权；
- route 的两端必须属于同一 workspace/world/worldline；
- 拓扑必须能表达一个 Canon origin 虚拟节点，不能让 root Packet 没有合法来源；
- 拓扑修改必须有明确 owner/治理入口或确定性种子来源，不能把测试 fixture 当生产拓扑。

拓扑快照在 enqueue 时物化进 job input，并带 `topologyVersion`：

- 一个 Campaign 的传播结果不因之后修改路线而改变；
- Worker 不在计算中临时读取一半新、一半旧的拓扑；
- 重试读取同一 job input，保证结果可重放；
- 新拓扑只影响之后的新 Campaign。

### 4.2 空拓扑与不完整拓扑

以下情况必须 fail-closed，不得静默生成空的「成功」传播：

- 找不到 Canon origin node；
- route 指向不存在节点；
- clearance/channel/distance 不符合类型约束；
- 拓扑版本无法生成稳定 digest；
- 作用域跨 world/worldline。

此时 Canon merge 事务整体回滚，或者（实现批若采用显式待处理状态）必须有可查询的 `campaign_creation_failed` 运行态记录；不能让 Canon 已提交但用户无法知道传播从未创建。首选方案是**在同一事务内校验并创建 job，校验失败则 Canon merge 一并失败**，因为「Canon 合并后必须进入传播队列」是本批产品动作的不变量。

## 五、Worker 进程拓扑与队列语义

### 5.1 独立进程

Worker 不在 `realm-dev.service`、Vinext route handler 或 SSE 连接内启动。未来实现新增独立入口：

```text
scripts/propagation-worker.mjs
  └─ modules/application/propagation-worker-runtime.ts
       ├─ createLocalPostgresPool(REALM_RUNTIME_DATABASE_URL)
       ├─ PropagationJobQueue
       ├─ PropagationRepository
       └─ PropagationTopologyProvider
```

systemd 形态：

```text
realm-propagation-worker.service
```

约束：

- 与 `realm-dev.service` 分离生命周期；Web 请求不能因为 Worker 卡住而占满请求池；
- 使用 loopback PostgreSQL 连接和 `realm_runtime` 最小权限；
- systemd `Restart=on-failure`，启动时先做健康检查和 stale 恢复；
- 当前阶段只支持单 Worker 实例，通过 PostgreSQL advisory lock 或等价的数据库单例闸门防止重复消费者；
- 不使用全局进程名杀任务，不在验收时手工杀未知进程。

### 5.2 领取作用域必须修正

当前 `claimNext(scope)` 只按 workspace 领取，未来实现必须改为以下二者之一：

- 推荐：`claimNext(workspaceId)` 返回 `{ scope: WorldScope, job: PropagationJob }`，SQL 在领取时返回 job 自带的 world/worldline；
- 或者：Worker 为每个 world/worldline 单独调用严格过滤的 `claimNext(scope)`。

不允许继续使用「调用方传入 scope，但 SQL 只按 workspace 领取」的形态。`markDone`、`markFailed`、`retry`、`saveRun` 必须使用被领取 job 的真实作用域。

### 5.3 生命周期

单例 Worker 的稳定循环：

1. 获取 workspace 级 advisory lock；失败则退出并由 systemd 不重复拉起；
2. 校验 migration ledger、runtime 连接和拓扑表可用；
3. 恢复本实例上次崩溃遗留的 running jobs；
4. 领取最早 pending job（`FOR UPDATE SKIP LOCKED`）；
5. 使用 job 内 immutable input 运行 `realm-propagate-v1`；
6. 在一个结果事务中写入 Campaign/Packets/Exposures（幂等）并标记 done；
7. 失败写入有限长度 `last_error`、attempts 和 failed 状态；
8. 无任务时采用有上限的 idle backoff；收到新任务或达到上限后再次检查。

当前阶段不做多副本横向扩展、不做跨机器队列、不把 Worker 调度塞入开发服务器。未来若需要多副本，必须先增加 lease owner、lease expiry、心跳和可证明的重复执行语义，不能只删除 advisory lock。

### 5.4 失败、重试与幂等

- 拓扑校验、输入 schema、数据库外键错误：job failed，不自动无限重试；
- 临时连接/模型相关错误不属于 propagation 引擎本身，但 Worker 仍只能有限重试；
- 默认最多 3 次尝试，重试间隔必须持久化或由稳定的 `attempts` 计算，不能依赖进程内计时器；
- `done` job 不重新执行；`failed` 只能通过明确 retry 入口重新置 pending；
- Exposure 唯一键与 Campaign/Packet 关系必须能抵御 Worker 在写结果后、标记 done 前崩溃的重放；
- 任何失败都不得更新正式 Canon、Claim 或 Worldline。

## 六、Semantic conflict 模型预算与降级

### 6.1 调用预算

单次显式语义复审的硬预算：

- 最多 **1 次**模型调用；不自动 retry；
- 有效请求超时 **8 秒**，取 `min(语义预算, 当前模型设置 timeoutMs)`；
- 输出上限按适配器真实下限执行（当前 OpenAI-compatible adapter 的有效最小 `max_tokens` 为 1024），不在任务书里写一个适配器不会遵守的 512；
- 输入 Claims 最多 64 条，每个字段在进入 prompt 前按既有长度边界截断；超限直接返回 `SEMANTIC_REVIEW_INPUT_TOO_LARGE`，不调用模型；
- 同一进程同一 workspace 同时最多 1 个语义复审；忙时返回可重试的运行态错误，不排队无限堆积；
- 不把用户配置的普通回合预算挪给语义复审；语义复审必须有独立常量/配置键和日志锚点。

未来实现应扩展 `ModelChatRequest`/OpenAI-compatible adapter 以传递可取消的 per-call deadline，或以等价方式让底层 fetch 在 8 秒时真正终止。仅用外层 `Promise.race` 不满足本规范。

### 6.2 调用前分流

```text
确定性 causal report
  ├─ none / hard       → 不调用模型，返回确定性结果
  └─ high-risk         → 最多一次模型调用
                         ├─ 合法 JSON → source=model
                         ├─ 超时/网关错误 → source=fallback
                         └─ 非法 JSON → source=fallback
```

模型结果必须经过现有 normalize schema：kind、severity、recommendation、rationale 全部合法且 rationale 非空才接受。模型输出不能把 `hard` 静默降成 `none` 并直接改变正式状态；它只是复审结果，正式写入仍由原有用户动作决定。

### 6.3 失败降级

模型失败的降级结果必须完整复制确定性结论：

- `source: "fallback"`；
- `model: "none"`；
- prompt version 与 input digest 仍落证据；
- severity 保持 deterministic severity；
- recommendation 保持当前安全建议（high-risk 默认 `branch`）；
- 不创建 Worldline、不写 Canon、不更新 Claim。

模型成功也只写 evidence，不把自然语言 rationale 当作正史。

### 6.4 Evidence 先于声称

现有 `semantic_conflict_evaluations` 表只有 workspace 作用域和随机 id。未来实现至少需要让 evidence 可审计地关联：

- workspace/world/worldline；
- request/correlation id；
- source/model/prompt version/input digest；
- normalized result；
- created_at。

建议 migration 0025 为表补齐 world/worldline/request 关联，并继续保持 append-only、FORCE RLS 与最小授权。Evidence 写入失败时：

- 不把未持久化的模型结果返回为「已完成复审」；
- 返回安全的 `SEMANTIC_REVIEW_UNAVAILABLE`；
- 不改变任何正式世界状态；
- 记录脱敏本机诊断，不打印 prompt、Claim 全文、密钥或连接串。

## 七、未来实现批的事务边界

### 7.1 Canon merge 与 Campaign enqueue 必须同事务

推荐把当前 Canon merge 的数据库事务扩展成一个显式 application transaction：

```text
BEGIN + realm.workspace_id
  1. 锁定并确认 proposal 仍为 pending
  2. 写 proposal = merged
  3. 写 promoted Claims
  4. 写 CanonRevision
  5. 读取/校验 public propagation topology snapshot
  6. 写 information_campaigns（确定性 campaign id）
  7. 写 root information_packets
  8. 写 propagation_jobs（确定性 job id，pending）
COMMIT
```

任何一步失败，以上全部回滚。`NOTIFY`、Worker 唤醒或 HTTP 响应都不能替代这个事务。

当前 `CanonRepository` 与 `PropagationJobQueue` 各自拥有事务的结构不能直接满足该边界。实现时必须引入可组合的 `PoolClient`/Unit-of-Work 入口，或者由一个明确的 application repository 在同一个 `PoolClient` 内完成全部写入；禁止「Canon commit 成功后 fire-and-forget enqueue」作为最终方案。

### 7.2 Semantic review 不参与 Canon 写事务

语义复审是只读事实读取 + 独立 evidence 写入：

- 不打开 Canon merge 事务等待模型；
- 不持有 worldline 写锁等待模型；
- 不让模型失败阻塞普通 causal preview；
- evidence 事务成功才可声称复审结果已记录；
- 用户后续的 merge/branch/reject 仍走各自既有显式路径。

## 八、未来实现批拆分与提交边界

T11-B 进入实现后，仍按一个批次一个 Kite 任务、一个主题一个提交推进。推荐顺序：

1. **规范/迁移与 schema contract**：拓扑表、Campaign/job 幂等约束、evidence 作用域、最小授权；
2. **Canon 原子入队**：可组合事务、确定性身份、public 边界、回滚零残留；
3. **拓扑 provider 与 Worker**：真实 PG 读取、作用域安全领取、单例锁、stale/重试/崩溃重放；
4. **semantic review**：独立 route、调用前分流、真正 timeout/cancellation、evidence/fallback；
5. **围栏与验收收口**：把 T10-B9-A 的 deferred 负断言翻转为正向调用图契约，补 PG/应用测试、STATUS 和迭代日志。

每一步只 add 指定文件，禁止 `git add -A`、禁止 squash、禁止 push。实现任务若发现 topology、visibility 或 migration grant 不能在同一批形成机器证据，必须停在部分实现，不能以存在 route/worker 文件宣布完成。

## 九、实现批最小验收门槛

### 9.1 传播链

必须有真实机器证据：

1. Canon merge 成功 → 同事务存在恰一个 Campaign、root Packet、pending job；
2. Canon merge 回滚 → 三者均不存在；
3. 同一请求重放 → 不重复创建；
4. proposal reject/defer/普通图谱写 → 不创建传播 job；
5. Worker 从独立进程启动，单例锁生效；
6. 多 world/worldline job 领取不会串作用域；
7. pending→running→done、failed→retry、stale 恢复、Worker 中途崩溃重放均有测试；
8. 拓扑快照版本固定，路线修改不改变既有 Campaign 结果；
9. `realm-propagate-v1` Exposure 逐字节重放一致；
10. 正式 Canon/Claim/Worldline 在传播失败时不被污染。

### 9.2 Semantic review 链

必须有真实机器证据：

1. legacy/causal preview 的既有 response 与零写库语义不回归；
2. none/hard 不调用模型；high-risk 最多调用一次；
3. timeout 真正终止底层请求，并写 fallback evidence；
4. 非法 JSON、网关不可用、evidence 写失败均有明确 fail-closed 证据；
5. evidence 包含正确 world/worldline/request 作用域，且不进入正式 Canon；
6. 模型建议不能直接 merge/branch/reject；
7. workspace/world/worldline 隔离和非成员 404/403 边界成立；
8. prompt/Claim 全文与凭据不进入日志、提交或响应中不必要的字段。

### 9.3 工程与环境

- T10-B9-A `m5-runtime-status` 从负断言同步翻转为正向接线契约；
- focused Core/application/PG/schema/static tests 原始 exit 0；
- 若新增 migration，所有受影响临时库迁移清单和 `realm_schema_migrations` 台账实查；
- GUI/PG 测试结束后按任务前缀清理，独立复核 worlds/accounts/相关新表/`realm_t%`；
- `npm run typecheck`、受影响 eslint、`git diff --check` 原始 exit 0；
- `realm-dev.service` 与 `realm-propagation-worker.service` 状态有时间戳证据；
- 工作树、任务进程、Kite inflight 和数据库临时对象全部收口；
- 未经独立验收不得 push。

## 十、失败矩阵

| 场景 | 正确结果 |
|---|---|
| Canon merge 事务中 Campaign/job 任一步失败 | 整个 Canon merge 回滚，不留半个传播任务 |
| 同一 revision 重放 | 返回/保留同一 Campaign/job，不重复执行 |
| 拓扑缺 origin 或跨 world | fail-closed，不能生成空成功传播 |
| Worker 领取其他 world 的 job | 测试必须失败；实现不得保留 workspace-only claim |
| Worker 在结果写入后、mark done 前崩溃 | 重启后幂等重放，Exposure 不重复 |
| 模型超时/不可用/非法 JSON | fallback evidence，正式世界状态不变 |
| evidence 写入失败 | 不声称复审完成，返回 unavailable，正式状态不变 |
| legacy/causal preview 未请求 semantic | 不调用模型，response 与 T10-B2 保持一致 |
| 非成员请求传播/语义 route | 不泄露世界存在性 |
| SSE/图谱写入 | 不触发 propagation，保持 T11-A2 与 M5 边界 |

## 十一、本批验收与结论

本设计批的验收范围：

- documentation-layout 原始 exit 0；
- 受影响 eslint 原始 exit 0；
- `git diff --check` 原始 exit 0；
- diff 名单仅本规范、docs 索引、STATUS、EXPERIENCE-ITERATION；
- 零数据库写入，零迁移变化；
- `realm-dev.service` 保持 active；
- 工作树 clean。

本批完成后只能声称：

> **T11-B 的产品触发、Worker 拓扑、拓扑数据源、模型预算/超时/降级与证据边界已完成前置设计；propagation/semantic-conflict 仍未进入生产接线。**

只有未来实现批同时满足第九章的传播链、Semantic review 链和环境收口证据，才能撤销 `contract-only-deferred`。
