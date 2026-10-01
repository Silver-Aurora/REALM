# T11-A2 · 知识图谱 graph-specific SSE 自动刷新（持久化失效账本 + LISTEN/NOTIFY 唤醒）

> 批次 T11-A2（T10-B22-A 评审 §六前置已由用户产品决策选定 SSE 方案；本批从 contract-only-deferred 推进为真实实现）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=1070bc8；架构方向（用户决策）：轻量图谱失效事件 + 既有 GET 权威回读——SSE 不传整张图，不复用 record/events SSE，不做轮询降级假接线。

## 一、写路径审计（当前 HEAD 实读，全部枚举）

图谱/CANON 写路径全集（除注明外均经 `withWorkspaceTransaction` 事务）：

| # | 写路径 | 位置 | 事务边界 |
|---|---|---|---|
| W1 | upsertEntity（实体新建/摘要编辑） | database/postgres/world-knowledge-repository.ts:19 | 独立事务 |
| W2 | appendClaim（提交 Claim） | world-knowledge-repository.ts:47 | 独立事务 |
| W3 | appendRelation（建立关系；service 层 projectRelation 调用） | world-knowledge-repository.ts:111 | 独立事务 |
| W4 | createArticle | world-knowledge-repository.ts:160 | 独立事务 |
| W5 | createProposal（canon propose） | database/postgres/canon-repository.ts:12 | 独立事务 |
| W6 | markDecided（canon reject/defer） | canon-repository.ts:143 | 独立事务 |
| W7 | mergeProposal（canon merge：UPDATE proposal + INSERT claims/articles/revisions） | canon-repository.ts:59 | 单事务多写 |
| W8 | 晶化入图谱（T9 产生侧：upsertEntity + 白名单 Claim） | modules/application/local-record-service.ts:1256–1290 | best-effort 独立步骤，**走 W1/W2 repository 方法**——repository 层打点自动覆盖 |
| W9 | 酒馆导入直写 world_articles | modules/application/tavern-import-service.ts:123 | 导入事务内直写（不经 W4），需在该事务内补失效记录 |

- 调用方覆盖：W1–W4 由 `app/api/world-knowledge/route.ts` POST 三 action
  与 W8 共用 repository 方法——**在 repository 方法内打点即覆盖全部
  调用方**，无漏接；W5–W7 由 `app/api/canon/route.ts` propose/decide
  驱动；W9 唯一直写点已定位。
- 不纳入（非图谱快照数据）：causal_edges（冲突预览用，不进 GET 快照）、
  worldline merge 拓扑写（worldlines/stories/records）、genesis/世界管理台
  写——面板快照 = entities/claims/relations/articles + canon proposals。
- 无实际变化不发事件：canon propose 低于审核层级返回 null（无写入）；
  POST 校验失败/409/500 无写入——打点只在事务内成功路径。
- 授权/迁移约束：realm_runtime 走最小授权（新表需 SELECT+INSERT grant +
  ENABLE/FORCE RLS + workspace policy，沿用 0016/0019/0020 形态）；新增
  正式迁移 0024，不改写历史迁移；不移除/恢复任何 D1/Drizzle tooling。

## 二、事件契约

- 表 `graph_invalidation_events`（迁移 0024）：`workspace_id` /
  `world_id` / `worldline_id` / `cursor bigint GENERATED ALWAYS AS IDENTITY`
  / `id text` / `kind text` / `reason text` / `created_at timestamptz`；
  PRIMARY KEY (workspace_id, cursor)（cursor 全表单调，重放按 cursor 排序）；
  workspace FK ON DELETE CASCADE；ENABLE+FORCE RLS +
  realm_workspace_isolation policy；append-only 守卫触发器（仿
  canon_revisions 0010 形态）；GRANT SELECT, INSERT TO realm_runtime。
- kind 枚举：`entity`（W1）/ `claim`（W2/W8 claims）/ `relation`（W3）/
  `article`（W4/W9）/ `canon_proposal`（W5）/ `canon_decision`（W6）/
  `canon_merge`（W7）。reason 为短动作标识（如 `upsertEntity`、
  `tavern-import`），**不携带**实体全集、Claim 内容、提案 rationale 或
  任何敏感数据。
- 原子性：失效记录与业务写**同一事务**插入；事务回滚则失效记录同样
  回滚——不存在「业务回滚但事件可见」。
- NOTIFY：事务内插入后 `SELECT pg_notify('graph_invalidation', …)`，
  payload 仅 {cursor, worldId, worldlineId, kind}；PG 语义保证提交后才
  送达。NOTIFY 只是低延迟唤醒——**持久化账本才是重连/漏事件恢复依据**，
  payload 不作为唯一事实来源（收到通知后按 cursor 查账本）。
- 同一事务多次写合并为一个事件（W7 merge 一个 `canon_merge`；W8 晶化
  每个 repository 调用各一事件——它们本就是独立事务；W9 导入事务末尾
  一个 `article` 事件），cursor 单调、回读最终状态正确。

