# REALM 全部后续开发待办收口计划

> **For Hermes:** 按当前仓库 HEAD 逐项执行并独立验收；不要把任何单测、导出或静态契约当成生产接线证据。

**Goal:** 完成 T11-H 之后仍明确存在、且当前具备实现条件的全部开发待办，不在 T11-I 处停下；对依赖外部产品/网关/凭据的项目做硬证据审计并明确封存边界。

**Architecture:** 保留 realm_runtime 最小权限。受众映射的 Web 管理面不得直接拿 owner `DATABASE_URL`；采用数据库 `SECURITY DEFINER` 的单动作追加函数，函数内部重新验证 owner、节点、continuity 和作用域，Web 只获得 `EXECUTE`，表本身仍不授 INSERT/UPDATE/DELETE。UI 只提供 list/add，保持 append-only 治理事实。GUI 验收优先在隔离临时数据库与无 token 本地会话进行，不改 `realm_dev`。

**Tech Stack:** TypeScript、vinext/React、PostgreSQL 17、Node test、Playwright Chromium/WebKit、systemd user service。

---

## 当前待办边界

1. **T11-I-A：受众映射产品面**：owner 在 Knowledge Graph/Canon 操作面查看拓扑，追加 node→continuity mapping；非 owner 只读；重复追加幂等；越界/失效 fail-closed；不提供删除/修改假象。
2. **T11-I-B：GUI 真实交互验收**：在隔离库运行 T11-H operator surface 和映射 UI 的真实 Playwright 点击；无凭据时不猜 token、不绕过门禁，改用明确关闭的 tokenless 本地隔离会话或把受保护路径标成外部阻塞。
3. **T11-I-C：记忆面刷新闭环**：回合提交后，后台 `sync_turn` 完成时 UI 能在不重进页面的情况下重读 memory；不增加模型调用，不改变记忆权限。
4. **T11-I-D：Worker 运行验证扩展**：隔离库双 Worker advisory-lock 互斥、stale recovery、永久失败和清理；不向 `realm_dev` 注入 Job，不修改常驻 Worker loop/unit。
5. **T11-I-E：T11 系列复盘与台账收口**：回读 T11-A2/B/C/D/E/F/G/H，修正文档过时状态，记录仍受外部前置阻塞的事项。
6. **外部前置项**：M4 tool-pause 需要供应商流式 tool-call 事件；secret 非 `private_letter` 需要产品/安全授权模型；生产压力/容量/多副本验证不能用本地 harness 冒充。为这些项补 capability/preflight 证据和明确 deferred，不造假接线。

## 执行纪律

- 每个行为变更严格 RED → GREEN → REFACTOR；先写一个会失败的测试并保存原始退出码。
- 每个主题独立提交，禁止 `git add -A`，不 push。
- 每批先 focused，再 typecheck/lint，再完整 `npm test`、render 和文档布局。
- 迁移后查 `realm_schema_migrations`；临时库按专属前缀盘点，未经授权不 DROP 未知数据库。
- 所有真实 GUI/Worker 测试后独立查询并清理数据；凭据只从本机环境读取，不输出、不写进日志。

## T11-I-A：受众映射产品面

### A1：数据库追加函数（RED → GREEN）

- Test：`tests/postgres-propagation-node-audiences-governance.test.ts`
- Modify：`database/postgres/migrations/0028_propagation_node_audiences_owner_append.sql`
- Contract：`SECURITY DEFINER`、固定 `search_path`、owner membership、同 workspace/world/worldline、active node、active continuity、重复幂等；`realm_runtime` 仅 EXECUTE，表仍无直接写授权。
- Verify：临时库迁移后分别以 owner/runtime/非 owner 运行成功与失败矩阵；更新迁移清单；确认触发器和 RLS 在位。

### A2：API/domain route

- Test：扩展 `tests/postgres-canon-qualification.test.ts` 与新增 governance route test。
- Create/modify：`database/postgres/propagation-node-audiences.ts`、`app/api/propagation/node-audiences/route.ts`、`database/postgres/public.ts`。
- Contract：GET qualification topology；POST add 只允许 owner；服务端从 session 解析 principal；请求体不接受 workspace；返回 stable JSON；不提供 update/delete。
- Verify：malformed body、跨 world、inactive、non-owner、duplicate、runtime direct INSERT 全部有证据。

