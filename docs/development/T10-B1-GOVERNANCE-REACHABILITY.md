# T10-B1 · 世界治理可达性审计 + Worldline Merge 作用域修复

> 批次 T10-B1（docs/development/EXPERIENCE-ITERATION.md T10 拆批，第二章缺陷 #13 的审计收口第一步）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=fbacdb4；本批不改迁移 0001–0021、不改 LM Studio、不改 T6–T10-A 行为、不跑 GUI/全量。

## 一、当前调用图逐项结论（2026-08-21 全库实查，纠正「worldline 全部零调用」的过时说法）

| 模块 | 生产可达性 | 结论 |
|---|---|---|
| `modules/worldline/canon.ts` | **生产可达**：`app/api/canon/route.ts`（T9 已去 demo 硬编码 + membership 校验），K 组图谱 UI 真实覆盖 | 已接线 |
| `modules/worldline/branching.ts`（classifyWorldlineConflict） | **生产可达**：`app/api/worldline/conflict/route.ts`（纯分类器 API） | 已接线 |
| `modules/worldline/merge.ts` | **生产可达**：`app/api/worldline/merge/route.ts`——但路由硬编码 `ws_demo/world_ember_coast` 且请求体可任意写 `operator` 审计主体 | 已接线但作用域/审计造假（本批修复） |
| `modules/worldline/conflict-detection.ts`（detectCausalConflicts 因果依赖图） | 零生产调用（merge 自带 plan 内冲突逻辑，不用它） | **承诺但缺 runtime 接线**（M5 第一批「确定性冲突检测」的因果增强件）——候选后续独立批次，本批不接不删 |
| `modules/worldline/semantic-conflict.ts` | 零生产调用（仅类型被引用 + repository 实现零实例化） | **M5 §9 明确第二批暂缓**——保留并如实标记 |
| `modules/propagation/public.ts` + `worker.ts` | 零生产调用（repository/测试除外）；无 enqueue 入口、无 worker 启动方 | **M5 §7 明确第二批暂缓**（离线调度）——保留并如实标记，不当死代码删 |

## 二、Merge 路由修复（唯一代码改动面）

`POST /api/worldline/merge` 现状三处造假/硬编码，修复契约：

1. **worldId 显式必填**（请求体 `worldId` 字段；缺失/空值 → 400 `INVALID_REQUEST`）。
   workspace 永远取当前本地 workspace 常量（`LOCAL_RECORD_SCOPE.workspaceId`），
   不信任客户端传 workspaceId。
2. **作用域校验**：复用 T9 的 `resolveWorldScopeForMember`（membership + 当前
   active 世界线解析）——世界不存在或非成员 → 404，不泄露存在性。
   `service.merge` 用解析出的 `scope.workspaceId/scope.worldId`。
3. **审计主体**：`operator` 恒等于已解析的 `principalId`；请求体 `operator`
   字段被忽略（不再写入审计）。
4. **连接池**：复用 T9 的 `getSharedRuntimePool`（按连接串进程级复用），
   不再每请求新建永不回收的 Pool。
5. **保留契约**：sourceA/sourceB/idempotencyKey 校验、幂等重放、dryRun 预览、
   rejected→409/merged→200、WorldlineMergeError→404 全部不动；合并算法零改动
   （`worldlineExists` 已按 worldId 过滤，无跨世界线缺口——实查
   `database/postgres/worldline-merge-repository.ts:143-152`）。

## 三、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 缺失/空 worldId | 400 INVALID_REQUEST |
| F2 | 世界不存在/非成员 | 404（不泄露存在性） |
| F3 | sourceA/sourceB 世界线不存在 | 404 WORLDLINE_NOT_FOUND（既有） |
| F4 | 请求体 operator 伪造 | 忽略；审计行 operator=已解析 principalId |
| F5 | 幂等键重放 | 返回既有合并结果（既有契约） |
| F6 | hard 冲突 | 409 + rejected 审计行（既有契约） |

## 四、验收标准

1. Route focused 回归（真实临时 PG 库，t.after 拆库，不污染开发库）：
   缺失 worldId 400；未知世界 404；合法成员 dryRun 预览 200 且真实 merge 的
   审计行 operator=已解析 principal（请求体 operator 不生效）；source 不存在
   404；幂等重放同 mergeId。
2. 既有 `tests/m5-batch2.test.ts` / `tests/postgres-m5-batch2.test.ts` 零回退。
3. typecheck/受影响 eslint/文档布局契约/git diff --check 全过。
4. 不跑 GUI、不跑全量；开发库基线核对（worlds/accounts/first_nights/
   action_receipts/图谱/canon/files/articles）。

## 五、交付步骤

1. 本规范（单独 commit）；
2. 路由修复 + route PG 测试（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。

## 六、验证期实锤追加（不改写上文结论）

- 路由修复后的 focused 测试实锤一个更深的存量缺口：真实合并（非 dryRun）
  的拓扑写入（createMergedTopology → INSERT worldlines/stories/records）在
  realm_runtime 受限角色下 permission denied——0004 硬化收回了这些表的
  INSERT，而 M5 batch2 的 PG 测试全部用 owner 池，生产路径从未真正可达。
  修复：新增迁移 0022（只补授三表 INSERT，0001–0021 不动），规范「不改迁移」
  指不改旧迁移内容；新增迁移的理由即本节。
