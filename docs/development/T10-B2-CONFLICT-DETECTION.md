# T10-B2 · 确定性因果冲突检测生产接线

> 批次 T10-B2（docs/development/EXPERIENCE-ITERATION.md T10 拆批第二批，T10-B1 调用图审计的后续项）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=f54796e；本批不改迁移 0001–0022、不改 LM Studio、不改 T6–T10-B1、不接 semantic-conflict/propagation/memory snapshot。

## 一、现状与缺口（读代码确认）

- M5 §4 承诺三层因果检测（确定性/依赖图/语义分级），实现于
  `modules/worldline/conflict-detection.ts` 的 `detectCausalConflicts`
  （纯只读函数：输入 changeSet/claims/edges/future 游标，不写任何世界事实）。
- 现状：仅 `tests/m5-world-governance.test.ts` 调用；`/api/worldline/conflict`
  只接 `classifyWorldlineConflict`（纯游标分类）。属「承诺但缺 runtime 接线」。
- 既有数据端口齐备：`WorldKnowledgeService.listClaims/listCausalEdges`（按
  WorldScope 读 world_claims/causal_edges）、T9 的 `resolveWorldScopeForMember`
  + `getSharedRuntimePool`（T10-B1 已证受限角色可用）。

## 二、设计：同路由双分支（legacy 不动 + causal 显式分流）

`POST /api/worldline/conflict`：

- **legacy 分支（零改动）**：无 `mode` 字段时走既有 `classifyWorldlineConflict`
  纯函数契约（pastChange + existingFuture + anchors），不要求 DB/成员身份。
- **causal 分支**：`body.mode === "causal"` 时进入因果预览——

请求体：

```jsonc
{
  "mode": "causal",
  "worldId": "world_…",            // 必填；缺失/空 → 400
  "changeSet": {
    "changes": [                  // 至少 1 条；逐条严格解析
      {
        "kind": "assert" | "terminate" | "supersede",
        "subjectEntityId": "entity_…",     // 非空 ≤160
        "predicate": "life_state",         // 非空 ≤120
        "objectValue": "dead",             // 字符串 ≤160（可空字符串）
        "targetClaimId": "claim_…",        // terminate/supersede 必填
        "effectiveCursor": { "tick": 150, "ordinal": 0 }  // 安全整数
      }
    ]
  },
  "existingFuture": { "tick": 300, "ordinal": 0 }   // 复用 legacy 游标解析
}
```

响应（200）：

```jsonc
{
  "ok": true,
  "mode": "causal",
  "scope": { "worldId": "…", "worldlineId": "…" },
  "algorithm": "realm-causal-conflict/v1",
  "report": { "deterministic": [], "dependency": [], "classification": {} }
}
```

硬门：
1. existingClaims/causalEdges **只能从解析出的 WorldScope 经
   `createWorldKnowledgeService(createPostgresWorldKnowledgeRepository(pool))`
   读库**——请求体没有也绝不会有 claims/edges 字段，客户端无法注入事实；
2. worldId 经 `resolveWorldScopeForMember`（membership + 当前 active 世界线），
   未知/非成员统一 404；
3. 只读：`detectCausalConflicts` 纯函数，路由零写库、零 proposal/merge/branch；
4. `REALM_RUNTIME_DATABASE_URL` 缺失 → 503；读库失败 → 500 安全文案，
   不返回半真报告；
5. 算法零改动（除非 focused regression 证明输入 bug 并在此记录——本次无）。

## 三、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | mode=causal 缺/空 worldId | 400 |
| F2 | 未知/非成员世界 | 404（不泄露） |
| F3 | changeSet 缺失/空数组/字段非法（kind 越界、terminate 缺 targetClaimId、游标非安全整数） | 400 |
| F4 | existingFuture 缺失/非法 | 400 |
| F5 | 读库失败/运行时未初始化 | 500 安全文案 / 503 |
| F6 | legacy 分支 | 契约不变，零回归 |

## 四、验收标准

1. Route focused（真实临时 PG，t.after 拆库）：legacy 结果不回归；causal
   400/404 各态；成员请求从 DB 读事实——life_state 终止冲突返回 hard、
   依赖边返回 dependency 冲突、报告含 DB 独有的 claim id（证明非客户端注入）；
   全程零写库（worldline_merges/canon_proposals 前后为 0）。
2. `tests/m5-world-governance.test.ts` 纯函数覆盖零回退。
3. typecheck/受影响 eslint/文档布局/git diff --check 全过；不跑全量/GUI。
4. 开发库基线核对。

## 五、交付步骤

1. 本规范（单独 commit）；2. 路由分支 + route 测试（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
