# T4 实施规范：规则系统真化（数据驱动规则包 + 引擎工具 + 状态账本闭环）

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 上位设计：[`M2-ORCHESTRATION-AND-TOOLS.md`](./M2-ORCHESTRATION-AND-TOOLS.md)（三类权限、Actor/Engine Tool、Action Receipt、原子发布）。
> 迭代背景：[`EXPERIENCE-ITERATION.md`](./EXPERIENCE-ITERATION.md) 第二章缺陷 #8-#11、第四章 T4 主轴。本批是 T6 导入卡参局的硬前置。

## 1. 现状根因（审计实锤）

1. **规则包写死**：全库唯一规则包 `createLocalRulePack`（`modules/actions/public.ts`）硬编码 1 技能 `careful_observation` + 1 资产 `signal_lantern` + 1 姿态 `guarded_watch`，判定数值写死（modifier 2 / target 12），并为 demo 硬编码 `letter_seal` 蜡封桥段；其他技能/资产/姿态一律 `FatalTurnError`。
2. **SQL 硬编码锁死**：`database/postgres/action-state.ts` 能力目录三处 `WHERE` 硬编码 `skill_key='careful_observation'` / `asset_key='signal_lantern'` / `effect_key='guarded_watch'`——表里有其他定义也查不出来，自建世界行动面板只剩「观察四周」。
3. **引擎工具零实现**：M2 设计的 `resolve_uncertainty / consume_resource / apply_effect` 只有 `ENGINE_TOOL_SPECS` 声明，代码零命中；判定散落在 `resolveDecision` 内联，扣减/效果散落在 `applyActionState` 内联。
4. **选择与提示词同样写死**：`local-record-service.ts` 的 `KNOWN_ACTION_AFFORDANCES` 静态名单与 `playerActionFromSelection` 硬编码 skillId/assetId/stanceId；`model-powered.ts` 的 `ACTOR_TOOLS` 把 `use_skill` 的 enum 钉死为 `careful_observation`、target 钉死为 `letter_seal/scene_surroundings`。
5. **表骨架现成**：`skill_definitions`（含 `rule_pack_key`/`metadata`）、`character_skills`（含世界游标）、`asset_definitions`（含 `consumable`）、`character_assets`（含 `quantity`）、`effect_definitions`、`character_effects`、`action_receipts` 全在（迁移 0006），真化不需要新表，需要解锁与数据驱动。

**根因一句话**：规则层把「演示世界的一套定义」误当成了「规则引擎本身」，定义、判定参数、能力目录、模型提示词四处复制同一份硬编码，PostgreSQL 账本表反而成了没人读的摆设。

## 2. 核心设计：定义即数据，引擎只认数据

### 2.1 分层与权限（对齐 M2 §2）

```text
世界数据（创世/导入生成）
  → skill_definitions / asset_definitions / effect_definitions   （世界级定义，含 metadata 判定参数）
  → character_skills / character_assets                           （记录级归属与余额）
规则引擎（数据驱动，不持有硬编码内容）
  → Rule Pack：按定义裁决 use_skill / use_asset / take_stance
  → Engine Tools：resolve_uncertainty（纯判定）/ consume_resource（账本扣减）/ apply_effect（账本状态）
```

- 角色/玩家只声明「用什么」（skill_key / asset_key / stance_key）；
- 是否判定、判定参数、花费与状态效果，全部由定义数据经引擎推导；
- DM 不代填任何规则值（M2 §6）；引擎工具只在发布事务内由已授权的 Receipt 驱动。

### 2.2 定义 metadata 契约（判定参数数据源）

`skill_definitions.metadata`（jsonb object）：

```jsonc
{
  "check": { "system": "d20", "modifier": 2, "target": 12, "partialMargin": 2 }, // 可缺省：无 check → 自动成功
  "defaultTargetId": "letter_seal",                           // 可缺省：能力目录 id 后缀与默认目标
  "outcomes": {                                               // 全部可缺省，缺省走通用模板
    "success": "...", "partial": "...", "failure": "...", "impossible": "...",
    "privateObservation": "...",
    "discovery": {
      "subject": "具体对象",
      "feature": "对象上可观察、可复述的特征",
      "nextCheck": "角色下一步可以实际执行的复核动作"
    }
  },
  "targets": { "<targetId>": { /* 同 outcomes 结构，按目标覆盖 */ } }
}
```

