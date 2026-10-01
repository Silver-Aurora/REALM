# T10-B4 · Memory Snapshot/Delta 生产入口 + D1 退役围栏

> 批次 T10-B4（docs/development/EXPERIENCE-ITERATION.md T10 拆批，第二章缺陷 #11 的 snapshot/delta 部分 + D1 遗留链决策）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=44524c0；本批不改迁移 0001–0022、不改 LM Studio、不改 T6–T10-B3。

## 一、全库调用图与 M3 契约（读代码确认）

- `modules/memory/public.ts`：`CharacterMemoryService.snapshot()/delta()`
  （:384-407）已实现——snapshot 从 recall 产物确定性生成 content/itemIds/
  tokenCount，写 `memory_snapshots`（不可变，append-only 触发器）；delta 读
  snapshot 并与当前 cache epoch 比对，epoch 变化 → stale=true。
- `database/postgres/memory-repository.ts`：`createSnapshotAuthorized/
  deltaAuthorized` 已实现且经 PG 测试（tests/postgres-m3-memory.test.ts）。
- **缺口**：全库无 `/api` 生产入口，游玩回合也不消费 snapshot/delta——
  属 M3 §2.2/§2.3 已承诺但零接线。
- M3 硬契约：快照不可变；Delta 只增量；epoch 在 update/retract 后递进并使
  旧 delta stale；作用域服务端解析，客户端不得注入。
- `/api/memory` 现状：每请求 `createLocalPostgresPool` 从不回收（T9 发现过
  同款泄漏并已在 canon/图谱修复）——本批对 memory 路径做最小同型修复。

## 二、Snapshot/Delta API（向后兼容，受限入口）

- `POST /api/memory/snapshot`：body `{recordId?, kind?}`。
  kind 仅 `"representation"|"recall"`（缺省 representation，越界 400）；
  scope 由会话 principal + recordId 经 record-scope 解析（workspace/world/
  worldline/characterInstanceId 全服务端）；调用真实 `service.snapshot()`，
  content/itemIds/cursor/cacheEpoch/tokenCount 全服务端生成——客户端传
  content/itemIds/cursor/epoch 一律忽略。201 `{ok, snapshot}`。
- `GET /api/memory/delta?recordId&snapshotId`：同一 scope 解析 +
  `service.delta()`；snapshot 缺失/越界/非成员 → 404；200 `{ok, delta}`
  （stale/cacheEpoch/items）。
- 无 REALM_RUNTIME_DATABASE_URL → 503；读失败 → 500 安全文案；不回显连接串/
  错误 message/scope 私密字段。
- 既有 GET /api/memory（representation+relationships）与 POST（summarize）
  契约不变；两路由的 Pool 改共享池（同型最小修复，证据见迭代日志）。

## 三、D1 退役结论（本批不删除）

- 决策：`db/record-store.ts`、`db/story-record-repository.ts`、`db/schema.ts`、
  `drizzle/`、package.json 的 `db:d1:legacy:generate` 本批**保留**，标记为
  compatibility/migration reference——删除前置条件：①其类型/工具零生产引用
  且契约测试另有替代锚点；②历史迁移参考另有存档；③独立批次评审。
- 围栏：扩展 `tests/api-core-wiring.test.mjs`——活动 `/api/record`（GET/POST/
  SSE）、`/api/library`、`/api/memory` 等新入口静态证明不导入
  `db/record-store`/`db/story-record-repository`/`drizzle`/`createD1`/
  `submitPlayerTurn`；且这些 D1 文件不被任何 app/ 生产文件 import
  （静态扫描断言，非字符串口号）。

## 四、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 缺运行时 DB | 503 |
| F2 | 无 scope（缺记录/非成员） | 404 |
| F3 | kind 越界/snapshotId 缺或伪造 | 400 / 404 |
| F4 | 客户端伪造 content/itemIds/cursor/epoch | 忽略（服务端生成） |
| F5 | 读失败 | 500 安全文案，不回显内部细节 |
| F6 | snapshot UPDATE/DELETE | DB 触发器拒绝（既有不变量，测试断言） |

## 五、验收标准

1. Route PG 测试（真实临时库，t.after 拆库，受限 realm_runtime）：
   snapshot 404/400/伪造字段不生效/合法创建全字段；delta 初次为空、
   新增结论后出现增量、update/retract 后 epoch 递进且旧 delta stale=true、
   伪造 snapshot 404；snapshot 表 UPDATE/DELETE 被触发器拒绝。
2. m3-memory-completion/memory-recall 契约零回退；api-core-wiring 围栏通过。
3. typecheck/受影响 eslint/文档布局/git diff --check 全过；不跑全量/GUI。
4. 开发库基线核对不变。

## 六、交付步骤

1. 本规范（单独 commit）；2. API + 共享池最小修复 + 围栏 + 测试（独立
   commit）；3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
