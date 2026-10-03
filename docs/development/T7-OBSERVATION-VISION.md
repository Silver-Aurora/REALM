# T7 · 观察者之眼看世界：世界自演与旁白推进

> 批次 T7（docs/development/EXPERIENCE-ITERATION.md 迭代主轴第七项，对应第二章缺陷 #3「观察者体验空白」与玩法设计风险 #3「自演叙事疲劳」）。
> 上位依赖：批次 S 观察者席位与姿态切换（docs/development/WORLD-ONBOARDING.md）、T3 角色在场（./T3-CHARACTER-PRESENCE.md）、T2 记忆管线（./T2-MEMORY-PIPELINE.md）、T5 骰点投影（./T5-DICE-RANDOMNESS.md）、T1 初夜异步任务（./T1-FIRST-NIGHT.md）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中

## 一、现状实锤（审计结论，勿重复调查）

1. **观察者能看，但世界不会动**。观察者席位（membership `role='observer'`、人类 narrator 席位、`omniscient_player_character=true` 不可变）与投影链路已完备（批次 S）：观察者打开记录走同一条 delivery projection SQL（`database/postgres/delivery-projection.ts` EVENTS_SQL `$5` 全见），SSE/GET 无分支。但**没有任何无人输入的推进器**——自治回合只有插话与在场两类，且都挂在玩家回合之后（`modules/application/local-record-service.ts:1424-1457`）。观察者不输入，世界永远静止：缺陷 #3 实锤。
2. **观察者可输入但语义错位**：观察者以 narrator 席位走 `player.utterance`，行动目录恒空（`listAuthorizedAffordances` 对空 characterInstanceId 返回 `[]`）。本批不改变观察者输入能力，只补「不输入也有内容可看」。
3. **可复用资产全部就位**：
   - 完整回合机械 `executeTurn`（计划→起草→校验→发布→事务提交→世界坐标→observations→outbox→SSE），插话/在场两个 payload 变体已示范「无玩家事件的自治回合」；
   - T3 版本冲突重读重试一次模式（`local-record-service.ts:1008-1032`）——回合外异步写者（场景晶化）推进 record_version 的竞态已有成熟收口；
   - T1 初夜「DB 状态行 + 单飞调度 + 懒重试 + 强制降级」异步任务模板（`modules/application/first-night.ts`、`database/postgres/first-night-store.ts`）；
   - T2 sync_turn 单飞/re-arm 调度器（`modules/memory/pipeline.ts`）；
   - 发言租约与冷却 `TurnControl`（`modules/runtime/turn-control.ts`）。
4. **前端观察者 UI 只有徽标**：header「✦ 执笔者」（`app/realm-client.tsx:761-765`），无任何自演控制与状态呈现。「世界持续运行中」是静态文案（`world-navigation.tsx:70`），不反映真实运行。

## 二、核心设计

### 2.1 最小纵向闭环

```
观察者打开记录（既有链路，omniscient 投影）
  → 记录页「世界自演」卡：状态 + 「开始演绎」入口（i18n 三语）
  → POST /api/record/self-play {action:"start"}（成员校验，幂等）
  → 会话账本落 running 行（beat_budget=3）→ 单飞调度器接管（fire-and-forget）
  → 每拍 = 一次完整 executeTurn 自治回合（payload.selfPlay）：
      DM 模型规划（激活 ≤2 角色，预算 1，Narrator 强制开启）
      → 角色 propose/react（可 use_skill，T4/T5 裁决与骰点物化）
      → 旁白推进 → 结构校验 + 模型复核 → 正式事件提交
  → 每拍落库即经 delivery outbox → SSE → 观察者时间线实时渐显（既有链路零改动）
  → 预算耗尽/停止/失败 → 会话落终态，信封投影自演状态，前端状态卡更新
```

「谁/什么推进世界」的答案：**与玩家回合同一套 DM + 角色 + 旁白编排**，只是没有玩家输入事件——不是第二套引擎，不是脚本播放器。