`asset_definitions.metadata` / `effect_definitions.metadata`：同 `outcomes` 结构（花费由行级 `consumable` 决定，状态由 `effect_kind` 决定，不进 metadata）。

**骰系字段预留**：`check.system` 类型为 `DiceSystem = "d20" | "2d6" | "percentile" | "pool" | "draw"`（T5 多骰系）。T4 只实现 d20 解算；定义声明其他骰系 → `RULE_DECISION_INVALID` fail-closed（不静默改判）。

**判定参数推导规则**：
1. `use_skill`：按 `skillId`（= `skill_key`）查世界定义。无定义 → `SKILL_NOT_AVAILABLE`（保持既有错误形态）。有定义：`metadata.check` 有效 → resolution=check，modifier/target 原样取自 metadata（整数、target≥1，非法 → `RULE_DECISION_INVALID`）；可选 `partialMargin=1..10` 使未达目标但未触发 fumble 的近失结果进入 `partial`，不宣称检定成功；无 check → resolution=automatic success。facts 按 `targets[targetId]` 覆盖 `outcomes`，再缺的格用通用模板（含技能 title 的确定性句式，不引入随机与模型调用）。`discovery` 只有在 success/partial 时进入对应角色的 private observation；Character Runner 只消费 `subject/feature/nextCheck/certainty`，没有结构化 discovery 时不得把 legacy `privateObservation` 原文变成对白。
2. `use_asset`：按 `assetId` 查定义。无定义 → `ASSET_NOT_AVAILABLE`。`consumable=true` → cost `{resourceId:"asset:<定义行 id>", amount:1}`；`consumable=false` → 无 cost。resolution=automatic success。
3. `take_stance`：按 `stanceId` 查 `effect_definitions`（`effect_kind='stance'`）。无定义 → `STANCE_NOT_AVAILABLE`。effect `{effectId:<定义行 id>, targetId:行动者自身, operation:"apply"}`。
4. `act`：恒定 automatic success，不读定义（自然行动无规则成本）。

### 动态 Discovery（技能执行时生成）

`discovery` metadata 只保留为没有注入 DynamicDiscoveryGenerator 时的兼容测试/本地 fallback；真实 PostgreSQL model path 不把基础 `keen_insight` 的默认 clue 当作当前世界事实。每次 `use_skill` 先完成规则 outcome，再以当前 Record 的 scene brief、world/story/canon、current worldline head 前最多 12 条公开事件、技能 key/title/description、targetId 与 Actor Tool `intent` 调用 DynamicDiscoveryGenerator。

- **基础技能禁止静态隐藏结果**：`realm.base.v1/keen_insight` 的 live metadata 只允许保留 `check` 与 `defaultTargetId`；不得写入 `outcomes`、`targets`、`privateObservation` 或 `discovery`。0038 对 0035/0036 已落库的旧定义做一次性清理，旧 migration SQL 作为不可变历史保留。
- **历史 Record 不回写**：已经提交的 action receipt/Event 仍按 append-only 规则保留；其中若含旧的固定线索，只代表过去那次演练的历史结果，不会被 0038 伪造改写。后续技能使用从清理后的定义开始。
- `success`/`partial` 才允许生成 `{subject, feature, nextCheck}`；`certainty` 由规则 outcome 回填，模型不能自行提高确定性。
- 玩家输入中的“仔细观察甲板/浓雾”等内容只作为观察焦点，不直接成为事实；没有明确焦点时生成器根据当前 objective/tension 提供一枚可继续追查的 hook。
- provider 超时、模型输出非法或发现不满足结构约束时，保留 action/public fact，private discovery 为空，不回退到旧的 deck-specific clue。
- Receipt 持久化结构化 discovery；同一 `callId` 重放走 action resolver 幂等缓存，不重新掷骰或重新生成。
- NPC discovery 只进入对应 NPC 的 private grounding；玩家 discovery 通过 viewer-local Delivery Projection 只向控制该角色的玩家展示，不扩散给其他角色。

