# 分支树演化模型调研（BRANCH-TREE-RESEARCH）

- 基线：HEAD `0a2559a`（main），源码级只读调研，不含实现。
- 范围：World / Worldline / Story / Record / 重演 Record 的真实关系审计；分支 canonical 语义推荐；world 级分支树可视化方案；数据/API/迁移方向；分阶段计划。
- 约束提醒：本文只做设计，不改代码、不改数据库、不改 RLS/GRANT、不改模型调用链与 Worker。

---

## 一、现状证据

### 1.1 表结构与外键：层级是六层，全部是独立表

真实层级为 `workspaces → worlds → worldlines → stories → records → scenes/events`，所有表主键均以 `workspace_id` 前缀（RLS 租户隔离）。

- `worlds`：PK `(workspace_id, id)`，列含 `status('active'|'archived')`、`settings jsonb`（`database/postgres/migrations/0001_runtime_contract.sql:19-38`）。
- `worldlines`：PK `(workspace_id, id)`，**分支列已经存在**：`parent_worldline_id text, fork_tick bigint, fork_ordinal bigint`（`0001:46-48`），head 游标 `head_tick, head_ordinal` 默认 0（`0001:49-50`）；自引用 FK `worldlines_parent_fk`（`0001:60-62`）；形状 CHECK 约束"parent 与 fork 游标同有同无、禁止自指"（`0001:65-72`），并有 `worldlines_parent_idx`（`0001:781-782`）。
- `stories`：**独立表**，不是 record 上的字段（`0001:77-109`）。列含 `status('draft'|'active'|'completed'|'archived')`、`start_tick/start_ordinal/end_tick/end_ordinal`；FK `stories_worldline_fk` 指向 worldlines 复合键（`0001:95-97`）。stories 表**没有任何 fork/parent/分支字段**。
- `records`：PK `(workspace_id, id)`，列含 `world_id, worldline_id, story_id, status('draft'|'active'|'closed'|'archived'), start_tick/start_ordinal/end_tick/end_ordinal`（`0001:111-143`）；FK `records_story_fk` 指向 stories 四元复合键（`0001:129-131`）。0008 增补 `timeline_kind ∈ ('primary','retrospection','merged')` 与 `linked_record_id` 自引用 FK（`0008_record_timeline_kind.sql:14,24-27`）。
- `scenes`：独立表，挂 records 四元复合键（`0001:145-179`）；`weather` 快照列（`0039_scene_weather_snapshot.sql:10`）、`display_time` 快照列（`0040_scene_display_time_snapshot.sql:12`）、`tension` 列（`0029:7`）都在 scenes 上。
- `events`：`scene_id NOT NULL`，列含 `record_version, record_ordinal, event_kind, world_tick, world_ordinal, causation_command_id, turn_run_id`（`0001:521-582`）；两条关键唯一约束：`UNIQUE (workspace_id, record_id, record_ordinal)`（record 内密集序号）与 `UNIQUE (workspace_id, worldline_id, world_tick, world_ordinal)`（`0001:549`，**世界游标在同一 worldline 内唯一**）。

### 1.2 游标体系：谁在推进、谁是权威

- `WorldCursor` 类型：`modules/context/public.ts:11-16`：`{ tick, ordinal, calendarId, display }`。
- **worldline head**（全局游标）：`worldlines.head_tick/head_ordinal`。唯一写入方是提交路径 `runtime-repository.ts:943-954`（`UPDATE worldlines SET head_tick…`），写入前 `FOR UPDATE` 锁 worldline 行（`runtime-repository.ts:655-661`）。
- **record head**：`record_heads` 每 record 恰好一行（`0001:584-613`），列 `record_version, next_record_ordinal, last_event_id, last_world_tick, last_world_ordinal`；提交事务内就地 UPDATE、version 单调 +1（`runtime-repository.ts:924-942`），CAS 由 `record_version` 承担（`runtime-repository.ts:687-692`）。record_heads **不是 append-only**，是单行就地更新。
- **effective cursor 的权威读法**：`record-scope.ts:152-153` `COALESCE(rhead.last_world_tick, record.start_tick)`，注释（`record-scope.ts:195-199`）明确"canon/lore 的 effective cursor 用 Record head……worldline 全局 head 不得把其他 Record 推进的 future claim 带进本 Record"。delivery-projection 同形（`delivery-projection.ts:860-862`）。
- **start_tick/start_ordinal**：records/stories/scenes 各自的起始游标，创建时写入后不再变；重演 record 的 start = fork 点（见 1.4）。
- **交付层游标是 viewer-local 密集序号**：`delivery-projection.ts:383-385` 注释明确"Delivery uses a dense, viewer-local cursor; write-side CAS is a separate authenticated control-plane contract"——`ordinal: index + 1`（`:415`），不暴露真实 `record_ordinal`。前端"第 N 条事件"不等于数据层游标。

### 1.3 创建与读取路径

