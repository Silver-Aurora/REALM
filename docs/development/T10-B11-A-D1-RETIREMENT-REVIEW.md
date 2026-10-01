# T10-B11-A · D1 退役可执行评审（只审计/围栏，不删除）

> 批次 T10-B11-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B4 留下的「D1 实际删除评审」前置条件收口）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=d500083；本批为评审+围栏批——不删除任何 D1 文件/依赖/脚本，不改 `db/`、`drizzle.config.ts`、package script、迁移、API 行为、类型文件本身。

## 一、逐文件调用图实锤（2026-08-21 全库静态复核）

生产源码 = `app/`、`modules/`、`database/`、`scripts/`；测试 = `tests/`；工具 = `drizzle.config.ts` 与 package script。

| 文件 | 引用方（当次 grep 实查） | 分类 |
|---|---|---|
| `db/index.ts`（getD1/getDb，D1 入口） | 仅 `db/record-store.ts`、`db/story-record-repository.ts`（`@/db`，legacy 链内部） | extract-then-delete（随 record-store 退役后失去最后引用方） |
| `db/record-store.ts` | 生产 2 处均为 `import type`：`modules/application/local-record-service.ts`、`database/postgres/delivery-projection.ts`；测试 5 处均为 `import type`：local-record-application / m4-record-service / memory-prefetch-application / presence-application / self-play-application；legacy 内部被 story-record-repository runtime 引用 | extract-then-delete（混合兼容边界，**当前不能直接删**） |
| `db/story-record-repository.ts` | 零生产引用、零测试引用；仅引用别人（`@/db`、`@/db/record-store`、`@/modules/story-record/public`） | safe-to-delete 候选（零外部引用；但删除会牵动 T10-B4 围栏措辞与本文档闸门，仍走批准流程，本批不删） |
| `db/schema.ts`（Drizzle sqlite 表） | 仅 `db/index.ts`（`./schema`）与 `drizzle.config.ts` | extract-then-delete（随 db/index 退役） |
| `db/d1-legacy-types.d.ts`（D1 全局类型） | 支撑 `db/index.ts` 的 `D1Database` 签名 | extract-then-delete（随 db/index 退役） |
| `drizzle/`（0000–0002 SQL + meta） | 无代码引用；历史迁移参考 | retain-as-archive（本批不动；archive 决策随 tooling 批落档） |
| `drizzle.config.ts` + `db:d1:legacy:generate` + drizzle-orm/drizzle-kit 依赖 | local-only-boundary.test.mjs 故意锚定（script 必须存在、`db:generate` 必须不存在、`db/index.ts` 必须含 "Legacy D1 storage is inactive"） | retain（tooling；退役需先迁移 local-only 契约锚点，见 §五） |
| `modules/story-record/public.ts` | `db/story-record-repository.ts`、`tests/core-runtime.test.ts`；与框架/DB 无关的 domain 契约（README 代码结构在册） | retain（非 D1 链；其唯一 repository 实现是 legacy D1 适配器） |

补充事实：

- 生产源码中 `drizzle-orm`/`drizzle-kit` import 仅出现在 `db/` 内部与 `drizzle.config.ts`（当次 grep 实查）。
- `tests/api-core-wiring.test.mjs` 的 T10-B4 围栏已证明：全部活动 API route 不导入 record-store/story-record-repository/db schema/drizzle，且无 createD1/submitPlayerTurn。本批 audit 在其上补「生产非 route 源码」与「import type 形态」两层，不重复造轮子。
- `tsconfig.json` exclude 的 `db/schema 2.ts` 为本地杂散文件（`.gitignore` 在册），不在评审面。

## 二、为什么当前不能直接删 record-store

- `RecordProjection`/`ProjectionEvent` 是生产运行时的真实类型来源：
  `local-record-service.ts`（application 命令/投影编排）与
  `delivery-projection.ts`（PostgreSQL 交付投影）都以 `import type` 消费。
  删除即 typecheck 断裂——「D1 零运行时调用」不等于「D1 类型零引用」。
- `import type` 在编译期擦除，所以 D1 runtime（getD1/drizzle-orm/d1）不会进
  生产 bundle；但这个「安全」完全依赖 import 形态保持 type——一旦有人改成
  值导入，`@/db` → `drizzle-orm/d1` 会被拉进生产路径。这就是 audit 必须守
  的形态红线。
- record-store.ts 同时承载 runtime 导出（DEMO_IDS、getRecordProjection、
  RecordStoreError 等）与类型导出；生产只用类型，runtime 导出仅 legacy 链
  内部使用。它是「混合兼容边界」文件，不是纯可删文件。

## 三、portable type boundary 提取前置（下一批的前置条件）

