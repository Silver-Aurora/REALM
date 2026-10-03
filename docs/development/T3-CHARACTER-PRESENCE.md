# T3 · 角色在场：回合内角色主动发声与环境/关系反应

> 批次 T3（docs/development/EXPERIENCE-ITERATION.md 迭代主轴第三项，对应第二章缺陷#2「角色在场感弱」）
> 立项：2026-08-20 +08:00 · 状态：规范定稿，实施中
> 一句话目标：角色从「等玩家触发才回应」变成「真实在场」——玩家回合提交后，未发声角色会基于叙事上下文（环境变化、其他角色行动、未响应钩子）主动发声，预算受控、fail-closed、走既有事件形态。

## 一、现状调研（读代码确认）

### 1.1 角色发声只有两条被动路径（本批根因）

`modules/orchestration/model-powered.ts` 中角色发声仅有：

- `CharacterRunner.propose`（约 :438）——玩家回合内，DM 激活角色后由玩家输入触发；
- `CharacterRunner.react`（约 :489）——同回合内对玩家输入/自身行动结果的回应。

两条路径全部挂在 `submitMessage → executeTurn` 的玩家回合关键路径上
（`modules/application/local-record-service.ts` draft 阶段），**没有任何角色发声
发生在玩家回合之外**。唯一的例外是：

- M4 自动插话（`modules/orchestration/interjection.ts`）——但四道门的第一道就要求
  「玩家文本逐字点名且未被激活」，强沉默偏置下绝大多数回合无人可插话；
- T1 初夜同行者发声（`database/postgres/first-night-store.ts`）——一次性开场，
  进场后角色回到被动。

后果即缺陷#2 的实锤：角色之间无互动（没有角色回应另一个角色的台词）、对环境
变化无反应（旁白写了天气/光线/剧情推进，无人接话）、彼此无关系表达
（CharacterMemoryService.relationships/recordRelationship 接口存在但游玩闭环零消费，
即缺陷#11 所列死接口之一）。

### 1.2 既有可复用资产

- **事件形态**：`utterance.committed` + presentation payload（语义段）+ 可见性策略，
  插话回合（releaseBuilder interjection 分支）已示范「无玩家事件、仅角色回应」的
  最小发布形态；
- **回合机械**：`executeTurn` 全套（计划→起草→校验→发布→事务提交→世界坐标分配
  →observations 落库→delivery outbox→SSE 投递），插话回合同型复用；
- **发言权**：`TurnControl`（`modules/runtime/turn-control.ts`）——Record 级发言租约
  （FIFO 队列）与角色级冷却（默认 60s），插话已在用；
- **T2 记忆管线**：回合开始 `memoryPrefetch.begin` 并行召回、请求体组装同步
  `consume`（fail-closed）；回合后 `memorySync.schedule` 异步萃取。角色在场发声
  必须带着召回记忆，且其产生的 observations 也要进入萃取循环；
- **关系接口**：`CharacterMemoryService.relationships`（主观关系投影视图）与
  `recordRelationship`（追加 relationship 类型结论），`modules/memory/public.ts`。

### 1.3 回合管线时序（现状 + 本批插入点）

```
玩家输入 → submitMessage
  → prefetch.begin（并行召回）
  → 可见性裁决（模型）→ executeTurn（DM 规划/propose/react/旁白/复核）
  → commitRelease（events + observations 落库）→ 信封读回
  → [M4] 插话评估：点名且未激活 → 插话回合（executeTurn）
  → [T3 插入点] 未插话 → 在场评估（本批新增）
  → sync_turn 萃取 / 设定结晶（既有）
  → 响应
```

在场评估与插话共享「回合后自治发声」时机：**插话优先**（玩家点名是显式诉求），
本回合已发生插话则在场跳过；两者共用发言租约与角色冷却，绝不并发写同一 Record。

## 二、设计方案

### 2.1 总体形态：presence pass（回合后在场评估）

```
玩家回合提交成功（非 replay、public 可见性）
  │  同步消费 prefetch 就绪结果（每个候选角色一份记忆文本）
  ▼
schedulePresence（fire-and-forget，任何失败只留日志）
  ├─ 确定性抽取：在场上下文（旁白环境/剧情段、角色台词、行动公开事实、未响应钩子）
  ├─ 确定性资格：候选 = 本场 AI 角色 − 本回合已激活/已发声 − 冷却中；预算余额 ≥1
  ├─ 模型门禁（1 次 json_object 调用）：选谁说、因何说；无充分动机 → 沉默
  └─ 若发声：取发言租约 → 读关系视图 → executeTurn（presence 回合）
       → 提交成功：记冷却 + 再触发 sync_turn + （可选）recordRelationship
       → 任何失败：日志收场，不回灌玩家回合
```

### 2.2 触发时机与确定性抽取

在场发声只由叙事上下文驱动，共三类触发素材（`extractPresenceContext`，纯函数）：

