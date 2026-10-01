# T10-B15-A · Drizzle/D1 历史迁移 archive 决策（只落档/围栏，不移动不删除）

> 批次 T10-B15-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B14-A 后 tooling/archive 批的前置决策①：drizzle/ archive 决策落档）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=b3baa6d；本批只写决策+围栏——不移动、不删除、不改写 `drizzle/`、meta、`drizzle.config.ts`、`db/index.ts`、package script/依赖或任何 PostgreSQL migration；不得宣称 D1/Drizzle 已退役。

## 一、两套迁移系统边界（不得混淆、不得互删）

| | D1/SQLite 历史（archive 对象） | PostgreSQL 权威（活动） |
|---|---|---|
| 路径 | `drizzle/`（SQL + meta） | `database/postgres/migrations/` |
| 内容 | 0000 原型基础表 / 0001 prototype_turn_commits / 0002 events payload+version 列 | 0001–0023 runtime contract 至 library_runtime_grants |
| dialect | sqlite（`_journal.json` version 7、三份 snapshot 均 `"dialect": "sqlite"`，当次实查） | postgresql |
| 生成/执行 | drizzle-kit（`db:d1:legacy:generate`，**已无活动数据库可执行**——db/index getD1 永久 throw） | `scripts/postgres-migrate.mjs`（migrationDirectory=database/postgres/migrations，realm_schema_migrations 台账） |
| 签名特征 | `--> statement-breakpoint`、反引号标识符 | PG 语法（grant/policy/RLS 等；**零** statement-breakpoint，当次实查） |

红线：drizzle/ SQL **不是**可执行 PG migration，任何批次不得把 drizzle/
文件复制/链接进 `database/postgres/migrations/` 或让 PG runner 消费；
反向亦不得用 PG migration 覆盖/替换 drizzle/ 历史。

## 二、逐文件 provenance（当次实查）

| 文件 | 内容 | 历史角色 |
|---|---|---|
| `drizzle/0000_orange_triathlon.sql` | CREATE TABLE character_definitions 等原型基础表（SQLite 语法） | D1 原型首个可玩切片 schema |
| `drizzle/0001_cynical_khan.sql` | CREATE TABLE prototype_turn_commits（幂等提交台账） | D1 原型回合提交幂等 |
| `drizzle/0002_chunky_spirit.sql` | ALTER TABLE events ADD visibility_payload/action_payload/record_version | D1 原型事件可见性与版本列 |
| `drizzle/meta/0000–0002_snapshot.json` | 三份 `"dialect": "sqlite"` schema 快照 | drizzle-kit 增量生成基线 |
| `drizzle/meta/_journal.json` | version 7、dialect sqlite、entries tags 与三 SQL 文件名一一对应 | drizzle-kit 迁移日志 |
| `drizzle.config.ts` | schema=./db/schema.ts、out=./drizzle、dialect=sqlite | legacy 生成入口配置 |
| `db/schema.ts` + `db/index.ts` + `db/d1-legacy-types.d.ts` | D1 表定义与 inactive 入口 | local-only 契约锚点宿主 |
| package `db:d1:legacy:generate` + drizzle-orm/drizzle-kit 依赖 | legacy 生成工具链 | local-only 契约锚定对象 |

## 三、决策：保留原位（in-place retain），只落档

- drizzle/ 及 meta 作为 D1 原型历史参考**继续留在原路径**，直到未来
  tooling 退役批。
- 理由：①provenance 价值——三 SQL + meta + config + script 是「D1 原型曾
  如何演化」的完整证据链，拆散即毁链；②local-only 契约（
  tests/local-only-boundary.test.mjs）故意锚定 legacy script 与 inactive
  文案，移动/删除需先迁移契约锚点；③保留原位零风险：无活动消费方
  （B11–B14 audit 链已证），占位成本仅三个小 SQL 与四个 JSON。
- 本批不改变任何文件位 置/内容；「archive」是决策定性，不是物理移动。

## 四、未来整体 archive move 的前置与回滚

未来若物理归档，必须满足：

1. **整体性**：SQL + meta（snapshots + _journal）+ drizzle.config.ts 的
   provenance 作为一个整体 Git move 到明确 archive 位置（如
   `docs/archive/d1-drizzle/` 或同级），不得只删 SQL、只搬 meta、或留
   config 指向不存在路径；
