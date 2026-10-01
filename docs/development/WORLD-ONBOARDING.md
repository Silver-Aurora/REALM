# 世界记忆 · LLM 创世引导 · 角色与观察者 — 新玩家体验实施规范（批次 S）

> 文档性质：实施规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 视觉约束一律以 [`../design/UI-DESIGN.md`](../design/UI-DESIGN.md)（纸墨纪事）为准。
> 文本三语约束遵循 [`I18N-CENTRALIZED.md`](./I18N-CENTRALIZED.md)：界面固定文案走 `modules/i18n` 注册表，模型动态文本跟随玩家语言。

## 1. 现状调研（真实链路与数据模型）

### 1.1 默认入口链路（痛点一的现场）

1. 首页 `app/page.tsx` 经 `requirePageSession` 门禁后渲染 `app/realm-client.tsx`。
2. `RealmClient` 挂载即调用 `loadRecord()`（不带 recordId）→ `GET /api/record`。
3. `app/api/record/route.ts` → `LocalRecordService.loadRecord(recordId?)`；recordId 缺省值硬编码为
   `LOCAL_RECORD_SCOPE.recordId`（`modules/application/local-record-service.ts`），即
   `POSTGRES_DEMO_IDS.record = record_first_watch`（演示世界「烬海诸国」）。
4. 结论：**任何账号、任何时候打开首页都进入演示记录**。没有「上次打开的世界」记忆，
   也没有「无世界 → 引导创建」分支——无世界时 `loadRecord` 只会报错屏（断/重试）。

### 1.2 身份与账号模型

- `accounts`（迁移 0012/0015）：`(workspace_id, principal_id)` 主键，`display_name` 即身份，
  无密码。批次 Q 为它增加了用户级 `ui_language` 列，并只授予 `realm_runtime`
  `UPDATE (ui_language)` 的最小权限——**用户级设置的既有模式**。
- 会话为无状态 HMAC cookie（`modules/identity/auth.ts`），cookie 只携带 principalId。
- 登录（`app/api/auth/login/route.ts`）幂等写入演示世界 membership
  （`ensureDefaultWorldMembership`）：首次加入角色为 **player**；既有 membership 按 `ON CONFLICT DO NOTHING` 保持原 role。
  2026-09-28 起登录不再自动赋予 owner；创建者通过自己的 world-create transaction 获 owner。

### 1.3 世界创建链路

三条入口（世界库面板）最终都汇聚到同一落库函数：

| 入口 | 路径 | 落库 |
|---|---|---|
| 司卷问答（N 批次） | 八步固定问答 → `WorldGenesisDraft` | `POST /api/world/generate` draft 分支 |
| 启笔铸界（L 批次） | 自由文本 → `POST /api/world/generate` prompt 分支生成手稿 → 微调确认 | 同上 |
| 工笔细琢 | 手动表单 `POST /api/library` kind=world/story/record/character/branch | `LibraryService.create` |

`LibraryService.createGenesis`（`modules/application/library-service.ts`）单事务原子创建：
world + 原初 worldline + owner membership（**恒为 role='owner'、omniscient=true**）
+ companions（≤2，`character_definitions` native）+ story + record + record_head
+ `assembleDefaultRecord` 装配阵容（玩家人类角色 + 自定义 AI 角色）+ 开幕场景 + public 可见性策略。

### 1.4 司卷问答现状（痛点二的现场）

`app/components/guided-genesis.tsx` 是**固定八步表单**（世界名/纪元/文风/底色/故事/身份/同行者/场景/总览）。
「AI 代笔」（`POST /api/world/suggest`，`modules/application/genesis-suggestions.ts`）只为单步生成
2–3 条候选，fail-closed。**没有对话、没有追问、不能接住自由输入**——本质仍是填空模板。

### 1.5 角色与观察者现状（痛点三的现场）

数据模型（迁移 0001）：

- `player_world_memberships`：`role CHECK IN ('owner','player','observer')`，
  `omniscient_player_character`（创建后不可变，触发器守护）、`can_view_dynamic_knowledge`。
  **`observer` 枚举位存在但从未被任何代码写入**。