**演示世界零回退**：demo 三种定义的 metadata 由共享模块 `modules/actions/demo-definitions.ts` 提供（`check:{d20,+2,12}`、`defaultTargetId:"letter_seal"`、蜡封桥段 facts 逐字落 metadata），`demo-seed.ts` 与本地规则包共用同一份数据；演示种子对 metadata 做幂等 upsert，存量库重跑种子即对齐。

### 2.3 规则包形态

- `createDataDrivenRulePack(provider)`：唯一裁决逻辑，provider 负责按世界加载三类定义（`modules/actions/rule-pack.ts`）。
- `createLocalRulePack()`：= 数据驱动包 + 内存 demo 定义 provider（纯内存测试组合继续可用，行为不变，未定义技能仍 fail-closed）。
- `createPostgresRulePack(database, {workspaceId, worldId})`（`database/postgres/rule-pack.ts`）：provider 走 PostgreSQL 读该世界全部三类定义（实例内缓存）。真实模型回合的 `orchestratorFactory` 注入该包（`allowStatefulReceipts: true`）。

### 2.4 能力目录解锁（action-state.ts）

- 移除三处硬编码 `WHERE`：技能/资产分支不再限定 key；姿态分支改为「本世界全部 `effect_kind='stance'` 定义且当前未 active」。
- 能力 id 数据驱动：`skill.<skill_key>[.<metadata.defaultTargetId>]` / `asset.<asset_key>` / `stance.<effect_key>` / `scene.observe_surroundings`（不变）。演示世界 id 保持 `skill.careful_observation.letter_seal` / `asset.signal_lantern` / `stance.guarded_watch`（前端与测试断言零变更）。
- 提交授权仍以「当前目录实时投影」为准（既有 1306 行路径）；静态 `KNOWN_ACTION_AFFORDANCES` 名单退役，规划期按目录解析选择并组装 Actor Tool Call（skillId/targetId/assetId/stanceId 全部从能力 id 反解），查无 → `INVALID_ACTION_SELECTION`（400，保持既有形态）。

### 2.5 创世与导入生成基础定义（数量受控）

`ensureWorldBaseRuleDefinitions(client, scope, world)`（library-service / tavern-import 共用）：

- 按世界文风（`worlds.settings->>'style'`，缺省 modern）生成确定性定义目录：**固定 2 技能 + 1 资产 + 1 姿态**（防泛滥；无模型调用）：
  - 技能一 `keen_insight`：带 `check`（d20 +1 / target 10 / partialMargin 2）和 `scene_surroundings` 目标结果，走判定分支；
  - 技能二 `steady_hand`：无 `check`，走自动成功分支；
  - 资产 `travel_kit`：`consumable=true`；
  - 姿态 `watchful_guard`：`effect_kind='stance'`。
  - title/description 按四种文风（modern/classical/western_fantasy/anime）各一套固定文案；`ON CONFLICT (workspace_id, world_id, key) DO NOTHING`，重复调用幂等。
- **生成时机**：
  - `createGenesis`（引导创世）→ 记录装配 `assembleDefaultRecord` 内确保定义存在并授权；
  - 手动「世界 → 故事 → 记录」流程同样经 `assembleDefaultRecord`，同一入口兜底；
  - 酒馆导入（角色卡/世界书）→ 目标世界确保定义存在（导入时不授权；授权发生在记录装配/角色挂记录时，`attachCharacterToRecord` 对挂入角色补授权）。
