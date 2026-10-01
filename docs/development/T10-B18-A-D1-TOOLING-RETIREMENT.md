# T10-B18-A · 执行 D1/Drizzle tooling 退役批（用户已明确授权）

> 批次 T10-B18-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B17-A §四用户确认闸门的执行批）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=83e5485；**用户已明确要求「继续完成剩余的 REALM 迭代工作」**，B17 已将执行清单与确认闸门逐条落档——本批按 B17 §二 A–D 执行，闸门各条与执行动作的对应关系见 §一。
> 本批完成后方可声称「D1/Drizzle tooling 退役完成」。

## 一、授权与闸门逐条对应

| B17 §四 确认点 | 本批执行 |
|---|---|
| A 删除 db 三件套 | `git rm db/index.ts db/schema.ts db/d1-legacy-types.d.ts` |
| B move 单元与 archive 目标路径 | `drizzle/` 三 SQL + 整个 `drizzle/meta/` 整体 `git mv` 至 `docs/archive/d1-drizzle/`（内容不改写） |
| drizzle.config.ts 删除 + provenance 入 archive README | `git rm drizzle.config.ts`；配置全文/生成命令/SQLite dialect/原 schema+out 路径记入 `docs/archive/d1-drizzle/README.md`，README 不留指向活动路径的假配置 |
| C package script/依赖移除 + lock 同步 | 删 `"db:d1:legacy:generate"`；`npm uninstall drizzle-orm drizzle-kit`（真实操作，不手改 lock 伪造）；`npm install --package-lock-only` 验证一致 |
| D 锚点迁移顺序 | 四个 D1 族测试同批改写为 post-retirement 形态（§四）；`tests/local-only-boundary.test.mjs` 不动（B16 已解耦） |

停止条件：若 drizzle-orm/drizzle-kit 仍被活动代码引用（B17 实锤为零，
执行前以 audit 复核），立即停止并报告，不强行删除。

## 二、保留项（硬红线，diff 名单核对为空）

`database/postgres/migrations/0001–0023`、`scripts/postgres-migrate.mjs` 与
realm_schema_migrations 执行链、`modules/story-record/public.ts` Core
contract、`modules/application/legacy-record-projection-types.ts`、
`app/`、`modules/` 活动生产代码、PG repository、vite/dev-server、GUI 测试
主体、`.env.local`、开发数据库、服务配置。

## 三、A–C 执行细节

- A：三文件 git rm 后 `db/` 目录在 Git 中消失；本地杂散
  `db/schema 2.ts`（gitignored）不在清单内，由用户自行处置。
- B：目标布局 `docs/archive/d1-drizzle/{0000,0001,0002}_*.sql` +
  `docs/archive/d1-drizzle/meta/{0000–0002_snapshot.json,_journal.json}`；
  archive README 记录 provenance 与退役批事实，链接须过
  documentation-layout；**不得**把任何 drizzle 文件放入
  `database/postgres/migrations/`。
- C：script 行从 package.json 移除（npm uninstall 不管 scripts）；
  npm uninstall 同步 package-lock；验证 `npm install --package-lock-only`
  exit 0 且不恢复已移除顶层依赖。

## 四、D：post-retirement 围栏迁移（同批，不留必红 stale 测试）

| 测试 | 迁移后形态 |
|---|---|
| `tests/d1-drizzle-archive-review.test.mjs` | archive 新路径 SQL+meta+journal provenance 完整（dialect/tags/关键头部/statement-breakpoint）；archive README 含 config provenance；原 `drizzle/`、`drizzle.config.ts`、db 三件套不存在；legacy script 消失、`db:generate` 仍不存在；PG migrations 不混入 |
| `tests/d1-retirement-review.test.mjs` | db 三件套 + drizzle.config 不存在断言；零引用（resolve 轨 + specifier 形态轨覆盖 resolve-null，含 drizzle 包）；deferred→已退役的表述更新；portable types 唯一类型来源与纯度断言保留；pair 零引用保留 |
| `tests/d1-pair-deletion-review.test.mjs` | deferred 五件套锚点改退役后不存在/零引用；pair 缺失、Core contract 独立性锚点保留 |
| `tests/d1-tooling-retirement-review.test.mjs` | 转为 post-retirement execution audit：待退役对象不存在、archive 完整、零 drizzle 依赖 import、保留项锚定；文件头注明由 B17 资格评审转换（历史形态见 Git） |
| `tests/local-only-boundary.test.mjs` | **不动**（generic runtime 边界，B16 已解耦；不得塞回 D1 断言） |

历史 B11–B17 规范文档不静默改写；docs/README.md 树结构与索引追加
archive 位置与本批文档。

## 五、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | move 单元拆散（SQL/meta 分离、config 留指向不存在路径） | archive audit provenance 锚定失败 |
| F2 | 删除后残留 db/drizzle import（含 resolve-null 悬挂） | typecheck + audit specifier 轨双失败 |
| F3 | lock 与 package 不一致或已删依赖被恢复 | `npm install --package-lock-only` 失败/审计 diff 核对 |
| F4 | 保留项被误动 | diff 名单核对 + 保留项锚定失败 |
| F5 | stale 测试留红或删断言造绿 | 本批验收六项原始 exit code 全零；审计断言数不减少语义覆盖面 |

## 六、验收标准（本批）

- §四全部测试 + local-only-boundary + documentation-layout 原始 exit 0；
  typecheck、受影响 eslint（全部改动测试文件）、git diff --check exit 0；
  `npm install --package-lock-only` exit 0；保留项 diff 名单为空；
  活动源码零 D1/Drizzle import；archive 文件完整；开发库计数
  2/1/0/0/0/0 不变；realm-dev.service 保持 active（无需重启）；
  工作树 clean；不跑 GUI/PG/全量；零 DB 写入。

## 七、回滚路径

`git revert` 本批各 commit 恢复全部文件与 package 记录；`npm ci` 按恢复
后 lock 重建依赖；纯源码/依赖/文档变更，零 DB 联动。

## 八、交付步骤

1. 本规范 + docs 索引（独立 commit）；2. A+B（删除三件套 + drizzle 整体
   归档 + archive README，独立 commit）；3. C（script/依赖移除 + lock
   同步，独立 commit）；4. D（四测试 post-retirement 改写 + docs 树，
   独立 commit）；5. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
   说明：D 之前的中间 commit 中 D1 族旧锚点测试暂时转红属批次内预期
   （旧链已删而锚点未迁），D commit 恢复全绿；每步 staged diff 逐一核对。