| 触发 | 来源 | 素材内容 |
| --- | --- | --- |
| `environment` | 本回合旁白的 environment/story 段 | 场景变化、剧情推进句 |
| `peer` | 本回合其他角色的台词与行动公开事实 | 谁说了/做了什么（摘要） |
| `hook` | `firstNight.load(recordId)` 的 hookContent（ready/degraded） | 开场钩子全文 |

- 三类素材全空（例如空旁白空回应的退化回合）→ **确定性沉默**，不发起模型调用；
- hook 素材存在与否只作为门禁上下文（「玩家是否已回应钩子」由门禁按语义判断，
  不做脆弱的关键词判定）。

### 2.3 模型门禁（createModelPresenceAssessor）

单次 `json_object` 调用（temperature 0，maxTokens 200，失败/越权/格式损坏一律
fail-closed 为沉默）：

- 输入：在场上下文 + 候选名单（characterInstanceId + displayName）+ 预算余额；
- 输出契约：`{shouldSpeak:boolean, characterInstanceId?:string, triggerKind?:'environment'|'peer'|'hook', reason:string}`；
- 裁决规则写进 system prompt：
  1. 强沉默偏置——只为「值得反应」的素材发声：场景出现实质变化、其他角色的
     言行值得接话、钩子仍悬置且角色有动机提醒；
  2. 有实质触发素材时通常应选一名角色给出简短反应（在场是常态，沉默是例外，
     但绝不刷存在感）；
  3. 只能从候选名单选人；反应不得复述秘密、不得替玩家做决定；
- 确定性校验：`shouldSpeak=true` 但 characterInstanceId 不在候选 / triggerKind 越界
  → 视为沉默。

### 2.4 预算与频率控制（防刷屏，硬性约束）

| 闸门 | 规则 |
| --- | --- |
| 回合预算 | 每玩家回合至多 `PRESENCE_MAX_PER_TURN = 1` 次在场发声（常量硬上限 2，依赖注入可降不可升） |
| 插话优先 | 本回合插话评估为 interject → 在场整体跳过（显式点名 > 自主反应） |
| 候选排除 | 本回合被 DM 激活或已发声的角色不进候选（不重复刷同一人） |
| 角色冷却 | 复用 `TurnControl.canInterject/recordInterjection`（默认 60s）；在场发声成功同样记冷却 |
| 发言租约 | 复用 `TurnControl.acquire`：在场回合持租约执行，与玩家回合/插话/下一在场互斥（FIFO） |
| 可见性门禁 | restricted 回合（秘密）之后不评估在场——场外角色未听见，反应即泄密；replay 回合不评估 |
| 失败即沉默 | 门禁/生成/校验/提交任一环节失败 → 本轮沉默，不重试刷存在 |

预算语义：预算是「每玩家回合的在场发声次数」，与插话合计后玩家回合后的自治
发声至多 1（插话发生）或 1（在场发生）次——房间里有人，但绝不两人抢话。

### 2.5 生成与落库（presence 回合）

presence 回合是一次完整 `executeTurn`，载荷标记 `payload.presence`，各阶段对齐
插话回合的最小形态：

- **planner**：固定计划——单一候选角色、`narratorEnabled=false`、
  `actionBudgetPerCharacter=0`（在场反应永不触发规则引擎，绕开 demo 写死规则包
  的 FatalTurnError 面，缺陷#8 免疫）；计划体携带 presence 上下文
  （triggerKind/triggerText/记忆文本/关系视图）供 drafter 使用；
- **drafter**：`orchestrator.draft`，`CharacterRunner.react` 新增 presence 分支：
  模型只输出 `{action, dialogue, relationship?}`——action 为不改变世界状态的轻微
  动作（≤320 字），dialogue 为角色本人台词（≤500 字，规整引号），relationship
  可选（`{target, note}`，仅当本次反应真实改变了对某人看法时输出）；
  提示词注入：触发素材、T2 prefetch 记忆文本、relationships 关系视图、世界 canon
  与文风（buildSceneCanon）；
- **validator**：仅结构校验（createRuleBasedDMController：恰好一条目标角色回应、
  零行动事务）——**不走模型复核**，杜绝 DM_OUTPUT_REJECTED（缺陷#12）面；
- **releaseBuilder**：`utterance.committed` + outbox delivery，与插话同型；
  载荷附 `presence: { trigger: triggerKind }` 标记（不是新事件类型，只是 payload
  元数据）；
- **提交管线**：既有 runtime repository 事务——events、observations（public-scene
  全员观察）、record_heads/worldlines 头指针推进、delivery outbox；SSE
  （/api/record/events，750ms 轮询）自动把在场事件投给在线玩家与观察者；
- **投影透出**：delivery projection 从 payload 解析 presence 标记 → 投影事件附
  `presence` 字段 → 前端 TimelineEvent 透出 → 时间线卡片附 `data-presence`
  属性（GUI 断言锚点，T7 观察者内容供给同源于此）；
- **关系落库**：presence 回合提交成功且 relationship 有效（target 必须精确匹配
  在场角色/玩家名，note ≤80 字）→ `characterMemory.recordRelationship`
  （fidelity 0.6），失败只留日志。

### 2.6 与记忆系统的接线（消费 T2，不另起炉灶）

