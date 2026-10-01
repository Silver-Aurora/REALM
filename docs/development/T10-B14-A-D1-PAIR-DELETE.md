# T10-B14-A · 执行 D1 pair 删除（用户已独立确认；只删两文件）

> 批次 T10-B14-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B13-A §七 pair 删除批准闸门的执行批）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=ba8aeb2；**用户已明确独立确认执行本删除批**（派单原话：「用户已明确确认执行本删除批」），满足 T10-B13-A §七闸门第 3 条。
> 本批是 **pair 删除**，不是 D1/Drizzle 全退役——`db/index.ts`、`db/schema.ts`、`db/d1-legacy-types.d.ts`、`drizzle.config.ts`、`drizzle/`、`db:d1:legacy:generate` script 与 drizzle-orm/drizzle-kit 依赖全部保留，属下一 tooling/archive 批。

## 一、精确删除边界（硬边界）

只允许从 Git 删除两文件：

- `db/record-store.ts`
- `db/story-record-repository.ts`

绝对不得删除/改动内容：deferred 五件套（db/index、db/schema、
db/d1-legacy-types、drizzle.config、drizzle/）、package.json/package-lock 的
D1/Drizzle script 与依赖、任何 migration、
`modules/story-record/public.ts` Core `StoryRecordRepository` contract、
`modules/application/legacy-record-projection-types.ts` portable types、
API/数据库/运行时行为。不「顺手清理」杂散文件（`db/schema 2.ts` 等本地
exclude/gitignore 项不动）。

## 二、资格依据（引用 T10-B13-A 实锤，不重复论证）

- pair 零 db/ 外部引用（B13 review audit 5/5，2026-08-21 实查）；
- delivery projection 类型已由 T10-B12-A 提取至 portable 模块，2 生产 +
  5 测试改道完毕，record-store 的 re-export 桥无外部消费方；
- record-store 残余 runtime 出边（semantic-segments 的
  `fallbackCompositeSemanticSegments`）随文件删除消失——presentation 模块
  有其他消费方，不受影响；
- `db/index.ts` 删除后失去全部源码引用方，但按 B13 §四分界**保留**（
  local-only 文案锚定 + tooling 批统一退役），不搭车。

## 三、audit 锚点同步方案（存在 → 删除后缺失）

`tests/d1-retirement-review.test.mjs`：

| 原断言（B11/B12 形态） | 删除后形态 |
|---|---|
| LEGACY_FILES 含 record-store/story-repository 存在性锚定 + record-store re-export/runtime 导出锚定 | pair 移出存在清单，改为**不存在**断言（删除后预期）；deferred 五件套 + drizzle/tooling 存在性锚定保留 |
| 「record-store 仅 db/ 内部引用」 | 「全扫描面（含 db/）对 pair 路径的 import/export/require 语句为零」——按 specifier 形态解析，resolve 不到也要命中 |
| 生产零 D1 runtime import / portable 唯一类型来源 / 允许列表 / local-only 锚点 | 原样保留（portable types、Core contract、deferred 链不受影响） |

`tests/d1-pair-deletion-review.test.mjs`：整体改写为「删除后纪事」形态——
pair 不存在 + 零引用；deferred 链（db/index → schema + drizzle-orm/d1、
drizzle.config → schema）锚定；Core contract 独立性与 core-runtime 走
in-memory 锚定；local-only 锚点复核。文件头注明本测试已由「资格评审」
转为「删除后围栏」，历史形态见 Git 记录，不静默改写 B11/B13 规范文档。

## 四、Core contract / in-memory 证明

- `modules/story-record/public.ts` 不删不改：`export interface
  StoryRecordRepository`（:141）与 `createInMemoryStoryRecordRepository`
  （:521）保持；Core 零 db/drizzle 反向依赖。
- `tests/core-runtime.test.ts` 走 Core in-memory 实现，不引用 pair——
  删除后必须原样 exit 0，作为「Core 不依赖 pair」的运行态证明。

## 五、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | pair 之外文件被删/被改（deferred、script、迁移、Core、portable） | git diff 名单核对 + audit 存在性锚定失败 |
| F2 | 删除后残留对 pair 的 import（typecheck 之外的形态） | audit 零引用断言失败 |
| F3 | Core contract/in-memory 被破坏 | audit 锚定 + core-runtime.test 双失败 |
| F4 | local-only 锚点被搭车改（script/inactive 文案） | local-only-boundary + audit 双失败 |
| F5 | 删断言/放宽扫描造绿 | 本规范禁止；audit 断言数不得少于删除前语义覆盖面 |

## 六、回滚与批准说明

- 回滚：`git revert` 本批删除 commit 即可完整恢复两文件（无 DB/迁移/
  依赖联动，纯源码删除）；
- 批准链：T10-B13-A 资格评审（零外部引用实锤）→ 用户独立确认（本批
  派单）→ 本规范 → 删除执行；后续 tooling/archive 批需各自独立评审与
  确认，本批不构成对它们的预先批准。

## 七、验收标准（本批）

- 两个 audit、local-only-boundary、api-core-wiring、documentation-layout、
  core-runtime 原始 exit 0；typecheck、受影响 eslint、git diff --check
  exit 0；pair 文件缺失、deferred 文件仍在；不跑全量/GUI/PG；零 DB 写入；
  realm-dev.service 保持 active；工作树 clean。

## 八、范围外清单（本批不做）

db/index/schema/d1-types 删除、drizzle/ archive、drizzle.config 删除、
package script/依赖移除、local-only 契约迁移、migration 改动、API/运行时
行为改动、杂散文件清理、全量/GUI/PG 回归。

## 九、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. 删除 pair + 两 audit 删除后形态
   （独立 commit）；3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