### 2.2 会话账本：`record_self_play_sessions`（新迁移 0020）

异步自演必须有可追踪状态与清理路径（任务书硬要求），T1 初夜已立「DB 状态行」先例。新建理由：自演会话与初夜状态行的生命周期不同（可重复开始/停止/完成多轮），复用初夜表会污染其 pending/ready 语义，故独立建表。0001–0019 不改动。

```sql
record_self_play_sessions (
  workspace_id, id, record_id,
  state text CHECK (state IN ('running','stopping','completed','failed','cancelled')),
  beat_budget integer, beats_completed integer DEFAULT 0,
  requested_by text,  -- principalId
  last_error text,    -- 失败码（fail-closed 证据）
  created_at, updated_at,
  PRIMARY KEY (workspace_id, id)
)
-- 部分唯一索引：同一记录至多一条活动会话
CREATE UNIQUE INDEX ... ON record_self_play_sessions (workspace_id, record_id)
  WHERE state IN ('running','stopping');
```

RLS 强制 workspace 隔离 + realm_runtime SELECT/INSERT/UPDATE（无 DELETE，审计行保留；测试清理由清理脚本负责）。stale 恢复：活动行 `updated_at` 超过 5 分钟无心跳即判死（懒恢复，见 2.4）。

### 2.3 自演拍：payload.selfPlay 自治回合

`PlayerUtterancePayload` 新增变体（与 interjection/presence 同型）：

```ts
selfPlay?: { beat: number; instruction: string };
```

- **planner**（`createLocalTurnDependencies`）：走真实 `orchestrator.plan`（模型 DM 规划，`playerText=instruction`），随后确定性规整：`narratorEnabled` 强制 true、`actionBudgetPerCharacter` 强制 1、激活集为空时兜底激活可用角色前 1 名——**每拍必有内容**（旁白 + 至多 2 名角色），空阵容世界由旁白单独推进（晶化批次已支持）。规整只收紧不放纵：DM 返回的激活集超界仍 DM_PLAN_INVALID。
- **instruction**：固定引导语，按世界界面语言取 i18n 模板（三语各一句，语义=「本轮没有玩家输入：让世界自行推进——角色继续他们的行动与对话，旁白描述环境变化」）。instruction 只作模型上下文，**不作为玩家事件落库**。
- **drafter / validator**：与玩家回合同路径（`payload.text=instruction`），**保留结构校验 + 模型复核**——自演回合改变世界状态（角色行动/资产扣减/效果），复核门槛不能降；复核三连否决（缺陷 #12 形态）→ 本拍失败 → 会话落 failed（fail-closed，不阻塞任何东西）。
- **releaseBuilder**：selfPlay 分支——**无玩家事件**；发布行动事务（含 T5 骰点物化）+ 旁白 + 角色事件，可见性恒 `{kind:"public"}`；`narration.committed` 与 `utterance.committed` 载荷附 `selfPlay: {beat}` 标记（payload 元数据，不是新事件类型，T3 presence 先例）。
- **透出**：`mapLocalFormalEvent` 把 `payload.selfPlay` 落入 `metadata.self_play`；delivery projection 解析（形状非法不透出，parsePresenceMarker 同型）→ ProjectionEvent/TimelineEvent 附 `selfPlay` 字段 → 前端时间线卡 `data-self-play` 锚点（GUI 断言用）。

### 2.4 调度器：单飞、预算、取消、重入幂等

`createSelfPlayScheduler`（modules/application/self-play.ts），形态对齐 T1/T2：

