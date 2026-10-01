# T10-B3 · Worldline 路由 500 脱敏可观测性

> 批次 T10-B3（docs/development/EXPERIENCE-ITERATION.md T10-B1/B2 共同发现：「route 层 500 无日志，调试期只能靠 scratch 复现」）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=a920f5b；本批不改业务响应契约/算法/scope/迁移/LM Studio/T6–T10-B2，零新增 DB 写入。

## 一、现状（读代码确认）

- `app/api/worldline/merge/route.ts`：非 WorldlineMergeError 的未知异常 →
  500 `INTERNAL_ERROR`，**零日志**（T10-B1 调试期被迫 scratch 复现permission
  denied）。
- `app/api/worldline/conflict/route.ts`：causal 分支读库失败 → 500 安全文案
  （40c91be 修复后）**零日志**；legacy 分支异常 → 400 invalid（输入错误语义，
  非内部故障）。
- 全 app/api 无共享安全 logger；既有 `console.warn` 携带 error.message
  （world/generate 等）——message 可能含连接串/内部细节，不能照搬到 worldline
  路由。

## 二、设计：共享脱敏诊断助手（零新依赖）

新增 `app/api/route-observability.ts`：

```ts
logRouteInternalError({ route, stage, error })
// 输出单行 console.error：
// [realm] route internal failure {"route":"worldline/merge","stage":"…","errorType":"Error","code":"ECONNREFUSED"}
```

字段白名单（只这些）：
- `route`：固定路由标识字符串（`worldline/merge` / `worldline/conflict`）；
- `stage`：失败阶段枚举（merge=`"request"`；conflict causal=`"causal-preview"`）；
- `errorType`：错误构造名经白名单规整（`/^[A-Za-z]*Error$/` 否则 `"Error"`；
  非 Error 取 `"unknown"`）；
- `code`：仅当 `error.code` 匹配 `/^[A-Z0-9_]{2,20}$/`（errno/SQLSTATE 形态）
  才带出，否则 `"unknown"`。

**严禁**进入日志：连接串/环境变量值、API key/token、error.message/stack
（可能带连接串）、请求 payload、claims/body/source id、完整 SQL、世界线内容。

业务结果不伪报 internal：WorldlineMergeError（merge 业务 404）与
CausalInputError（conflict 输入 400）不进内部故障日志。

## 三、接线点（仅两处 500 分支）

- merge route：catch 中非 WorldlineMergeError → `logRouteInternalError` 后照旧 500；
- conflict route：causalPreview 读库 catch → `logRouteInternalError` 后照旧 500。
- HTTP status 与 response body 逐字节不变。

## 四、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | merge 内部失败（断库/权限） | 500 不变 + 脱敏结构化日志 |
| F2 | conflict causal 读库失败 | 500 不变 + 脱敏结构化日志 |
| F3 | merge 业务失败（世界线不存在等） | 404 照旧，无 internal 日志 |
| F4 | conflict 输入非法 | 400 照旧，无 internal 日志 |
| F5 | 错误对象携带敏感 message/stack/连接串 | 日志只含白名单字段 |

## 五、验收标准

1. Route focused（真实 loopback 断库地址 `127.0.0.1:1`，开发库零污染）：
   - merge 断库 → 500 不变；conflict causal 断库 → 500 不变；
   - 捕获的 console.error 恰一行、JSON 可解析、仅含 route/stage/errorType/code，
     不含连接串/message/stack/请求 body/claim id；
   - merge 业务 404（世界线不存在）与 conflict 输入 400 不产生 internal 日志。
2. 既有 route PG 测试（postgres-worldline-merge-route / postgres-conflict-route）
   零回退。
3. typecheck/受影响 eslint/文档布局/git diff --check 全过；不跑 GUI/全量。

## 六、交付步骤

1. 本规范（单独 commit）；2. 助手 + 两路由接线 + 测试（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
