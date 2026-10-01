# T10-B9-A · M5 propagation / semantic-conflict 运行态围栏

> 批次 T10-B9-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；缺陷 #13 的 M5 暂缓模块定性与防漂移围栏）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=f184ed1；本批为 contract-only-deferred 标记批——不接线、不新增 API、不加假调用点、不改模块行为、不改迁移/网关/既有 API 契约。

## 一、当前调用图实锤（2026-08-21 全库静态复核）

| 模块 | 已实现 contract | 生产调用方 | 结论 |
|---|---|---|---|
| `modules/propagation/public.ts` | 确定性传播引擎（`propagate`，realm-propagate-v1） | 零（仅自身/repository/测试） | contract-only-deferred |
| `modules/propagation/worker.ts` | 队列 worker（`createPropagationWorker`，领取/stale 恢复/重试） | 零（无 enqueue 入口、无启动方） | contract-only-deferred |
| `modules/worldline/semantic-conflict.ts` | 模型评估器（`createModelSemanticConflictAssessor`）+ `needsSemanticReview` 分流 + 证据表契约 | 零（仅类型被引用；repository 实现零生产实例化） | contract-only-deferred |

- 扫描证据：`app/` 与 `modules/application/` 下对 `propagate(`/
  `createPropagationWorker(`/`createModelSemanticConflictAssessor(`/
  `needsSemanticReview(` 的生产调用为零（当次 grep 实查）。
- `app/api/worldline/conflict/route.ts` 继续只走 T10-B2 的确定性 causal
  preview 语义，不引入 semantic-conflict/模型网关。
- 既有 focused 单测（tests/m5-world-governance.test.ts、tests/m5-batch2.test.ts）
  覆盖引擎/worker/评估器行为——单测存在不等于生产接线，本文档与围栏把
  两者区分写死。

## 二、contract-only-deferred 语义

- 三模块保留：契约完整、测试覆盖、M5 §7/§9 明确为第二批（离线调度/语义
  评估）设计内暂缓；不是死代码，不是已完成。
- 未来接线前置条件（引用 T10-B1 §一表与 T10-B5 §二同型）：enqueue 入口
  （哪个用户动作产出 campaign）、worker 启动方（进程拓扑）、语义评估的
  模型调用预算与降级策略。满足前任何接线都属假实现。

## 三、围栏（静态契约测试 `tests/m5-runtime-status.test.mjs`）

负断言（说明：这些是「当前不得接线」的防漂移断言，不是功能断言——
未来真要接线时应先修规范与本文档，再同步更新本测试）：
1. 扫描 `app/**/*.ts(x)` 与 `modules/application/**/*.ts` 生产源码：不得出现
   对 `propagate`/`createPropagationWorker`/`createModelSemanticConflictAssessor`/
   `needsSemanticReview` 的 import 或调用；
2. `app/api/worldline/conflict/route.ts` 不得引入 `semantic-conflict` 模块或
   模型网关（`model-powered`/`inference`）；
3. 正向锚定：三模块文件存在且继续导出上述符号（防悄悄删除）；
4. 既有 M5 单测保持全过（行为零改动）。

## 四、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 生产代码悄悄接线 | 围栏失败 |
| F2 | 模块被删除/导出消失 | 围栏失败（正向锚定） |
| F3 | conflict 路由引入模型评估 | 围栏失败 |
| F4 | 引擎/worker 行为回归 | 既有 M5 单测失败 |

## 五、验收标准

- `tests/m5-runtime-status.test.mjs` exit 0；m5-world-governance 与
  m5-batch2 原始 exit 0；typecheck/受影响 eslint/文档布局/git diff --check
  exit 0；不跑全量/GUI/PG 长回归；开发库不变；realm-dev.service 不需重启。

## 六、范围外清单（本批不做）

queue persistence 生产接线、worker scheduler 启动方、worldline 路由接模型
评估、图谱 UI 刷新、D1 删除、库授权文档化。

## 七、交付步骤

1. 本规范（单独 commit）；2. 围栏测试（独立 commit）；3. STATUS/
   EXPERIENCE-ITERATION 收口（独立 commit）。
