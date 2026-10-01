# T6 实施规范：导入卡参局

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 上位设计：[`TAVERN-IMPORT.md`](./TAVERN-IMPORT.md)（导入格式）、[`T4-RULE-REALIZATION.md`](./T4-RULE-REALIZATION.md)（数据驱动规则包与授权）、[`T5-DICE-RANDOMNESS.md`](./T5-DICE-RANDOMNESS.md)（多骰系）。
> 迭代背景：[`EXPERIENCE-ITERATION.md`](./EXPERIENCE-ITERATION.md) 第二章缺陷 #6「导入的角色卡不参局」。

## 1. 现状实锤（审计结论，勿重复调查）

1. **进阵容已有两条路**：新建记录装配（`assembleDefaultRecord` 把本世界全部自定义角色——含导入卡——入阵）与既有记录挂入（`attachCharacterToRecord`，continuity key `custom_<definitionId>` 查重幂等）。T4 已在这两条路上给 AI 实例授权**基础技能**（`grantBaseSkillsToInstances`）。
2. **资产未授权**：两条路径都不给 AI 实例授权基础资产——`character_assets` 只有玩家实例行（quantity 2）。导入角色的资产授权缺失。
3. **卡自带技能定义来源不存在**：酒馆卡是纯文本人设，parser（`modules/import/tavern-parser.ts`）不读 V2 标准扩展点 `data.extensions`，「卡携带技能定义」目前无落点、无授权、无参局。
4. **导入卡无「挂入既有记录」入口**：library UI 的挂入（attachRecordId）只对新建原生角色开放；导入卡只能等新建记录装配进阵容。
5. **参局面已就绪待验证**：AI 角色 `use_skill` 提示词按持有技能生成（T4 characterSkillProvider）、数据驱动规则包裁决、T5 骰点投影——导入角色经授权后即具备参局条件，本批闭环验证。

## 2. 核心设计

### 2.1 卡携带技能来源：extensions.realm_skills

SillyTavern V2 标准扩展点 `data.extensions.realm_skills`（JSON 数组）：

```jsonc
{
  "data": {
    "extensions": {
      "realm_skills": [
        {
          "skillKey": "cartography",          // 合法标识（小写字母/数字/下划线）
          "title": "制图术",
          "description": "描绘并判读地形。",
          "check": { "system": "2d6", "modifier": 1, "target": 8 }  // 可缺省=自动成功；五骰系同 T5 契约
        }
      ]
    }
  }
}
```

- **解析**：parser 透传 `extensions.realm_skills`（`TavernCharacterCard.realmSkills`）。逐条校验：skillKey 合法标识（`/^[a-z][a-z0-9_]{0,49}$/`）、title/description 非空、check 经 `parseCheckSpecification` 预验——**非法条目跳过并把 skillKey 列入导入报告 warnings**（显式报告，不阻断导入，不静默参局）。
- **落库**：导入事务内落 `skill_definitions`（`rule_pack_key='realm.imported.v1'`，id `imported_skill_<skillKey>_<worldId 尾>`），`ON CONFLICT (workspace_id, world_id, skill_key) DO NOTHING`（与世界既有定义同 key 时保留既有行，重复导入幂等）。
- **归属记录**：character profile 增 `realm_skill_keys: string[]`——该卡合法技能 key 列表，授权时的卡→技能关联键（不改表结构，无新迁移）。

### 2.2 授权闭环（装配与挂入统一收口）

新增授权 helper `grantCharacterRuleAssets`（library-service 内），装配（`assembleDefaultRecord` 自定义角色分支）与挂入（`attachCharacterToRecord`）统一调用，对**自定义角色实例**（含导入卡）：

1. 基础技能：`grantBaseSkillsToInstances`（既有，幂等）；
2. 卡自带技能：按角色 profile `realm_skill_keys` 查定义行 → `character_skills` 授权（acquired (0,0)，`ON CONFLICT DO NOTHING`）；
3. 基础资产：`character_assets` 授权 quantity **1**（AI 实例新增；玩家实例 quantity 2 不变——玩家主角余量差异为定案），`ON CONFLICT DO NOTHING`。