2. **契约同步**：local-only-boundary 锚点（script + inactive 文案）与新
   位置在同一批次改写落地，不留无锚点窗口；
3. **文档同步**：docs/README.md、README、本决策文档增补移动事实
   （追加不改写）；
4. **审计同步**：本批 archive review 与 B11/B14 audit 的 drizzle 锚点
   同步更新为新位置；
5. **回滚**：Git move 本身可 revert；若连同 script/依赖移除（tooling
   退役批），回滚需同时恢复 package.json/package-lock，须在该批规范中
   写明。

## 五、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | drizzle/ SQL/meta/config 被移动/删除/改写（本批红线） | review 存在性 + provenance 锚定失败 |
| F2 | drizzle 文件混入 PG migrations 或 PG runner 消费 drizzle | review 边界断言失败（statement-breakpoint 交叉检测） |
| F3 | PG migration 被当 D1 历史改动/删除 | review PG 目录完整性断言失败 |
| F4 | local-only 锚点被搭车改（script/inactive 文案） | review + local-only-boundary 双失败 |
| F5 | 只搬一部分（SQL 走了 meta 留下等）破坏证据链 | 未来 move 批评审打回；本批整体锚定防渐进拆散 |

## 六、本批围栏（`tests/d1-drizzle-archive-review.test.mjs`）

1. 存在性正向锚定：三 SQL、三 snapshot、_journal、drizzle.config.ts 全部
   在位；`db/index.ts` inactive 文案、package legacy script 锚定；
2. provenance 锚定：_journal dialect=sqlite 且 entries tags 与三 SQL 文件
   名一一对应；三 snapshot dialect=sqlite；三 SQL 各含
   `--> statement-breakpoint` 与标志性语句头（0000 character_definitions /
   0001 prototype_turn_commits / 0002 visibility_payload）——证明内容未
   被改写（存在性/关键头部/journal tags 实锚，不生成 hash 伪证据）；
3. 边界断言：PG migrations 目录在、0001–0023 齐全、全部 .sql 零
   statement-breakpoint；drizzle/ SQL 零 PG 专有构造（CREATE POLICY /
   ENABLE ROW LEVEL SECURITY / GRANT … TO realm_）；postgres-migrate.mjs
   的 migrationDirectory 指向 database/postgres/migrations；
4. 只解析真实文件内容与 import 语句，不做全仓 `drizzle` 字符串禁用——
   历史文档/测试里的 drizzle 字样合法。

## 七、验收标准（本批）

- 新 archive review、d1-retirement-review、d1-pair-deletion-review、
  local-only-boundary、documentation-layout 原始 exit 0；typecheck、
  受影响 eslint、git diff --check exit 0；不跑全量/GUI/PG；零 DB 写入；
  realm-dev.service 无需重启；工作树 clean，未移动/删除/改写任何文件。

## 八、范围外清单（本批不做）

drizzle/ 物理移动、drizzle.config.ts 删除、package script/依赖移除、
db/index/schema/d1-types 删除、local-only 契约迁移、PG migration 任何
改动、tooling 退役批本身、全量/GUI/PG 回归。

## 九、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. archive review 围栏（独立
   commit）；3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。

## 十、状态更新（批次 T10-B16-A，2026-08-21 追加，不改写上文历史）

- §四.2「契约同步」前置的**解耦部分已核销**：`tests/local-only-boundary.test.mjs`
  已只守 generic 活动 runtime 边界（无 Sites/Worker/Cloudflare、回环绑定、
  启动路径），不再读取 `db/index.ts`、不再断言 legacy script/inactive 文案；
  该组 D1 历史链契约由 `tests/d1-drizzle-archive-review.test.mjs` 的专门用例
  承担（指定锚点，见 docs/development/T10-B16-A-LOCAL-ONLY-DECOUPLING.md）。
- 未来 tooling 退役批的锚点迁移面因此收窄为：archive review 指定锚点 +
  B11/B13 D1 族 audit 的同类断言；通用 runtime 边界不再在迁移面内。
- 未核销：§四 的整体 move 其余前置（文档/审计同步、回滚含 package 恢复）
  与 tooling 退役批本身，仍需独立规范与确认。
