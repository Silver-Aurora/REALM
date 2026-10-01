# Shared World Memory v2 · 实施计划（调研 + Plan，不含实现）

> 基线 HEAD=4ec4e0f。本文只做有证据的调研与计划；不改任何实现。
> 证据全部为当前仓库 file:line；无证据的路径标注「未发现」。
> 凭据/endpoint/连接串一律 [REDACTED] 或不出现。

## 1. Goal / Non-goals

**Goal**：让 Shared World Memory 中**授权 Canon claims 与合格 article excerpt**以权限安全、预算有界、时序明确的方式进入生产模型 prompt；`world_relations` 在 v2 中定位为**选择/来源关联的底层 substrate**（图谱结构与 provenance 元数据），不直接拼进 prompt。把上一轮 Clean-up 实锤的「world 侧只注入授权 Canon claims」缺口补齐为有证据的垂直闭环。

**硬性前置（独立验收修订，两道 no-go 闸门）**：
1. **security**：在能证明 promoted claim 的 security provenance（public eligibility）之前，不得启用任何 article excerpt，也不得扩大/复制现有通用 Canon 注入（Task 0）。
2. **temporal（本轮复核新实锤）**：现有 `readCanonLines` **不做** valid_from/valid_to 时间过滤（record-scope.ts:352-408 的 SQL 只过滤 workspace/world/worldline + truth_status + supersedes，调用不传 effective cursor；valid_from_tick 仅用于排序）——Canon/lore 注入必须绑定 Record 当前 effective world cursor，过滤未来/已失效 claim；该修复完成前不得启用 lore。若它改变现有通用 Canon 注入结果，属于 security/temporal precondition 的一部分。
两道闸门任一未通过时的安全结果是**零新增 lore 注入**，而不是沿用不明状态的现状。若现有 Canon 通用注入本身被证明泄漏 restricted/secret 晋升 claim 或未来/失效 claim，必须先修（见 Task 0）。

**Non-goals**：
- 不改 Character Memory 链路（上一轮已证明闭环）；
- 不接回已删除的 Context Ledger/compiler（orphan 结论保持）；
- 不改 visibility/audience/recipient/Exposure/Canon/append-only 任何权限语义；
- 不做 propagation 内容注入（传播引擎只搬运 claim id，不读内容——见 §2.8）；
- **`world_relations` 不直接拼进 prompt**（复核修订）：v2 中 relations 是选择/来源关联的 substrate（图谱结构与 provenance 元数据），如需消费须另行补查询/预算/权限/测试；
- 本轮不执行任何 migration 或数据 backfill；
- 不追求跨调用 prompt prefix 缓存（无基础设施，结构性记录）。

## 2. 已证实事实（调研证据）

### 2.1 表结构（全部 `database/postgres/migrations/0010_world_governance.sql`）

- `world_entities`（0010:9-35）：PK (workspace,id)，双 FK CASCADE，kind CHECK 六值，valid_from/to_tick。
- `world_claims`（0010:37-87）：scope CHECK record/story/world；truth_status CHECK 八值；supersedes 自引用 FK；索引 `world_claims_scope_status_idx(workspace_id, worldline_id, subject_entity_id, truth_status, valid_from_tick)`（80-87）；append-only 触发器（89-103）。
- `world_relations`（0010:105-130）：source_claim FK → world_claims；UNIQUE 含 source_claim_id。
- `world_articles`（0010:132-153）：title/body/claim_ids text[]/source_event_ids text[]；claim_ids 无 FK（文本数组）。
- RLS：五表 ENABLE+FORCE，`realm_workspace_isolation`（348-381）；realm_runtime：entities SELECT/INSERT/UPDATE、其余 SELECT/INSERT（418-422）。
- `0025:4` 明令禁止用 world_relations 冒充传播拓扑。

### 2.2 写入来源（谁写图谱）