幂等：挂入的 continuity 查重早退（既有）+ 授权全部 ON CONFLICT——重复装配/挂入不产生重复行。

### 2.3 后续挂入通道（导入卡可挂入既有记录）

- `LibraryCreateCommand` 新增 `{ kind: "attach-character"; worldId; recordId; definitionId }` → 复用 `attachCharacterToRecord`（含 2.2 授权）。definition 不存在/不属于该世界 → `LibraryServiceError("WORLD_NOT_FOUND" 形态沿用)` fail-closed。
- API `POST /api/library` 请求体增 `recordId` / `definitionId` 透传（既有字段校验形态）。
- UI：library 世界卡的角色行，当该世界是当前打开记录所属世界、且角色不在阵容时，显示「入阵容」按钮（三语 i18n `ui.library.attachToCast`）；点击后发 attach-character 命令，成功重载当前记录（沿用 onRecordReload）。

### 2.4 参局闭环（无新代码，验证收口）

授权后导入角色即具备参局条件（T4/T5 链路）：AI 角色 react/propose 提示词按持有技能（基础 + 卡自带）生成；`use_skill` 经数据驱动规则包裁决，`metadata.check` 走 T5 多骰系解算；骰点随 Receipt 物化并进 Delivery Projection 玩家可见。本批以 PG 集成 + GUI 真实模型验证闭合。

## 3. 失败矩阵（fail-closed）

| 场景 | 形态 |
|---|---|
| realm_skills 条目非法（key/title/check） | 跳过该条 + warnings 显式报告（不阻断导入） |
| realm_skills 非数组 | 视为无卡技能（缺省） |
| attach-character 的 definition 不存在/不属该世界 | `WORLD_NOT_FOUND` LibraryServiceError |
| attach-character 重复挂入 | continuity 查重幂等跳过（既有形态） |
| 导入角色调用未持有技能 | `CHARACTER_ACTION_INVALID` / `SKILL_NOT_AVAILABLE`（既有形态） |
| 卡技能 check 非法 | 导入期 parseCheckSpecification 预验拦截 → warnings，不落库 |

## 4. 交付步骤与 commit 纪律

1. 规范（本文档）单独 commit。
2. **授权闭环**：parser realm_skills 透传；导入落 skill_definitions + profile.realm_skill_keys + warnings；grantCharacterRuleAssets（基础技能 + 卡技能 + 基础资产 quantity 1）接入装配与挂入两路径。
3. **挂入通道**：attach-character 命令 + API 透传 + UI 入阵容按钮 + i18n。
4. **测试收口**：PG 集成（导入→授权幂等→裁决参局→fail-closed）+ parser/应用层 + GUI t6-tavern-import.spec.ts 真实模型；STATUS 三段式与迭代日志；清理回基线。

禁 `git add -A`；每步独立 commit。

## 5. 验收标准

1. 导入角色授权幂等（重复装配/挂入不重复授权行；卡技能与基础定义两条来源都正确落库）。
2. GUI 真实模型：导入角色（带 realm_skills 自定义检定技能）回合内使用技能成功落库，骰点可见（延续 T5 投影锚点）。
3. 非持有技能 fail-closed 报错形态明确。
4. demo 零回退：既有断言不改通过。
5. 范围化回归（总纲第五章）：t6 新 spec + r-tavern-import + c-actions + t4-rule-realization + t5-dice + 冒烟 a-library/b-record，双引擎；失败逐一复跑定性，数据敏感用例复跑前先清理。
6. 每次 GUI/集成写入后执行 `scripts/clean-gui-test-data.sql`，计数回基线（worlds=2 / accounts=1 / first_nights=0 / action_receipts=0）。
