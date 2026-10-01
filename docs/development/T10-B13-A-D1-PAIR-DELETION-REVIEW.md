# T10-B13-A · D1 pair 删除资格评审（只评审不删除）

> 批次 T10-B13-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B11-A 删除批准闸门逐条推进：B12 完成闸门第 2 条后，对第一对删除候选的资格评审）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=feb44ab；本批为评审+围栏批——不删除任何 D1/Drizzle 文件、依赖、脚本，不改 db/、迁移、API 行为、类型文件。

## 一、当前调用图实锤（2026-08-21 全库静态复核，B12 提取后）

**删除候选 pair**（预期零外部引用，当次 grep 实查确认）：

| 文件 | 外部引用方（app/modules/database/scripts/tests） | 内部出边 | 结论 |
|---|---|---|---|
| `db/story-record-repository.ts` | **零**（仅注释提及） | `@/db`（getD1）、`@/db/record-store`、`@/modules/story-record/public`（Core 契约）、`@/modules/context/public`（VisibilityPolicy 类型） | 有资格进入下一删除批 |
| `db/record-store.ts` | **零**（仅注释提及） | `@/db`（getD1）、`modules/presentation/semantic-segments.ts`（runtime fallback 函数）、`modules/application/legacy-record-projection-types.ts`（import type + re-export 兼容桥） | 有资格进入下一删除批 |

**retain/deferred**（不随 pair 删除）：

| 文件 | 引用方 | 结论 |
|---|---|---|
| `db/index.ts`（getD1/getDb + inactive 文案） | 仅 pair 两文件；local-only-boundary 锚定其 inactive 文案 | deferred——pair 删除后失去最后引用方，但随 tooling 批统一退役，不搭车 |
| `db/schema.ts` | `db/index.ts`、`drizzle.config.ts` | deferred（随 db/index） |
| `db/d1-legacy-types.d.ts` | 支撑 db/index 签名 | deferred（随 db/index） |
| `drizzle.config.ts`、`drizzle/` SQL+meta、`db:d1:legacy:generate`、drizzle-orm/drizzle-kit 依赖 | local-only-boundary 故意锚定（script 存在 + `db:generate` 不存在） | retain——archive 决策与 local-only 契约迁移是 tooling 批前置，本评审不触碰 |

**Core contract retain**：`modules/story-record/public.ts:141` 的
`export interface StoryRecordRepository` 是 persistence-neutral domain 契约，
且 Core 自带 `createInMemoryStoryRecordRepository`（public.ts:521）——
`db/story-record-repository.ts` 只是该契约的唯一 **D1 持久化**适配器，
不是唯一实现；`tests/core-runtime.test.ts` 走 Core in-memory 实现，不依赖
pair。删除 pair 不影响 Core 契约与其测试。

## 二、孤儿实现与 Core contract 的区分

- 「唯一 D1 实现消失」不等于「契约失去意义」：StoryRecordRepository 的方法
  集（load/appendAuthorized/compileSharedTurnContext 等）定义的是 domain
  读写面，in-memory 实现与 core-runtime 单测证明契约独立成立。
- 红线：任何批次不得因 pair 删除而删/改名 `StoryRecordRepository`
  interface 或 Core 的 in-memory 实现。

## 三、删除前后预期

**删除前（本批锚定的现状）**：pair 存在；record-store 保留 re-export 桥与
runtime 链（DEMO_IDS/getRecordProjection/RecordStoreError）；legacy 内部链
（story-record-repository → record-store → db/index）完整；core-runtime、
B11 audit、local-only、wiring 全绿。

**删除后（下一批预期，本批不执行）**：

- pair 两文件消失；`db/index.ts` 失去全部源码引用方（仅剩 local-only
  文案锚定）——**保留**，随 tooling 批处理；
- record-store 的 re-export 桥随文件消失——B12 已完成 2+5 改道，无外部
  消费方，零影响；
- `tests/d1-retirement-review.test.mjs` 与本评审 audit 需同步更新锚点
  （pair 存在性正断言改为不存在负断言或退役对应用例）——同步方案必须在
  删除批规范中写明，不留无锚点窗口；