- **单飞**：进程内 `inFlight: Set<recordId>` + DB 部分唯一索引双闸；start 时已有活动会话 → 返回现有会话（幂等重入，200），不另起。
- **拍循环**：每拍前重读会话行——`stopping` → 落 `cancelled` 收束；`beats_completed >= beat_budget` → 落 `completed` 收束。取发言租约 → `loadConsistentSnapshot` → executeTurn（幂等键 `selfplay-${sessionId}-${beat}`）→ 成功 `completeBeat`（beats_completed+1 + 心跳 updated_at）→ 触发 `memorySync.schedule` 与场景晶化（与玩家回合后同型）。
- **版本冲突**：撞 RECORD_VERSION_CONFLICT（场景晶化/初夜/玩家回合并发推进）→ 换新幂等键重读快照重试一次（T3 模式原样复用），再撞 → 本拍失败，会话落 failed。
- **取消**：POST `{action:"stop"}` → `stopping`；在途拍跑完（不打断回合内模型调用），拍边界收束为 cancelled。无活动会话时 stop 幂等返回最近状态。
- **fail-closed**：模型/结构化输出/提交任何失败 → 会话落 failed + last_error 记失败码，只留日志；绝不阻塞页面、玩家回合与其他会话。
- **懒恢复**：信封投影读取会话时发现活动行心跳超 5 分钟（进程崩溃/重启）→ 落 `failed`（last_error=SELFPLAY_STALE）后返回终态；下次 start 可正常开新会话。
- **预算常量**：`SELF_PLAY_BEAT_BUDGET = 3`（每次 start 的拍数，依赖注入可降），硬上限 `SELF_PLAY_BEAT_HARD_CAP = 5`；预算语义=「每次显式触发最多演绎几拍」，重复 start（上一会话终态后）开新会话，不设全局速率限制。

### 2.5 API 与权限

`POST /api/record/self-play`，请求体 `{recordId, action: "start"|"stop"}`：

- 会话鉴权 + principal 解析（既有 `resolveRequestPrincipal`）；记录所属世界的 membership 必须存在（player 与 observer 均可触发——自演对玩家同样有意义，观察者是第一公民场景）；无 membership → 404 形态（LOCAL_RUNTIME_NOT_INITIALIZED 沿用）。
- 返回当前会话状态 `{state, beatBudget, beatsCompleted, lastError}`；start 幂等（活动会话在→返回现状），stop 幂等。
- action 非法 → 400；fail-closed，不暴露内部细节。

### 2.6 投影与前端

- 信封投影增 `selfPlay: {state, beatBudget, beatsCompleted, lastError} | null`（无会话行 → null，完全向后兼容；懒恢复在此触发）。
- 前端记录页增「世界自演」卡（直角纸墨、无语义底色，沿用既有卡片规则）：状态行（演绎中 n/budget · 已完成 · 已停止 · 已失败）+ 主按钮（空闲/终态=「开始演绎」，running=「停止」）+ `data-self-play-panel` 锚点。
- running 时前端 2.5s 轮询信封刷新状态（T1 firstNight 轮询先例）；事件内容本身走既有 SSE 实时渐显，不自造推送。
- 观察者界面语言、世界语言、i18n key 三语齐备；观察者视角与其他成员视角同卡同态（状态是记录级的）。

### 2.7 可见性边界（绝不泄露）

- 自演拍可见性恒 public，无玩家秘密输入参与，不产生 restricted 内容；
- 投影链路零改动：观察者 omniscient 全见是既有成员标志语义；非 omniscient 视角的 scene/restricted/private 过滤规则一行不动；
- 自演不向任何人透出控制面事件：无骰点的 `action.transaction.committed` 仍被 EVENTS_SQL 排除，带物化骰点的判定事件沿用 T5 放行形态。

## 三、失败矩阵（fail-closed）

