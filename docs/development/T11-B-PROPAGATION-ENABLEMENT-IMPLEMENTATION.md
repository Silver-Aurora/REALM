# T11-B · propagation + semantic-conflict enablement 实现规范

> 批次：T11-B implementation（前置设计 docs/development/T11-B-PROPAGATION-ENABLEMENT-PREFLIGHT.md 的唯一实现批；用户已明确要求真实实现）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=e435bf9。本规范只记录前置设计未钉死的**具体绑定决策**；产品/架构语义以 preflight 为准，冲突处回读 preflight。

## 一、public 边界的具体形态：merge 显式 attest

preflight §3.1「成功 merge 产生 Campaign」与 §3.3「无法证明 public 则不创建 job」的绑定：

- `POST /api/canon` `decision=merge` 接受可选字段 `propagate: "public"`——
  这是当前 schema 下**唯一**的 public 证明（用户对本次合并的显式公开
  传播裁决）；缺省/缺字段 = 无法证明 public = fail-closed 不创建传播任务，
  Canon 正史照常成立（§3.3 明文允许）。
- `propagate: "public"` 且本次 merge 有 ≥1 条 promoted Claim → 同一事务
  创建 Campaign/root Packet/pending job；拓扑校验失败 → **整个 merge
  回滚**（preflight §4.2 首选方案）。
- `propagate: "public"` 但 promoted Claims 为空 → 只写 CanonRevision，
  不创建空 Campaign（§3.1），merge 正常成功。
- `reject`/`defer`/`propose`/普通图谱写/晶化/worldline merge 一律不触发。

## 二、迁移 0025（0025_propagation_topology_semantic_scope.sql）

1. `propagation_nodes`：workspace/world/worldline + `node_key` +
   `clearance`（CHECK public/restricted/secret）+ `active boolean`；
   PK (workspace_id, world_id, worldline_id, node_key)；workspace 与
   worldlines 复合 FK；FORCE RLS + realm_workspace_isolation；
   **realm_runtime 仅 SELECT**（拓扑治理写入走 owner 种子，见 §三）。
2. `propagation_routes`：同作用域 + `id` + `from_node`/`to_node` +
   `channel` CHECK + `distance numeric` + `recipient text`（private_letter
   必填 recipient 的 CHECK）；双端点复合 FK 到 propagation_nodes 同作用域
   键（数据库级保证两端同 workspace/world/worldline）；FORCE RLS；
   realm_runtime 仅 SELECT。
3. Canon origin 虚拟节点：约定 `node_key = 'canon_origin'`（clearance
   public 的活跃节点），provider 校验其存在。
4. `information_campaigns` ADD `canon_revision_id text`（可空）+ 部分唯一
   索引 `(workspace_id, world_id, worldline_id, canon_revision_id) WHERE
   canon_revision_id IS NOT NULL`——Campaign↔Revision 幂等的数据库闸门。
5. `propagation_jobs` ADD 唯一索引 `(workspace_id, world_id, worldline_id,
   campaign_id)`——job↔campaign 幂等闸门（不依赖应用层先查再插）。
6. `semantic_conflict_evaluations` ADD `world_id text`、`worldline_id
   text`、`request_id text`（三列可空，兼容既有行）；append-only 守卫与
   FORCE RLS 不动。

## 三、拓扑治理来源

- 生产/开发拓扑写入入口 = owner 通道的稳定种子：
  `database/postgres/demo-seed.ts` 的 `seedPostgresDemo` 增补 demo 世界
  拓扑（canon_origin + 两个社会节点 + 两条路线，幂等 ON CONFLICT）；
  治理写入口审计 = 种子脚本本身（postgres-seed-demo.mjs 要求 migration
  owner 权限）。无测试 fixture 充当生产拓扑。
- realm_runtime（路由快照读取 + Worker）只读。

## 四、Canon 原子 enqueue 绑定

- Core `CanonRepository.mergeProposal` input 增加可选
  `propagation?: { enqueue(client: unknown): Promise<void> }`——opaque
  client 由 PG 实现在同一事务内传递 PoolClient；抛出即整事务回滚
  （promoted claims/revision/campaign/packet/job 全部回滚）。
- `createCanonService` options 增加可选 `propagation` 端口（拓扑快照
  在事务内加载校验 → 确定性身份 → 写入）。`decide` input 增加
  `propagatePublic?: boolean`。
- 确定性身份：`campaign_canon_<revisionId>` / `job_<campaignId>` /
  `packet_<campaignId>_root`；root packet channel=official_bulletin、
  framing=neutral、claimIds=本次 promoted、contentHash 与引擎
  packetHash 同构（sha256 id:claimIds:omitted:framing）。
- Campaign：securityClass='public'、effectiveTick=revision.effectiveTick、
  salience/complexity 用表默认 0.5、canon_revision_id=revision.id。
- 重放安全：proposal 非 pending → 409 零写入（既有守卫）；同 revision
  二次创建撞唯一索引 → 事务回滚报错（不会产生第二个 Campaign）。
- T11-A2 图谱失效事件（canon_merge）保留不变。

## 五、队列与 Worker 绑定