1. 选址：`RecordProjection`/`ProjectionEvent` 等被生产消费的类型提取到中立
   地（候选：`database/postgres/` 或 `modules/application/` 内的纯类型文件），
   不得反向依赖 `db/`。
2. 改道：2 处生产 + 5 处测试的 `import type` 指向新址；record-store 内保留
   re-export 或同步删除，由当批规范定。
3. 验收：typecheck + 受影响 application/frontend-contract 测试 exit 0，
   本评审 audit 同步更新允许清单。
4. 完成后 record-store 失去生产引用方，才进入 safe-to-delete。

## 四、退役顺序（后续批次，本批不执行）

1. extract portable types（§三）→ 2. 移除 `db/record-store.ts` 与
   `db/story-record-repository.ts`（旧 repo）→ 3. 移除 `db/index.ts`、
   `db/schema.ts`、`db/d1-legacy-types.d.ts` → 4. Drizzle tooling 退役：
   `drizzle.config.ts`、`db:d1:legacy:generate` script、drizzle-orm/drizzle-kit
   依赖、`drizzle/` archive 决策。每步独立批次、独立规范与围栏更新。

## 五、package script 与 local-only 契约迁移计划

- 现状：`tests/local-only-boundary.test.mjs` 故意要求
  `"db:d1:legacy:generate"` 存在、`"db:generate"` 不存在、`db/index.ts` 含
  "Legacy D1 storage is inactive"——即「D1 inactive + legacy script retained」
  是本地边界契约的一部分，不能随删除悄悄改。
- 迁移计划（tooling 退役批）：先改写 local-only-boundary 锚点（例如改为
  「drizzle 依赖不存在 + db/ 目录不存在」的负断言），评审通过后才允许移除
  script/依赖/db 文件；同一批次内契约测试与删除同步落地，不留无锚点窗口。

## 六、本批围栏（`tests/d1-retirement-review.test.mjs`）

1. 正向锚定：db 四件套 + d1-legacy-types + drizzle.config.ts + drizzle/ SQL
   全部存在（防「删文件冒充退役」）；
2. 生产/工具源码（app/、modules/、database/、scripts/）不得出现对
   `db/index`、`db/schema`、`story-record-repository`、`drizzle-orm`/
   `drizzle-kit` 的 import/require（无论 type 与否）；
3. 生产/工具源码对 `db/record-store` 的引用必须是 `import type`/`export type`
   形态，runtime import 即失败；
4. 允许列表：引用 db 四件套的源码文件只能位于 `db/` 内部、`drizzle.config.ts`
   或 `tests/`；
5. local-only 锚点复核：package.json 含 `"db:d1:legacy:generate"` 且不含
   `"db:generate"`；`db/index.ts` 含 inactive 文案；
6. 扫描只匹配 import/export/require 语句形态——注释、规范文档、测试里的
   `d1`/`drizzle` 字符串合法，不做全仓字符串禁用。

围栏只守评审事实，不代表删除已完成。

## 七、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 活动 app/API 或生产源码引入 D1 runtime import/call | audit §六.2 失败（T10-B4 围栏同守 route 层） |
| F2 | db/index、db/schema、story-record-repository 被允许列表外文件引用 | audit §六.4 失败 |
| F3 | record-store 生产引用从 import type 变为 runtime import | audit §六.3 失败 |
| F4 | local-only 契约被单方面破坏（script 删除/inactive 文案改动） | audit §六.5 与 local-only-boundary 双失败 |
| F5 | 评审被偷换成删除（db 文件/drizzle/tooling 消失） | audit §六.1 正向锚定失败 |

## 八、删除批准闸门（逐项满足后才允许开删除批）

1. 本评审文档与 audit 持续绿至少一个批次周期；
2. §三 portable type boundary 提取完成、引用改道、本 audit 允许清单同步；
3. `drizzle/` 迁移 archive 决策落档（保留位置与形态）；
4. §五 local-only 契约迁移方案成文；
5. 独立删除批次，逐文件勾选 §一分类表，删除后 audit 更新为「不存在」
   负断言形态。

## 九、验收标准（本批）

- 新 audit、`tests/local-only-boundary.test.mjs`、
  `tests/api-core-wiring.test.mjs`、`tests/documentation-layout.test.mjs`
  原始 exit 0；typecheck、受影响 eslint、git diff --check exit 0；
  不跑全量/GUI/PG；零 DB 写入；realm-dev.service 无需重启；
  工作树 clean，未删除任何 D1 文件/依赖/脚本。

## 十、范围外清单（本批不做）

删除任何 db/ 文件、drizzle/、drizzle.config.ts、package script、npm 依赖；
修改类型文件内容；portable types 实际提取；local-only 契约改写；
modules/story-record 的存废评审（retain，非 D1 链）；全量/GUI/PG 回归。

## 十一、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. audit 测试（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