- **授权（记录级）**：装配时本世界全部基础技能授权给玩家实例与 AI 角色实例（`character_skills`，acquired (0,0)，`ON CONFLICT DO NOTHING`）；基础资产仅授权玩家实例（quantity 2）。演示世界不受影响（demo 记录不走装配路径，其定义与授权由种子提供）。
- **角色提示词数据驱动**：`model-powered.ts` 的 `use_skill` enum 改为按角色实际持有技能生成（provider 走 `character_skills`+`skill_definitions`，含世界游标门禁；加载失败 fail-closed 为空集 = 只能 `act`）；`parseActorToolCall` 以持有集校验 skillId，target 不再限定枚举（任意稳定标识，规则包按定义裁决）。角色未持有任何技能时工具列表只保留 `act`。

## 3. 引擎工具语义（M2 §4 落地）

三个引擎工具收拢到 `modules/actions/engine.ts`，全部引擎权威——DM/角色均不代填：

| 工具 | 形态 | 语义 |
|---|---|---|
| `resolve_uncertainty` | 纯函数（无数据库） | 输入 seed + check 参数，输出 MechanicDetail 与成败。T4 仅 d20（确定性哈希，T5 换真随机与多骰系）；未知骰系 fail-closed `RULE_DECISION_INVALID`。`resolveDecision` 的 check 分支改为调用本工具。 |
| `consume_resource` | 账本事务内函数（PoolClient） | `character_assets.quantity` 原子扣减（`quantity >= amount` 守卫 + revision++）；余额不足 → `RESOURCE_UNAVAILABLE`；非 `asset:` 资源种 → `RESOURCE_KIND_NOT_AVAILABLE`；定义不存在或非 consumable → `RESOURCE_DEFINITION_UNKNOWN`。 |
| `apply_effect` | 账本事务内函数（PoolClient） | `character_effects` apply（唯一 active 约束，重复 → `EFFECT_ALREADY_ACTIVE`）/ remove（无 active → `EFFECT_NOT_ACTIVE`）；目标必须是行动者自身（`EFFECT_TARGET_NOT_AUTHORIZED`）；effect 定义必须存在于本世界（`EFFECT_DEFINITION_UNKNOWN`）。 |

- `runtime-repository.ts` 的 `applyActionState` 不再内联扣减/状态 SQL，改为顺序调用 `consume_resource` / `apply_effect`；三者仍与源事件、Receipt、Head 同事务原子提交（M2 §8）。
- `action_receipts` 保持 append-only；Receipt 的 `effects/costs` 只来自 Rule Pack 决定，引擎工具不接受 DM 或角色直填。

## 4. 状态账本闭环

1. **apply**：`take_stance` 成功 → `character_effects` 落 active 行（source_action_receipt_id 回指）。
2. **可见性回流**：能力目录的姿态分支排除已 active 的效果——同一姿态不可重复进入（目录层隐藏 + 账表层 `EFFECT_ALREADY_ACTIVE` 双保险）。
3. **扣减**：`use_asset` 成功 → quantity 真实扣减；quantity=0 → 目录层隐藏 + 账表层 `RESOURCE_UNAVAILABLE` 双保险。
4. **expire/remove**：`apply_effect` 支持 remove 语义（引擎工具完备）；主动移除的 Actor 入口不在 T4 范围（姿态随场景演进的自然过期留待后续批次）。
5. **幂等**：同一 Call ID + 同一参数重放同一 Receipt（解析器内存指纹，既有机制）；发布层 turn 完成态短路（既有 `commitRelease` completed 分支），重放不重复扣减、不重复施加效果。

## 5. 失败矩阵（全部 fail-closed，绝不静默改判）

