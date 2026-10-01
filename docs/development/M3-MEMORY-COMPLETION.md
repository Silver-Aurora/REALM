# M3 补全实施规范：关系状态、分域摘要与记忆快照

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 前置文档：[ROADMAP M3](../development/ROADMAP.md)、[模型与记忆运行时](../architecture/MODEL-AND-MEMORY-RUNTIME.md)、[M2 实施规范](./M2-ORCHESTRATION-AND-TOOLS.md)。
> 开发边界：全部数据面位于本机 PostgreSQL；不引入新的外部供应商；不改动 0001–0008 既有迁移。

## 1. 核心原则

M3 已有混合召回与跨 Record 继承。本批次补全四个剩余交付，且不推翻既有正确性地基：

1. Relationship 是**主观认知状态**，不属于客观世界事实；客观世界关系属于 M5 世界图谱，本批次不建立。
2. 记忆结论（`memory_conclusions`）保持 append-only 证据属性；Relationship 状态是**可修订的派生投影**，每次修订必须回写一条证据结论。
3. 分域摘要只是 Summary 结论的 `domain` 维度，复用既有四作用域与安全过滤，不新建召回通道。
4. Snapshot 不可变、Delta 只增量、Cache Epoch 只在历史解释被改写时递进；与 Context Ledger 的衔接停留在“提供 epoch 字符串与快照游标”，不改 `modules/context` 的既有契约。

## 2. 数据模型（迁移 0009_memory_m3_completion.sql）

### 2.1 relationship_states（主观关系状态）

| 列 | 含义 |
|---|---|
| `workspace_id / world_id / worldline_id` | 既有作用域三元组 |
| `observer_continuity_id` | 持有该关系认知的角色连续体 |
| `target_entity_key` | 关系指向的对象键 |
| `relation_kind` | `trust / hostile / alliance / kinship / debt / acquaintance / note` |
| `content` | 当前主观关系描述文本 |
| `fidelity` | 保真度 0–1 |
| `source_record_id / source_observation_id` | 最近一次修订的来源（可空） |
| `occurred_* / available_from_*` | 发生游标与可用游标 |
| `created_at / updated_at` | 创建与最近修订时间 |

- 主键 `(workspace_id, observer_continuity_id, target_entity_key, relation_kind)`：同一观察者对同一对象的同类关系只有一条当前状态。
- 该表允许 UPSERT（`updated_at` 递进），是记忆中唯一可变的派生投影；其余记忆表继续 append-only。
- RLS、`realm_runtime` 授权与既有记忆表一致。

### 2.2 memory_snapshots（不可变快照）

| 列 | 含义 |
|---|---|
| `observer_continuity_id` | 快照所属连续体 |
| `snapshot_kind` | `representation / recall` |
| `content` | 冻结文本 |
| `item_ids text[]` | 入选结论的不可变清单 |
| `cursor_tick / cursor_ordinal` | 拍摄时的有效知识游标 |
| `cache_epoch` | 拍摄时的缓存代数 |
| `token_count` | 估算 Token 数 |
| `created_at` | 拍摄时间 |

- append-only 触发器与 `memory_conclusions` 同款；任何修改必须拍摄新快照。

### 2.3 memory_cache_epochs（缓存代数）

- 主键 `(workspace_id, observer_continuity_id)`，单调整数 `epoch`。
- `add` 结论不递进 epoch（增量由 Delta 覆盖）；`update / retract` 结论递进 epoch（历史解释被改写，旧快照失效）。
- Context 编译器需要记忆缓存代际时读取该值作为 `cacheEpoch` 的来源之一；本批次不改 `ContextRequest` 契约。

## 3. 服务契约（modules/memory/public.ts 扩展）

```text
recordRelationship(input, relationKind, targetKey, content, fidelity?)
  → UPSERT relationship_states
  → 同步 append 一条 memory_kind=relationship 证据结论

relationships(input)
  → 优先读 relationship_states（观察者主观状态）
  → 无状态时回退到既有 observed_entity_key 推导线索
  → 输出必须保持主观措辞，不得表述为客观世界关系

summarizeDomain(input, domain, scope?)
  → domain ∈ episodic / semantic / decision
  → 分别使用“事件经过 / 稳定事实 / 目标承诺”召回意图
  → 持久化为 kind=summary 结论，metadata 携带 summaryScope + summaryDomain

snapshot(input, kind?) → { snapshotId, cacheEpoch, content, itemIds, cursor }
  → 以当前有效游标拍摄不可变召回/表示快照

delta(input, snapshotId)
  → epoch 一致：返回快照游标之后新增的合法结论（增量）
  → epoch 不一致：返回 stale，调用方必须重新拍摄

representation(input, depth, tokenBudget?)
  → 在既有三档 limit 之上增加固定 Token 预算裁剪，超长条目截断
```

所有新入口继续遵守既有顺序：作用域由服务端解析 → 时间/世界线/continuity 硬过滤 → 相关性排序。

## 4. 验收标准

1. Relationship 状态可写入、可修订、可按观察者隔离召回；`relationships()` 不再把关系状态混同为客观事实。
2. 三类分域摘要可生成、可持久化、可通过召回读到对应 `summaryDomain`。
3. 快照不可变（UPDATE/DELETE 被触发器拒绝）；`add` 后 delta 返回增量；`update/retract` 后 epoch 递进且旧 delta 判定 stale。
4. 安全退出标准契约测试全部通过：
   - 新 Record 不继承未来、平行世界线或无权记忆；
   - 关键词与向量召回不能扩大 ACL（精确关键词命中未授权内容时结果为空）；
   - 固定 Token 预算下长 Record 的表示输出不超过预算且保留最新条目。
5. `npm test` 全绿；`tests/gui/e-memory.spec.ts`（Chromium）不回归。

## 5. 明确不做

- 不建立客观世界关系图谱（M5）。
- 不引入异步 LLM 结论推断与自动矛盾合并（后续批次）。
- 不改 `modules/context` 的编译器、Manifest 或既有 cacheEpoch 语义。
- 不改前端记忆面板交互；`/api/memory` 仅以可选 `domain` 参数暴露分域摘要。