| 来源 | 写入 | 证据 |
|---|---|---|
| 创世（library-service/world-genesis） | 不写图谱（未发现） | library-service.ts 全部 INSERT 不含图谱四表 |
| Tavern 导入 | 只写 world_articles | tavern-import-service.ts:125-136 |
| 晶化入图谱 | entities + claims（record_confirmed/record scope，白名单谓词） | local-record-service.ts:1285-1339（appendClaims 批量）；scene-crystallization.ts:267-273 |
| Canon approve/merge | promoted claims + canon_revisions；article 恒 null（死分支） | canon.ts:249-360；canon-repository.ts:80-146 |
| worldline merge | 不复制图谱数据（合并世界线无 claims 继承） | worldline-merge-repository.ts:30-227 |
| propagation | 只搬 claim id（text[] 无 FK），不读内容 | canon-propagation.ts:128-199；propagation/public.ts:196-242 |
| 人工编辑 API | entities/claims/relations（无 createArticle action） | app/api/world-knowledge/route.ts:139-215 |

### 2.3 读取来源（谁读图谱）

- 回合主链唯一读取：`readCanonLines`（record-scope.ts:352-408）——truth_status ∈ (story_canon, world_canon)、supersede 链取最新、本 worldline、LIMIT 12、单条 ≤160 字、独立只读事务、fail-closed 空串 → `brief.canon` → 所有 model-powered 链的 `[World and scene]` user 块（model-powered.ts:166-180, 383）+ discovery 链（383）。**已证实缺口：无时间过滤**——SQL 无 `valid_from_tick <= cursor`/`valid_to_tick` 条件，调用不传 cursor（valid_from_tick 只用于 ORDER BY）。
- 图谱面板：/api/world-knowledge GET 全量列表（无 truthStatus 过滤，membership 门禁，route.ts:99-121）；/api/canon GET 提案；SSE 失效事件（只传标记不传内容）。
- 冲突检测读 claims+causal_edges（app/api/worldline/conflict*/route.ts）。
- `world_articles` 的读者：仅图谱面板（knowledge-graph-panel.tsx:206/400-416）；**prompt 消费为零**（README.md 当前边界条已声明；本计划不改变这一事实的口头状态，只给出落地路径）。

### 2.4 Character Memory vs World Knowledge 生命周期差异

- Character Memory：observations（record 作用域）→ sync_turn 萃取 → memory_conclusions（observer continuity + occurred/availableFrom 双 cursor + fidelity）→ 下一回合 prefetch 召回（memory-repository.ts:71-195）。垂直闭环已在 postgres-local-record-application.test.ts 第三用例证明。
- World Knowledge：append-only claims + truthStatus 阶梯（TRUTH_LADDER：mentioned→record_confirmed→story_canon→world_canon；rumor/hypothesis/disputed/deprecated 侧态）+ supersede 链；无 per-observer 维度；读侧按 worldline + truthStatus + supersede 过滤——**但不按时间过滤（已证实缺口）**：`readCanonLines` 不传 effective cursor，无 `valid_from_tick <= cursor` / `valid_to_tick` 条件（record-scope.ts:352-408），未来生效或已失效的 canon claim 当前都会进入注入。
- 关键差异：World Knowledge 无 availableFrom/observer 概念，可见性 = worldline scope + truthStatus +（T11-G 起）canon revision security_class；Character Memory 有 consumer 维度。

### 2.5 当前每个模型阶段实际收到的 context（生产装配实读，非类型推断）

- 全部 model-powered 链 system = 静态英文 policy（Prompt System v2）；user = `[World and scene]`（brief 九字段 + canon 存在时并入）+ 各链自有块。
- canon 对所有链一视同仁（含 Character propose/react）——它是本 worldline 已成立公共正史，对全部 world member 可见；Character 对白 grounding 仍只许本人合法 observation（`renderDiscoveryDialogue` 路径不变）。
- presence gate 额外收触发素材；narrator 收公开结果 + forbiddenNames；review 收候选摘要。
- scene crystallization extract/adjudicate 不读 canon（user 只有当前场景/玩家原话/回合叙述/候选增量）。
- first-night 不读 canon（创世时点无世界线正史）。
- 回合 DB 预算基线（Clean-up Phase 2 实测围栏 tests/postgres-turn-efficiency.test.ts）：scope resolve 1、canon 1、全量事件扫描 2、总查询 170。

### 2.6 任务书与代码的冲突标注

- 无实质冲突。任务书把 snapshot/delta 称作「控制面而不是 prompt cache」与仓库一致；「world 侧当前注入授权 Canon claims」一致。

## 3. 设计问题回答（任务书 §B 逐条）

