# REALM Clean-up Iteration · Phase 0 审计（性能 × 记忆/上下文时序 × 输出容错）

> 基线 HEAD=442b69c。本文件是 Phase 0 的硬证据审计：全部为真实代码路径计数与调用图证据（文件:行号），无测量依据的项标注「结构性观察」。实现决策见各节末尾。

## 一、真实模型调用清单（一次玩家回合，happy path，2 AI 角色）

同步路径（阻塞响应）：

| Stage | 模型调用 | DB | 证据 |
|---|---|---|---|
| 预读快照 loadConsistentSnapshot | 0 | 3 事务/7 查询 | local-record-service.ts:2531-2560 |
| resolveRuntimeScope | 0 | 2 事务/6-7 查询（canon 独立第 2 事务） | record-scope.ts:118-345, 334-408 |
| memory prefetch begin | 0 | 后台并行：每角色 1 事务/2 查询 | local-record-service.ts:1628-1638 |
| visibility assessor | 1 | 0 | model-powered.ts:421-462 |
| DM plan | 1 | 2 事务 | model-powered.ts:557-637 |
| 每角色 propose（并行） | ≤2 | 每角色 listSkills 1 事务 | model-powered.ts:849-900 |
| actionResolver（串行） | 每个 skill 行动 +1（discovery） | 每 resolve 1 事务（实例内缓存） | rule-pack.ts:63-93 |
| narrator | 1（越权重试 ≤2） | 0 | model-powered.ts:1088-1195 |
| 每角色 react（并行） | 2 | 0（记忆走 prefetch consume） | orchestration/public.ts:237-246 |
| 结构校验 + 模型 review | 1（三连否决 ≤3） | 2 事务 | model-powered.ts:643-805 |
| commitRelease | 0 | 1 事务/~20+ 查询 | runtime-repository.ts:545-898 |
| 回合后 envelope | 0 | 再次 resolveScope（canon 第 2 次）+ 快照 7 查询 + affordances + firstNight + selfPlay | local-record-service.ts:1731, 2562-2588 |

同步模型调用典型 8 次，带 repair/否决最多 13+。executeTurn 账本全程 11 事务/50+ 查询。

异步尾流（fire-and-forget）：presence/interjection（至多一条整 executeTurn 重跑）、crystallization（第 3 次完整快照读取 + extract/adjudicate 2 次模型调用 + applyDelta 1 事务 + 图谱 claim N+1）、memorySync 萃取（1 事务，逐条 INSERT ON CONFLICT）。

输出 schema 与失败路径：全部 JSON 链走 `requestStructuredObject`（容错解析 + 恰好一次 repair）→ 各链 fail-closed（fatal/null/silent/fallback/degraded），详见 docs/development/PROMPT-SYSTEM.md §四-五。

## 二、记忆/世界知识数据流（表 → repository → application → prompt）

**已生产接线（证据）**：
- 写：`memory_conclusions` ← `extractAuthorized`（memory-repository.ts:197-314，幂等 ON CONFLICT）← sync_turn 调度器（memory/pipeline.ts:23-88）← 4 处回合提交后触发（local-record-service.ts:1789/950/1138/1450）。`relationship_states` ← presence react 的 relationship 输出（local-record-service.ts:1166-1201）与 POST /api/memory/relationship。
- 读进 prompt：回合开始 `prefetch.begin`（每角色并行召回，`recallAuthorized` READ ONLY，worldline head cursor + availableFrom 过滤 + keyword/vector/fidelity/recency 混合排序，memory-repository.ts:71-195）→ propose/react/presence 组装时 `consume`（同步、fail-closed ""）；presence react 的 relationships 文本直读 PG（local-record-service.ts:1054-1069，回合内唯一串行记忆 DB 读）。
- 世界侧：`readCanonLines`（record-scope.ts:334-408，独立只读事务、上限 12×160）→ `brief.canon` → 每条链 user 的 `[World and scene]` 块（T9 接线）。晶化入图谱写入侧（upsertEntity + per-claim）在回合后。

**有表/有路由但无 prompt/UI 消费者**：`memory_snapshots`、`GET /api/memory/delta`、`POST /api/memory/snapshot`、`POST /api/memory/relationship`（UI 无调用方）。`memory_cache_epochs` 仅服务于 delta 路由的 stale 判定（真实消费，但整条链对 prompt 无贡献）。
**纯死接口**：`summarizeForDM/Public/Restricted`（modules/memory/public.ts:309-319，全仓零调用方）。
**已知简化**：萃取的 `observed_entity_key` 硬编码为 continuity_id（memory-repository.ts:281）；embedding 是本地 lexical hash（接口留了 EmbeddingProvider 端口未注入外部实现）。

