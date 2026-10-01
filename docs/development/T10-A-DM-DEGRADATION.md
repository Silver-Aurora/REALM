# T10-A · DM 审查降级路径（缺陷 #12）

> 批次 T10-A（docs/development/EXPERIENCE-ITERATION.md 第二章缺陷 #12，T10 拆批第一批）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=cbbf946（LM Studio 切换已收口）；本批不改 provider/模型配置/迁移 0001–0021/T6–T9 功能边界。

## 一、当前失败链（读代码确认，勿重复调查）

`createModelDMController.validate`（modules/orchestration/model-powered.ts:441-457）：

```
structuralDM.validate（确定性硬门：权限/回执/预算/旁白权限/结构完整）
  → 通过 → reviewCandidateWithModel（:462-520，至多 3 次模型复核）
      ├─ 任一复核 accepted+goalSatisfied+worldCompatible 全真 → 放行
      ├─ 复核 JSON 损坏：第 1 次重试，持续损坏 → 直接放行（不具结论不拦截）
      └─ 三次一致否决 → throw DM_OUTPUT_REJECTED → 回合裸挂 422
```

- 调用图实锤：玩家回合与 self-play 拍共用此路径——`createLocalTurnDependencies`
  的 validator 只对 presence/interjection 载荷走纯结构校验，普通玩家回合与
  `payload.selfPlay` 回合都走 `orchestrator.validate`（model-powered）。
- 既有缓冲：三次独立复核、第三次升温放行偏好、JSON 损坏不视为否决。
- 实锤阵发：T1 验收 O3（连续三次 goalSatisfied=false）、T6/T7/T9 回归期
  DM_OUTPUT_REJECTED 与 DM_OUTPUT_INCOMPLETE 多次出现（STATUS/迭代日志在案）。

## 二、硬不变量（不可动）

1. 确定性结构校验是**唯一硬门**：结构失败（DM_OUTPUT_INCOMPLETE 等）继续
   fail-closed，本批零改动。
2. 降级绝不产生未验证世界事实：候选已先通过结构硬门（工具回执、段落归属、
   预算、旁白只用公开 Receipt 事实）——降级只是「不再让软性模型复核单独
   否决一个硬件合格的候选」，不是把模型文本当事实写库。
3. 成功路径输出与错误码兼容；新字段向后兼容（validation body 可选字段）。
4. 降级有界：复核次数上限不变（3），降级不触发任何额外模型调用或重试。

## 三、降级策略（选定：确定性接受 + 审计标记）

候选方案对比：
- **A. 确定性接受已通过结构校验的候选**（选定）：候选已过全部硬门；模型复核
  自 2026-08-13 起定位就是「非阻塞兼容性检查」（STATUS 在案：模型复核误否决
  合法输入的修复史）。降级 = 接受 + 打审计标记，内容与正常路径逐字节同构。
- B. 只由 receipts/brief 生成保守旁白：要替换角色回应为确定性占位文本，既丢
  内容又引入「降级专用叙事模板」第二套生成面；不接受。

三类复核结果区分：
| 类别 | 判定 | 行为 |
|---|---|---|
| 结构校验失败 | structuralDM.validate 抛错 | 照旧 fail-closed（不进入复核） |
| 模型复核否决 | 三次一致 {accepted/goalSatisfied/worldCompatible} 非全真 | **降级接受** + 审计 |
| 复核非法/不可用 | JSON 持续损坏 / gateway 调用抛错 | **降级接受** + 审计（与「否决」区分记因） |

## 四、可观测性（零迁移）

- validation body 增可选字段 `review: { mode: "model" | "degraded", vetoes: number, reason: string }`——
  随 turn_runs checkpoint 的 validation JSONB 持久化（既有结构，向后兼容，
  旧行无此字段）。降级回合在库可查，不伪装成模型复核成功（mode=degraded）。
- 控制面日志：`[realm] review degraded: <reason>（vetoes=n）——accepting structurally validated candidate`（console.warn，失败码不入库新表）。

## 五、失败矩阵

| # | 场景 | 行为 |
|---|---|---|
| F1 | 结构校验失败 | fail-closed 照旧（DM_OUTPUT_INCOMPLETE 等） |
| F2 | 复核一次否决后放行 | 照旧（review.mode="model"） |
| F3 | 复核三次一致否决 | 降级接受，review={mode:"degraded",vetoes:3,reason:"triple-veto"} |
| F4 | 复核 JSON 持续损坏 | 降级接受，reason="invalid-json"（既有「损坏不拦截」语义的显式化） |
| F5 | 复核 gateway 抛错（服务不可用/超时） | 降级接受，reason="review-unavailable" |
| F6 | 降级 | 不新增模型调用、不重试、不影响已提交内容形态 |

## 六、验收标准

1. Core（model-inference）：三次一致否决 → validate 返回 accepted 且
   review.mode="degraded"/vetoes=3，不抛 DM_OUTPUT_REJECTED；单次否决后放行
   照旧（review.mode="model"）；复核 JSON 持续损坏与 gateway 抛错分别降级且
   reason 正确；结构失败仍抛错不降级。
2. 应用层（self-play-application）：self-play 拍经共享 orchestrator 三连否决 →
   拍落库、会话 completed（调用图：payload.selfPlay → orchestrator.validate）。
   玩家回合侧由 Core 层 model-powered 测试覆盖同一函数。
3. 受影响回归：m2-orchestration、self-play 既有用例零回退。
4. typecheck/eslint 干净；文档布局契约过。

## 七、交付步骤

1. 本规范（单独 commit）；
2. 实现（reviewCandidateWithModel 返回复核证据 + validate 写入 validation body）；
3. 测试与收口（Core/应用测试 + STATUS/迭代日志）。
