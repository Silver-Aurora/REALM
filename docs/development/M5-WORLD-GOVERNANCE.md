# M5 补全实施规范：世界知识、Canon 治理与社会传播

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 前置文档：[ROADMAP M5](../development/ROADMAP.md)、[总体设计方案](../architecture/SYSTEM-DESIGN.md)（§4 世界线/Canon、§5 冲突、§7 世界记忆、§8 信息传播）、[M3 补全规范](./M3-MEMORY-COMPLETION.md)、[M4 补全规范](./M4-STREAMING-INTERRUPTION.md)。
> 开发边界：全部数据面位于本机 PostgreSQL；情报传播用确定性规则实现，不用 LLM 模拟；不改动 0001–0009 既有迁移。

## 1. 核心原则

1. **Canon Claim、传播中的 Packet、角色 KnowledgeClaim 三者严格分离**：客观事实、传播中的信息、角色认知不混用一张表。
2. **普通 Record 事实不逐条打扰用户**：只有 scope 为 story/world 或真值晋升到 story_canon 以上的高层变化才产生 Canon Proposal。
3. **历史不可变**：已有 Record 与其事件永远不被改写；过去重大变更通过分支而不是改写解决。
4. **传播必须可复现**：同一 Campaign、同一网络、同一算法版本必须产生完全相同的 Exposure 结果；任何“随机”都由内容哈希驱动。
5. 本批次不做两个正式 Worldline 的通用自动合并；不建立跨进程传播 Worker。

## 2. 数据模型（迁移 0010_world_governance.sql）

### 2.1 世界知识图谱

| 表 | 职责 | 关键列 |
|---|---|---|
| `world_entities` | 实体（地理/历史/设定/势力/人物/其他） | `entity_kind`、`name`、`summary`、有效期游标 |
| `world_claims` | 最小事实断言（append-only，可 supersede） | `subject_entity_id`、`predicate`、`object_value`、`scope`(record/story/world)、`truth_status`、`confidence`、`valid_from/to`、来源 |
| `world_relations` | 由 Claim 投影的实体关系 | `subject_entity_id`、`predicate`、`object_entity_id`、`source_claim_id` |
| `world_articles` | 世界观文章，引用覆盖的 Claim 与来源事件 | `title`、`body`、`claim_ids`、`source_event_ids` |
| `causal_edges` | 因果边（Claim/Event 间依赖） | `from_ref`、`to_ref`、`edge_kind`(enables/contradicts/supersedes/context) |

- `truth_status` 晋升阶梯：`mentioned → record_confirmed → story_canon → world_canon`；旁支状态 `rumor / hypothesis / disputed / deprecated`。
- `world_claims` append-only（触发器），修订以 `supersedes_claim_id` 追加新结论；其余表按职责允许受控更新（entities summary 等），RLS 与授权口径同既有表。

### 2.2 Canon 治理

| 表 | 职责 |
|---|---|
| `canon_proposals` | DM 提案：目标层级（story/worldline）、关联 Article 与 Claim 集、状态 `pending / merged / rejected / deferred`、提出者与决定者 |
| `canon_revisions` | 不可变正史版本：`worldline_id`、`parent_revision_id`、`effective_change_cursor`、`accepted_proposal_id`、`content_hash`、`committed_at` |

- 合并提案 → 在同一事务内：晋升其 Claim 至目标层级、写入 Article、生成 CanonRevision。拒绝/暂存不产生任何正史变更。

### 2.3 社会传播

| 表 | 职责 |
|---|---|
| `information_campaigns` | 一次高层事件的传播战役：根 Claim、显著度、复杂度、安全级、`algorithm_version` |
| `information_packets` | 不可变信息包，谱系经 `parent_packet_id`；含 `framing`、`omitted_claim_ids`、`semantic_fidelity_to_parent`、`content_hash` |
| `propagation_exposures` | 传播结果：节点 × Packet × Channel、到达游标、保真度、算法版本 |

传播网络节点与路由不进表（v1 作为引擎入参）：节点含身份/兴趣/智力/位置；路由含渠道、距离、质量与访问要求。

## 3. 确定性传播引擎（modules/propagation/public.ts）

```text
propagate(campaign, rootPackets, nodes, routes)
  → 从根 Packet 沿合法 ChannelRoute 广度传播
  → 每跳：arrivalTick 累加渠道延迟 × 路由距离；fidelity 按渠道基础保真逐跳衰减
  → 转述产生子 Packet（不可变、记谱系）：rumor 渠道按内容哈希确定性省略部分 Claim 并改写 framing
  → 无路由或访问不符的节点不产生 Exposure
```

确定性规则（algorithm_version = `realm-propagate-v1`）：

| Channel | 基础延迟 | 基础保真 | 失真规则 |
|---|---:|---:|---|
| `official_bulletin` | 2 | 0.95 | 不失真，仅公共安全级内容 |
| `private_letter` | 4 | 0.9 | 不失真，仅指定收件节点 |
| `market_rumor` | 1 | 0.6 | 每跳按内容哈希省略 1/3 Claim，framing 改写为 rumor |