- **创世 → 首个 Record**：`POST /api/world/generate`（`app/api/world/generate/route.ts:56-66`）→ `createGenesis`（`modules/application/library-service.ts:844-1008`），单事务依次 INSERT worlds → worldlines（原初，head (0,0)，`:871-876`）→ memberships → character_definitions → stories → records（`timeline_kind='primary'`，`:921-934`）→ record_heads → `assembleDefaultRecord`（含首个 scene 快照，`:1613-1641`）→ 开场 `narration.committed` 事件（世界坐标 (0,1)）。
- **RecordView 读取**：`GET /api/record` → `loadRecord`（`modules/application/local-record-service.ts:1860-1875`）→ `loadConsistentEnvelope`（`:2979-3018`）→ `delivery-projection.ts:158-196`，单只读事务四段查询（META/CAST/EVENTS/navigation）。导航数据 `loadNavigation`（`delivery-projection.ts:505-557`）只含**当前 worldline** 的全部 stories + 当前 story 的非归档 records——**跨 worldline 导航不存在**。
- **Library 快照**：`GET /api/library` → `library-service.ts:257-410`，返回 worlds → worldlines（含 `parentWorldlineId`，`:333`）→ stories → records 嵌套；record 条目为 `{ id, title, status, timelineKind, linkedRecordId }`（`:27-33`），**不含所属 worldlineId/storyId 扁平字段**，前端需自行拼图。归档 record 被列表 SQL 排除（`:368`）。

### 1.4 重演（duplicate）的真实语义——它是"从起点重演"，不是分支

`POST /api/record/duplicate`（`app/api/record/duplicate/route.ts:30-37`）→ `duplicateRecordInTransaction`（`library-service.ts:1087-1269`）：

- **输入只有 recordId**，无任何起点/游标/场景参数（`duplicate/route.ts:22-28`）。
- **fork 游标固定取源 record 自己的 `start_tick/start_ordinal`**（`library-service.ts:1100-1101`），不是当前 head，更不是任意中间事件。
- 创建四行：新 worldline（`parent_worldline_id=源 worldline`，`fork_*=源 record 起点`，`library-service.ts:1190-1204`，label `回溯：{源 record 标题}`）→ 新 story（`重演：…`）→ 新 record（`timeline_kind='retrospection'`、`linked_record_id=源 record`）→ record_heads（last_world_*=fork）。
- **继承**：源 record 最早一条 scene 的 title/location/objective/tension/weather/display_time 快照（`:1121-1130,1258-1265`）、AI 阵容（`:1158-1183`）、observer 姿态/playerRole、角色 instance 的 `inheritance_cutoff_*`（`:1486-1505`）。
- **不继承**：events（新 record 从 0 事件起步）、canon/memory（走新 record 自己的 worldline 作用域投影）。
- 另一个分叉入口：`POST /api/library {kind:"branch"}`（`library-service.ts:691-727`）——只插 worldline 行，**parent 固定取最早创建的 worldline、fork 取其 head**（`:700-707`），不建 story/record，创建后没有任何后续玩法路径。

### 1.5 worldline merge / conflict / 归档

- **merge**：审计表 `worldline_merges`（`0011_worldline_merge_semantic_propagation_jobs.sql:9-34`，immutable trigger）；`planWorldlineMerge`（`modules/worldline/merge.ts:56-154`）纯函数去重/分类（同游标同载荷 none、同 speaker 同游标 hard 拒绝、异 speaker bridgeable 顺移）；可合并时 `createMergedTopologyAndAudit` 新建 worldline（**不设 parent_worldline_id**，`worldline-merge-repository.ts:108-121`）+ story + 每源 record 一条 `timeline_kind='merged', status='archived'` 的 record 行，**不复制 events**，合并时间线只存在于 `manifest` jsonb（`merge.ts:11-12` "Merging only ever inserts new rows"）。幂等键重放（`merge.ts:243-249`）。API `POST /api/worldline/merge` **UI 未直接接**，唯一消费方是 retrospection 正史入口（`app/api/record/retrospection/commit/route.ts:107-133`）。
- **conflict**：`modules/worldline/branching.ts` 全文 50 行，只有纯分类器 `classifyWorldlineConflict`（`:11-16`），输出 `{conflict, reason, shouldBranch}`——**不是 branch 实现**。`/api/worldline/conflict` UI 未调用。
- **归档**：record 归档 = `UPDATE records SET status='archived'`（`library-service.ts:591-597`），owner-only、防活跃 self-play/turn，**无物理删除、无恢复入口**（`0037_record_archive_grant.sql:3-6` "Record deletion is implemented as an owner-authorized archive/hide operation. Committed Events remain append-only"）；读侧 fail-closed（`delivery-projection.ts:874-876`、`record-scope.ts:189`、写路径 `runtime-repository.ts:633-636`）。world 归档/恢复双向（`library-service.ts:610-639`）。

### 1.6 知识图谱 / Canon / Memory / Propagation 与 Record 的边界

