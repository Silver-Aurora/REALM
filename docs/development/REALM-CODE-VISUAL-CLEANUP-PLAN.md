# REALM 全局代码 + 可视化审计与清理计划（REALM-CODE-VISUAL-CLEANUP-PLAN）

- 基线：`ef8552b`（origin/main 同步，工作树 clean）。本文件为纯审计/计划，不含生产代码改动。
- 审计方法：四域只读源码审计（冗余 / 性能可靠性 / 模型容错 / 安全边界，四个独立子审计各带 file:line 证据）+ 质量门实测 + 隔离环境真实 Chromium 可视化审查（截图在 `/tmp/realm-visual-audit/`）。
- 未覆盖边界（不夸大）：Android 真机/WebView、WebKit、两台物理设备 LAN、公网/OS app link 均未验收；性能数字只有既有 baseline 脚本实测，新增热点处标注「尚无测量」。

## 0. 质量门与可视化证据（本批实测）

- `npm run typecheck` exit 0；`npm run lint` exit 0；lobby/origin/launcher/desktop/genesis/theme 六组契约 **31/31 exit 0**；`npm test` **exit 0**（0 失败标记）。
- 可视化（隔离 server + scratch PG + 双身份，真实 Chromium）：11 张截图于 `/tmp/realm-visual-audit/`（desktop-record/library/lobby-create/longname/share/branch-tree/record-night/onboarding + mobile-record/lobby/login）。375px 下 scrollWidth−innerWidth ≤ 1（record/lobby/login），pageerror=0，无非预期 console error。长房名（40+ 字符）正确换行不溢出（desktop-lobby-longname.png、mobile-lobby.png）。
- 可视化观察备注（非 bug）：大厅「共享世界」为原生 `<select>`（直角样式，与设置页一致，沿用既定表单约定）；Tauri 壳为中文主体验（无 i18n 基建，不伪造三语）。

## 1. P0/P1/P2 问题总表

> 证据列格式：`文件:行`；类别：冗余/性能/可靠性/模型容错/安全；性质：fact（代码可证）/hypothesis（需测量确认）。工作量为粗粒度（S<0.5d，M≤2d，L>2d）。

### P0（阻塞主线，必须先做）

| # | 类别/性质 | 证据 | 影响 | 建议动作 | 验证 | 工作量 |
|---|---|---|---|---|---|---|
| P0-1 | 性能 fact | `app/api/record/events/route.ts:109,134-139` + `modules/application/local-record-service.ts:2272-2286` | record「SSE」实为 750ms 轮询：每拍全量投影（约 5 查询/拍/连接），JS 内再过滤丢弃；每个打开的标签页常驻 | record-projection baseline 曲线（事件数→查询数/payload）→ 改 LISTEN/NOTIFY 唤醒 + ordinal 增量 SQL | 既有 m4-streaming/interruption 测试须保留选项面；新增增量游标 PG 测试 | L |
| P0-2 | 性能 fact | `database/postgres/delivery-projection.ts:970-985` | EVENTS_SQL 无 LIMIT：envelope 全量事件随时间线线性无界；被 SSE 合并刷新（`app/realm-client.tsx:851-861`）、自演/初夜轮询、preview 授权、每回合 ×2 反复搬运 | 同上 baseline → 窗口化/增量 envelope 协议（viewer 序号与 recordEnvelopesEqual 语义须先设计） | 大量 GUI 假设全量时间线，需增量协议设计稿先行 | L |
| P0-3 | 模型容错 fact | `modules/orchestration/model-powered.ts:1690-1699` | 401/缺 key/配置错误一律包成 RetryableTurnError「稍后重试」并触发一次注定失败的 retryTurn（`modules/application/local-record-service.ts:2063`）；玩家得不到「检查 API key」的可操作诊断 | `modelCall` 对 `ModelConfigurationError`/`MODEL_AUTH_FAILED` 转 FatalTurnError 保留原 code；Retryable 仅限 429/5xx/timeout/network | mock gateway 抛 MODEL_AUTH_FAILED 断言 disposition=failed 且 diagnostic.code 保留 | M |

> P0-1/P0-2 的代码事实已确认，但当前没有新增热点的 before/after 基线；在事件数曲线、查询数、payload 和真实耗时跑完前，它们只算“候选阻塞项”，不得直接进行高风险增量协议改造。若基线未证明达到阻塞门槛，则降为 P1 性能批次；P0-3 可独立先修。

### P1（隔离可实现/验证）