### A3：UI 与 contract

- Test：`tests/frontend-record-contract.test.ts`、新增 `tests/propagation-node-audiences-ui-contract.test.ts`。
- Modify：`app/components/knowledge-graph-panel.tsx`、`app/components/knowledge-graph-types.ts`、`modules/i18n/public.ts`、必要 CSS。
- Contract：owner 看到节点映射治理折叠面；列表显示 active/inactive 状态；追加选择只来自 qualification payload；malformed payload 隐藏写控件；追加后重新读取资格；非 owner 只读。
- Verify：三语 key、键盘可操作、失败不乐观更新、重复追加显示“已存在”。

## T11-I-B：GUI 验收

- Test：新增 `tests/gui/t11-i-audience.spec.ts`，复用 `tests/gui/helpers.ts`。
- Setup：创建专属临时数据库并迁移/seed；启动无 `REALM_ACCESS_TOKEN` 的隔离 dev server；`GUI_BASE_URL` 指向隔离端口；测试后精确清理并盘点。
- Flow：打开图谱 → 正史审核 → owner 资格面 → 选择 node/continuity → add → list 反映 mapping → 重复 add 幂等 → 非 owner 上下文只读 → malformed API payload 不显示写控件。
- Verify：Chromium 必跑；WebKit 作为跨引擎补充；pageerror=0、console 无新增 error、截图/trace 仅保留失败证据。

## T11-I-C：记忆面刷新

- Test first：`tests/memory-prefetch-application.test.ts` 增加“committed event 后调度 memory reread，重复事件去抖”的失败用例；前端 contract 增加取消卸载与动态知识关闭用例。
- Modify：`app/realm-client.tsx`，抽出小型可测试 `scheduleMemoryRefresh`/cleanup 逻辑；不改 `/api/memory` 权限和数据格式。
- Verify：真实 API/PG 现有 memory pipeline 回归；GUI 用隔离回合证明无需重进页面即可看到更新，失败只保留旧快照。

## T11-I-D：Worker 运行验证扩展

- Test：`tests/postgres-propagation-worker-isolation-acceptance.test.ts`。
- Harness：同一临时库启动两个真实 `scripts/propagation-worker.mjs` 子进程；创建显式 restricted、secret/private_letter、非法 secret channel、stale job；断言单锁、done/permanent_failed、零错误产物和退出清理。
- Verify：任务库清零、runtime 连接释放、常驻 systemd 服务不受影响；不修改生产 Worker。

## T11-I-E：复盘与最终收口

- Modify：`STATUS.md`、`docs/development/EXPERIENCE-ITERATION.md`、`docs/README.md`。
- Record：每个主题的真实提交、原始退出码、数据库/服务边界、GUI 是否受凭据墙限制、外部前置事项。
- Final：`npm test`、`npm run lint`、`npx tsc --noEmit`、文档布局、`git diff --check`、migration audit、inventory、systemd 状态、clean worktree。

## 外部前置封存标准

- M4：当前 gateway 能力若仍只有完整响应/文本 chunk，则只追加 capability audit 与明确阻塞，不引入模拟事件或假调用方。
- secret 扩展：没有 recipient/audience/clearance 产品决策时保持 `private_letter` 唯一合法 secret channel；用 negative acceptance 证明拒绝路径。
- 生产规模：隔离 harness 只证明语义和资源边界；容量、网络故障、多副本需独立运维窗口，不在本地开发批次伪造“生产已验收”。

## 完成定义

所有当前可执行项均有代码、测试、独立运行证据和文档收口；不可执行项均有当前 HEAD capability/preflight 证据、阻塞原因和 fail-closed 行为。最终状态不再使用“下一阶段”掩盖未处理的已知待办。

## 当前执行记录（2026-08-22 18:36 +08:00）

本节是对上方计划的追加事实，不改写 T11-H 历史记录。当前代码基线为 `6012495`；本节写入时工作树含本批 GUI/Worker 验收测试与一个 UI 状态修正，尚未形成最终收口提交。