- **作用域全部是 worldline 级**：world_entities/claims/relations/articles/causal_edges 的 UNIQUE 含 `worldline_id`（`0010:21-22,54-55`）；canon 读取谓词 `workspace_id+world_id+worldline_id`（`canon-repository.ts:47-49`）；memory 谓词再加 `observer_continuity_id`（`memory-repository.ts:136-151`）；propagation topology 快照 `WHERE … worldline_id=$3`（`propagation-topology.ts:56-69`）。
- **依赖 vs 投影**：events/observations/context_snapshots 是 per-record 产物（各 record 自带，分支不需继承）；canon/memory/propagation 是 worldline 级**依赖**——分支化时新 worldline 天然隔离，但"继承父线分叉前正史"没有现成机制（`SHARED-WORLD-MEMORY-V2-PLAN.md:78` 已记录当前语义："branch：重演 fork 出新 worldline → 只见本 worldline canon"）。
- **图谱与叙事 Record 共享实体数据但不同源**：claims 由 scene crystallization 从 Record 回合写入（record scope），`GraphArticle.sourceEventIds` 引用 Record 事件（`app/components/knowledge-graph-types.ts:34`）——**知识图谱是内容关系图，不是叙事分支树**，两者不能混为一谈。
- 图谱 UI 现状：入口在 world 级（Library 世界卡 `library-panel.tsx:163-171`、WorldView header `world-view.tsx:69-75`），overlay 形态；但面板头部**硬编码"原初世界线"标签**（`knowledge-graph-panel.tsx:232`），无 worldline 切换器；渲染是手写 SVG 环形布局（`knowledge-graph-panel.tsx:490-607`），**全仓无任何图布局依赖**（package.json dependencies 仅 fflate/pg/react/react-dom/vinext）。

### 1.7 权限、append-only 与锁顺序（分支设计的硬约束）

- **角色**：`realm_runtime`（`0002:471-477`，NOBYPASSRLS，列级最小授权、无 DELETE/TRUNCATE，`0004:68-71`）、`realm_control`、`realm_transfer`（`0042:17-42`，纯 SELECT + 13 个受控 SECURITY DEFINER 函数）。**无独立 app 用户角色**。
- **RLS 只做 workspace 隔离**：统一策略 `realm_workspace_isolation`，谓词 `workspace_id = realm_current_workspace_id()`（`0002:442-465`、`0010:348-416` 等）；account/world 级隔离全在应用层（`player_world_memberships`，`0001:300-311`）与席位解析（`record-scope.ts:302-321`，找不到席位 = 404 等价）。
- **append-only**：events 有 trigger `guard_event_append_only` + REVOKE UPDATE/DELETE（`0002:9-22,537-540`）；world_claims/canon_revisions/information_packets 等同样 immutable。**历史不可改写，分支只能以新 worldline/record 行 + 追加写实现**。
- **锁顺序**（`world-write-gate.ts:9-13` 注释）：`set_config → worlds(KEY SHARE) → records(FOR UPDATE) → record_heads → worldlines → …`。`gateWorldWrite`（`world-write-gate.ts:32-52`）是内容面写事务第一锁。创建分支可复用 `withWorkspaceTransaction` + `gateWorldWrite` + 既有 INSERT 授权（`0022:9` 已授 realm_runtime INSERT ON worldlines 等）——canon merge（`canon-repository.ts:68-162`）与 worldline merge（`worldline-merge-repository.ts:156-166`）都是现成的"单事务原子拓扑写入"范本。
- **导出/导入**：`.realm` v1 已把 `parent_worldline_id` 列为一等公民（`modules/world-transfer/export-matrix.ts:74-77` rewriteColumns；导入侧自引用拓扑排序 `world-import-service.ts:104-142`）；template 模式遇分支 409 `TEMPLATE_UNSUPPORTED_BRANCH`（`world-transfer-repository.ts:1154-1167`）；selection 闭包沿祖先链补齐（`:412-431`）。**只利用既有分支列不破坏 v1 兼容；新增表/列才需要 bump version 与 requiresMigrations。**

---

## 二、推荐模型：先回答模型问题，再谈 UI

### 2.1 五种"关系"必须分开，不能揉成一团

| 关系 | 载体（现状） | 性质 |
| --- | --- | --- |
| branch lineage（分支谱系） | `worldlines.parent_worldline_id + fork_tick/fork_ordinal`（`0001:46-48`） | 树（每节点至多一个 parent） |
| story membership（归属） | `records.story_id → stories → worldline_id`（`0001:129-131`） | 严格层级，非谱系 |
| worldline causal order（因果序） | `events` 的 `UNIQUE(worldline_id, world_tick, world_ordinal)`（`0001:549`） | 线内全序，跨线无序 |
| knowledge graph edge（内容关系） | world_relations / causal_edges（worldline 级，`0010`） | 任意图，与叙事结构无关 |
| record navigation edge（导航） | delivery `loadNavigation`（`delivery-projection.ts:505-557`） | 纯展示投影 |

结论：**真实关系是"主分支树（worldline lineage）+ 少量跨引用 overlay"**，不是单一树也不是全 DAG：
- 树骨架 = worldline 父子链（schema 已保证每 worldline 至多一个 parent、禁自指）。
- 跨引用 overlay 只有两类，都是有向、可标注边：① `records.linked_record_id`（重演/merged record → 源 record，0008）；② `worldline_merges` 审计行（merge 结果 ← 两个源 worldline，`0011:9-34`）。
- 知识图谱边只属于内容层，作为节点详情的关联信息出现，**不进分支树骨架**。

### 2.2 Canonical 分支语义（推荐唯一语义）

**Branch root = worldline。** 分支不是 record 的属性，是 worldline 的属性；record 只是 worldline 上的游玩单元。这与现有 schema 完全一致，不需要新发明概念。