1. **最小消费单元**：混合——Canon claims 做硬事实（已接线、有 truthStatus/supersedes/scope），article excerpt 做受控背景（长文 lore 的唯一缺口）。理由见 §4 方案比较。
2. **各阶段可见内容**：DM/Narrator/Discovery = canon +（闸门通过后）受控 article excerpt；Character Runner = canon + excerpt + 本人合法 memory/observation；Character 对白 grounding 不变（无合法 privateObservation 只能 action-only；publicFacts 永不进对白）。**scene crystallization 默认排除**（独立验收修订）：文章是背景 lore，不是当前回合的事实证据——extract/adjudicate 不接收 excerpt；若未来要给 adjudicate 只读冲突参考，必须单独定义输入标签（标注为不可晶化的参考资料）、不作为可晶化事实来源，并补 fail-closed/grounding 测试后方可启用。
3. **scope/时间**：claims/articles 按 workspace/world/worldline。**temporal 硬要求（复核修订，原为缺口暗示）**：Canon 与 article eligibility 必须绑定 Record 当前 effective world cursor——`valid_from_tick <= cursor`（inclusive）且（`valid_to_tick IS NULL` 或 `valid_to_tick > cursor`，exclusive；NULL=开放有效）。未来生效/已失效 claim 不得注入。branch：重演 fork 出新 worldline → 只见本 worldline canon（fork 起点为空，worldline merge 不继承 claims——已实锤，产品可接受则保持）；replay 不触发新读取。failing test 必须覆盖 future/expired/same-tick 边界（Task 0/1）。
4. **visibility/audience/recipient/Exposure/truthStatus/supersedes 进入查询的方式**：truthStatus 阶梯 + supersede 最新链 + worldline scope 是既有查询语义（record-scope.ts:363-393）。**security hard gate（独立验收修订，取代原「保持现状」表述）**：promoted claims 落入 world_claims 时不携带 revision 的 security_class（canon.ts:286 只写 canon_revisions）——因此：在能证明每条注入 claim 的 public eligibility 之前，不得启用 article excerpt，也不得扩大/复制现有通用 Canon 注入；若验证证明现有通用 Canon 注入已可能含 restricted/secret 晋升 claim，则该注入本身是**已记录的安全缺口**，必须先修 security provenance（Task 0，必要时设计新增 migration——本轮只规划不执行），不得称为完成。Exposure 是传播侧角色知识视角，不进 prompt 拼接。
5. **article 资格与长度预算（复核修订：去掉无条件 recency 兜底 + 补时间资格）**：文章进入 excerpt 的资格条件（缺一不可，fail-closed 排除）：① 属于当前 worldline；② 能证明 public eligibility——`claim_ids` 链接到当前 worldline 的 public/world_canon claim（链接经 Task 0 的 security 验证），或来自明确的公共导入 provenance；③ **时间资格**：文章自身无 valid cursor，必须继承其合格链接 claim 的 validFrom/validTo——只取至少一条链接 claim 在当前 effective cursor 有效的文章（future/expired claim 链接不算资格）；无 claim 链接的 Tavern article 即使内容看起来公共也不得进入 prompt。**Tavern 导入的文章 `claim_ids` 默认为空（tavern-import-service.ts:125-136），默认不得进入 prompt**——直到后续补齐 provenance/qualification 规则。来源 title 只是展示标签，不构成安全资格。预算：每回合最多 2 条 excerpt、每条 ≤600 字符、合计 ≤1200 字符（截断保留 title 作为来源标签）；无合格文章时零注入；查询失败 fail-closed 空。
6. **输入编排顺序**：system（稳定英文 policy + schema）→ user：`[World and scene]`（含 canon）→ `[World lore excerpts]`（Task 0 闸门通过后、且仅在非空时出现）→ 角色记忆/关系块 → 当前动作/玩家原话 → 任务块。稳定段可复用（无 prefix 缓存基础设施，结构性记录）；memory 段必须按角色隔离。
7. **持久化与时序**：本轮全是读路径——无新写、无 durable queue；晶化/canon merge 的写入既有；excerpt 查询与 canon 读共用 resolve 的同一事务（不新增事务）；异步失败 fail-closed 空串（与 readCanonLines 同款）。
8. **query/cache**：v1 不加跨回合缓存——lore 查询进 canon 同一事务（+1 查询、+0 事务；**该查询只有在 Task 0 的 security gate 通过后才计入预算**，闸门失败时安全结果是零 lore 注入而非复用不明安全状态）；索引可复用 worldline 维度主键/UNIQUE；不做 N+1（单条聚合查询）。若日后加缓存，epoch 可锚 graph_invalidation_events 的 max id（T11-A2 账本已是单调 cursor）。
9. **模型输出容错**：lore 是输入侧，不产生新输出链；既有 requestStructuredObject 语义不变；lore 读取失败=空注入，绝不阻塞回合、绝不补造事实。
10. **migration**：**条件性**（复核修订，不再无条件「不需要」）——Task 0 能从现有 canon_revisions/information_campaigns 反查 security provenance、且现有 Canon 注入无污染时，v2 article 读路径无需 migration（四表与索引均已存在）；若 provenance 不可反查，或需要 claim 级 security/temporal 标记，则本计划只输出 migration 设计并停在 no-go，绝不在本轮执行。