**决策**：prompt 路径刻意走「每回合授权召回」而非 snapshot 缓存——snapshot/delta/epoch 是已测试的控制面 API，保留但不假装是 prompt 缓存（本节即记录）；`summarizeFor*` 死接口删除（Phase 2）。萃取窗口期（提交→结论可见）由 UI memory-refresh 三次重读缓解，模型侧下回合 fail-closed 缺席——保留（fire-and-forget 是 T2 冻结语义，硬等会破坏回合预算）。

## 三、wired / compat / dead 分类

- **orphan（零生产导入，反向围栏在）**：`modules/story-record/public.ts` 整文件（869 行）——`createStoryRecordContextCompiler`/`compileSharedTurnContext`/`submitPlayerTurn` 等只被 core-runtime.test.ts 与自身引用；`tests/api-core-wiring.test.mjs:25,85` 反向断言活动路由不导入它。
- **几乎全死**：`modules/context/public.ts`——compileContext/ContextLedger/ContextManifest/cacheEpoch/stablePrefix 唯一消费者是 story-record 的 compat compiler 与 context-compiler.test.ts；唯一生产消费是 `WorldCursor` 类型（worldline/conflict-detection.ts:13、branching.ts:1 的 import type）。另有第三份 WorldCursor 拷贝在 modules/memory/public.ts:4 与 app/components/semantic-review-types.ts:12（漂移）。
- **生产真实路径**：app/api/** → local-record-service.ts → orchestration → model-powered.ts（内联 prompt-kit 组装）。两套上下文系统并存；孤儿系统的语义（cursor 时序/可见性过滤）生产侧由 PG 查询层承担。

**决策**：安全删除 orphan（story-record 整模块 + context 裁剪为 types-only，WorldCursor 单源化），删除/改写对应 dead contract 测试（core-runtime.test.ts 相关用例、context-compiler.test.ts）与 d1 review 围栏里的引用。不接 PG ledger 重构（收益不抵风险，生产过滤已在查询层）。见 Phase 2。

## 四、性能怀疑点（证据分级）

硬证据候选（Phase 2 落地，带 before/after 查询计数）：
1. **submitMessage 重复解析**：scope resolve 每回合 2 次（canon 双事务）；完整 projection 快照每回合 3 次（提交前/信封/晶化），presence 尾流再 1-2 次。→ 请求级 memo + 晶化改为消费回合内数据。
2. **commitRelease observations 逐条 INSERT**（12-15 条/回合，同事务）→ 单条 unnest 批量。
3. **晶化图谱 claim N+1**（每 claim 1 事务）→ 单事务批量。
4. **EVENTS_SQL 无 LIMIT 全量历史扫描** × 每回合 3-6 次执行（delivery-projection.ts:835-909）——线性增长热点，但投影是完整时间线的产品语义，**不能擅自截断**；本批只做执行次数削减（热点 1），窗口化留给产品决策（记录在案）。

结构性观察（不声称数值 ROI）：
- worldContextBlock 逐字复制进 8-12 次模型调用（无跨调用 prefix 缓存机制，结构所限）；
- recentPublicEvents（≤4800 字符）随 command payload + turn_runs checkpoint 被 7+ 次 UPDATE 重复物化（replay 自包含要求，改动风险高，保留）；
- saveRun 双写 plan/candidate 独立列 + run_checkpoint 整包（重复物化，改 schema 风险，保留并记录）；
- executeTurn 11 事务账本粒度（合并事务=改运行时状态机语义，超出本批风险预算，记录为后续候选）。

## 五、输出容错残留（Phase 3 清单）

1. first-night 的 repair schemaInstruction 是手写简写串，与 system 的 jsonOutputInstruction 不同源 → 共享常量。
2. genesis-chat 手写 repair 是 requestStructuredObject 的复制 → 接入 helper（message 角色放宽到 assistant；空响应三级阶梯保留为 call 包装）。
3. discovery normalizer 双实现逐行重复（actions/public.ts:346 ↔ model-powered.ts:387）→ 单源导出。
4. genesis-suggestions 的 system 无正式 schema 块 → 补 jsonOutputInstruction 并与 repair 同源。
5. 两个注入过滤器（isInternalDiscoveryText / isInternalInstructionText）词表核心漂移风险 → 抽共享核心词表。
6. 已核实无问题：无绕过 helper 的模型响应 JSON.parse；tools 链未误接 JSON repair；presence×2/review×3/narrator×2/流式×3+回退的重试分层正确（注释完备）。

## 六、现状明确缺口（"有持久化表但主请求未消费"）

- `memory_snapshots`/`memory_delta`/`POST snapshot`/`POST relationship`：无 prompt/UI 消费者（决策：保留为控制面 API，不是 prompt 缓存——见 §二）。
- `world_articles`：由 canon/晶化流程写入，回合 prompt 不消费（决策：本批不发明注入点，记录为后续产品决策——文章是长文 lore，注入需预算与产品语义）。**已由 Shared World Memory v2 覆盖（367fe64/8d8a424/48aff13 + 验收修正 c33b891/3cb5b04/d316e97/d437259/ecc01b9）：合格 article excerpt 经双重闸门（security+temporal，cursor=Record head）后以背景参考身份进入 user context，消费者显式 opt-in（DM plan/Narrator/Character Runner/Discovery；visibility/presence/reviewer/晶化无 lore）；文章资格 fail-closed（claim_ids 非空且全部引用合格，混合引用整篇排除）；Tavern 空链接文章默认排除。实现偏差如实记录：lore 查询未新增 WorldKnowledgeService repository 方法，而是并入 record-scope 的 canon 事务（+1 查询 +0 事务、资格单源 SQL CTE，避免 +1 事务与双实现漂移）。**
- `modules/story-record` + `modules/context` 的编译管线：orphan（决策：删除，见 §三）。

## 七、实施收口（Phase 1–3 实际结果）

- **Phase 1（b8a1b71）**：删除 orphan 上下文管线（modules/story-record 整模块 + core-runtime/context-compiler 死契约测试；三道 D1 围栏更新为删除后事实）；context/public.ts 收缩为 WorldCursor 类型单源（semantic-review-types 复用；memory 的两字段同名类型语义不同、保留）；删除死接口 summarizeForDM/Public/Restricted。新增生产装配垂直证明（postgres-local-record-application 第三用例）：回合 1 提交（prefetch 时记忆未物化，fail-closed 空）→ sync_turn 物化 → 回合 2 召回命中且角色 react prompt 的 user context 真实携带记忆块。
- **Phase 2（f0391b0）**：临时库计数池实测 before→after：scope resolve 2→1、canon 2→1、全量事件扫描 3→2 + 晶化授权瘦读 1、observations INSERT 9→1、总查询 201→170。围栏：tests/postgres-turn-efficiency.test.ts（登记进 test:postgres-runtime）。结构性观察未动：executeTurn 11 事务账本、saveRun 双写、recentPublicEvents 随 payload 重复物化（replay 自包含要求）、EVENTS 全量截断（产品语义，需产品决策）。
- **Phase 3（e57c61f）**：first-night schema 同源（FIRST_NIGHT_SCHEMA）；genesis-chat 手写 repair 合入共享 helper（assistant 历史支持，退化阶梯包进 call）；discovery normalizer 单源（normalizeDiscoveryDraft 共享）；genesis-suggestions system 补正式 schema 块；注入词表核心共享（internal-text-core.ts）。T9 图谱测试暴露批量化误并失效事件基数，已修正为同事事务逐 claim 追加。

## 八、SWM 下一阶段收口（article qualification G0–G6，2026-09-03）

- **资格模型换轨**：lore 注入资格由「claim 链接 ∈ eligible canon」换轨为「owner attestation + immutable qualification ledger」（attested-only，全部 AND 无旁路）。claim 链接合格 ≠ 正文安全的结论落地；canon-generated 自动路径当前生产 writer = none，不进 query。
- **0041 migration（只入库 + 隔离应用）**：`article_qualifications`（append-only 资格账本，复合 FK + 全具名 CHECK + seq 唯一键）与 `article_import_entries`（file_exact source identity：source_namespace=bundle sha256、identity=entry ordinal，schema CHECK 锁死）；pgcrypto 扩展；两表 FORCE RLS + realm_runtime 最小 SELECT/INSERT。0041 不含事务控制语句，由 runner per-file transaction 包裹；共享 realm_dev 未应用（未来部署另审批）。
- **Tavern 导入**：精确同文件去重（SAVEPOINT-per-entry，冲突撤销 candidate 无孤儿，DB readback 判 duplicate）；uid 仅信息列、filename/keys 不作身份；跨上传 content_changed 诚实不支持；导入文章默认 pending_review。未应用 0041 的库回落旧行为（文章永不可注入，fail-closed）。
- **Attestation API**：固定 `POST /api/world-knowledge/articles/qualify`；owner-only 由 session + membership role 服务端判定；worldline 行锁串行化 + seq MAX+1（realm_runtime 对 world_articles 无 UPDATE 授权，article 行锁以 worldline 锁替代）；duplicate 幂等；正文被改且仍 qualified_public → 409 HASH_MISMATCH；revoke 立即全隐藏。
- **lore 谓词**：Record head tuple `(tick, ordinal) >= (available_from_tick, available_from_ordinal)`，不 retroactive；content hash DB-side（pgcrypto digest/encode）read-time 比对；claim_ids 必须为空；预算 80/600/2/1200 不变；turn-efficiency 围栏 total ≤172、loreRead=1、模型调用数不增。
- **GET 控制面矩阵**：pending/rejected 正文 owner-only、非 owner metadata-only；qualified_public 全 member；revoked 全员 metadata；非 member 404；缺 0041 的库全部按 pending_review 处理不 500。图谱面板新增「文章」页签（状态徽标 + owner 授权/拒绝/撤销按钮，前端不发身份字段）。
- **能力边界（诚实声明）**：restricted/secret lore 不做；source_event_ids legacy 不参与资格；entry 内混合权限不支持；模型不参与授权；lore 不进晶化；opt-in 默认 false；GUI attestation 用例因共享库写禁令真实阻塞（route 级矩阵已覆盖）。
- 测试登记：postgres-article-qualification / postgres-article-qualification-migration / article-qualification-design 入 test:postgres-runtime；d1 archive/tooling review 的 migration 计数登记到 41；schema-contract 登记两表。

## 九、v37 轨道 X 收口（2026-09-05）

- qualification 串行点由 worldline FOR UPDATE 换为 per-article advisory + worlds/worldlines KEY SHARE（§E.1 正确锁矩阵）；Record single-writer / Event append-only / seq 连续不变（UNIQUE 兜底 + MAX+1 锁内计算保留）。
- propagation queue 契约升级为 lease 双分量（Z1 旧形态 completeRun(scope, jobId, run) 已废止）；recoverStale 由「恢复即全收」改为 5min 阈值（与 completeRun 行锁串行无穿插）。
- 统一锁序落地：set_config → worlds(KEY SHARE) → records(FOR UPDATE) → record_heads → worldlines → jobs → advisory/其余；worlds 行 FOR UPDATE 单锁点 = 仅 world-archive/delete-world。
- 测试清单治理：六处 era 固定迁移清单（dice/rule-realization/self-play/world-admin/presence/first-night/m4-interjection/local-record-application/memory-pipeline）升级为全链 readdir（0037 列级授权是 gateRecordActive 的协议前提，era 固定清单与新协议不兼容）；runtime-migration-hardening 链尾补 0037。turn-efficiency 预算 172→214（v37 gate 两拍 +42 确定性开销）。
- 观察项（未动）：executeTurn 11 事务账本、saveRun 双写、recentPublicEvents 重复物化等 Phase 2 既有记录不变；不同 Record 同 worldline 回合并行仍为 out-of-scope（K.2 只断言不回归）。

## 十、v37 轨道 Y 收口（2026-09-05）

- `.realm` 包 ABI：manifest version 1 字段序冻结；tables[] 恒三字段（wireDigest 是 dry-run 派生的内部对象，不进包）；双 codec（pack camelCase/ordinal/类型编码 ↔ insert snake_case/值形态收敛）；digest 链 JS↔PL 双侧逐字节一致（向量 1–9 锚定）。
- realm_transfer 权限面：42 内容表 + 5 ledger SELECT-only + 13 受控函数 EXECUTE；helper×4 与守卫×4 零授权；一切写经函数（direct INSERT/UPDATE 全拒）。accounts 不授权——memberships 展示名由调用方 runtime 池预解析。
- 后续登记（明确入口，非缺口）：member/viewer clearance 投影导出；不同 Record 同 worldline 回合并行；导入世界多账号加入；目标 world owner 查看 import jobs 的产品决策；.realm 加密导出；公网暴露面设计；瞬态表导出价值重估。
- GUI attestation/传输面：GUI 级（浏览器）端到端用例未跑（需独立 dev server + 临时库的 harness——本批以 route 级 PG 全矩阵 + UI 静态契约代替，STATUS 如实标记）。