- **parent branch**：`worldlines.parent_worldline_id`（已存在）。
- **parent record / 分叉来源记录**：新 worldline 首个 record 的 `records.linked_record_id` 指向"发起分叉时所在的 record"（复用 0008 既有列，语义从"重演来源"推广为"分叉来源"）。
- **fork cursor**：`worldlines.fork_tick/fork_ordinal`（已存在），语义 = **父 worldline 因果序上的一个已提交世界游标**。分叉后新线 head 初始化为 fork 点（duplicate 已这么做，`library-service.ts:1194`）。
- **分支 = 新 Record 继承，不是 copy-on-write 也不是 snapshot**：历史 events 因 append-only 不可复制（`0002:9-22`），新线从 0 事件起步、只读侧按 worldline 作用域隔离。所谓"继承"只发生在四个明确点位：scene 快照、AI 阵容、角色 instance 的 `inheritance_cutoff_*`、姿态/playerRole——即现有 `duplicateRecordInTransaction` 的装配逻辑（`library-service.ts:1245-1267`）。
- **从中间事件/中间场景拉分支**：schema 的 fork CHECK 允许任意 fork 点，限制在应用层（duplicate 恒取源 record 起点）。推荐语义：fork 点必须满足 **落在某个已提交 event 的 `(world_tick, world_ordinal)` 上**（可精确校验存在性），且 source record 的 effective cursor ≥ fork 点；scene 起点取"fork 点之后父线上第一个 scene 边界"或显式新建 scene，**不复制父线 events**。这是 P1 的新能力，P0 不做。
- **从重演起点再分支**：允许——重演 record 本身在一条 worldline 上，再分叉就是该 worldline 的 child，谱系自然递归，无需特例。

### 2.3 责任边界与生命周期

- **World**：命名空间与权限边界（membership、owner、归档）；分支树的最大作用域。生命周期 active/archived。
- **Worldline**：谱系节点 + 因果序载体（head 游标唯一推进者）；canon/memory/propagation/图谱的作用域。状态字段当前允许 `active/frozen/archived`（`0001:63-64`），其中 `frozen` 的产品/写入语义尚未在本功能中冻结，**不可删除**（无 DELETE 授权）。
- **Story**：worldline 内的叙事分段（起止游标），纯粹的组织单元；同一 worldline 下多个 story 顺序排列；**story 不参与谱系**。
- **Record**：一次可游玩的连续会话，挂在 story 下；`timeline_kind` 区分 primary / retrospection / merged；生命周期 draft→active→closed→archived（归档不可逆）。
- **Replay/Retrospection Record**：`timeline_kind='retrospection' + linked_record_id` 的 record，是"分支的一种既有特例"——从源 record 起点重演，正史回写走 worldline merge。**它已经是分支，不应再发明第二套**；分支树功能应把它泛化（任意 fork 点、可选手动命名），而不是并列新增。
- **Merged Record**：`timeline_kind='merged', status='archived'` 的拓扑占位行，只承载 merge manifest 的挂点；不可游玩、不可再分叉（建议禁止，理由：其 events 未物化，effective cursor 无语义）。
- **跨 Story 分支**：天然支持（分支在 worldline 层，新线自建 story 链）；UI 上表现为新子树。
- **归档/恢复**：分支（worldline）不提供删除；record 归档维持现状（owner、不可逆）；P2 可考虑 record 恢复，但与分支树无关，不单列。
- **比较**：只读 diff 两条 worldline 在共同 fork 点之后的 events/canon 差异，复用 `planWorldlineMerge` 的分类器做展示，不写入。
- **合并**：见 5.3，P2 且不承诺——现有 merge 产出"不可游玩的归档拓扑 + manifest"，不是玩家意义上的"合并后继续玩"，语义缺口大，过早做会把分支树拖入合并地狱。

### 2.4 重演语义审查：哪些可复用、哪些不能伪装成 branch

可复用（已经是分支语义的正确子集）：
- 新 worldline + fork 游标 + head 初始化（`library-service.ts:1190-1204`）。
- scene/weather/display_time 快照继承（`0039/0040` 的列 + `:1258-1265`）。
- 角色 continuity 经 `inheritance_cutoff_*` 隔离（`:1486-1505`）——角色记忆只看分叉点之前，这是正确的 continuity 语义。
- 游标隔离：新线 events 的 `(world_tick, world_ordinal)` 唯一约束只在本线内（`0001:549`），新线从 fork 点之后分配，天然不与父线冲突。
- canon 隔离：新线只见本线 canon（`SHARED-WORLD-MEMORY-V2-PLAN.md:78`），record-scope 的 effective cursor 用 record head（`record-scope.ts:195-199`）防止 future claim 泄漏。

绝不能伪装成 branch 的：
- **worldline merge 不是分支合并的完成态**：merged record 是归档占位，events 在 manifest jsonb 里未物化；把"merge"当"branch 的逆操作"宣传是语义造假。
- **`classifyWorldlineConflict` 不是分支创建器**：它只是分类器，输出 `shouldBranch` 建议（`branching.ts:11-16`）。
- **`{kind:"branch"}` 不是可用分支**：它创建没有 story/record 的空 worldline、parent 恒为最早线（`library-service.ts:700-707`），玩家无法在其上游玩——P1 必须改造或废弃此入口，否则分支树上会出现"幽灵节点"。
- **知识图谱的实体关系边**不能当叙事分支边渲染。

