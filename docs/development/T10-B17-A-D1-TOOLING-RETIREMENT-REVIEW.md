# T10-B17-A · D1/Drizzle tooling 退役资格评审与执行方案（只评审，不删除/不移动）

> 批次 T10-B17-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；tooling 退役批的资格评审与执行方案落档——前置已由 T10-B14-A/B15-A/B16-A 全部核销）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=77f2858；本批只评审+围栏——不删除、不移动、不改写任何待退役文件，不改 package.json/package-lock、PG migrations、活动生产代码；结论只能是「具备进入下一执行批的资格/待用户确认」。

## 一、退役资格实锤（2026-08-21 全库静态复核）

| 待退役对象 | 当前引用面（当次 grep 实查） | 资格 |
|---|---|---|
| `db/index.ts` | 零源码 import（B14 pair 删除后失去全部引用方）；仅 D1 族测试锚定（B11/B13/archive review）与注释提及 | 具备 |
| `db/schema.ts` | 零源码 import；仅 `drizzle.config.ts` 配置字符串 `./db/schema.ts` 指向 | 具备（随 config 同步处理） |
| `db/d1-legacy-types.d.ts` | 零引用（全局声明文件，仅支撑 db/index 签名） | 具备 |
| `drizzle/` SQL+meta、`drizzle.config.ts` | 零活动消费方（drizzle-kit 生成入口，无活动数据库）；B15 provenance 围栏在守 | 具备（整体 move 单元） |
| package `"db:d1:legacy:generate"` script | 仅 archive review 指定锚点 + B11/B13 同族断言 | 具备（锚点迁移面已显式枚举，B16 核销） |
| `drizzle-orm`（dependencies）、`drizzle-kit`（devDependencies） | 消费方仅 `db/index.ts`（drizzle-orm/d1）、`db/schema.ts`（drizzle-orm/sqlite-core）、`drizzle.config.ts`（drizzle-kit）——全在待退役面内 | 具备（随三件套+config 删除后零消费方） |

- Core/PG/active runtime 无反向依赖：`modules/story-record/public.ts`、
  `database/postgres/`、`modules/application/` 对 db//drizzle 的 import 为零
  （B11–B16 audit 链持续在守，本批 review 复核）。
- `tests/api-core-wiring.test.mjs` T10-B4 围栏的路由扫描正则含
  `db\/schema|drizzle` 形态——退役后该正则依然有效（负断言形态与文件
  存废无关），**无需改动**。
- 本地杂散提示：`db/schema 2.ts`（gitignored、tsconfig exclude）不在 Git
  删除清单内；db/ 三件套删除后该本地文件由用户自行处置，执行批不动它。

## 二、未来执行批精确清单（本批不执行）

**A. 删除（git rm，三文件）**：`db/index.ts`、`db/schema.ts`、
`db/d1-legacy-types.d.ts`。删除后 `db/` 目录在 Git 中自然消失。

**B. 整体 move（git mv，一个单元）**：`drizzle/0000–0002 SQL` +
`drizzle/meta/`（三 snapshot + `_journal.json`）→ 明确 archive 位置
（建议 `docs/archive/d1-drizzle/`，执行批规范定稿路径）。
`drizzle.config.ts` 本体**删除**——它是只对 drizzle-kit 有意义的活配置，
move 后会指向已删除的 `./db/schema.ts`；其 provenance（三行配置全文 +
生成命令 + dialect 说明）记入 archive 位置的 README，满足 B15 §四.1
「SQL + meta + 配置 provenance 作为一个整体」且不留指向不存在路径的
config。

**C. package 变更**：移除 `"db:d1:legacy:generate"` script；移除
`drizzle-orm` 与 `drizzle-kit` 依赖；`package-lock.json` 经
`npm uninstall`（或等价）同步，执行批必须验证 package 与 lock 一致。
影响面：node_modules 变化，无其他消费方（§一实锤）。

**D. 锚点迁移（与 A–C 同批落地，不留无锚点窗口）**：
1. `tests/d1-drizzle-archive-review.test.mjs`：指定锚点（script/inactive
   文案）改为「已退役」形态——script 不存在、db/index 不存在、archive
   新位置 provenance 锚定（SQL+meta+journal 在新路径完整）；