| 项目 | 当前事实 | 证据 |
|---|---|---|
| T11-I-A 受众映射产品面 | 已实现并通过局部 PG/API/UI contract 验证 | `a9dc74b`；`T11I_AUDIENCE_GREEN_EXIT=0`、`T11I_AUDIENCE_API_GREEN_EXIT=0`、`T11I_DB_GOVERNANCE_EXIT=0`、`T11I_UI_CONTRACT_EXIT=0`、`T11I_UI_TSC_EXIT=0` |
| T11-I-B GUI | 修正追加后刷新清掉成功状态的真实 UI 缺陷后，隔离 Chromium 2/2 通过 | `tests/gui/t11-i-audience.spec.ts`；`T11I_GUI_HARNESS_EXIT=0`；`2 passed`；临时库 `realm_t11i_gui_*` 残留 0；`realm-dev.service` 恢复 `active` |
| T11-I-C memory refresh | 已实现并提交，committed event 后有界去抖重读，卸载/动态知识关闭均有边界 | `6012495`；`T11I_MEMORY_REFRESH_GREEN_4_EXIT=0`、`T11I_MEMORY_TSC_2_EXIT=0` |
| T11-I-D Worker 扩展 | 真实入口同库双进程互斥与 stale recovery 已加入 acceptance 并通过 | `tests/postgres-propagation-worker-runtime-acceptance.test.ts`；`T11I_WORKER_MULTIPROCESS_EXIT=0`；日志含 `advisory lock acquired`、`recovered 1 stale job(s)`、`job done`、`exit code=0`；T11E/T11H/T11I 临时库残留均为 0 |

### 仍未扩大或冒充完成的边界

- 当前 GUI owner 验收使用关闭 token gate 的隔离本地会话；真实多人 non-owner 角色矩阵仍不能宣称完成。现有登录流程会给新 principal 自动授予默认世界 `owner`，这仍是多人治理发布前的授权语义阻塞。
- secret 继续只允许 `private_letter`；`official_bulletin`、`market_rumor` 不因本批验收而开放。
- M4 tool-pause、生产吞吐/多副本/网络故障/容量验证仍是外部前置；本地 acceptance 不替代生产验收。
- 完整 `npm test`、全量 lint/typecheck、迁移台账和最终文档/工作树收口在本记录写入时仍待执行。

## 最终收口记录（2026-08-22 19:41:27 +08:00）

本批当前可执行项已完成代码、真实隔离运行、全量回归和 live 边界核对；以下事实均来自本次现场命令，不采信旧异步审计回执。

- GUI：`T11I_GUI_HARNESS_EXIT=0`，Chromium `2 passed`；修复了追加后 qualification 刷新清掉成功状态的真实缺陷。测试使用隔离 `realm_t11i_gui_*` 数据库与 `10099` 端口，残留 0，`realm-dev.service` 测后为 `active/enabled`。
- Worker：`T11I_WORKER_MULTIPROCESS_EXIT=0`；真实入口双进程同库竞争，第二进程 lock error 退出，首进程恢复 stale `running` Job 后完成并 clean exit。T11E/T11H/T11I 临时库盘点均为 0，常驻 Worker 仍为唯一进程。
- 全量门禁：`npm test` exit 0——core `183/183`、contracts `72/72`、application `33/33`、frontend `30/30`、PostgreSQL `64/64`、build complete、render `3/3`；独立 `npm run lint`、`npm run typecheck`、documentation-layout、`git diff --check` 全部 exit 0。
- live migration：在 loopback `realm_dev` 上只应用 `0028_propagation_node_audiences_owner_append.sql`，runner exit 0；台账 `0028`/28 条。权限实查：函数 `SECURITY DEFINER=true`、固定 `search_path=pg_catalog, public`、`realm_runtime EXECUTE=true`；治理表 `SELECT=true`、`INSERT=false`。
- live inventory：临时库 0；`realm_dev` worlds=2、accounts=1、propagation nodes/routes=3/2，campaign/packet/exposure/job/semantic evidence 全 0；`realm-dev.service` 与 `realm-propagation-worker.service` 均 `active/enabled`。

### 最终边界

- GUI 真实证据是隔离 tokenless 单用户 owner 路径；当前登录流程给新 principal 自动授予默认世界 `owner`，因此多人 non-owner/operator 治理语义仍是发布前安全决策，不包装成已解决。
- secret 仍仅允许 `private_letter`；M4 tool-pause 缺供应商流式 tool-call 能力；生产吞吐、多副本、网络故障、容量验证仍需独立运维窗口。