1. **记忆注入**：submitMessage 在提交成功后、prefetch 会话 finally end 之前，
   同步 `consume` 每个候选角色的预取记忆文本并带入 presence 载荷——在场发声与
   回合内发声消费同一份 T2 召回结果；prefetch 生命周期（begin/end 各一次）不变；
2. **关系读取**：presence 回合执行前调用 `CharacterMemoryService.relationships`
   （主观关系视图）注入提示词——角色带着既有关系认知说话；
3. **关系写入**：见 2.5 关系落库——走 `recordRelationship` 既有接口；
4. **萃取循环**：presence 回合的 observations 随提交落库，成功后再触发一次
   `memorySync.schedule`——在场发声本身也会沉淀为记忆（sync_turn 幂等，re-arm
   语义已由 T2 保证）。

### 2.7 可见性与观察者模式

- presence 事件一律走 public 可见性策略（在场反应是公开场景内容；restricted
  回合后根本不评估在场，见 2.4）；
- 观察者模式：presence 事件即普通 public `utterance.committed`，delivery
  projection 对 observer membership 同权可见——T7 观察者的内容供给直接复用，
  无需观察者专用通道。

## 三、失败矩阵

| # | 失败点 | 行为 | 玩家影响 |
| --- | --- | --- | --- |
| F1 | prefetch consume 无会话/未就绪 | 记忆文本为空串，角色「没有额外记忆」也发声 | 无 |
| F2 | firstNight.load 抛错 | hook 素材按空处理（fail-closed） | 无 |
| F3 | 门禁模型调用失败/超时 | RetryableTurnError 就地重试一次；仍失败 → 本轮沉默 | 无 |
| F4 | 门禁输出格式损坏/越权（选人越候选、triggerKind 越界） | 确定性判为沉默 | 无 |
| F5 | relationships 读取失败 | 关系视图按空注入 | 无 |
| F6 | presence react 模型失败/输出不合格 | FatalTurnError → presence 回合 failed → 日志 | 无 |
| F7 | presence 回合提交撞版本（玩家已提交下一回合） | RECORD_VERSION_CONFLICT → 日志，事件不落库 | 无 |
| F8 | recordRelationship 写入失败 | 日志；发声事件本身已提交 | 无 |
| F9 | schedulePresence 任意未捕获异常 | 外层 try/catch 日志收场 | 玩家回合响应绝不受影响 |
| F10 | restricted 回合后误入评估 | 调用方门禁直接跳过（不进 schedulePresence） | 秘密不泄 |

## 四、验收标准

### 4.1 Core（触发机制与门禁）

1. `extractPresenceContext`：旁白环境/剧情段、角色台词、公开事实、钩子各自独立
   抽取；全空上下文确定性沉默；
2. 候选过滤：本回合已激活/已发声角色、冷却中角色不进候选；
3. 模型门禁（fake gateway）：合法选择透传；选人越候选/triggerKind 越界/格式损坏
   → 沉默；
4. presence react 分支（fake gateway）：输出 action+dialogue 语义段；relationship
   有效时附带；dialogue 引号规整。

### 4.2 应用层（触发时机与 fail-closed，内存仓储）

1. 回合提交后触发在场评估：非 replay、public、未插话、有触发素材、有候选 →
   presence 事件落时间线（speaker=选定角色，role=character，presence 标记）；
2. 预算：同一玩家回合至多 1 条 presence 事件（门禁连续给出发声决定也只落 1 条）；
3. 插话优先：点名回合走插话，presence 不发生；
4. restricted 可见性回合与 replay 回合不评估在场；
5. fail-closed：门禁抛错/发声回合失败 → 玩家回合结果完整、时间线无 presence 事件、
   无未捕获异常。

### 4.3 PG 集成（在场事件落库与预算控制）

1. presence 回合经真实 PostgreSQL 运行时原子提交：events 表 `utterance.committed`
   + payload metadata 含 presence 标记，record_heads/worldlines 头指针推进连续，
   不产生玩家事件；
2. delivery projection 透出 presence 字段；观察者 membership 同权可见；
3. 预算控制：确定性门禁连发两次发声决定，落库 presence 事件恰 1 条；
4. observations 随在场事件落库（萃取管线可消费）。

### 4.4 GUI 真实模型（T3 组，不新增 mock）

1. T3-1：真实模型回合（不点名任何角色）提交后，无额外玩家输入，时间线出现带
   `data-presence` 的角色事件（event-character，committed，含 dialogue 段）；
2. 预算断言：单回合后 `data-presence` 事件数 ≤ 1；
3. 全程真实模型（门禁与发声均为真实模型调用），HOST_BIND=192.0.2.10。

## 五、实施顺序（commit 链）

1. 本规范（单独 commit）；
2. 触发机制：presence 上下文抽取 + 候选过滤 + 模型门禁 + presence react 分支
   （Core 测试随码）；
3. 事件落库与预算：schedulePresence 装配、executeTurn presence 分支、
   metadata/投影/前端透出、记忆与关系接线（应用层测试随码）；
4. 测试收口：PG 集成 + GUI T3 组 + STATUS/迭代日志。
