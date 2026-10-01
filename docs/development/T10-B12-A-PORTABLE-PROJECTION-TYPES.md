# T10-B12-A · 提取 D1/PG 共享 delivery projection 纯类型边界

> 批次 T10-B12-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B11-A 删除批准闸门 §三「extract portable types」执行）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=75d3fad；本批只做类型提取与引用改道——不删 db/、drizzle/、package script/依赖、迁移，不改 API 行为、JSON 形状、runtime 逻辑、D1 文件内容语义。

## 一、两套 RecordProjection 不能混淆（红线）

| 类型 | 出处 | 语义 | 消费方 |
|---|---|---|---|
| Core `RecordProjection` | `modules/story-record/public.ts:53` | persistence-neutral Core 契约：recordId/workspaceId/worldline/nextOrdinal/`RecordEvent[]`，面向上下文编译与回合编排 | `tests/core-runtime.test.ts`、legacy `db/story-record-repository.ts` |
| delivery `RecordProjection` + `ProjectionEvent` | `db/record-store.ts:24,50` | 交付/UI 投影：世界/故事/记录导航树、cast、events（segments/presence/dice/selfPlay）、stories/records | 生产 2 处 + 测试 5 处（见 §二），均为 `import type` |

两者同名不同语义，本批**不合并、不改名、不互相 re-export**；delivery
类型迁入新模块后文件级注释写死「delivery projection，非 Core projection」。

## 二、提取方案

- 新模块：`modules/application/legacy-record-projection-types.ts`
  （纯类型文件——零 runtime 导出、零值导入；名字记录其来源：从 legacy
  record-store 提取的 delivery projection 类型）。
- 迁移内容：`ProjectionEvent`、`RecordProjection` 及其字段注释原样迁移
  （含 T3 presence / T5 dice / T7 selfPlay 标注）。
- 依赖边界：只允许 `import type` 依赖其他模块的纯类型——
  `SemanticSegment`（modules/presentation/semantic-segments.ts）、
  `MechanicDetail`（modules/actions/public.ts）；**不得** import
  db/、drizzle、database/postgres、HTTP/路由，不得出现任何值导入
  （值导入即生产 runtime 边，违背纯类型边界定义）。
- 不并入 `modules/story-record/public.ts`：那是 Core 契约宿主，delivery
  形状进入会造成 §一的语义混淆。

## 三、引用改道（2 生产 + 5 测试，当次 grep 实锤清单）

- `database/postgres/delivery-projection.ts:5`（RecordProjection）
- `modules/application/local-record-service.ts:1`（RecordProjection + ProjectionEvent）
- `tests/local-record-application.test.ts:3`
- `tests/m4-record-service.test.ts:3`
- `tests/memory-prefetch-application.test.ts:3`
- `tests/presence-application.test.ts:9`
- `tests/self-play-application.test.ts:11`

全部改指新模块，保持 `import type` 形态与导出名不变（最小 diff）。

## 四、旧 record-store 保留策略

- `db/record-store.ts` 不删除：DEMO_IDS、getRecordProjection、RecordStoreError、
  ensurePrototypeSchema 等 runtime 导出与 D1 链原样保留（T10-B11-A 结论：
  当前不能直接删除）。
- 类型定义改为 `import type` 新模块 + `export type { … }` re-export——
  单一类型来源，legacy 内部消费方（`db/story-record-repository.ts` 经
  `@/db/record-store` 引用 ProjectionEvent/DeliveryRecordProjection）零改动。
- re-export 是兼容桥，不是新引用入口：生产/测试不得再 import
  `db/record-store`（audit 守）。

## 五、与 B11 删除批准闸门的关系

- 本批完成闸门第 2 条「portable type boundary 提取完成、引用改道、audit
  允许清单同步」的类型侧；record-store 自此失去全部生产/测试引用方，
  仅剩 db/ 内部 legacy runtime 引用。
- 闸门其余条目不变：archive 决策（drizzle/）、local-only 契约迁移、
  独立删除批次。record-store 实际删除仍需后续批次按闸门执行——本批
  **不删除任何文件**。
- B11 audit 同步更新：生产零 import `db/record-store` 负断言 + 新模块为
  唯一生产类型来源的正向锚定 + 新模块自身纯度断言（零值导入、零 db/
  drizzle 引用）；旧 D1 文件与 legacy script 存在性锚定不动。

## 六、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | delivery 类型并入/改名 Core RecordProjection | 规范红线；typecheck 语义混淆即评审打回 |
| F2 | 新模块出现值导入或 db/drizzle/PG/HTTP 依赖 | audit 纯度断言失败 |
| F3 | 生产/测试仍 import `db/record-store`（或改回 runtime import） | audit 负断言失败 |
| F4 | record-store runtime 导出被删/D1 链内容被改 | audit 存在性与导出锚定失败；B11 闸门打回 |
| F5 | 类型漂移（新模块与 record-store re-export 不一致） | 单一来源 + typecheck 5 个应用套件失败 |

## 七、验收标准（本批）

- 更新后 audit、`tests/local-record-application.test.ts`、
  `tests/m4-record-service.test.ts`、`tests/memory-prefetch-application.test.ts`、
  `tests/presence-application.test.ts`、`tests/self-play-application.test.ts`、
  `tests/local-only-boundary.test.mjs`、`tests/api-core-wiring.test.mjs`、
  `tests/documentation-layout.test.mjs` 原始 exit 0；typecheck、受影响
  eslint、git diff --check exit 0；不跑全量/GUI/PG；零 DB 写入；
  realm-dev.service 无需重启；工作树 clean，未删除任何 D1 文件/依赖/脚本。

## 八、范围外清单（本批不做）

删除 record-store/story-record-repository/db/index/schema/d1-types；
drizzle/ archive 决策；local-only 契约改写；package script/依赖移除；
Core RecordProjection 的任何改动；runtime 行为、JSON 形状、API、数据库、
迁移改动；全量/GUI/PG 回归。

## 九、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. 类型提取 + 改道 + record-store
   re-export + audit 更新（独立 commit）；3. STATUS/EXPERIENCE-ITERATION
   收口（独立 commit）。