| 场景 | 错误码 | 形态 |
|---|---|---|
| use_skill 无定义 | `SKILL_NOT_AVAILABLE` | FatalTurnError（保持既有） |
| use_asset 无定义 | `ASSET_NOT_AVAILABLE` | FatalTurnError（保持既有） |
| take_stance 无定义 | `STANCE_NOT_AVAILABLE` | FatalTurnError（保持既有） |
| metadata.check 非法（非整数/target<1） | `RULE_DECISION_INVALID` | FatalTurnError |
| check.system 非 d20（T5 预留） | `RULE_DECISION_INVALID` | FatalTurnError |
| 余额不足 | `RESOURCE_UNAVAILABLE` | FatalTurnError（保持既有） |
| 非 asset: 资源种 | `RESOURCE_KIND_NOT_AVAILABLE` | FatalTurnError（保持既有） |
| 扣减目标定义缺失/非 consumable | `RESOURCE_DEFINITION_UNKNOWN` | FatalTurnError（新增） |
| 效果定义不在本世界 | `EFFECT_DEFINITION_UNKNOWN` | FatalTurnError（新增） |
| 重复进入姿态 | `EFFECT_ALREADY_ACTIVE` | FatalTurnError（保持既有） |
| 移除未 active 效果 | `EFFECT_NOT_ACTIVE` | FatalTurnError（保持既有） |
| 效果目标非行动者自身 | `EFFECT_TARGET_NOT_AUTHORIZED` | FatalTurnError（保持既有） |
| 过期/伪造能力选择 | `INVALID_ACTION_SELECTION` | LocalRecordServiceError 400（保持既有） |
| 角色调用未持有技能 | `CHARACTER_ACTION_INVALID` | FatalTurnError（保持既有） |
| 纯内存组合启用 stateful 工具 | `STATEFUL_ACTION_NOT_AVAILABLE` | FatalTurnError（保持既有） |

## 6. 交付步骤与 commit 纪律

1. 规范（本文档）单独 commit。
2. **解锁与数据驱动**：action-state.ts 三处 WHERE 移除 + 能力 id 数据驱动；规则包数据驱动（本地 demo provider + PG provider）；demo 种子 metadata 幂等 upsert（共享 demo-definitions）；选择解析目录化（KNOWN_ACTION_AFFORDANCES 退役）；创世/装配/导入生成基础定义与授权；角色 use_skill 提示词按持有技能生成。
3. **引擎工具**：engine.ts 三工具落地；resolveDecision check 分支改走 resolve_uncertainty。
4. **账本闭环**：applyActionState 改走 consume_resource/apply_effect；定义权威校验（RESOURCE_DEFINITION_UNKNOWN/EFFECT_DEFINITION_UNKNOWN）；真实模型路径注入 PG 规则包。
5. **测试收口**：PG 集成 + 应用层 + GUI 真实模型 t4 spec；STATUS 三段式与迭代日志；清理回基线。

禁 `git add -A`；每步独立 commit。

## 7. 验收标准

1. `npm test` 全链绿（typecheck + core + contracts + build + render）；`npm run lint` 当次实测 0 error。
2. PG 集成新增并通过：
   - 数据驱动能力目录——自定义技能/资产/姿态定义落库后目录可见（非 demo key）；
   - 创世装配后基础定义与授权存在（数量恰为 2 技能 + 1 资产 + 1 姿态）；
   - 引擎工具账本闭环——use_asset 回合后 quantity 真实扣减、action_receipts 落库；take_stance 回合后 character_effects active；重复进入 `EFFECT_ALREADY_ACTIVE`；余额归零后 `RESOURCE_UNAVAILABLE`；重放幂等不重复扣减。
3. 应用层新增并通过：判定参数推导（metadata.check 生效、无 check 自动成功、非法 check fail-closed、targets 覆盖 facts、未定义 fail-closed）。
4. GUI 真实模型（不 mock）：自建世界（非 demo）使用生成的技能成功落库（时间线可见）；资产扣减可见（目录「剩余 n 次」2→1）。
5. 演示世界行为零回退：C 组/o-action-suggestions 既有断言不改或仅按新事实同步；demo 蜡封桥段、信号灯扣减、警戒姿态照旧。
6. 范围化回归（总纲第五章）：本批新增 t4 spec + c-actions、o-action-suggestions、l-genesis、n-guided-genesis、t1-first-night + 冒烟 a-library、b-record；失败逐一复跑定性，数据敏感用例复跑前先清理。
7. 每次 GUI/集成写入后执行 `scripts/clean-gui-test-data.sql`，计数回基线（worlds=2 / accounts=1 / first_nights=0）。