| # | 失败点 | 行为 |
|---|---|---|
| F1 | start 时已有活动会话 | 幂等返回现状，不重复起拍 |
| F2 | DM 规划模型失败/输出非法 | 本拍失败 → 会话 failed + last_error，日志收场 |
| F3 | 角色 propose/react 或旁白模型失败 | 同 F2 |
| F4 | 模型复核三连否决（缺陷 #12 形态） | 同 F2（DM_OUTPUT_REJECTED 入 last_error） |
| F5 | 提交撞 RECORD_VERSION_CONFLICT | 重读快照重试一次；再撞 → 会话 failed |
| F6 | stop 时在途拍执行中 | 在途拍跑完，拍边界落 cancelled |
| F7 | 进程崩溃/重启，活动行残留 | 信封懒恢复：心跳超 5 分钟落 failed（SELFPLAY_STALE） |
| F8 | 无 membership/记录不存在 | 404 形态，不建会话 |
| F9 | 调度器任意未捕获异常 | 外层 try/catch 日志收场，会话尽力落 failed |
| F10 | 会话行写库失败 | start 返回 500 安全文案，不起拍 |

## 四、验收标准

### 4.1 Core（纯函数与契约）

1. 会话状态机：start/running→(completed|cancelled|failed) 迁移合法集、终态不可逆、stopping 只从 running 进入；
2. 预算钳制：注入预算夹取 [0, SELF_PLAY_BEAT_HARD_CAP]；
3. 自演计划规整：narratorEnabled 强制 true、预算强制 1、空激活兜底前 1 名、超界激活仍 DM_PLAN_INVALID；
4. selfPlay 标记规整：投影解析形状非法不透出（parseSelfPlayMarker）。

### 4.2 应用层（内存仓储）

1. start→拍循环→completed：事件落时间线（旁白+角色，无玩家事件，metadata.self_play 携带拍号），record 头指针连续；
2. stop 于拍间：会话落 cancelled，已完成拍事件保留；
3. start 幂等重入：活动会话在→同一会话，不重复起拍；
4. 版本冲突重读重试一次（确定性注入冲突），再撞 fail-closed；
5. 拍模型失败 → failed + last_error，主路径（玩家回合 submitMessage）不受影响。

### 4.3 PG 集成

1. 迁移 0020 应用幂等；RLS workspace 隔离；realm_runtime 授权真实存在（REALM_RUNTIME_DATABASE_URL 重定向验证）；部分唯一索引拒绝并发双活动会话；
2. 真实 PG 全程：start→≥1 拍落库→completed，events/observations/头指针/世界线头连续，metadata.self_play 落库，delivery 投影透出 selfPlay 字段；
3. stale 恢复：手工老化心跳 → 投影读取落 failed；
4. 观察者 membership（role='observer'）投影同权可见自演事件；restricted 既有事件过滤不受自演影响（对照断言）。

### 4.4 GUI 真实模型（t7-observation spec，不新增 mock）

1. T7-1：创世/建仓世界+记录 → 姿态切观察者（UI 或 API）→ 打开记录 → 点「开始演绎」→ 无键盘输入，时间线实时渐显带 `data-self-play` 的旁白/角色事件（≥1 拍），状态卡从演绎中走到终态，拍数 ≤ 预算；
2. 全程真实模型；chromium 主测，改动跨引擎面（时间线渲染）加跑 webkit；
3. HOST_BIND=192.0.2.10，跑后执行清理脚本回基线。

### 4.5 回归圈（总纲第五章）

t7 新 spec + 直接关联 b-record / g-realtime / c-actions / d-visibility / o-action-suggestions + 冒烟 a-library；npm test 全链必跑。

## 五、交付步骤与 commit 纪律

1. 本规范（单独 commit，含 docs/README.md 索引登记与 T4–T6 索引补登）；
2. 迁移 0020 + 会话仓储 + schema 契约断言（独立 commit）；
3. 自演回合管线：payload 变体 + planner/drafter/validator/releaseBuilder 分支 + metadata/投影透出 + 调度器 + 服务/API 接线（Core/应用测试随码，独立 commit）；
4. 前端自演卡 + i18n + 时间线锚点（独立 commit）；
5. 测试收口：PG 集成 + GUI t7 spec + 范围化回归 + 清理回基线 + STATUS/迭代日志（独立 commit）。

禁 `git add -A`；每步独立 commit；不 push。