## 三、SSE 路由契约（`app/api/world-knowledge/events/route.ts`，Node runtime）

- 鉴权：`resolveRequestPrincipal` + `resolveWorldScopeForMember`（与
  world-knowledge GET 同源）——缺 worldId 400、未知/非成员世界 404
  不泄露存在性、未认证 401；同源会话凭据，不引入 URL token。
- 作用域：连接绑定 workspace+world+worldline（active 世界线解析与 GET
  一致）；**其他世界/世界线的事件绝不发送**（服务端按作用域查账本过滤，
  不依赖客户端过滤）。
- 事件形态：`event: graph-invalidation`、`id: <cursor>`、
  `data: {cursor, worldId, worldlineId, kind}`、`retry: 5000`；心跳
  `: ping` 每 25s。
- Last-Event-ID 重放：连接建立后按当前作用域查 `cursor > lastEventId`
  的账本记录依次补发；**初次连接竞态**——先 LISTEN 再查账本补发，
  期间到达的 NOTIFY 触发按 `cursor > lastSentCursor` 的重查去重，
  不丢不重。
- 生命周期：每连接一个专用 `pg.Client`（REALM_RUNTIME_DATABASE_URL，
  realm_runtime；LISTEN 无需额外授权）——专用连接避免长占共享池；
  request abort 时清理 heartbeat 定时器、NOTIFY 监听回调、`client.end()`；
  LISTEN 连接断开依赖客户端 EventSource 自动重连（携带 Last-Event-ID
  从账本恢复）。
- 背压：事件只发失效标记，频率天然受写路径频率限制；路由不做无限
  缓冲，通知处理为重查账本（有界）。

## 四、前端契约（knowledge-graph-panel.tsx）

- 按 worldId 建立一个 graph-specific EventSource
  （`/api/world-knowledge/events?worldId=…`）；worldId 变化或卸载时
  close；**绝不订阅 /api/record/events**。
- 收到 graph-invalidation 事件 → 触发既有 `load()`（GET 仍是权威快照），
  不相信 SSE payload；短时间多事件 300ms 防抖合并为一次刷新。
- 复用 T10-B10 全部语义：loadSeqRef 序号竞态、mountedRef 卸载防护、
  Promise.all 双端点整体成败、旧快照保留、selectedId 保留/清空、
  POST 回读、手动刷新按钮。
- 可见性（克制）：连接正常不显示任何常驻指示；EventSource 断开/重连
  中显示一行 role=status「实时同步已断开，正在重连…可手动刷新」——
  不把断线伪装成「数据为空」，不加轮询定时器，轮询不作隐形 fallback。
- SSE 故障不阻塞编辑与手动刷新（两者本就走独立路径）。

## 五、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 写事务回滚但事件可见 | PG 测试：rollback 后账本零记录 |
| F2 | 跨世界/世界线泄露 | PG+路由测试：异世界写入不产生本连接事件 |
| F3 | 断连丢事件 | Last-Event-ID 重放测试：重连补发不丢不重 |
| F4 | SSE 故障阻塞编辑/手动刷新 | 前端契约：独立路径；K7 回归 |
| F5 | 事件携带敏感内容 | schema 单测 + 契约断言 payload 白名单字段 |
| F6 | 某写路径漏接 | 写路径覆盖测试：W1–W9 逐路径事件证据 |

## 六、验收标准

1. 事件 schema/cursor 单测；2. 写事务成功产生事件/回滚不产生事件；
3. Last-Event-ID replay、重复/乱序去重、跨 world/worldline 不泄露；
4. SSE 鉴权/成员边界/断开清理/heartbeat；5. W1–W9 逐写路径事件证据；
6. 前端只回读 GET、突发合并、旧响应不覆盖新响应、卸载不 setState
   （静态契约 + K8）；7. T10-B22 围栏翻转为正向 contract（graph SSE
   存在、record/events 未被冒充、面板订阅正确路由、cursor/replay 证据）；
8. K8 Chromium（外部写入→不点刷新→新节点出现）+ K7 回归；
9. typecheck/受影响 eslint/git diff --check/schema contract（0024）/
   focused PG 全部原始 exit 0；10. 清理后 DB 基线 2/1/0/0/0/0、
   realm_t% 临时库 0、realm-dev.service active、工作树 clean。

## 七、范围外清单（本批不做）

T11-B propagation/semantic-conflict 接线、M4 模块、record/events 与
record/preview 语义、轮询 fallback、causal_edges/拓扑写入的事件化、
退役链历史文档改写、GUI 全量、push。

## 八、交付步骤

1. 本规范 + docs 索引；2. 迁移 0024 + 失效账本助手 + 九条写路径打点；
3. SSE route + replay/auth/生命周期 + PG/路由测试；4. 前端 EventSource/
   竞态/状态 UI；5. 围栏翻转 + K8 + K7 回归；6. STATUS/EXPERIENCE 收口。