## 4. 方案比较（任务书 §C）

| 方案 | 收益 | 成本/输入负担 | 权限风险 | 实现复杂度 | 结论 |
|---|---|---|---|---|---|
| A. claims-only（现状） | 零新风险；已接线 | 最低（12×160 字） | 无新增 | 零 | 基线保留；不足：长文 lore（文章）永远缺席，世界质感弱 |
| B. article excerpt-only | 文章被消费 | 高（相关性难控、易与 canon 冲突） | 中：excerpt 无 truthStatus 语义，可能把未成立 lore 当事实 | 中 | **不选**：单独使用会让软文本压过硬事实 |
| C. hybrid（claims 硬事实 + 受控 excerpt 背景） | 硬事实与背景分层；文章获得最小消费闭环 | +≤1200 字符/回合、+1 查询（gate 通过后） | 中：excerpt 明确标注为「背景参考」且必须让位于 canon；**security hard gate（Task 0）是启用前提，未通过则零注入** | 低-中（纯读路径、无迁移） | **选定**，分两阶段交付（Task 0 安全闸门 → lore 读取/注入） |

排序：C > A > B。C 以 Task 0 安全闸门为绝对前置；闸门未通过时 hybrid 不启用，安全结果是零 lore 注入。

## 5. 目标 Context Manifest（统一拼接契约）

每回合每条模型链的 user message 段序（既有 Prompt Kit `composeContext`）：

| 段 | 内容 | 来源 | 预算 | 消费者 |
|---|---|---|---|---|
| stable policy | 英文 system policy + schema | prompt-kit | — | 全部 |
| `[World and scene]` | brief 九字段 + canon（12×160） | record-scope resolve | ≤~2000 字符 | 全部链 |
| `[World lore excerpts]`（新，Task 0 闸门通过后才启用） | ≤2 条合格 article excerpt（≤600/条，带 title 来源标签） | canon 同事务 lore 查询 | ≤1200 字符 | DM/Narrator/Character/Discovery（**晶化默认排除**，见 §3.2） |
| `[Character]` / memory/relationship | 本人合法记忆与关系 | prefetch consume / PG 直读 | 既有 | 角色链 |
| 任务块/玩家原话 | 各链自有 | — | 既有 | 各链 |

契约不变量：excerpt 永不覆盖 canon（canon 是 binding，lore 是 background）；空 lore 时段结构性缺席；Character 无合法 privateObservation 时 action-only 语义不变。

## 6. 分阶段任务（bite-sized，可独立提交；Task 0 是 no-go 闸门）

### Task 0 — security + temporal provenance 闸门（no-go precondition，先行）

- **目的**：证明或证伪两道闸门：① 进入通用 Canon/lore 注入的每条 claim 都具备 public eligibility；② Canon/lore 注入已绑定 effective world cursor（无未来/已失效 claim）。
- **修改点**：只读验证脚本/临时库测试（不先改生产）：实查 canon merge 的 promoted claims 能否经 `canon_revisions`/`information_campaigns` 反查出 security_class；实查现有 `readCanonLines` 是否已可能含 restricted/secret 晋升 claim、未来生效（valid_from_tick > cursor）或已失效（valid_to_tick ≤ cursor）的 claim。
- **结果分支**：
  - 可反查且现有注入无污染 → canon/lore 查询统一加 security eligibility + temporal 过滤（Task 1 内实现），继续；
  - 可反查但现有注入已被污染（security 或 temporal 任一）→ 把「修 readCanonLines 的过滤」列为本批最高优先实现项（独立 commit），完成前 Task 1/2 不得启动；
  - security 不可反查 → 设计「晋升 claim 带 security 标记」的新增 migration（只规划不执行）并把本计划停在「安全缺口已记录、lore 不启用」；**不得**沿用现状称为完成。