- `PropagationJobQueue.claimNext(workspaceId: string)` 返回
  `{ scope: WorldScope, job } | null`——SQL 领取时返回 job 真实
  world/worldline；claimable 条件含 attempts 退避
  `finished_at + LEAST(POWER(2, attempts), 60) * INTERVAL '1 second' <= now()`
  （attempts 持久化，重试间隔稳定）。markDone/markFailed/retry/stats 用
  被领取 job 的真实 scope；recoverStale(workspaceId)。
- `PropagationJobInput` 扩展 `nodes` + `topologyVersion`（immutable
  快照）；Worker 不再向调用方要 resolveNode——从快照懒物化可达节点。
- 队列新增 `completeRun(scope, jobId, run)`：Campaign/Packets/Exposures
  幂等插入 + job 标记 done **同一事务**——结果写完未标 done 的崩溃窗口
  消失；崩溃于事务前则 stale 恢复后重放，幂等插入不重复。
- `modules/application/propagation-worker-runtime.ts`：专用 client 持
  advisory lock（pg_try_advisory_lock(hashtext('realm_propagation_worker'))，
  抢不到即退出）；启动健康检查（migration 台账含 0025、拓扑表可读）；
  启动时逐 workspace recoverStale；主循环逐 workspace claimNext（取
  workspaces 表枚举）；临时错误（连接类）attempts<3 自动 failed→pending，
  拓扑/schema/FK 类永久错误留 failed；idle backoff 1s→15s 上限；
  stop() 供测试与 SIGTERM。
- `scripts/propagation-worker.mjs`：env-file 装载 + loopback runtime URL
  校验 + SIGTERM/SIGINT 优雅退出。systemd：`scripts/systemd/
  realm-propagation-worker.service` 仓库模板（Restart=on-failure、
  依赖 PostgreSQL、与 realm-dev 分离生命周期）；**不安装、不改用户
  unit**——安装待 Iris 验收。

## 六、Semantic review 绑定

- 新路由 `app/api/worldline/conflict/semantic/route.ts`（POST）：
  与 causal 分支同源的作用域/changeSet 严格解析（复用同形态校验，
  worldId 必填 400、非成员 404）；服务端重读 Claims/CausalEdges；
  detectCausalConflicts 确定性报告先行。
- 分流：`classification.conflict` 为 none/hard → 直接返回确定性结论
  （`semantic: null`，不调模型不落 evidence）；high-risk → 进程内
  per-workspace 单飞闸门（忙 409 SEMANTIC_REVIEW_BUSY 可重试）→
  最多 1 次模型调用。
- 真实 deadline：`ModelChatRequest` 增加 `timeoutMs?`，OpenAI 兼容
  adapter 的 requestJson 生效 `min(request.timeoutMs, settings.timeoutMs)`
  并驱动底层 AbortController（chat 路径；既有行为默认不变）；评估器
  增加 `timeoutMs` 选项，语义复审常量 8000ms。
- 输入边界：Claims > 64 → 400 SEMANTIC_REVIEW_INPUT_TOO_LARGE 不调模型；
  changeSet 解析与 causal 同构（字段截断同 causal 分支）。
- evidence：`SemanticConflictEvidence` 增加可选 `scope`（workspace/world/
  worldline）与 `requestId`；PG store 写入新列。evidence 落库失败 →
  503 SEMANTIC_REVIEW_UNAVAILABLE，不返回复审结果，正式状态不变。
- 模型输出合法 JSON + normalize 全字段合法才 source=model；超时/网关
  错误/非法 JSON → source=fallback（assessor 既有降级形态）；
  recommendation 只是 evidence，绝不 merge/branch/reject。
- 既有 `/api/worldline/conflict` legacy/causal 零改动（回归测试守）。
- 日志：沿用 route-observability 白名单脱敏；不写 prompt/Claim 全文。

## 七、围栏翻转（m5-runtime-status）

负断言 → 正向接线契约：四符号的允许调用点白名单恰为
canon 路由（propagation enqueue 经 service 端口）、
propagation-worker-runtime/scripts（worker）、semantic route（assessor +
needsSemanticReview）；conflict 路由保持零模型/零 semantic-conflict；
其余生产面维持零接线。

## 八、失败矩阵（增补 preflight §十之外）

| 场景 | 正确结果 |
|---|---|
| merge+attest 但拓扑缺 origin | 整个 merge 回滚，409/500 安全文案，Canon 未成立 |
| merge 无 attest | Canon 成立，零 Campaign/job |
| 同 proposal 重复 merge | 409 PROPOSAL_NOT_PENDING，零写入 |
| Worker 与 Web 同进程 | 禁止——worker 仅独立脚本入口 |
| evidence 落库失败 | 503 unavailable，不返回复审结果 |

## 九、交付步骤

1. 本规范 + docs 索引；2. 迁移 0025 + topology provider + schema 契约；
3. Canon 原子 enqueue + route/PG 测试；4. 队列 scope 修正 + Worker +
   脚本/模板 + 测试；5. semantic route + deadline + evidence + 测试；
6. 围栏翻转 + 验收收口。