- `participants`：`participant_kind IN ('character','narrator')`；narrator 时
  `character_instance_id` 必须为 NULL（shape check）；`controller_mode IN ('human','ai','hybrid')`；
  `is_active` 布尔；FK 指向 membership（principal 席位）与 character_instances。
  **人类 narrator 参与者（执笔者席位）在模型上是合法的，但从未被创建**。
- 装配（`assembleDefaultRecord`）**永远**给玩家建一个 `controller='human'` 的角色席位；
  没有「不扮演角色」的形态。
- 运行时席位解析（`database/postgres/record-scope.ts`）只查 `participant_kind='character'`，
  且 `playerActor` 缺失即返回 null（记录不可玩）；**不按 is_active 过滤**。
- 回合管线：`listAuthorizedAffordances` 把 `playerActor.characterInstanceId` 传给
  `action-state.ts`，后者 `validateScope` 要求所有字段非空——观察者（无角色实例）会直接抛错。
- 角色创建：`kind=character` 只插 `character_definitions`；**已打开的记录阵容不会变化**
  （阵容只在创建记录时装配）。入口埋在「工笔细琢」抽屉的五个 tab 之一。
- 交付投影（`delivery-projection.ts`）：cast 只含 character 参与者；meta 不带 membership.role；
  视角由 membership.omniscient 决定（现存世界恒为 true）。

### 1.6 多世界既有数据

库中权威世界：`world_ember_coast`（烬海诸国，演示种子）、`world_d89eae19bf8a495799`（界核，用户真实世界）。
清理脚本 `scripts/clean-gui-test-data.sql` 只保留这两个世界与 Lyle 账号。本批次不得改变该约束。

## 2. 三个痛点的根因

| 痛点 | 根因 |
|---|---|
| ① 每次打开都是 demo 世界 | `GET /api/record` 缺省 recordId 硬编码为演示记录；accounts 上没有任何「最近打开」状态；前端无 onboarding 分支 |
| ② 司卷问答太弱 | 八步固定表单 + 单步候选补全；LLM 只参与「代笔候选」，不参与提问/接话/提案，无对话上下文 |
| ③ 无法添加角色、无法当观察者 | 角色定义与记录装配脱节（加角色不进当前阵容）；membership 的 observer 枚举与 narrator 参与者形态从未接入装配/运行时/UI；affordance 校验不容忍无角色实例的席位 |

## 3. 设计方案

### 3.1 需求一：世界记忆与默认入口

#### 3.1.1 数据模型（迁移 0017 `account_last_opened.sql`）

沿用批次 Q 的 accounts 列模式（用户级设置、最小列级授权）：

```sql
ALTER TABLE accounts ADD COLUMN IF NOT EXISTS last_world_id text;
ALTER TABLE accounts ADD COLUMN IF NOT EXISTS last_record_id text;
-- 记录被删自动置空，避免悬空引用
ALTER TABLE accounts ADD CONSTRAINT accounts_last_record_fk
  FOREIGN KEY (workspace_id, last_record_id)
  REFERENCES records (workspace_id, id) ON DELETE SET NULL;
GRANT UPDATE (last_world_id, last_record_id) ON accounts TO realm_runtime;
```

同迁移内做一次性**回填**（老用户升级首次打开合理化）：为每个账号挑选其
membership 所辖**非演示世界**中最近创建的记录写入；仅演示世界成员资格的账号保持 NULL
（首次打开进引导，演示世界仍可从引导屏/世界库进入）：

```sql
UPDATE accounts a SET last_world_id = p.world_id, last_record_id = p.record_id
FROM (
  SELECT DISTINCT ON (m.workspace_id, m.principal_id)
         m.workspace_id, m.principal_id, r.world_id, r.id AS record_id
  FROM player_world_memberships m
  JOIN records r ON r.workspace_id = m.workspace_id AND r.world_id = m.world_id
  WHERE m.world_id <> 'world_ember_coast'
  ORDER BY m.workspace_id, m.principal_id, r.created_at DESC, r.id DESC
) p
WHERE a.workspace_id = p.workspace_id AND a.principal_id = p.principal_id
  AND a.last_record_id IS NULL;
```

#### 3.1.2 接口

- `LocalRecordService` 新增 `openDefaultRecord(principalId)`：
  1. 读 `accounts.last_record_id`（account-repository 新增 `findLastOpened` / `saveLastOpened`）；
  2. 有值 → 按该 recordId 加载（加载失败/记录已不存在 → 视为无记忆，走 3）；
  3. 无记忆 → 返回 onboarding 信号（不抛错）。