### 2.5 为复杂演化保留的性质

- **不可变 provenance**：fork 游标 + parent 指针 + linked_record_id 全部写入后不改（worldlines 行本身除 head/status 外无 UPDATE 路径；fork/parent 列无授权 UPDATE——P0 需核实并在设计稿中冻结"provenance 列永不 UPDATE"为不变量）。
- **有效游标**：任何分支节点都能回答"我从哪来（fork 点）、我到哪了（head）"，两个游标都在父线/本线因果序上可解释。
- **可重放性/确定性**：fork 点 + append-only events ⇒ 任何分支的历史前缀 = 父线 ≤fork 点的前缀 + 本线追加，重放只需读两条有序流，无隐藏状态。
- **抽象形态**：对外呈现是 **lineage tree（谱系树）**；因果分析（merge/conflict）才落到 causal graph。**不要强行把 merge 边塞进树骨架**，用 overlay 边表达。

---

## 三、可视化产品方案（world-level branch explorer）

### 3.1 形态与挂载点

- **world 级入口**，三处：WorldView header（与"图谱"按钮并列）、Library 世界卡、Record 顶栏面包屑的 world 节点（作为"回到世界"的扩展）。Record 内**不**放完整树，只在 header 给一个"来源/分支"摘要链接（P1）。
- **容器范式**：复用现有 overlay 模式（`graph-overlay` + fixed panel，`globals.css:3947-3950`），避免新增路由；窄屏占满 `100dvh - 36px`。
- **默认树状**：骨架 = worldline 谱系树（纵向时间流、横向分叉）；**混合形态** = 树骨架 + overlay 跨引用边（重演 linked 边、merge 边）用虚线/点线叠加， hover/聚焦时才高亮，避免常态视觉噪声。
- **不是知识图谱**：分支树与 knowledge-graph-panel 是两个独立视图；节点详情里可给"查看此线图谱"跳转（图谱面板届时需要 worldline 切换器——当前硬编码"原初世界线" `knowledge-graph-panel.tsx:232`，列为 P1 前置小改）。

### 3.2 节点/边的视觉规则（不靠单一颜色）

- **节点类型**（形状 + 颜色 + 文本徽标三重编码）：
  - worldline 根（原初世界线）：实心方块 + "原初" 徽标；
  - 分支 worldline：直角边框方块 + 分支名；
  - record 节点挂在所属 worldline 泳道内：primary=实心、retrospection=斜纹/双线边框 + "回溯" 徽标、merged=灰色 + "合并" 徽标（现状 StoryView 已有这三个徽标样式可复用，`story-view.tsx:91-105`）；
  - 当前所在 record：粗边框 + "在读"（复用 world-navigation 的 `.is-current` 语义）；
  - 归档 record：降低对比度 + "已归档"文本（不是只变灰）；
  - 不可访问（无席位/无权限）：锁形图标 + 文本，不渲染标题内容。
- **边**：谱系边 = 实线直角折线（保持纸墨直角语言，禁曲线/圆角）；重演 linked 边 = 虚线；merge 边 = 点线汇聚到 merged 节点；线粗区分"主游玩路径"（从根到当前 record 的路径加粗）。
- **冲突/待合并状态**：只读期不展示；P2 若有冲突检测结果，用节点角标 + 文本，不只靠红色。

### 3.3 动作主次级（P0 只读 / P1 才写入）

- **P0（只读）**：浏览树、节点详情（fork 点/head/创建时间/谱系路径）、进入 record（继续游玩 = 主按钮）、回到父分支、路径高亮、搜索/过滤。所有动作导航到既有页面，**不新增任何写 API**。
- **P1（写入，全部走显式确认）**：从当前 record 创建分支（fork=当前 head，等价于泛化 duplicate）、从选中事件/游标创建分支（二级入口，需确认对话框说明"历史不复制、从该点重新演绎"）、选择 Replay 起点、重命名分支 label（worldlines.label 的 UPDATE 授权需新增，列入 migration 计划）、归档 record（复用既有入口）。
- **P2**：比较两个分支（只读 diff 视图）、合并（见 5.3 的保留意见）、record 恢复。
- 确认语义：所有写动作两段式确认（沿用 `RecordDeleteButton` 模式，`library-panel.tsx:690-739`）；分支创建结果必须 `openRecord(新 recordId)` 走既有"创建成功 → 进入新 Record"路径，不发明第二条回家路。

### 3.4 大树性能与可访问性

- **折叠/聚焦**：默认只展开"根 → 当前 record"的祖先链 + 一级兄弟；其余子树折叠为计数徽标（"3 条分支"）。
- **布局**：不引入 dagre/d3（全仓零图依赖，`package.json:35-41`）；树用确定性分层布局（DFS 序号 × 层级），手写 SVG 或 DOM+CSS，直角折线。节点数 > ~200 时按子树虚拟化（只渲染展开部分）。
- **搜索/过滤**：按 worldline label / record 标题过滤，命中路径自动展开。
- **键盘/ARIA**：树语义 `role="tree" / treeitem / aria-expanded / aria-level`，方向键导航，Enter 进入 record；窄屏替代视图 = 缩进列表（复用 WorldView 的 `view-record-list is-nested` 模式，`world-view.tsx:136-150`），保证 720px 下面包屑被隐藏时（`globals.css:2630-2633`）仍有可达路径。
- **状态**：loading（骨架行）/ empty（"还没有分支，从当前记录创建第一条"）/ error（安全文案 + 重试）三态齐备，沿用现有 notice 语义，不暴露技术字段。

