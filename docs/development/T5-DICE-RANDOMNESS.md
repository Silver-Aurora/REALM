# T5 实施规范：骰子真随机与多骰系

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 上位设计：[`M2-ORCHESTRATION-AND-TOOLS.md`](./M2-ORCHESTRATION-AND-TOOLS.md) §4（五种骰系）/§5（Action Receipt）。
> 前置批次：[`T4-RULE-REALIZATION.md`](./T4-RULE-REALIZATION.md)（引擎工具、metadata.check 契约、骰系字段预留）。

## 1. 现状根因（审计实锤，勿重复调查）

1. **伪随机查表**：`deterministicInteger(seed, 20)`（`modules/actions/engine.ts`）由 seed 做 FNV-1a 哈希——同一行动重放永远同结果。这不是骰子，是查表。
2. **骰系只有 d20**：M2 §4 设计 d20 / 2d6 / 百分骰 / 骰池 / 抽牌五种，`parseCheckSpecification` 对其他四种一律 `RULE_DECISION_INVALID`。
3. **判定结果玩家不可见**：mechanic 已随 Receipt 落库（`action_receipts.receipt` jsonb + 事件 `metadata.actionTransaction`），但 Delivery Projection 读侧（`database/postgres/delivery-projection.ts`）不投影 mechanic，玩家时间线只有结论文案，看不到骰点过程。
4. **物化链路现成**：`mapLocalFormalEvent` 已把完整 Receipt（含 mechanic）与 `action.transaction.committed` 事件、action_receipts 行同事务原子落库——真随机不需要新表、不需要新迁移。

## 2. 核心设计：掷骰一次物化，重放只读库

### 2.1 真随机源与重放幂等的共存方案

- 随机源换成密码学安全随机（`node:crypto` `randomInt`），`deterministicInteger` 删除（全库仅 engine.ts 一处使用，无其他调用方）。
- **掷骰时机**：只发生在行动裁决时刻（`resolveDecision` 候选装配期）。一次生成，结果即进 MechanicDetail，随 Receipt 在发布事务内原子落库。
- **重放幂等三层共存**（M1 铁律不破）：
  1. 进程内：解析器 callId + fingerprint 去重，重复 resolve 返回同一 Transaction（既有机制，不变）；
  2. 回合级：重放走发布层 completed 短路，不重新解析、不重掷（既有机制，不变）；
  3. 跨进程/展示：任何读侧（Delivery Projection、回读、测试核对）一律取已落库的 Receipt/事件 metadata 中的物化骰点，**绝不重掷**。
- 未提交候选的骰点随候选废弃而消失（不可见、无副作用），重新裁决重新掷——这不违反幂等：幂等约束的是已落库结果的重放。
- `resolveUncertainty` 的 `seed` 参数退役；为保持语义可测，函数接受可选 `randomInt` 注入（生产缺省 CSPRNG，测试注入确定性序列）——注入只用于测试，生产路径（解析器/规则包）不传。

### 2.2 多骰系契约（metadata.check 数据驱动，延续 T4）

公共字段：`system`（必填）、`modifier`（整数，缺省 0）、`target`（整数 ≥1，必填）。各骰系：

| 骰系 | 掷法 | 成败 | 暴击规则 |
|---|---|---|---|
| `d20` | 1d20 | `roll+modifier ≥ target` | 天然 20 必成功（critical）；天然 1 必失败（fumble） |
| `2d6` | 2d6 求和 | `sum+modifier ≥ target` | 双 6 必成功；双 1 必失败 |
| `percentile` | 1d100，target=成功率（1..100） | `roll ≤ clamp(target+modifier, 1, 100)` | roll ≤5 必成功；roll ≥96 必失败 |
| `pool` | N 颗 sides 面骰，每颗 ≥successOn 记 1 成功 | `成功数+modifier ≥ target`（target=需要成功数） | 全部骰子成功必成功；0 成功必失败 |
| `draw` | 从 deck 不放回抽 1 张 | 抽中 ∈ successCards 成功 | 无暴击 |

骰系专属字段：

- `pool`：`dice`（整数 1..50，必填）、`sides`（整数 ≥2，缺省 6）、`successOn`（整数 1..sides，缺省 = sides）。
- `draw`：`deck`（字符串数组，非空、去重、≤100，必填）、`successCards`（字符串数组，非空且必须 ⊆ deck，必填）；**不允许** `modifier`/`target`（出现即 `RULE_DECISION_INVALID`）。

**抽牌不放回**：`RuleDefinitionProvider` 新增可选方法 `listDrawnCards({skillKey, actorCharacterInstanceId})`。PG 实现查 `action_receipts` 中本记录、本行动者、本技能已物化的 `mechanic.drawnCard`（`createPostgresRulePack` scope 扩 worldlineId/recordId）；规则包裁决 draw 时以 `deck − 已抽` 为剩余牌堆，剩余为空 → `DECK_EXHAUSTED` FatalTurnError。内存 demo provider 不实现该方法 → 每次完整牌堆（文档化的组合限制，纯内存组合本无跨进程账本）。MechanicDetail 对 draw 附带 `skillKey`（历史查询关联键）、`drawnCard`、`deckRemaining`（抽后剩余）。