- `GET /api/record`（不带 recordId）：走 `openDefaultRecord`；onboarding 时返回
  `200 { ok: true, onboarding: true }`（无 record 字段）。
- **记忆写入**：`GET /api/record` 每次成功加载（含显式 recordId 打开）后，把
  `(worldId, recordId)` 写回 accounts（fail-closed：写失败只 warn，不影响读取）。
- 深链支持：前端首屏读取 `?recordId=` 查询参数直接打开（供引导回跳、测试与分享）。

#### 3.1.3 UI：引导屏（无记录时的首页）

新增 `app/components/world-onboarding.tsx`（全屏 state-screen 同族，纸墨风格）：

- 主入口：**「司卷对谈」LLM 引导创建**（见 3.2）+ 次入口「司卷问答·逐步填写」（旧八步，保留）。
- 若账号已有可进入的世界（library 快照非空）：列出「继续上次 / 进入已有世界」
  （每个世界显示名字与最近记录，点击打开该世界最近记录）；演示世界一并列出，不特殊化。
- 已有世界数据不受影响：本屏只是入口，不改动任何既有世界。

`RealmClient` 增加 `onboarding` 状态分支：`GET /api/record` 返回 onboarding 时渲染该屏；
创世完成或点选已有世界后 `openRecord(recordId)`。

### 3.2 需求二：LLM 驱动的创建引导（司卷对谈）

#### 3.2.1 定位

新增**司卷对谈**（`GuidedGenesisChat`）作为推荐主入口：司卷（LLM）与玩家自由对话，
动态追问 → 生成世界提案（含角色与开场）→ 玩家可改可聊 → 落笔入界。
旧八步问答与启笔铸界**全部保留**为可切换路径（可跳过/可修改的纪律）。

#### 3.2.2 接口契约 `POST /api/world/genesis-chat`

无状态（与既有 API 一致，不落会话表）：客户端携带截断后的对话历史。

请求：

```jsonc
{
  "message": "玩家本轮输入（首轮可为空，表示请求司卷开场）",
  "transcript": [ { "role": "user" | "scribe", "content": "..." } ],  // 服务端再截断至最近 16 条、单条 ≤500 字
  "draft": { /* 已定提案快照，WorldGenesisDraft 同构（可缺省字段） */ }
}
```

响应（成功）：

```jsonc
{
  "ok": true,
  "reply": "司卷的回应文本（追问/点评/说明）",
  "draftPatch": { /* WorldGenesisDraft 局部字段；null 表示本轮不定稿 */ },
  "phase": "exploring" | "proposing" | "ready",
  "opening": "开场白（仅 phase=ready 可提供；≤300 字）"
}
```

- `draftPatch` 合并语义：只覆盖出现的字段；经 `normalizeGenesisDraft` 同源的
  限长/结构规整（世界名 ≤40、同伴 ≤2 等）。消毒后无可识别字段、或合并结果
  整体非法（如探索期尚未谈出世界名）→ 丢弃本轮 patch（按 null 处理）、
  对谈照常继续；响应本体（phase/reply）非法才整体 fail-closed。
- fail-closed 矩阵与 AI 代笔一致：模型错误/超时/非法 JSON/schema 不符 →
  `ok:false`，前端不弹报错，提示「司卷暂时沉默」并可一键转旧表单/重试；不阻塞创建。
- 模型通道：`getModelSettingsService().gateway()`，`responseFormat: "json_object"`；
  prompt 只含对话历史与已定草稿，绝不含密钥。
- 文风与语言：复用 `modules/style` 文风系统——司卷提示玩家选择/推断文风（style 进 draft）；
  回应语言跟随玩家输入语言（中/英/日），与既有 suggest/genesis 约束一致。
- 司卷人格与节奏（system prompt 要点）：一次只问一个焦点；接住自由输入并顺势追问；
  2–5 轮内收束；玩家明确要求时立即给出完整提案；同伴角色禁用「塞娜/弥洛/洛川」；
  输出单一 JSON，无多余文字。

#### 3.2.3 提案与落库

- `phase=proposing/ready` 时前端渲染**世界提案卡**：全部字段可就地编辑
  （世界名/纪元/文风/底色/故事/你的定位/同行者/场景），并显示开场白预览。