---

## 四、数据 / API / 迁移方向（只设计，不实现）

### 4.1 原则：复用 records/worldlines，不新增 branch_nodes/branch_edges

理由：
1. 谱系列已存在且有 CHECK/索引（`0001:46-48,60-72,781-782`）；新建 branch_* 表会造成同一事实两个真相源（worldline 父子 vs branch 边表），迁移期一致性风险高。
2. append-only 约束下分支本来就是"新 worldline + 新 record 行"，没有独立实体需要表达。
3. `.realm` v1 已支持 parent 重映射与祖先闭包（`export-matrix.ts:74-77`、`world-import-service.ts:104-142`），不动表结构就**不需要 bump 格式版本**。

需要新增的 only 是**投影/查询与少量列**：

- **P0（只读树投影，零 migration）**：新 API `GET /api/world/branch-tree?worldId=`——服务层在 `withWorkspaceTransaction({readOnly:true})` 内读 worldlines（parent/fork/head/label/status）+ records（story 归属/timeline_kind/linked_record_id/status/起止游标）+ stories，组装树 JSON。归档 record 带 tombstone 节点（区别于 Library 列表的排除语义，`library-service.ts:368`），保证树结构完整；非成员 404（membership 校验范本 `worldline/merge/route.ts`）。
- **P1（创建分支，小 migration）**：
  - 泛化 duplicate：`POST /api/record/branch`，入参 `{ recordId, fork?: {tick, ordinal} | {eventOrdinal}, label?, storyTitle? }`，缺省 fork=当前 head（与 duplicate 的"起点重演"区分；duplicate 保留原语义作为"重演"快捷方式）。
  - fork 点校验：必须 ≤ source record effective cursor、且对应本线已提交 event；通过 `events` 的 `(worldline_id, world_tick, world_ordinal)` 唯一索引精确判定存在性。
  - 可选列：`worldlines.created_by_record_id`（provenance 冗余，`SYSTEM-DESIGN.md:378-394` 的设计稿已提及 `created_by_record_id/fork_event_id`，实现时对齐）、`records` 无需新列（linked_record_id 复用）。
  - 重命名：`UPDATE worldlines SET label` 的列级 GRANT + owner/member 校验（新 migration，锁序遵守 `world-write-gate.ts:9-13`）。
- **幂等与并发**：branch 创建带 `idempotency_key`（客户端生成 UUID，参考 `worldline_merges` 的 UNIQUE idempotency_key，`0011:9-34`）；同事务内 `gateWorldWrite` → 源 record `FOR UPDATE`（防 fork 期间归档）→ 按锁序插 worldlines/stories/records/record_heads；record_version/head 的 CAS 语义不变。
- **防幽灵分支/错误传播**：创建事务任何一步失败整体回滚（canon merge 范本 `canon-repository.ts:68-162`）；分支创建后 record_first_nights/开场事件的补写走既有 first-night 事务外调度，失败不污染谱系。
- **权限**：创建分支要求 owner/player membership（对齐 `{kind:"branch"}` 的 `assertWorldContentMember`，`library-service.ts:693`）；observer 只读树不可分叉；非成员整树 404；跨 workspace 由 RLS 天然拦截；未来多人扩展时树投影无需改（worldline 无 owner 概念，权限在 world 层）。

### 4.2 明确不改的边界

- 不改 events/record_heads 的 append-only 与 CAS 语义；不改 delivery 投影的 viewer-local 游标；不改 RLS 策略与角色体系；不改 canon/memory/propagation 的 worldline 作用域；不改 merge 语义；不改 `.realm` v1 格式（P1 若新增 `created_by_record_id` 列需评估 requiresMigrations 追加，不动 version）。
- **tavern-import 只写首条 worldline**（`tavern-import-service.ts:103-115`）：分支世界里导入角色卡落到 root 线——P1 需在文档/UI 标注该限制，暂不扩展。

---

## 五、分阶段计划与验收

### 5.1 P0：冻结语义 + 只读树投影（本研究的直接后续）

- 范围：`GET /api/world/branch-tree`（新）+ branch explorer overlay（新组件，只读）+ 三处入口；**零 migration、零写路径**。
- 源码范围：`app/api/world/`（新 route）、`modules/application/library-service.ts` 或新 `branch-tree-service.ts`（只读查询）、`app/components/`（新 branch-tree-panel.tsx + types）、`app/realm-client.tsx`（overlay 状态）、`modules/i18n/public.ts`（三语）、`app/globals.css`。
- 明确不改：duplicate/retrospection/merge/`{kind:"branch"}` 行为；图谱面板；Record 玩家路径。
- 测试：树组装契约测试（多分支/重演/merged/归档/空世界/非成员 404）；Chromium+WebKit GUI：WorldView 入口 → 树打开 → 进入 record → 返回；窄屏列表视图；键盘导航。
- 风险/回滚：纯增量，回滚 = revert。