**骰系选择完全由 metadata.check.system 分流**——不新增任何硬编码分支；未知骰系/非法字段一律 fail-closed。

### 2.3 统一 MechanicDetail

```ts
type MechanicDetail = {
  system: DiceSystem;
  rolls: number[];        // 骰点明细；draw 为空数组
  modifier: number;       // draw 恒 0
  target: number;         // percentile=成功率, pool=需要成功数, draw 恒 0
  total: number;          // d20/2d6: 合计+修正; percentile: roll; pool: 成功数+修正; draw 恒 0
  success: boolean;
  critical?: true;
  fumble?: true;
  skillKey?: string;      // draw：历史查询关联键
  drawnCard?: string;     // draw
  deckRemaining?: number; // draw：抽后剩余
};
```

既有 `roll: number` 字段由 `rolls: number[]` 取代（结构统一；受影响的 Core 断言本批同步）。outcome 语义不变（success/failure 二值进 Receipt.outcome；partial 仍只存在于定义 outcomes 文案层）。

## 3. 判定结果玩家可见（Delivery Projection）

- **读侧投影**：`delivery-projection.ts` 对 `action.transaction.committed` 事件从 `payload.actionTransaction.receipt.mechanic` 解析出 `dice` 块投到事件上（既有事件形态，不加新事件表——T3 presence 先例）。SSE 与 GET 共用同一 RecordProjection，天然同链。
- **前端契约**：`TimelineEvent.dice`（record-types.ts normalize 透传），字段同 MechanicDetail。
- **UI 形态**：系统事件卡追加一行骰点明细（如 `🎲 d20 14+2=16 ≥ 12 · 成功`；pool 列各骰；draw 展示抽中牌与剩余），锚点 `data-dice-system` / `data-dice-outcome`；三语文案走 i18n `ui.dice.*`。**前端不自行模拟骰点**——只渲染投影值。
- M2 §4「玩家默认不读内部机械细节」在本批由任务书显式开启（骰点过程可见），规则卡/内部 Transaction 其余部分仍不透出。

## 4. 失败矩阵（T4 §5 之上新增/修订，全部 fail-closed）

| 场景 | 错误码 | 形态 |
|---|---|---|
| 未知骰系 | `RULE_DECISION_INVALID` | FatalTurnError（既有） |
| 公共字段非法（modifier 非整数/target 非正整数） | `RULE_DECISION_INVALID` | FatalTurnError（既有语义） |
| pool 缺 dice/dice 越界/sides<2/successOn 越界 | `RULE_DECISION_INVALID` | FatalTurnError（新增） |
| percentile target 非 1..100 | `RULE_DECISION_INVALID` | FatalTurnError（新增） |
| draw deck 空/重复/超 100、successCards 空或非子集 | `RULE_DECISION_INVALID` | FatalTurnError（新增） |
| draw 携带 modifier/target | `RULE_DECISION_INVALID` | FatalTurnError（新增） |
| 牌堆抽空 | `DECK_EXHAUSTED` | FatalTurnError（新增） |

## 5. 交付步骤与 commit 纪律

1. 规范（本文档）单独 commit。
2. **真随机源与统一结构**：engine.ts 换 CSPRNG + `randomInt` 注入点 + 统一 MechanicDetail；public.ts 类型同步；受影响 Core 断言同步（roll→rolls）。
3. **多骰系**：parseCheckSpecification 五骰系推导；engine.ts 五骰系解算；PG provider 抽牌历史（scope 扩 record 级 + listDrawnCards）。
4. **投影与 UI**：delivery-projection dice 投影；record-types/event-timeline/i18n。
5. **测试收口**：PG 集成（分布非恒定/重放幂等/五骰系语义/抽牌不放回）+ 应用层（推导与 fail-closed）+ GUI t5-dice.spec.ts 真实模型；STATUS 三段式与迭代日志；清理回基线。

禁 `git add -A`；每步独立 commit。

## 6. 验收标准

1. 大样本掷骰分布非恒定（伪随机查表特征消失）。
2. 同回合重放读同一骰点结果（物化幂等）。
3. 五种骰系各至少 1 项集成测试通过。
4. GUI 真实模型：自建世界检定骰点玩家可见且落库。
5. demo 零回退：C1/C3 既有断言不改通过。
6. `npm test` 全链绿；`npm run lint` 当次实测 0 error。
7. 范围化回归（总纲第五章）：t5 新 spec + c-actions + o-action-suggestions + t4-rule-realization + 冒烟 a-library/b-record，双引擎；失败逐一复跑定性，数据敏感用例复跑前先清理。
8. 每次 GUI/集成写入后执行 `scripts/clean-gui-test-data.sql`，计数回基线（worlds=2 / accounts=1 / first_nights=0）。