- 玩家可继续聊天修改（「把名字换成…」→ 司卷回 draftPatch），也可直接编辑后确认。
- `WorldGenesisDraft` 扩展两个可选字段（向后兼容，旧路径不受影响）：
  - `playerStance: "player" | "observer"`（缺省 player，见 3.3）；
  - `opening: string`（开场白，≤300 字）。
- 「落笔入界」仍走既有 `POST /api/world/generate` draft 分支（单事务原子）。
  `opening` 非空时，`createGenesis` 在同事务内插入开场事件
  （`narration.committed`，speaker 旁白，presentation 单 story 段，public 策略，
  world 坐标 (0,1)，record_heads 置 version=1/next_ordinal=2）——新记录开局即有卷首。

#### 3.2.4 Core 契约（tests/genesis-chat.test.ts）

- draftPatch 合并/限长/非法结构 → null（fail-closed）；
- transcript 截断（≤16 条、单条 ≤500 字）；
- 响应规整：phase 枚举、opening 限长、非法 phase 拒绝；
- 纯函数可测，模型调用在路由层注入。

### 3.3 需求三：角色管理与观察者

#### 3.3.1 添加角色并进当前阵容

- `LibraryCreateCommand` character 分支扩展可选 `attachRecordId`：
  创建 `character_definitions` 后，在**同一事务**内把该角色装配进目标记录
  （continuity（key `custom_<definitionId>`，复用既有装配约定）→ character_instance（ai）
  → participant（kind='character'、controller='ai'、speaking_order 取当前最大值+1））。
  记录已在阵容中（按 continuity_key 查重）→ 幂等跳过。
- 前端：世界库每个世界卡片上新增显眼的「添角色」快捷表单（名字/定位/一句话侧写）；
  若当前打开的记录属于该世界，自动携带 `attachRecordId`，创建成功后重载当前记录，
  新角色立即出现在右侧阵容。工笔细琢抽屉的旧角色 tab 保留。
- 不新建表、不改 character_definitions 结构；native 与酒馆导入（source_format）角色
  走同一装配路径。

#### 3.3.2 观察者（不扮演任何角色）

模型依据：membership.role='observer' 与人类 narrator 参与者均为既有契约内的合法形态，
本批次把它们接通，不新增冲突字段。

- **创世选择**：司卷对谈提案卡、旧八步总览、启笔铸界手稿均新增「你的姿态」选择：
  入局（默认）/ 观察者（执笔者，不扮演角色）。写入 draft.playerStance。
- **落库（createGenesis）**：observer 时 membership role='observer'（omniscient 仍 true），
  不创建玩家角色定义/实例/参与者，改为创建**人类 narrator 参与者**
  （participant_kind='narrator'、character_instance_id NULL、controller_mode='human'、
  principal_id=创建者、speaking_order=0）。
- **运行时接通**：
  - `record-scope.ts`：席位解析先找本人的人类 narrator 席位（displayName 取账号昵称），
    再走原角色席位逻辑；actors 查询过滤 `is_active = true`；
    playerActor 允许无 characterInstanceId（以空串占位，类型不变）。
  - `listAuthorizedAffordances`：playerActor 无角色实例时直接返回空 affordances
    （观察者没有技能/道具/姿态按钮），不再触碰 action-state 的非空校验。
  - 交付投影：meta 增选 `membership.role`，viewer 增补 `membershipRole`；
    cast 仍只列 character 参与者（观察者在阵容外）。
- **既有世界切换姿态**（世界库世界卡「我的姿态」开关）：
  - 转观察者：UPDATE membership role='observer'（触发器只禁 omniscient 变更，role 可变）；
    该世界活跃主记录中本人的人类 character 参与者置 `is_active=false`
    （不删除——events 有 FK 引用），补建人类 narrator 席位（已有则激活）。
  - 转入局：role='player'；narrator 席位置 is_active=false；无活跃人类角色席位时
    按 `resolvePlayerPersona` 规则补建（char_def_player_<world> 约定）。
  - 通过 `POST /api/library` 新命令 `kind: "player-stance"`（worldId + stance）落库。
