# M2 实施规范：角色行动与规则裁决

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 开发边界：本阶段可使用本机 Fake Provider，或由用户显式配置并启用的模型供应商；外部生成只发送经过时间、可见性和角色知识过滤的最小上下文。规则裁决与 PostgreSQL 事实源仍位于本机，不接入外部消息网关。

## 1. 核心原则

角色负责决定“我要做什么”，规则系统负责决定“这件事如何裁决”，DM 负责决定“谁被激活、回合是否完整可信”。

```text
玩家 Command
→ DM 激活少量角色并给出 Goal / 约束
→ Character 通过 Actor Tool 声明自然行动
→ Action Resolver 查询 Rule Pack
→ Engine Tool 在内部完成不确定性与效果裁决
→ 生成 Action Receipt / Action Transaction
→ Narrator 只叙述公开世界事实
→ Character 只根据自己的 Observation 回应
→ DM 检查身份、事务闭合与输出边界
→ Runtime 原子发布正式 Event
```

禁止以自然语言关键词、正则或“是否出现观察/骰子”等词语决定规则动作。DM 不把角色文本翻译成骰点，也不代理角色选择技能。

## 2. 三类权限

| 概念 | 决策者 | 结构 |
|---|---|---|
| 行动、技能、资产、主动姿态 | 玩家或 Character | Actor Tool Call |
| 是否判定、使用哪种规则机制 | Action Resolver + Rule Pack | Engine Tool / Rule Decision |
| 状态、资源和世界事实变化 | 已完成的裁决结果 | Effect / Cost / Action Receipt |

技能、判定和状态不处于同一权限层。角色可以主动使用技能，但不能自行指定 DC、骰点或成功；角色可以主动进入警戒姿态，但不能自行给敌人添加眩晕或给自己移除中毒。

## 3. Actor Tool

第一批角色可见工具保持极少：

| 工具 | 含义 | 第一批状态 |
|---|---|---|
| `act` | 尝试一个自然行动 | 启用 |
| `use_skill` | 使用角色确实拥有的技能 | 启用 |
| `use_asset` | 使用道具、装备或有限资源 | 对服务端授权的玩家行动启用 |
| `take_stance` | 主动进入角色可控制的姿态 | 对服务端授权的玩家行动启用 |

角色不调用 `resolve_check`，也不填写骰面、属性修正或难度值。

普通行动示例：

```json
{
  "name": "act",
  "arguments": {
    "intent": "确认蜡封是否被人动过",
    "targetId": "letter_seal",
    "approach": "careful"
  }
}
```

技能示例：

```json
{
  "name": "use_skill",
  "arguments": {
    "skillId": "careful_observation",
    "targetId": "letter_seal",
    "intent": "辨认被重新封装的痕迹"
  }
}
```

Core 绑定 CharacterInstance、Participant、Record、Worldline 与时间，不让模型重复填写运行时身份和游标。

## 4. Engine Tool 与 Rule Pack

引擎内部工具不进入角色提示词：

| 工具 | 用途 | 第一批状态 |
|---|---|---|
| `resolve_uncertainty` | 由规则包解决不确定行动 | 启用 |
| `consume_resource` | 扣除次数、物品、法力或其他资源 | 通过 Action State Ledger 启用 |
| `apply_effect` | 添加、更新或移除状态效果 | 通过 Action State Ledger 启用 |

Rule Pack 可以把同一个 `use_skill` 决定为：

- 自动成功或自动失败；
- d20、2d6、百分骰、骰池或抽牌；
- 对抗判定；
- 成功、部分成功、失败或不可能；
- 需要确认、资源或额外前置条件。

Character 和玩家默认都不读取内部机械细节。规则算法、原始随机值、修正和目标值仅保存在受保护 Receipt 中，除非未来显式开启规则诊断视图。

## 5. Action Receipt

```ts
interface ActionReceipt {
  callId: string;
  status: "resolved";
  resolution: "automatic" | "check" | "contest";
  outcome: "success" | "partial" | "failure" | "impossible";
  publicFacts: string[];
  privateObservations: ObservationDraft[];
  effects: EffectDraft[];
  costs: CostDraft[];
  mechanic?: MechanicDetail;
}
```

- Narrator 只读取 `publicFacts`；
- Character 只读取属于自身的 `privateObservations`；
- 状态与资产服务只处理已授权的 `effects / costs`；
- `mechanic` 不进入普通玩家时间线和角色台词；
- 同一个 Call ID 与同一参数必须重放同一 Receipt，更换参数必须冲突。

第一批仍正式保存 `action.transaction.committed`，但它是控制面审计事件。玩家 Delivery Projection、GET、POST 与 SSE 只显示玩家发言、公开旁白和角色回应，不显示规则卡、骰点或内部 Transaction。

## 6. DM 的职责边界

DM 可以：