- local-only 锚点（db/index inactive 文案 + legacy script）不受影响；
- core-runtime.test.ts 不受影响（走 Core in-memory）。

## 四、archive/local-only 前置的分界

- pair 删除**不触碰** db/index、drizzle/、drizzle.config、package script——
  因此 pair 删除批的前置**不包括** drizzle archive 决策与 local-only 契约
  迁移；那两项是 tooling 退役批（db/index/schema/d1-types/drizzle/script/
  依赖）的前置，B11 §四退役顺序第 3、4 步。
- 对 B11 删除批准闸门的修正说明：B11 §八把 archive/local-only 列为统一
  前置，本评审按「影响面」细化——pair 批与 tooling 批各有闸门，避免
  「全部前置做完才能删任何文件」的过度阻塞，也避免「pair 删完顺手清
  tooling」的搭车。

## 五、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | pair 被 app/modules/database/scripts/tests 外部引用「复活」 | review audit §六.2 失败 |
| F2 | pair 或 deferred 文件被删除（本批红线：零删除） | review audit 存在性锚定失败 |
| F3 | Core `StoryRecordRepository` interface 被删/改名 | review audit 锚定 + core-runtime.test 双失败 |
| F4 | legacy 内部链被破坏（re-export 桥消失致 story-record-repository 编译断） | typecheck 失败 |
| F5 | local-only 锚点被单方面改（script/inactive 文案） | review audit §六.5 与 local-only-boundary 双失败 |

## 六、本批围栏（`tests/d1-pair-deletion-review.test.mjs`）

1. 存在性正向锚定：pair 两文件 + deferred 五件套（db/index/schema/
   d1-types、drizzle.config、drizzle/ SQL）全部存在——防「删文件冒充评审」；
2. pair 零外部引用：解析真实 import/export/require，任何 app/modules/
   database/scripts/tests 源码对 pair 的引用即失败（注释/文档合法）；
3. legacy 内部链锚定：story-record-repository → `@/db` + `@/db/record-store`
   引用存在；record-store → `@/db` + re-export 桥 + runtime 导出
   （DEMO_IDS/getRecordProjection/RecordStoreError）存在；
4. Core contract 锚定：`modules/story-record/public.ts` 存在且含
   `export interface StoryRecordRepository` 与
   `createInMemoryStoryRecordRepository`；core-runtime.test.ts 仍引用
   story-record/public；
5. local-only 锚点复核：package.json 含 `"db:d1:legacy:generate"` 且不含
   `"db:generate"`；db/index.ts 含 inactive 文案。

围栏只证明「pair 有资格进入下一删除批」，不代表删除已批准或已完成。

## 七、下一批删除批准闸门（pair 批）

1. 本评审文档与 review audit 持续绿；
2. 删除批规范先行：明确只删 pair 两文件；写明 B11 audit 与本 audit 的锚点
   同步形态；附 core-runtime 不受影响证明；
3. 派单方/用户独立确认后执行；
4. 删除后 typecheck + core-runtime + 全 audit 链 + local-only/wiring/docs
   focused 全绿；
5. db/index/schema/d1-types/drizzle/tooling 不搭车，仍属 tooling 批。

## 八、验收标准（本批）

- 新 review audit、`tests/d1-retirement-review.test.mjs`、
  `tests/local-only-boundary.test.mjs`、`tests/api-core-wiring.test.mjs`、
  `tests/documentation-layout.test.mjs`、`tests/core-runtime.test.ts`
  原始 exit 0；typecheck、受影响 eslint、git diff --check exit 0；
  不跑全量/GUI/PG；零 DB 写入；realm-dev.service 无需重启；
  工作树 clean，未删除任何文件。

## 九、范围外清单（本批不做）

删除 pair 或任何 D1/Drizzle 文件；删除/改名 StoryRecordRepository Core
interface；drizzle/ archive 决策；local-only 契约迁移；package script/依赖
移除；任何 db/ 文件内容改动；全量/GUI/PG 回归。

## 十、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. review audit（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