| # | 类别/性质 | 证据 | 影响 | 建议动作 | 验证 | 工作量 |
|---|---|---|---|---|---|---|
| P1-1 | 性能 fact | `modules/application/local-record-service.ts:1884-1894` + `app/api/record/preview/route.ts:45` | preview 授权检查跑完整投影（含全量 EVENTS 扫描），每次连接/重连触发 | 窄化到 META-only 授权查询 | preview-cancel-auth 语义不变 | S |
| P1-2 | 可靠性 fact | `app/api/record/preview/route.ts:62-68` | 心跳 enqueue 抛错只 clearInterval，不 unsubscribe/close——previewHub 监听器泄漏（与已修复模式对比的漏网） | catch 内走完整清理（与 lobby/world-knowledge 路由同构） | preview-cancel-auth + 新增泄漏计数断言 | S |
| P1-3 | 性能 fact | `app/components/lobby-panel.tsx:133-151` | lobby SSE effect 依赖 `expanded`：每次展开/收起成员都拆建 EventSource + 2 次全量 GET + LISTEN 重建 | expanded 移出依赖改 ref 读取 | lobby GUI 既有套件 | S |
| P1-4 | 性能 fact | `app/api/lobby/events/route.ts:48,88-101` | 每个大厅 SSE 连接独占 pg.Client 且各跑 reap 写事务（N 客户端=N×reap/25s + notify 放大回路） | reap 单飞（advisory lock 或仅首连接执行） | postgres-lobby + 连接计数断言 | M |
| P1-5 | 性能 fact | `modules/application/lobby-service.ts:216-245` | 房间列表无 LIMIT 含全部 closed；notify 唤醒即全量重读 | open 优先 + LIMIT/游标（LAN 规模下影响小，但与 #6 联动） | lobby-baseline.mjs 验证 + 断言 closed 可见的测试同步 | S |
| P1-6 | 模型容错 fact | `modules/orchestration/model-powered.ts:1101-1150` | actor(propose) 阶段游离于 MODEL_STAGE_POLICIES 之外（无 timeoutMs/maxTokens/thinking 覆盖；观测 stage 不在枚举内） | 策略表新增 actor 项并 spread | model-stage-policy 测试模式复用 | S |
| P1-7 | 模型容错 fact | `modules/orchestration/model-powered.ts:1724-1743` 等 | 无 stage/turn 级总时限：空流重试链理论最坏 ~16 次 provider 请求×90s（~24 分钟）后才 fail | chatWithStream 重试链加总 elapsed 上限（或 turn deadline） | fake gateway 恒空流测 narrate 失败耗时 | M |
| P1-8 | 冗余 fact | `app/chatgpt-auth.ts:1-90` | 整文件死代码（Next 脚手架残留，引用计数 0，认证实际走 modules/identity） | 删除 | typecheck + api-core-wiring | S |
| P1-9 | 冗余 fact | `database/postgres/propagation-topology.ts:18`；`modules/actions/public.ts:11,42` | 死常量：topology 算法版本（活源在 modules/propagation/public.ts:13）、两份工具目录常量（活定义在 model-powered 内联） | 删除死常量（或收编，取前者） | postgres-propagation-*、action-suggestions | S |
| P1-10 | 冗余 fact | `app/components/record-types.ts:12,13,96,106`；`knowledge-graph-types.ts:17`；`guided-genesis-steps.ts:20` | 6 个引用计数为 0 的导出 | 删除/去 export | typecheck + frontend-record-contract | S |
| P1-11 | 安全 fact | `app/api/world/transfer-shared.ts:78` | 非领域错误回显原始 error.message（pg/zlib 细节可外泄给已登录用户） | 领域错误才透传，其余静态文案（merge/route.ts:96 范式） | export/import route 负例测试 | S |
| P1-12 | 安全 fact | `modules/application/world-import-service.ts:593` | 残留调试日志 `console.error("SCAN", …)` 输出表结构/行数 | 删除或结构化单行 | world-import 测试 | S |
| P1-13 | 模型容错 fact | `modules/orchestration/model-powered.ts:451-461` vs `modules/inference/model-call-observer.ts:88-99` | 观测 errorCode 脱敏两口径分裂（actor 侧未走白名单） | 统一 `sanitizeObservationErrorCode` | 注入自定义 code 断言归一 | S |
| P1-14 | 模型容错 fact | `modules/orchestration/model-powered.ts:474` + `modules/orchestration/public.ts:598` | narrator 权限校验正则纯中文：英文世界「environment 不得含人物指代」失效（越权叙事 fail-open） | 按输出语言补英文人称/称谓词表 | 英文 environment 含 "she turns to the player" 断言拒绝 | M |