### 5.2 P1：真实创建分支与继续游玩

- 范围：`POST /api/record/branch`（泛化 duplicate，任意 fork 点）+ 从树/Record 事件时间线发起分叉的 UI + worldline 重命名 + 图谱面板 worldline 切换器（去掉 `knowledge-graph-panel.tsx:232` 硬编码）+ 改造或废弃 `{kind:"branch"}` 空 worldline 入口。
- migration：`worldlines.created_by_record_id`（可选）、label UPDATE GRANT、幂等键表或列。
- 测试：PostgreSQL 一次性隔离库（参照 tests/gui 与 postgres 测试的 disposable 模式，**不碰共享库**）：fork 中间点创建、无效 fork 拒绝、幂等重放、并发分叉、归档源拒绝；契约：events 不复制、scene 快照、inheritance cutoff；GUI：创建分支 → 进入新 Record → 首回合 → 树上出现新节点。
- 风险/回滚：写路径新但独立于既有回合写；回滚 = 关闭入口 + revert migration（列可留空）。

### 5.3 P2：比较、重演分支体验、合并（明确保留）

- 比较（只读 diff）：复用 `planWorldlineMerge` 分类做双栏对照。
- **合并不应过早做，理由**：① 现有 merge 产出归档占位 + manifest，events 未物化（`merge.ts:11-12`），"合并后继续玩"需要新的物化语义，牵动 canon/memory/propagation 三线继承；② retrospection commit 已覆盖唯一被验证的合并场景（正史回写）；③ 合并 UI 会诱导玩家把分支当 git 用，与"append-only 叙事"心智冲突。P2 先做比较与更清楚的重演入口，合并是否做、做成什么形态，届时单独评审。
- blocked/deferred：**record 恢复（unarchive）deferred**（与分支树无关，需单独安全评审）；**跨 world 分支 blocked**（违反 world 权限边界，无正当场景）；**分支删除 blocked**（无 DELETE 授权且破坏 provenance，归档替代）。

### 5.4 验收矩阵（每阶段）

- 契约测试：树投影形状、谱系完整性（parent/fork/linked 一致）、权限负例。
- PostgreSQL 隔离测试：一次性库跑 migrations 0001–0043 + seed + 分支场景 SQL，验证 CHECK/FK/RLS 行为。
- GUI（真实 Chromium，WebKit 快速可跑则补）：P0 三条只读路径 + 窄屏；P1 创建分支全链路；所有用例一次性隔离数据，跑完清理，不动共享库。
- 静态：`api-core-wiring` / `documentation-layout` / lint / typecheck / `git diff --check`。

---

## 六、风险与未决问题（需 Lyle 决策）

### 最大 P0 风险
**"branch" 一词当前有三个互不兼容的实现**：① duplicate/retrospection（从起点重演，可用但 fork 固定）；② `{kind:"branch"}`（空 worldline，孤儿节点）；③ worldline merge（归档拓扑）。P0 树投影会把这三者同时可视化出来，若不在 UI 文案与入口上先区分清楚，"分支树"会把既有语义混乱直接暴露给玩家。P0 必须先冻结术语（分支/重演/合并三词分工）再做视图。

### 需决策的未决问题
1. **术语**：UI 上"重演（retrospection）"与"分支（branch）"是合并为一个概念（分支=可任意 fork 点的重演），还是保持两个入口？推荐合并为一个"分支"概念，重演正史回写作为分支上的特殊操作保留。
2. **`{kind:"branch"}` 空 worldline 入口**：P1 改造（创建时强制带 story+record）还是废弃下线？涉及 Library 手动表单既有行为。
3. **fork 点粒度**：第一版任意 fork 点是否只开放"事件边界"（某条已提交 event 之后），还是允许 scene 边界快捷选项？推荐事件边界 + UI 默认推荐 scene 边界。
4. **canon 继承策略**：新分支完全不见父线 canon（现状语义）是否符合"从中间分叉"的玩家预期？若需要"分叉点之前的父线 canon 只读可见"，这是 P1 之后最大的语义扩展点，需单独设计（涉及 record-scope 的 worldline 谓词与 effective cursor 双重闸门）。
5. **merged 节点**：树上是否展示 merged record 占位节点？推荐展示为只读灰色节点（可查看 manifest 摘要），不可进入。

### 其余风险
- 树投影性能：worldlines/records 全量读，大世界的节点数上界未量化；P0 需加节点数软上限 + 折叠默认值（P0）。
- 归档 record 在树上可见 = 信息暴露面变化（Library 列表是排除的，`library-service.ts:368`）；需确认 owner/player/observer 看到的 tombstone 粒度（P0）。
- label 重命名授权是新写面，需走与 0037 同级的列级 GRANT 评审（P1）。

---

## 七、明确不做事项（本功能全周期）

- 不改 events/record_heads 的 append-only 与 CAS 语义；不实现任何"改写历史"的分支。
- 不把知识图谱当分支树，也不在分支树里渲染图谱实体边。
- 不做跨 world / 跨 workspace 分支；不做分支物理删除。
- 不做"合并后可继续游玩的物化时间线"（除非 P2 单独评审通过）。
- 不引入图布局第三方依赖（dagre/d3/reactflow 等）；不做圆角/玻璃拟态等偏离纸墨直角语言的视觉。
- 不改 `.realm` v1 格式版本；不动 RLS/GRANT 角色体系（新增列级 GRANT 除外且需独立 migration）。
- 桌面原生壳（Tauri/Electron）不在本功能范围；安装后形态维持"本地服务 + 系统浏览器"。