- **UI 语义**：
  - 记录页 header：观察者显示「✦ 执笔者」徽标（新 i18n 键），区别于上帝/角色视角。
  - 阵容卡：本人席位标注「你」；观察者世界阵容只列 AI 角色。
  - 世界库角色列表：标注来源（native/酒馆导入，source_format 透出），
    并显示该角色是否在当前记录阵容中——回答「谁在世界里、我扮演谁」。

#### 3.3.3 约束与兼容

- 不改 `player_world_memberships.omniscient_player_character` 的不可变约束；
- 不删除任何参与者行（FK events.actor_participant_id）；
- 烬海诸国/界核既有数据零变更（不回填 stance、不改 membership）。

### 3.4 i18n

新增界面文案全部进 `modules/i18n` 注册表（zh-CN/en/ja 三语键集一致，契约测试强制）：
引导屏、司卷对谈、提案卡、观察者徽标、添角色表单、姿态开关、角色来源标注等。
模型动态文本（司卷话语/提案内容/开场白）不进资源文件，语言跟随玩家。

## 4. 验收标准（Playwright 可断言）

### 4.1 既有回归

- `npm test` 全绿（typecheck/Core/契约/应用/前端/PG/渲染/构建）。
- GUI 既有 A–R 组全部通过（helpers `openDemoRecord` 改为深链 `?recordId=` 后语义不变）。
- schema 契约登记迁移 0017；accounts 新列不破坏既有 accounts 测试。

### 4.2 新增 S 组（tests/gui/s-onboarding.spec.ts，真实模型）

完整游玩闭环（一个用例内走完）：

1. **新用户无记忆 → 引导屏**：全新测试账号打开 `/`，断言不出现演示记录时间线，
   出现引导屏（司卷对谈入口可见）。
2. **LLM 对谈创建**：真实调用模型，≥2 轮自由对话（含接住自由输入的追问），
   得到世界提案卡（字段非空且与对话意向一致），可编辑；
3. **观察者落界**：提案中选择观察者姿态 → 落笔入界 → 进入新记录：
   面包屑为世界名，header 显示执笔者徽标，阵容不含「你」的角色席位；
   开场事件（卷首旁白）在时间线可见。
4. **添加角色**：世界库内为该世界添一名角色 → 当前记录阵容立即可见。
5. **进世界游玩**：以执笔者身份提交一条真实回合，AI 角色响应出现在时间线（TURN_TIMEOUT）。
6. **关闭重开回到上次世界**：重新 `goto("/")`（无参数）→ 自动回到该世界该记录（非演示世界）。
7. **姿态切换**：世界库切换「入局」→ 当前记录阵容出现「你」的席位；切回观察者 → 席位退出。

辅助用例（确定性路由拦截，不耗真实模型）：

8. genesis-chat fail-closed（模型端点被拦截为 500/坏 JSON）→ 前端静默降级，
   可转旧表单完成创建；
9. draftPatch 合并/截断边界（单测已在 Core 层覆盖，GUI 只验降级路径）。

### 4.3 清理与纪律

- GUI 全量后执行 `scripts/clean-gui-test-data.sql`；S 组产生的新账号/新世界均落入
  脚本既有删除规则（非保留世界/非 Lyle 账号一律删除），无需新规则。
- 报告附清理后计数：worlds / accounts / world_files / world_articles。

## 5. 交付物清单

| 交付物 | 说明 |
|---|---|
| 迁移 0017 | accounts.last_world_id/last_record_id + 回填 + 列级授权 |
| `database/postgres/account-repository.ts` | findLastOpened / saveLastOpened |
| `modules/application/local-record-service.ts` | openDefaultRecord；观察者席位与 affordance 兼容 |
| `app/api/record/route.ts` | 默认入口解析 + 记忆写入 + onboarding 应答 |
| `modules/application/genesis-chat.ts` + `app/api/world/genesis-chat/route.ts` | 司卷对谈契约与路由 |
| `modules/application/library-service.ts` | character attachRecordId 装配；player-stance 命令；opening 落库 |
| `app/components/world-onboarding.tsx` / `guided-genesis-chat.tsx` | 引导屏与对谈 UI |
| `app/components/library-panel.tsx` / `scene-inspector.tsx` / `realm-client.tsx` | 添角色、姿态开关、阵容语义、执笔者徽标 |
| tests | Core：genesis-chat 契约；PG：library 装配/stance；GUI：S 组 |