- **先写 failing test**：`tests/postgres-canon-security-lore-gate.test.ts`（临时库）：restricted merge 晋升 claim → 通用 canon/lore 查询不得返回它；future claim（valid_from > cursor）/expired claim（valid_to ≤ cursor）/same-tick 边界（valid_from = cursor 注入、valid_to = cursor 排除）逐一断言。
- **建议 commit**：`test: Shared World Memory v2 Task 0——canon security+temporal provenance 闸门验证（no-go precondition）`。

### Task 1 — lore 读取路径（仅 Task 0 通过后启动）

- **修改点**：`database/postgres/record-scope.ts`（readCanonLines 传入 Record 当前 effective cursor 并加 security + temporal 过滤；同事务增加 lore excerpt 查询；`brief.worldLore` 新字段——WorldSceneBrief 类型在 record-scope.ts:41-53）；`database/postgres/world-knowledge-repository.ts`（新增 `listArticleExcerpts(scope, {eligibleCanonClaimIds, cursor, limit, perExcerptChars})`，单条聚合查询）；`modules/world-knowledge/public.ts`（服务透出）。
- **article 资格（无 recency 兜底 + 时间资格）**：只取当前 worldline 且 ① claim_ids 链接到当前 worldline 的 public/world_canon claim，或 ② 有明确公共导入 provenance 的文章；且 ③ 链接 claim 中至少一条在当前 effective cursor 有效（valid_from ≤ cursor < valid_to 或开放）；Tavern 导入空 claim_ids 文章默认排除；title 不构成资格；无法证明 → fail-closed 排除。
- **先写 failing test**：`tests/postgres-world-lore.test.ts`（临时库 `realm_worldlore_test_*`，finally 强制拆库）：合格/不合格文章混合种子 → 断言只取合格者、claim 链接优先、预算截断、来源 title 保留、零合格文章零注入、worldline 隔离、重演 fork 不继承、Tavern 空链接文章被排除、链接 claim 未来生效/已失效时文章被排除（same-tick 边界：valid_from = cursor 有效、valid_to = cursor 失效）。
- **命令**：`node --env-file-if-exists=.env.local --experimental-strip-types --test --test-concurrency=1 tests/postgres-world-lore.test.ts` → 先红后绿。
- **回滚边界**：纯新增读取；删除新字段/方法即回滚，无 schema 变更。
- **建议 commit**：`feat: Shared World Memory v2 Task 1——worldline 作用域合格 article excerpt 读取（public eligibility 硬资格、预算有界、fail-closed）`。

### Task 2 — prompt 注入与审计（默认排除晶化）

- **修改点**：`modules/orchestration/model-powered.ts`（worldContextBlock 并入 lore 段——仅 DM/Narrator/Character/Discovery 链；**scene crystallization extract/adjudicate 不注入**；系统规则加英文静态行："Canon is binding; lore excerpts are background reference and must yield to canon."）。
- **先写 failing test**：`tests/prompt-system-audit.test.ts` 增加用例——带 lore 的 brief 驱动 orchestrator plan，断言 `[World lore excerpts]` 在 user、不在 system；晶化链 user 中无 lore 段；canon 优先规则在 system；lore 为空时段缺席。`tests/canon-readback.test.ts` 保持绿。
- **命令**：`npm run test:core`（含 audit）。
- **回滚边界**：删 user 段与静态规则行。
- **建议 commit**：`feat: Shared World Memory v2 Task 2——合格 lore excerpt 注入 user context（canon 优先、晶化排除、空缺席、审计锁定）`。

### Task 3 — 回归矩阵 + 性能围栏

