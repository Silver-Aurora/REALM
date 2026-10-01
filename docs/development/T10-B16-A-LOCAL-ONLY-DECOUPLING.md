# T10-B16-A · 迁移 local-only 契约锚点（generic 与 D1 archive 解耦）

> 批次 T10-B16-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B15-A 后 tooling 退役批的最后前置：local-only 契约迁移）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=83ddcdd；本批只迁移测试契约的责任边界——不删除/移动/改写 drizzle/、meta、drizzle.config.ts、db/index.ts、db/schema.ts、db/d1-legacy-types.d.ts、package script/依赖、任何 PG migration；不是 D1/Drizzle 退役批。

## 一、当前耦合（当次实查）

`tests/local-only-boundary.test.mjs` 同时守两类事实：

1. **generic 活动 runtime 边界**（其本职）：无 Sites/Worker/Cloudflare
   adapter 文件、vite.config/package.json 无 cloudflare/wrangler 痕迹、
   HOST_BIND/PORT 默认回环、dev-server.mjs 启动路径；
2. **D1 历史链契约**（耦合点）：读取 `db/index.ts` 并断言
   `Legacy D1 storage is inactive` 文案、断言 package.json 含
   `"db:d1:legacy:generate"` 且不含 `"db:generate"`。

后果：通用运行时边界的存废被绑在 D1 历史 tooling 上——未来 tooling
退役批要动 script/db/index 就必须先改通用边界测试，契约责任不清
（T10-B15-A §四.2 把「契约同步」列为 move 前置的根因）。

## 二、解耦方案

- `tests/local-only-boundary.test.mjs`：只守第 ① 类事实。移除对
  `db/index.ts` 的读取与全部 D1 历史链断言（inactive 文案、
  `"db:d1:legacy:generate"`、`"db:generate"` 配对负断言）。
  说明：`db/index.ts` 上原有的 `cloudflare:workers` 负断言随读取一并
  移除——该文件是永久 throw 的 legacy 入口，不是活动 runtime 的 DB
  通路（活动通路是 database/postgres），通用边界无需为它背书；
  「无 Cloudflare adapter」事实由文件缺失断言 + vite.config/package.json
  扫描完整覆盖。
- `tests/d1-drizzle-archive-review.test.mjs`：成为 legacy script 与
  inactive 文案的**指定 focused 锚点**。原测试 1 中的三条历史链断言
  独立成专门用例并注明所有权（B16 迁移自 local-only-boundary；
  tooling 退役批改锚点必须先改这里）。不新增第三套扫描逻辑——
  B11/B13 D1 族 audit 的同类断言属同族回归网，保持不动。
- 责任边界落档：

| 契约 | 承担者（本批后） |
|---|---|
| 无 Sites/Worker/Cloudflare、回环绑定、启动路径 | `tests/local-only-boundary.test.mjs`（generic） |
| legacy script 存在 + 无 `db:generate` + inactive 文案 | `tests/d1-drizzle-archive-review.test.mjs`（指定锚点） |
| D1 退役面（pair 缺失/零引用、deferred 链、Core 契约） | `tests/d1-retirement-review.test.mjs` + `tests/d1-pair-deletion-review.test.mjs`（同族回归网） |

## 三、对 T10-B15-A 决策的前置核销

- B15 §四.2「契约同步」前置中「local-only 锚点与历史链解耦」部分由本批
  完成：未来 tooling 退役批只需改写 archive review（指定锚点）+ B11/B13
  同族断言，不再触碰通用 runtime 边界。
- 尚未核销：tooling 退役批本身（script/依赖/db/index/schema/d1-types/
  drizzle/ 的移除或整体 move）仍需独立规范与确认。

## 四、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | local-only 重新引入 D1 历史链断言（耦合回潮） | 本规范 + archive review 所有权注释；评审打回 |
| F2 | 解耦时误删 generic 断言（Sites/Worker/回环/启动路径失守） | local-only 断言覆盖面评审核对（只减 D1 三条，generic 一条不少） |
| F3 | archive review 丢失 script/inactive 锚点 | archive review 专门用例失败 |
| F4 | 搭车改动 drizzle/、db/、package、PG migration | 硬边界；git diff 名单核对 |
| F5 | 把解耦宣称成 D1/Drizzle 退役 | 验收红线；STATUS 只记「契约已解耦」 |

## 五、验收标准（本批）

- local-only-boundary、d1-drizzle-archive-review、d1-retirement-review、
  d1-pair-deletion-review、documentation-layout 原始 exit 0；typecheck、
  受影响 eslint（两测试文件）、git diff --check exit 0；
  package/db/drizzle/PG migration 文件内容零改动；不跑全量/GUI/PG；
  零 DB 写入；realm-dev.service 保持 active；工作树 clean。

## 六、范围外清单（本批不做）

tooling 退役（script/依赖移除、db/index/schema/d1-types 删除、drizzle/
整体 move）、local-only 通用边界的新增断言、任何生产代码改动、
B11–B15 历史文档改写（B15 文档仅追加明确标注的状态更新）。

## 七、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. local-only 解耦 + archive
   review 锚点强化 + B15 文档状态更新（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