- 规划回合 Goal 和硬约束；
- 激活少量角色并保持强沉默偏置；
- 验证 Actor Tool 是否确由该 CharacterInstance 提出；
- 验证每个已批准行动都有且只有一个完成 Receipt；
- 验证 Narrator、Character 的语义片段和身份边界；
- 拒绝未闭合、越权或违反世界硬约束的候选。

DM 不可以：

- 从自然语言正则猜测技能、判定或状态；
- 代替角色选择行动或技能；
- 因模型漏调工具而静默代理不可逆操作；
- 把内部机械信息写进角色台词；
- 把控制面候选、Goal 或 Context Manifest 发送给玩家。

## 7. 弱工具模型降级

```text
原生 Tool Call
→ 直接校验 Actor Tool

结构化输出可靠
→ 输出 ActorActionProposal
→ Adapter 转为同一 Actor Tool

两者都不可靠
→ 独立 Intent Compiler 做窄范围结构化
→ 低置信度、状态变更或高价值操作一律拒绝/确认
```

不使用关键词表冒充语言理解。缺少必须的 Actor Tool 时进行一次窄范围重试；仍失败则明确失败，不能让 DM 静默制造角色意图。

## 8. 原子发布与恢复

通过 DM 验收后，一个回合按固定顺序组成同一发布包：

1. 玩家 `utterance.committed`；
2. 控制面 `action.transaction.committed`；
3. Narrator `narration.committed`；
4. Character `utterance.committed`；
5. 合法 Observation、Record/Worldline Head 与 Outbox。

模型和规则计算不持有数据库事务。状态工具启用前，Receipt、资源变化、效果变化、Event 与 Head 必须进入同一 PostgreSQL 事务，不能依赖进程内缓存。

## 9. 分批实施

### 批次 A：行动与裁决分层

- Actor Tool、Action Resolver、Rule Pack 和 Receipt；
- Fake Character 始终输出结构化 `act`，不解析玩家措辞；
- `use_skill` 可触发内部确定性判定；
- 状态工具失败关闭；
- 内部 Transaction 不进入玩家时间线。

### 批次 B：真实模型与 Context

- Character 原生选择 `act / use_skill`；
- Provider 能力探测与 Intent Compiler 降级；
- 各组件分别编译最小合法 Context；
- 稳定缓存前缀与动态尾部隔离；
- 不让全知玩家投影污染 Character Context。

### 批次 B2：原生行动面板

- 自然语言输入始终是主入口，不要求玩家记忆命令或手写结构；
- 服务端按 Principal、CharacterInstance、Scene、Record 和世界时间投影当前可用 `ActionAffordance`；
- 前端只做渐进披露和稳定 ID 选择，不自行推断技能、库存或合法目标；
- 已选择项与原始自然语言分别提交，Core 重新校验稳定 ID 后才生成 Actor Tool Call；
- 玩家主动行动与 AI 角色自主行动分别保留来源和权限，不允许 DM 代填；
- 只开放已有 Rule Pack 和持久化账本能真实裁决的技能、物品、姿态与场景目标。

```ts
interface ActionAffordance {
  id: string;
  kind: "skill" | "asset" | "stance" | "scene";
  actorCharacterInstanceId: string;
  title: string;
  description: string;
  suggestedText: string;
}

interface ActionSelection {
  affordanceId: string;
}
```

`ActionAffordance` 是针对当前玩家投影的短期能力目录，不是永久事实源。提交时必须重新授权；不能把客户端显示过某个选项当作仍然可用的证据。

### 批次 C：资产、状态工具与确认

- 技能归属、资产余额、效果和 Action Receipt 使用 Workspace 隔离的持久化账本；
- `use_asset / take_stance / consume_resource / apply_effect` 与来源 Event、Observation、Head 和 Outbox 原子提交；
- ActionAffordance 从当前 Principal 控制权、CharacterInstance、Record、库存余额和活动效果实时投影；
- 重放不重复扣除物品或施加效果，余额不足、效果已存在和过期选择全部失败关闭；
- 仅歧义、不可逆和高价值操作二次确认；
- 确认超时、拒绝、重试和并发冲突处理。

### 批次 D：创建流程与规则包

- World、Character、Story、Record 的最小创建界面；
- 通用轻规则包与可替换 Rule Pack；
- Perspective 创建时锁定，动态知识面板可变；
- 不承诺完整 D&D 规则兼容。

## 10. 批次 A 验收标准

- 项目不存在以玩家自然措辞触发骰子或判定的实现；
- Character 通过 Actor Tool 自己声明行动，DM 不代理；
- `act` 不强制判定，`use_skill` 是否判定由 Rule Pack 决定；
- 内部机械值不进入 Narrator、Character、GET、POST、SSE 或普通时间线；
- Action Transaction 仍与旁白和角色回应原子提交、幂等恢复；
- 状态变更工具只有在注入持久化账本的运行组合中启用；纯内存测试组合继续失败关闭；
- Narrator 越权、身份替换、重复行动或缺少 Receipt 时被拒绝；
- 真实 PostgreSQL、应用服务、构建与内置浏览器回归通过。