- **回归矩阵（PG/RLS/append-only/visibility）**：postgres-canon-security-lore-gate（Task 0）、postgres-world-lore（Task 1）、postgres-canon-readback、postgres-scene-crystallization（证明晶化无 lore 段）、postgres-retrospection-origin、postgres-canon-security-propagation（restricted/secret 不泄漏）、prompt-system-audit、`npm run test:postgres-runtime` 全量。
- **性能预算**：回合查询 170 → ≤172（lore 查询并入 canon 事务，+1 查询 +0 事务；**仅 Task 0 闸门通过后计入**；闸门失败=零 lore 注入=预算不变）。模型调用数不变；context 增量 ≤1200 字符。before/after 采样：扩展 `tests/postgres-turn-efficiency.test.ts` 标记（canon 事务内 lore 查询计数）——先红后绿。
- **建议 commit**：`test: Shared World Memory v2 Task 3——回归矩阵与性能预算围栏`。

### Task 4 — 文档与 STATUS

- `docs/development/CLEANUP-ITERATION.md` 的「world_articles 未注入」缺口条标注已由本计划覆盖；STATUS 追加实现批事实；README 边界条在实现后更新（本计划不预先改写）。

## 7. 生产接线验收（实现批必须满足）

- 用 fake gateway 捕获真实 ModelGateway messages（prompt-system-audit 模式），不是只测 helper；
- 临时 PG 证明：写入合格 article+canon → 下一回合 DM/Character 的 user context 出现 canon 与 excerpt、无权限越界；
- **Task 0 闸门证据**：restricted/secret revision 晋升的 claim 不出现在通用 canon/lore注入（无法证明时实现批不得启动 Task 1/2，只交付闸门测试与缺口记录）；canon/lore 查询绑定 effective cursor——**future/expired claim 不注入（no-future/no-expired），same-tick 边界（valid_from = cursor 注入、valid_to = cursor 排除）有断言**；
- 不合格文章（无链接、无公共 provenance、跨 worldline、链接 claim 未来/失效）一律 fail-closed 排除；
- relations 不出现在任何 prompt 段（仅作选择/provenance substrate）；
- 晶化链的 user context 无 lore 段；Character 对白 grounding 约束不变；
- 空文章世界零额外 prompt 段、创世/Tavern 导入路径不受影响。

## 8. Rollout / 监控 / 失败恢复 / 回滚

- Rollout：Task 0 只读验证先行；lore 注入为纯读路径 + prompt 段新增，默认无 migration（Task 0 分支三的 migration 只产出设计，另行评审）；先临时库再共享 dev 观察（只读）。
- 监控：turn-efficiency 围栏（查询数）+ prompt audit（段位置）+ 既有 SSE 失效账本可观测 lore 变更频率。
- 失败恢复：lore 查询失败 fail-closed 空注入，回合不受阻；无持久状态需要回补；**闸门失败时的安全结果是零 lore 注入**。
- 回滚：Task 2 单点删除 user 段即回到现状；Task 1 为纯新增。

## 9. 假设与未决产品决策（分栏）

**已证实事实**：见 §2（全部带行号）。
**假设**：① canon 对全部 world member 可见的产品语义继续成立；② 合格文章在多数世界初期稀少（资格硬门槛下注入量小）。
**未决产品决策**（下一轮依赖）：
1. 晋升 claim 是否允许带 security 标记（Task 0 分支三时才需要 migration 设计——本轮只规划不执行）；temporal 过滤（cursor 绑定）不依赖产品决策，是必须修复的缺口；
2. 重演 fork 的 canon 空起点是否符合产品预期（worldline merge 不继承 claims 是现状）；
3. Tavern 导入文章是否需要补 provenance/qualification 规则以获得 prompt 资格（当前默认排除）；
4. excerpt 预算数值（1200/600/2 条）是否需要按世界风格可调。

## 10. 本轮明确不做

- 不实现任何代码（本文件即全部交付）；
- **security 或 temporal 闸门任一未通过时不启用任何 lore 注入，也不扩大/复制现有通用 Canon 注入**；
- 不把 relations 文本拼进 prompt；
- 不给 scene crystallization extract/adjudicate 注入 excerpt（默认排除）；
- 不接回 Context Ledger/compiler；不做 prompt prefix 缓存基础设施；
- 不动 Character Memory、propagation、Exposure、semantic review；
- 不执行 migration/backfill；不把 snapshot/delta 改成 prompt cache；
- 不写凭据/endpoint；不 push。