- 同一输入重放结果逐字节一致；失真选择由 `fnv1a(packetId + claimId)` 驱动，不使用随机数。
- 引擎只产出角色认知（Exposure），绝不回写 `world_claims`。

## 4. 确定性冲突检测（modules/worldline/conflict-detection.ts）

`CausalChangeSet` = 一组新增/终止/修订的 Claim + 生效游标。检测分三层：

1. **确定性冲突**：人物生死、实体有效期等直接矛盾——变更在过去终止某人物，而存在更晚游标的 Claim 以该人物为活动主体 → `hard`。
2. **依赖图冲突**：`causal_edges` 中依赖被 supersede/终止 Claim 的后继 Claim → 计入依赖冲突并至少 `bridgeable`。
3. **语义分级**：沿用既有 `classifyWorldlineConflict`（`none / bridgeable / hard / high-risk`），过去重大变更默认 `shouldBranch: true`。

未来保护：冲突检测只读不写；Record/事件不可变由既有触发器保证，本批次以契约测试锁定。

## 5. 服务与接口

- `modules/world-knowledge/public.ts`：Entity/Claim/Article/CausalEdge 的创建、晋升、查询端口；`database/postgres/world-knowledge-repository.ts` 实现。
- `modules/worldline/canon.ts`：Proposal 提出（带层级门禁）、决定（merge/reject/defer）、Revision 生成；`database/postgres/canon-repository.ts` 实现。
- `/api/canon`：`GET` 列出待审提案；`POST { action: "propose" | "decide" }`。玩家界面不新增入口。
- 传播引擎纯函数 + `database/postgres/propagation-repository.ts` 持久化结果。

### 5.1 Web 身份与治理授权边界

- HTTP principal 必须来自服务端验证的 session；`decidedBy`、world scope 与角色身份不得由请求体授权。`/api/world-knowledge` 新 Claim 入口只允许初始真值 `mentioned` / `record_confirmed`；`story_canon` 与 `world_canon` 必须通过 Canon 决策流程晋升。
- T11-H 产品语义保持：普通成员可提出与执行 `none/public` Canon 合并；restricted/secret 合并要求当前 world membership 为 `owner`。角色检查不能只依赖 route preflight：repository 必须在同一写事务内重新读取并锁住操作者 membership，覆盖并发降权。
- 创建新世界时 creator membership 为 owner；新账号自动加入共享默认世界时只能获得 player。旧 membership 由 `ON CONFLICT DO NOTHING` 保留；批量修正历史角色需要单独审批与数据迁移，不隐式发生。
- **数据库限制（显式未达成独立操作者授权）**：`0010_world_governance.sql` 对 `realm_runtime` 的治理表写 grants 与 workspace-only RLS 不绑定 session principal。以上授权成立于受信 Web 服务遵守 session/scope/repository gate 的架构内；不能据此声称数据库 credential 本身可抵御任意受信服务端代码绕过。若要隔离该威胁，需单独实现并实测 principal-bound DB capability/control plane；不要用 runtime 可自行设置的 GUC 伪造这一边界。
- **2026-09-28 进展（仅 membership 一条垂直切片，0053）**：`player_world_memberships` 的 runtime 直写（INSERT/UPDATE role）已撤销，写入收口为 `membership_capability_grant` 窄通道；actor 由 **DB 可验证会话证明**推导——应用会话签名密钥（env/安装级 0600 文件）的副本存于 `realm_capability_keys`（purpose='session'，runtime 零表授权），definer 独立重算 HMAC 并从证明取 principal；调用方自报 principal 仅作一致性比对（不符即 CAPABILITY_ACTOR_MISMATCH）。该锚的强度等于登录策略强度：**无密码账户的证明只证明"应用按其策略签发过会话"**——要让 DB 锚覆盖"知道昵称即可登录"的账户，需要产品决策（如 LAN 强制密码），本批不做。
- **会话密钥首写权威（0053 修订）**：runtime 没有任何 key provisioning/bootstrap/轮换入口（无函数、无表授权；ACL 实测钉住）；唯一首写路径是 `scripts/local-provision-capability-session.mjs`（owner/provisioning 连接 + loopback 限定 + 拒绝开发兜底常量；幂等同值重放、分歧 fail-closed 不覆盖），接线在 launcher/两个 scratch runner/`db:postgres:bootstrap` 的 migrate+seed 之后。缺 key 一律 CAPABILITY_SESSION_KEY_MISSING。每 workspace 至多一条 active 会话密钥（部分唯一索引）。开发兜底常量（公知值）不得用于 provisioning。
- 其余治理表（world_claims/canon/audiences/proposals 等）的同类收口**仍未做**；0042 导入 bootstrap（`realm_import_begin_bootstrap`）是显式登记的替代受控入口：owner membership 只写给 job operator（任务台账绑定），pack 不得携带 memberships（函数级禁止），回归见 `tests/realm-transfer-cli.test.ts` 与 `tests/postgres-capability-actor-proof.test.ts`。

## 6. 验收标准