2. `tests/d1-retirement-review.test.mjs`：db 三件套存在性锚定改为不存在
   断言 + 零引用断言（specifier 形态）；允许列表断言退役；
3. `tests/d1-pair-deletion-review.test.mjs`：db/index 链锚定改为不存在
   断言；其余（pair 缺失、Core 契约）保留；
4. `tests/local-only-boundary.test.mjs`：**不动**（B16 已解耦，generic
   边界不在迁移面内）；
5. `docs/README.md` 与 documentation-layout：archive 位置若含 md 需过
   链接完整性；索引登记 archive 位置。

**E. 保留项（执行批不得触碰）**：`database/postgres/migrations/`（0001–
0023）、`scripts/postgres-migrate.mjs`、`modules/story-record/public.ts`
Core contract、`modules/application/legacy-record-projection-types.ts`、
vite.config、dev-server、全部活动 API 与生产代码。

**F. 回滚路径**：`git revert` 执行批各 commit 恢复全部文件；
`npm ci` 按恢复后的 package-lock 重建依赖；纯源码/依赖变更，零 DB 联动。

## 三、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | move 单元被拆散（只删 SQL 留 meta、或留 config 指向不存在路径） | archive provenance 锚定失败；执行批评审打回 |
| F2 | 删除后残留对 `@/db`/`db/schema`/`d1-legacy-types` 的 import | typecheck + audit specifier 形态扫描双失败（resolve 盲区已覆盖） |
| F3 | 锚点迁移留无锚点窗口（先删文件后改测试） | 执行批规范禁止：锚点同步与删除同批落地 |
| F4 | package.json 与 package-lock 不一致 | 执行批 `npm ci` 验证失败 |
| F5 | 误动保留项（PG migrations/runner/Core/portable/活动代码） | diff 名单核对 + 保留项锚定失败 |

## 四、用户确认闸门（执行批批准点）

执行批开工前必须逐条向用户展示并获得明确确认：§二 A 删除三文件、
B move 单元与 archive 目标路径、drizzle.config.ts 删除+provenance 入
archive README 的方案、C package script/依赖移除、D 锚点迁移顺序。
本批（T10-B17-A）**不构成**执行批准；未经确认不得动任何文件。

## 五、本批围栏（`tests/d1-tooling-retirement-review.test.mjs`）

1. 待退役对象全部在位（本批零删除）：db 三件套、drizzle SQL+meta、
   drizzle.config、legacy script、两 drizzle 依赖；
2. db 三件套零活动生产/测试 import：行首锚定语句解析 + specifier 形态
   匹配（`@/db`、`db/index`、`db/schema`、`d1-legacy-types`），resolve
   盲区覆盖；config 字符串引用单独断言（仅 drizzle.config.ts）；
3. drizzle-orm/drizzle-kit 包引用方 ⊆ {db/, drizzle.config.ts}
   （specifier 形态，resolve 无关）；
4. move 单元完整性：SQL+meta+config 齐全（与 archive review 互证——
   archive review 守内容未改写，本测试守退役资格）；
5. 保留项锚定：PG migrations 0001–0023 齐全、postgres-migrate.mjs 指向
   PG 目录、Core contract 与 portable types 存在且 Core 零 db/drizzle
   反向依赖；
6. 不做全仓字符串禁用：注释/文档/测试里的 drizzle 字样合法。

围栏只证明「具备进入执行批的资格」，不代表退役已批准或已完成。

## 六、验收标准（本批）

- 新 review 原始 exit 0；archive review、d1-retirement-review、
  d1-pair-deletion-review、local-only-boundary、documentation-layout
  原始 exit 0；typecheck、受影响 eslint、git diff --check exit 0；
  禁动文件与 package/PG migration 的 diff 名单核对为零；不跑全量/GUI/PG；
  零 DB 写入；realm-dev.service 保持 active；工作树 clean。

## 七、范围外清单（本批不做）

§二 A–D 的全部执行动作；package.json/package-lock 任何改动；PG
migration 任何改动；活动生产代码改动；宣称 D1/Drizzle 已退役。

## 八、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. review 围栏（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
