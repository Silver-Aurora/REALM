# T10-B21-A · propagation / semantic-conflict 真实接线资格复核（结论：仍 contract-only-deferred）

> 批次 T10-B21-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B9-A §二接线前置的当前 HEAD 复核）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=2201e5a；本批只做资格复核/决策——不接线，不改 app/、modules/application/、modules/propagation/、modules/worldline/semantic-conflict.ts、API route、inference gateway、SQL/migrations。

## 一、复核方法

从当前 HEAD 重新静态扫描（非引用 T10-B9-A 的旧证据）：

- 生产调用图：`grep -rn "propagate|createPropagationWorker|createModelSemanticConflictAssessor|needsSemanticReview" app/ modules/application/`（2026-08-21 实查 exit 1，零命中）；
- 模块引用面：`grep -rln "modules/propagation|semantic-conflict" app/ modules/ database/ scripts/`；
- repository 实例化面：`grep -rln "createPostgresPropagationRepository|createPostgresPropagationJobQueue|createPostgresSemanticConflictEvidenceStore"`；
- 前置①：`grep -rn "enqueue|campaign" app/api/`；前置②：`grep -rn "Worker|scheduler|setInterval" scripts/dev-server.mjs app/ modules/application/`；前置③：`grep -rn "semantic" app/api/settings/ modules/inference/` + `createModelSemanticConflictAssessor` 签名实读。
- 纪律：focused 单测（tests/m5-*、postgres-m5-batch2）存在只证明契约与测试态实例化，**不得**推导生产接线（T10-B9-A 已写死此区分）。

## 二、生产调用图复核（当前 HEAD 实锤）

- `app/` + `modules/application/` 对四符号（propagate /
  createPropagationWorker / createModelSemanticConflictAssessor /
  needsSemanticReview）的 import 与 call 仍为**零**（grep exit 1）。
- repository 层事实（接线面 ≠ 调 用面，如实记录）：
  - `database/postgres/public.ts:45,47` re-export
    `createPostgresPropagationRepository` / `createPostgresPropagationJobQueue`
    ——仅导出面，全生产面无实例化调用方；
  - `createPostgresSemanticConflictEvidenceStore` 仅被
    `modules/worldline/semantic-conflict.ts` 自身引用（类型/定义点），
    零生产实例化；
  - `modules/propagation/worker.ts` 是 PropagationJobQueue 接口的唯一
    消费方（模块内部）。
- `app/api/worldline/conflict/route.ts` 仍只走 T10-B2 确定性 causal
  preview（B9 围栏在守）。

## 三、三项前置逐项结论（T10-B9-A §二）

| # | 前置 | 结论 | 机器证据（2026-08-21 实查） |
|---|---|---|---|
| ① | 哪个用户动作产出 campaign 的 enqueue 入口 | **FAIL（缺失）** | `app/api/` 零 propagation job/campaign enqueue——路由内 `enqueue` 命中全部是 SSE `controller.enqueue`（record/events、record/preview，语义无关）；`modules/application/` 零 propagate 调用；无任何用户动作产出 propagation job |
| ② | worker 启动方/进程拓扑 | **FAIL（缺失）** | `scripts/dev-server.mjs`、`app/`、`modules/application/` 零 createPropagationWorker 启动；生产面唯一 `setInterval` 是 preview 路由 SSE heartbeat（无关）；无 worker 进程拓扑 |
| ③ | 语义评估模型调用预算与降级策略 | **FAIL（缺失）** | `createModelSemanticConflictAssessor` 签名实读仅 `{ getGateway, evidenceStore? }`——无预算/超时/降级参数；`app/api/settings/` 与 `modules/inference/` 零 semantic 装配（grep exit 1）；证据落库工厂零生产实例化 |

无 UNKNOWN 项——三项均有明确缺失证据。

## 四、结论

**当前仍 `contract-only-deferred`，资格前置未满足**（三项全 FAIL）。
本批不构成「具备进入实现批资格」；未来任何接线必须先以独立规范满足
T10-B9-A §二三前置（enqueue 入口、worker 启动方、模型预算与降级），
未满足前的任何生产调用属假实现，由 m5-runtime-status 围栏拦截。

## 五、围栏扩展（tests/m5-runtime-status.test.mjs 第四测试）

在既有三测试（导出锚定/生产零调用/conflict 路由纯洁）上补「前置缺失
锚定」（只读负断言，接线时应先改规范再同步本测试）：

1. `app/api/` 不得 import propagation-job-queue / propagation-repository /
   semantic-conflict-repository（enqueue 入口缺失锚定）；
2. `scripts/dev-server.mjs` 与 `app/` 不得出现 createPropagationWorker
   调用（worker 启动方缺失锚定）；
3. `app/api/settings/` 与 `modules/inference/` 不得引用 semantic-conflict
   （模型预算/降级装配缺失锚定）。

## 六、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 生产代码悄悄接线四符号 | 既有围栏测试 2 失败 |
| ②-1 | 路由层出现 propagation enqueue 入口（未过前置评审） | 新第四测试失败 |
| ②-2 | dev-server/app 出现 worker 启动 | 新第四测试失败 |
| ②-3 | settings/inference 出现 semantic 装配 | 新第四测试失败 |
| F5 | 从单测存在推导电 claiming 已接线 | 本评审 §一纪律；围栏只扫生产源码 |

## 七、验收标准（本批）

- `tests/m5-runtime-status.test.mjs`、`tests/m5-world-governance.test.ts`、
  `tests/m5-batch2.test.ts`、`tests/documentation-layout.test.mjs` 原始
  exit 0；typecheck、受影响 eslint、git diff --check exit 0；
  不跑 PG/GUI/全量；零 DB 写入；realm-dev.service 保持 active；
  工作树 clean。

## 八、范围外清单（本批不做）

任何生产接线（enqueue/worker/语义评估装配）、模块行为改动、API 改动、
inference gateway 改动、SQL/migration 改动、「具备进入实现批资格」的
声称（三前置未满足）、执行方案撰写（前置满足时才需要）。

## 九、交付步骤

1. 本评审 + docs 索引（独立 commit）；2. 围栏扩展（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