1. 知识图谱五类表可写可查：实体→Claim→Article 引用链完整，Relation 由 Claim 投影生成。
2. 只有高层变化产生 Proposal；record 级事实不产生；merge 生成不可变 CanonRevision 并晋升 Claim；reject 不改任何正史。
3. “过去杀死未来仍存活人物”被检测为 hard 冲突；依赖被改写的后继 Claim 被检测为依赖冲突；过去重大变更默认建议分支。
4. 既有 events 表 UPDATE/DELETE 被拒绝（未来保护）。
5. 官方公告、商队传闻、密信对同一 Campaign 产生不同且逐字节可复现的 Exposure；无路由节点无 Exposure。
6. `npm test` 全绿；`tests/gui/a-library.spec.ts` 与 `b-record.spec.ts`（Chromium）不回归。

## 7. 明确不做（第一批）

- 两个正式 Worldline 的通用自动合并（第一批暂缓，第二批补齐，见 §8）。
- 多尺度 SocialNode 的懒物化与人口群体模拟（第一批节点由调用方给出，第二批补懒物化，见 §10）。
- DM 语义冲突的模型评估（第一批保留分级器接口，第二批补齐，见 §9）。
- 传播 Worker 与离线调度（第二批补齐，见 §10）。
- Canon 审核的玩家界面入口（归入 UI 批次）。

## 8. 通用 Worldline 合并（第二批）

`mergeWorldlines({ sourceA, sourceB, idempotencyKey, operator, dryRun })`：

1. 取两条 Worldline 的全部 Record 与已提交事件，按 `(world_tick, world_ordinal)` 归并排序，同源同游标时 A 先于 B。
2. 冲突分级（确定性规则）：
   - 同游标且载荷一致 → `none`（去重保留其一）；
   - 同游标、同发言者、不同内容 → `hard`（同一主体同一时刻两个矛盾动作，不可自动调和）；
   - 同游标、不同发言者 → `bridgeable`（B 侧事件顺移到下一个空闲 ordinal，只改排序不改内容）。
3. 存在任何 `hard` 冲突 → 拒绝合并，结果携带完整冲突清单；`dryRun: true` 只产出报告不落库。
4. 合并产出：新 merged Worldline（parent 为空、标注来源）、新 Story「合并时间线」、每个来源 Record 对应的 merged Record（`timeline_kind='merged'` + `linked_record_id`），以及不可变的合并清单 `worldline_merges`（来源、操作者、幂等键、冲突报告、manifest）。manifest 为合并后时间线：`[{ ordinal, sourceWorldlineId, sourceRecordId, eventId, resolution }]`，ordinal 从 1 连续且唯一。
5. 幂等：`(workspace_id, idempotency_key)` 唯一；重放返回既有合并结果，不重复创建。
6. 来源 Record 与事件零改写：合并只新增行。

服务接口：`POST /api/worldline/merge`（`dryRun` 预览）。

## 9. DM 语义冲突模型评估（第二批）

`createModelSemanticConflictAssessor({ getGateway })`：

- 只对确定性检测无法判定的候选（分类为 `high-risk` 或无锚点的 `bridgeable`）调用；确定性 `hard`/`none` 不调用模型。
- 输出契约：`{ kind: 'life_state' | 'dependency' | 'worldview' | 'other', severity: ConflictClass, recommendation: 'merge' | 'branch' | 'reject', rationale }`；severity 与既有分级统一。
- 证据单独持久化到 `semantic_conflict_evaluations`（来源、模型、prompt 版本、结果 JSON），不进入 `world_claims` 等正式状态。
- 模型调用失败或输出无效 → 降级为确定性判定结果，evidence 标记 `source: 'fallback'`，不阻塞流程。
- prompt 版本常量 `SEMANTIC_CONFLICT_PROMPT_VERSION = 'semantic-conflict-v1'`。

## 10. 离线传播 Worker 与懒物化（第二批）

- `propagation_jobs` 队列表：Campaign 创建后入队 `pending`；Worker `runOnce` 领取（单进程串行）、计算、持久化 Exposure、标记 `done`；失败记 `failed` 与错误。
- 恢复：`recoverStale()` 把卡在 `running` 的任务重置为 `pending`；进程重启后调用即可续跑；`retry(jobId)` 让 `failed` 重新入队。
- 懒物化：Worker 先从根节点沿路由做可达性遍历，只对实际触达的节点键调用 `resolveNode(key)` 物化 SocialNode；未触达的目录节点不进内存。
- 确定性不变：引擎与算法版本不变（realm-propagate-v1），同一输入重放一致。
- 状态查询：`stats()` 返回各状态计数。

## 11. 第二批验收标准

1. 通用合并：bridgeable 自动顺移、hard 拒绝并给清单、幂等重放返回同一结果、来源零改写、dry-run 不落库。
2. 语义评估：输出与确定性分级统一；证据独立成表；模型失败优雅降级。
3. Worker：pending→running→done 全链路、stale 恢复、失败重试、懒物化只物化触达节点、确定性重放。
4. `npm test` 全绿；`tests/gui/a-library.spec.ts`（Chromium）不回归。