### P2（后续清单，暂不实现）

| # | 类别/性质 | 证据 | 说明 |
|---|---|---|---|
| P2-1 | 性能 fact | `modules/application/library-service.ts:317-430` + `app/realm-client.tsx:278,405,458,493,521` | Library 巨型 jsonb_agg 快照 + 每次管理操作全量重读；规模小可接受，benchmark 后定 |
| P2-2 | 性能 fact | `database/postgres/memory-repository.ts:217-312` | sync_turn pending observations 无 LIMIT + 逐行 INSERT（积压场景 N+1） |
| P2-3 | 性能 hypothesis | `app/components/event-timeline.tsx` + `app/realm-client.tsx:1597-1601` | 时间线无虚拟化、分支对话框每事件一个 option；数百事件后成本待测 |
| P2-4 | 冗余 hypothesis | `modules/runtime/public.ts:572` `resumeTurn` 生产零调用 | 恢复 API 未接线——先确认是保留契约还是未完成功能缺口（非纯清理） |
| P2-5 | 冗余 fact | Library/BranchTree 类型服务端/客户端双份定义（`app/components/library-types.ts:47-107` vs `modules/application/library-service.ts:40-84`；`app/components/branch-tree-types.ts:6-57` vs `modules/application/branch-tree.ts:20-60`） | 属有意 wire 契约 + fail-closed normalize，建议保留并加形状对拍测试替代人肉同步 |
| P2-6 | 冗余 fact | theme-night-contract B 节仅字符串钉住 THEME_INIT_SCRIPT 与 applyTheme 等价性 | 补 jsdom 执行等价测试 |
| P2-7 | 冗余 hypothesis | `drizzle/` 空目录残留（git ls-files 为空） | rmdir 即可；4 个 d1-*-review 退役审计测试有意保留 |
| P2-8 | 安全 fact | `modules/inference/local-settings.ts:158` apiKey 末 4 位随 GET 下发 | LAN 共享主机跨会话可见；Stripe 同款可接受，可选收紧为仅 loopback 单用户返回 |
| P2-9 | 安全 hypothesis | Tauri webview 导航到远端 URL 后 10 个自定义 command 仍可 invoke（csp: null） | 实际危害限于停本地服务/开浏览器（入参均校验）；可用 isolation/CSP 收紧 |
| P2-10 | 性能 fact | `app/api/record/events` 自演 2.5s 全量 envelope 轮询与 SSE 并存（`app/realm-client.tsx:747-756`） | 随 P0-1/P0-2 主线一并设计瘦状态端点 |

## 2. 「不要动」清单

- DB schema/migration/RLS/GRANT、realm_transfer 受控函数、events append-only、record_heads CAS、锁序。
- 大厅/邀请/租约的授权语义（session principal、scrypt 密码、owner-only 世界绑定、白名单字段）。
- 模型 provider 目录/端点策略/key 处理（apiKey 只进 Authorization header）。
- 纸墨直角视觉体系与 light/night 双主题变量结构（除非有具体修复证据）。
- 既有 API 对外协议（record envelope 的 viewer-local 游标、writeToken、SSE 选项面 `pollIntervalMs/maxPolls`）——P0 主线只改内部实现与增量协议，不改对外语义。
- Tauri/Rust 命令清单与校验规则（只增不减）。

## 3. 分阶段实施路线（性能/可靠性/模型容错/冗余分开）

### 阶段 R1：低风险独立修（P1-1、P1-2、P1-3、P1-11、P1-12）
- 每项一个 commit + 对应 focused 测试先行（preview 授权窄化、监听器泄漏计数、lobby effect 依赖、transfer 错误回显、SCAN 日志）。
- 回滚点：单 commit revert 即可，互无依赖。

### 阶段 R2：冗余清理（P1-8、P1-9、P1-10、P2-7）
- 删除前置：引用搜索证据已在上表（每项均给 grep 计数）；逐文件单独 commit 便于定点回滚。
- 门：typecheck + lint + api-core-wiring + 相关 focused 套件 + 全量 npm test。

### 阶段 R3：模型容错（P0-3 → P1-6 → P1-13 → P1-14 → P1-7）
- 顺序：错误分类（401 不再误标可重试）→ actor 策略补齐 → 观测统一 → narrator 英文词表 → 重试总时限。
- 每项 fake-gateway 行为测试先行；不改 provider 目录/端点/语义。