---

## 八、实现记录（P0+P1 垂直切片，2026-09-14）

> 本节是实施后的补充，不改动上文原始证据。commit 链：
> `0881403`（只读投影）→ `ec8a1fe`（可玩分支）→ `233155d`（explorer UI）
> → `e0e1a7a`（PG/GUI 测试）→ `f84639e`（围栏对齐）。

### 落地语义（最终决策）

- **timeline_kind 扩展为四值**：`primary | retrospection | merged | branch`
  （migration `0044_record_branch_timeline_kind.sql`，仅追加枚举，旧数据
  不受影响）。必要性：retrospection 的「写入正史」流程
  （`app/api/record/retrospection/commit/route.ts:81`）只对
  `timeline_kind='retrospection'` 放行——若分支复用该枚举，分支记录会
  被误导入正史回写流程，或被误标为「回溯」。`linked_record_id` 语义从
  「重演来源」推广为「分叉来源」（retrospection/merged/branch 共用）。
- **branch root = worldline**；分支创建 = 单事务原子写入新 worldline
  （parent=源 worldline，fork=给定已提交游标）+ 新 story + 新 record
  + record_heads + 场景/角色装配（`createRecordBranchInTransaction`，
  `modules/application/library-service.ts`）。events 不复制；场景取
  「覆盖 fork 点的最近 scene」；角色 inheritance cutoff = fork。
- **fork 校验**：缺省 = 源 record effective head；eventId 形式必须是源
  record 自己的已提交事件（服务端读 canonical 游标，viewer-local
  delivery ordinal 不得冒充）；显式游标必须命中源 record 一条已提交
  事件或等于 record start；未来/越界/空洞/他 record 事件/归档源/非
  成员全部 fail-closed（INVALID_FORK/RECORD_ARCHIVED/WORLD_NOT_FOUND）。
- **幂等**：确定性 record id（sha256(workspace:source:key)）；幂等身份不
  随默认 head 改变，源 record FOR UPDATE 是并发串行点，同键并发恰一套
  拓扑，重放 200；重放读回首次落库的 fork 游标。
- **kind:"branch" 幽灵路径处置选择 = 改造**：命令新增必填
  `sourceRecordId`，委托同一 helper 产出可玩拓扑（fork=源 head）；
  Library 手动表单的「分支」tab 已移除（表单无来源选择器，统一走
  Record 页/分支树入口）。回归测试：缺 sourceRecordId → INVALID_COMMAND。
- **合并不做**：worldline merge 维持归档拓扑/manifest 语义，未扩展。

### API / UI

- `GET /api/world/branch-tree?worldId=…[&recordId=…]`：只读树投影
  （`modules/application/branch-tree.ts`），任一 membership 角色可读，
  非成员 404；归档 record/worldline 以 tombstone 保留（status 显式），
  merges 审计作 overlay 列表；recordId 经服务端归属校验后回显 current。
- `POST /api/record/branch`：body `{recordId, fork?, label?, storyTitle?,
  recordTitle?, idempotencyKey(必填)}`；201 创建 / 200 重放 / 400 形状
  / 403 observer / 404 不存在 / 409 状态冲突（INVALID_FORK 等）。
- UI：`app/components/branch-tree-panel.tsx`（world 级 overlay，纯 CSS
  正交连接线树，无图布局依赖；Worldline→Story→Record 三层、role=
  tree/treeitem + roving tabindex + 方向键；≤720px 同 DOM 退化缩进列表）；
  入口 = WorldView header + Library 世界卡 + Record 页「创建分支」对话框
  （head/事件边界两种 fork，事件选择用 canonical event id）。创建对话框
  具备标题/说明 ARIA 关联与初始键盘焦点。Library/StoryView/Record 徽标已
  区分 branch/retrospection/merged；节点详情的父线与来源 Record 标签分开。

### 验证

- `tests/postgres-branch-lineage.test.ts`：一次性 scratch 集群 + 全链
  0001–0044 + seed（3/3 通过，覆盖树投影/权限负例/fork 校验/原子性/
  幂等/并发/父子隔离/duplicate 回归/幽灵路径回归/路由状态映射；
  runtime 池实测 realm_runtime 角色足够完成分支写）。
- `tests/gui/v-branch.spec.ts`：隔离 dev server + 一次性 PG，Chromium
  与 WebKit（V1 head 分叉全链路、V2 事件边界分叉、V3 Library 入口 +
  ARIA/键盘 + 375px 无横向溢出）。
- 围栏对齐：world-write-gate-fence 新增 branch helper 锁序/门禁断言；
  d1 两个评审测试的迁移台账更新为 0001–0044。
- `.realm` v1 兼容：format version 不变；`REQUIRES_MIGRATIONS` 与导出
  manifest 增加 0044（world-import-service 台账探针同步），旧包导入
  新库不受影响（subset 断言），含分支记录的包导入未应用 0044 的库会
  以 MIGRATION_REQUIRED 提前 fail-closed。
