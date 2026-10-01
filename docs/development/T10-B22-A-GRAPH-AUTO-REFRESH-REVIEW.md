# T10-B22-A · 图谱自动刷新（轮询/SSE）资格评审（结论：维持手动刷新，自动刷新 deferred）

> 批次 T10-B22-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B10-A §八范围外项「自动轮询/SSE」的资格评审）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=329fab7；本批只评审——不加轮询定时器、EventSource、SSE route、API 字段、数据库列；不改图谱组件/API/repository/migration/outbox/record SSE。

## 一、既有手动刷新闭环复核（当前 HEAD 实查在位）

`app/components/knowledge-graph-panel.tsx`（T10-B10-A）：

- 初次 load + 显式「刷新图谱」按钮（`aria-label="刷新图谱"`），共用同一条
  `load()`：`Promise.all` 双 GET `/api/world-knowledge` + `/api/canon`，
  显式 worldId + `cache: "no-store"`；
- loading/loadedOnce 状态机、请求序号竞态（loadSeqRef）、mountedRef
  卸载防护、失败 notice 可重试、旧快照保留、编辑 POST 成功后回读。
- K7（tests/gui/k-graph.spec.ts）验证手动刷新契约；本批不跑 GUI，锚点由
  静态围栏守（§六）。

## 二、失效信号面实锤（2026-08-21 重扫，文件/符号级）

| 候选信号 | 实查结论 | 证据 |
|---|---|---|
| 图谱读 API 游标/版本 | **无**——GET 返回裸数组（entities/claims/relations/articles），无 snapshotVersion/graphCursor/etag；仅 `Cache-Control: no-store` | `app/api/world-knowledge/route.ts:99-122` |
| Canon 读 API 游标 | **无**——GET 返回 proposals 数组，无游标 | `app/api/canon/route.ts:85-103` |
| 行级时间戳 | world_entities 有 `updated_at`（0010:20），但**未暴露**为图谱级失效游标（无 MAX(updated_at) 端点） | `database/postgres/migrations/0010_world_governance.sql:20` |
| 图谱 SSE route/topic | **无**——全 app/api 仅两条 SSE：record/events（记录级 committed 事件）与 record/preview（回合预览），均与图谱无关 | `grep -rln "text/event-stream" app/api/` |
| 客户端 EventSource | 仅 realm-client.tsx 两处，均指向 `/api/record/events`——**不是**图谱推送 | `app/realm-client.tsx:369,470` |
| DB LISTEN/NOTIFY | **无**——database/postgres 与 app/modules 零 LISTEN/NOTIFY/pg_notify | grep 实查 |
| outbox 失效事件 | knowledge/canon repository 零 outbox 关联（outbox 属回合/交付事件链） | `database/postgres/world-knowledge-repository.ts`、`canon-repository.ts` |

**结论：当前不存在 graph-specific invalidation contract**——没有可关联
world-knowledge/canon 写路径的失效语义。`/api/record/events` 的 SSE 与
普通 `controller.enqueue` 均不得当作图谱推送信号。

## 三、polling 方案评估（若未来选轮询）

- 触发信号：定时器——实现最简单，但每个 interval 都是两次 GET 的固定
  成本（空负载也付）；必须有**请求预算/频率**与**焦点策略**（页面不可见
  时暂停）的产品决策，当前不存在。
- worldId/成员权限：每次请求服务端复核（既有 resolveScope 在守），轮询
  不削弱；但面板打开期间 worldId 不变，无需额外契约。
- visibility/缓存：`cache: "no-store"` 保证读新；双端点 Promise.all 整体
  成败语义（B10）可直接复用。
- 竞态：编辑 POST 回读与轮询响应交叠——B10 的请求序号机制（旧响应不得
  覆盖新响应）已覆盖此场景，可复用。
- 退后台/卸载/重连/背压：卸载由 mountedRef 覆盖；退后台需要
  document.visibilitychange 策略（缺失）；轮询天然无重连问题，但失败
  退避（连续失败降频）策略缺失。
- **缺口汇总**：轮询预算/间隔、焦点暂停策略、失败退避——均为产品决策，
  不是技术阻塞。

## 四、SSE 方案评估（若未来选推送）

- 需要新建 graph-specific 事件契约：写路径（knowledge/canon POST 与晶化
  回读管线）发出失效事件、事件携带 worldId 与游标、按连接做成员权限
  复核、断线重连与退避、与编辑 POST 回读的竞态规则。
- 现状缺口：无写路径事件源（零 NOTIFY/outbox 关联）、无图谱 topic
  route、无游标定义——**是契约级缺失，不是包一层 EventSource 能补的**。
- record/events SSE 的 cursor/重连模式可作设计参考，但语义是记录级
  committed 事件，不可复用为图谱失效。

## 五、结论

**维持手动刷新；自动刷新（轮询/SSE）为 deferred。** 没有 graph-specific
invalidation contract，不得声称「自动刷新已具备」。T10-B10-A 的手动
刷新闭环是当前且唯一的刷新语义。

## 六、最小未来实现批前置（均需先立产品决策，当前未实现）

1. 产品决策二选一：轮询（预算/间隔、页面不可见暂停、失败退避参数）
   或 SSE（图谱失效事件契约：事件源、游标、topic route）；
2. 权限复核：SSE 按连接成员校验 / 轮询沿用逐请求 resolveScope；
3. 竞态规则：自动响应 vs 编辑 POST 回读 vs 手动刷新的统一序号语义
   （可沿用 B10 loadSeqRef 机制，需写入规范）；
4. 重连/退避（SSE）或退后台/降频（轮询）策略成文；
5. 验收 GUI：新增 K8（外部写入 → 不点刷新 → 节点按策略出现）+ K7 不
  回归；不跑通不声称完成。

## 七、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 把 record/events SSE 或 controller.enqueue 当图谱推送 | 评审 §二实锤 + 围栏断言（面板零 EventSource） |
| F2 | 评审/文档把自动刷新写成已具备 | 围栏 deferred 语义锚定失败 |
| F3 | 手动刷新闭环被本批破坏 | 围栏手动锚点（按钮/双端点/竞态/卸载）失败 |
| F4 | 无产品决策直接加定时器/EventSource | 硬边界；围栏负断言（零 setInterval/EventSource）失败 |
| F5 | 用全仓字符串禁用误伤注释/测试字样 | 围栏只解析目标文件的目标原语 |

## 八、验收标准（本批）

- 新 review fence、api-core-wiring、documentation-layout 原始 exit 0；
  typecheck、受影响 eslint、git diff --check exit 0；diff 名单仅
  docs/test/STATUS/EXPERIENCE；不跑 GUI/PG/全量；零 DB 写入；
  realm-dev.service 保持 active；工作树 clean。

## 九、范围外清单（本批不做）

轮询定时器、EventSource、SSE route、API 字段、数据库列、图谱组件/
API/repository/migration/outbox/record SSE 任何改动、K8 实现、
自动刷新的任何声称。

## 十、交付步骤

1. 本评审 + docs 索引（独立 commit）；2. 围栏测试（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