### 阶段 R4：性能主线（P0-1、P0-2，联动 P1-4、P1-5、P2-10）
- 先写 baseline：扩展 `scripts/record-projection-baseline.mjs` 为事件数曲线（100/500/1000 事件 × 查询数/payload/耗时）；`scripts/lobby-baseline.mjs` 加多连接 reap 场景。
- 设计稿先行：增量事件协议（viewer 序号兼容、recordEnvelopesEqual、GUI 全量假设清单）+ LISTEN/NOTIFY 唤醒与轮询兜底。
- 分两个 commit 阶段：preview 授权/轮询路径先切增量（低风险面），envelope 窗口化最后动（高风险面）。

## 4. 冗余清理候选与删除前置条件

- `app/chatgpt-auth.ts`：grep 全仓引用计数 0（含 tests）→ 删除。
- `database/postgres/propagation-topology.ts:18` PROPAGATION_TOPOLOGY_ALGORITHM：零读取，活源 `modules/propagation/public.ts:13` → 删除该行。
- `modules/actions/public.ts:11,42` ACTOR_TOOL_SPECS/ENGINE_TOOL_SPECS：零引用 → 删除。
- `record-types.ts` 4 个 / `knowledge-graph-types.ts` 1 个 / `guided-genesis-steps.ts` 1 个死导出：零引用 → 删除。
- `drizzle/` 空目录：git ls-files 为空 → rmdir。
- 保留并加测试而非删除：Library/BranchTree 双份 wire 类型（P2-5）、runtime/public 公共契约面、4 个 d1-*-review 退役审计测试、5 条「API 先行 UI 未做」路由（memory delta/snapshot/relationship、worldline merge、propagation exposures——需在 docs 标注其定位而非下线）。

## 5. 性能 benchmark 方案（before/after 指标与采集方法）

| 域 | 脚本 | 指标 | 方法 |
|---|---|---|---|
| record 投影/SSE | `scripts/record-projection-baseline.mjs`（扩展） | 事件数 100/500/1000 × 查询数/payload 字节/avg ms（20 次取均值） | scratch PG + demo + 合成事件（既有形态） |
| lobby 列表/reap | `scripts/lobby-baseline.mjs`（扩展） | N 房间 × M 成员列表查询数/耗时；K 个 SSE 连接时 reap 写事务次数/25s | 同上 + 计数代理连接池 |
| Library 快照 | 新增 `scripts/library-baseline.mjs` | 世界数 5/20/50 × 查询数/payload/耗时 | 同上 |
| 模型调用 | `scripts/prompt-budget-baseline.mjs`（既有） | 每回合调用数/system+user 字符/dupBlocks | fake gateway（零真实 provider） |
| 前端渲染 | 新增 Playwright 采集（不入回归门） | 长记录（500 事件）首屏 TTI 近似、时间线滚动帧时间 | trace + performance.now 采样 |

门槛规则：只有 before/after 同一环境同一负载的差值 ≥20% 才保留改动；否则回滚并记录「无可证提升」。

## 6. GUI 回归矩阵与截图验收标准

- 现有回归：y/z/z2/z3/z4/z5/w/x 系列（大厅/租约/邀请/壳/响应式/主题）+ t12/l/s 系列，全隔离 runner。
- 视觉抽查（每阶段末复用本轮一次性 spec 形态重跑）：viewport 375/768/1440 × record/lobby/library/branch-tree/onboarding/login；验收标准：scrollWidth−innerWidth≤1、无白块/黑字黑底、焦点可见、pageerror=0、长文本换行、overlay 互斥。本轮采集 spec 为一次性审计工具（`tests/gui/va-visual-audit.spec.ts` 形态，运行后未入库），截图证据在 `/tmp/realm-visual-audit/`。
- 截图不入库（/tmp），引用 evidence 路径于阶段验收记录。

## 7. 未完成边界与下一轮建议

- 未验收：Android 真机/WebView、WebKit、两台物理 LAN、公网/OS app link——本轮 GUI 全部是单机 Chromium + 隔离环境。
- 尚无测量：event-timeline 长列表渲染、Library 快照放大、preview 授权窄化后的实际收益（先 R1 再测）。
- 下一轮建议：R1 快修批（P1-1/2/3/11/12/13）→ R2 冗余删除批 → R3 模型容错批 → R4 性能主线（设计稿先行）。
