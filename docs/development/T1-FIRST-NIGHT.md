# T1 · 世界初夜：创世落笔后的第一轮可玩密度

> 批次 T1（docs/development/EXPERIENCE-ITERATION.md 迭代主轴第一项）
> 立项：2026-08-20 +08:00 · 状态：规范定稿，实施中
> 一句话目标：落笔入界的那一刻，世界就要有密度、可玩、想让人行动——场景即时生成、角色在场、钩子事件开场。

## 一、现状调研（读代码确认）

### 1.1 落笔入界到底初始化了什么

两条创世入口（逐步引导 guided-genesis、司卷对谈 guided-genesis-chat）最终都走
`POST /api/world/generate { draft }` → `library-service.createGenesis`，单事务写入：

| 数据 | 内容 | 叙事含量 |
| --- | --- | --- |
| worlds | 名称/摘要/settings{era,style,weather,tension} | 字段级 |
| worldlines | 「原初世界线」head=(0,0) | 无 |
| player_world_memberships | owner 或 observer 席位 | 无 |
| character_definitions | 同行者至多 2 名（名字/身份/侧写） | 仅档案 |
| stories / records / record_heads | 标题与前提 | 无 |
| assembleDefaultRecord | 玩家角色（或观察者 narrator 席位）+ 同行者实例与参与者 + scenes 行（title=「开幕」，仅 location/objective 字段）+ public 可见性策略 | 无 |
| insertOpeningEvent | **仅当 draft.opening 非空**时插入卷首旁白（世界坐标 (0,1)） | 唯一叙事 |

关键事实：**只有司卷对谈路径会在 phase=ready 产出 opening**；逐步引导
（guided-genesis.tsx）固定传 `opening: ""`。也就是说逐步引导创世的世界，
落库后时间线事件数为 0。

### 1.2 首屏玩家看到什么（GET /api/record?recordId=）

- **时间线**：0 条（逐步引导）或 1 条卷首旁白（对谈）；`.timeline-empty`
  「新的一页，还没有内容」是常见首屏。
- **场景卡（SceneInspector）**：scene.location / worlds.settings 的 weather/tension /
  scene.objective——全部来自草稿字段，留白步骤则为空；没有「画面」。
- **阵容卡**：玩家 + 同行者名单（名字/身份）——在场名单，不在场发声。
- **行动卡（MessageComposer affordances）**：能力目录 SQL 只产出
  skill/asset/stance/scene 四类；新世界没有任何技能/资产/姿态台账，只剩
  scene 类「观察四周」，文案由 scene_location/weather 套模板，留白时落到
  `action.observe.suggested.blank` 泛化兜底——即「泛化行动卡」。
- **下一步提案（suggestions）**：仅在 submitMessage 回合响应里携带
  （旁白结构化输出），loadRecord 永远不带——首屏为 0。

### 1.3 行动卡场景快照怎么算

`database/postgres/action-state.ts` CAPABILITIES_SQL：以参与者所在记录的最新
active scene（start≤世界头≤end）取 scene_location/scene_weather，连同
worlds.settings.style/language 交给 `composeObserveSurroundingsCopy` 生成文案。
新世界 scenes 行只有草稿 location/objective，weather 在 worlds.settings，
无钩子、无事件可引用——快照天然浅，提案退化成「观察四周」式空话。

## 二、根因

1. **创世只写结构，不写叙事**：createGenesis 的产出是骨架（表行/席位/空场景行），
   唯一叙事（卷首旁白）还是可选的。场景只有字段，没有「定格描述」。
2. **角色只进名单**：同行者被装配为参与者后没有任何首轮发声机制——角色声音
   只能等玩家第一回合触发 DM Controller 激活。
3. **钩子机制不存在**：数据模型没有钩子位置，UI 没有钩子呈现，提案机制是
   回合驱动的（玩家先动，世界才给提案），开场没有「下一步想做什么」。
4. **行动提案素材浅**：新世界的行动卡只能吃 scene_location/weather，没有事件/
   钩子可引用，于是退化为泛化卡。

## 三、设计方案

### 3.1 总体形态：事务内保底 + 事务外增密

- **落笔事务内（同步、确定性、零模型调用）**：永远插入开场旁白事件
  （draft.opening 或草稿场景字段合成的风格化定格句）+ 写入初夜状态行
  （state=pending）。保证落笔响应 <1s、首屏永不空白。
- **落笔事务后（异步、真实模型、fail-closed）**：一次性模型调用生成
  「初夜包」（场景定格 / 角色发声 / 钩子+提案），以标准事件形态追加进同一
  记录，推进 record_heads 与世界线头，状态行转 ready。前端轻量轮询，内容渐显。
- **懒重试自愈**：记录打开时若状态仍 pending（进程重启等）且尝试次数未超限，
  自动补跑；超限则落确定性降级包（state=degraded），绝不留白。

### 3.2 数据模型

新迁移 `0019_record_first_nights.sql`：

```sql
CREATE TABLE IF NOT EXISTS record_first_nights (
  workspace_id text NOT NULL,
  record_id text NOT NULL,
  world_id text NOT NULL,
  state text NOT NULL CHECK (state IN ('pending','ready','degraded')),
  attempts integer NOT NULL DEFAULT 0,
  hook_content text NOT NULL DEFAULT '',
  suggestions text[] NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (workspace_id, record_id),
  CONSTRAINT record_first_nights_record_fk
    FOREIGN KEY (workspace_id, record_id) REFERENCES records (workspace_id, id)
    ON DELETE CASCADE
);
-- RLS：realm_workspace_isolation（与 accounts 同型）
-- GRANT：owner 角色全量；realm_runtime SELECT/INSERT/UPDATE（懒重试与投影读取）
```

- 事件不加表不加列：初夜事件全部走既有 events 形态（narration.committed /
  utterance.committed + presentation payload + public 策略），与演示种子
  OPENING_EVENT_SQL 同型；record_heads/worldlines 头按既有规则推进。
- 提案与钩子正文镜像存于状态行（投影读取快路径），事件 payload 内同时携带
  `hook: true` 元数据标识。
- 旧世界无状态行 → 投影 firstNight=null，前端行为完全不变（向后兼容）。

### 3.3 生成管线（modules/application/first-night.ts）

- `generateFirstNightPack(gateway, input)`：单次 json_object 调用。System 注入
  世界提案（名/纪元/底色/文风 describeWorldStyle/故事/玩家定位/同行者/初始场景/
  已有卷首旁白）+ PLAYER_LANGUAGE_RULE 同型语言规则；输出：

```json
{
  "scene": { "environment": "…", "story": "…", "fact": "…" },
  "characters": [ { "name": "…", "utterance": "…", "action": "…" } ],
  "hook": { "content": "…", "suggestions": ["…", "…"] }
}
```

- `normalizeFirstNightPack(raw, draft)`：严格规整——characters 只保留名字与提案
  同行者精确匹配者；scene 三段至少一段非空；hook.content 非空；suggestions
  过滤空值取 2–3 条；长度截断。整体无效返回 null → 降级包。
- `fallbackFirstNightPack(draft)`：确定性降级——scene 由草稿场景字段经风格模板
  （world-style 新增 `first-night.*` keys，四风格×三语）合成；characters 空
  （模型才能给角色声音）；hook 由 tension/objective 合成（无则空）；suggestions
  由 objective/location 模板合成。保证「至少场景+旁白可进场」。

### 3.4 落库（library-service）

- `createGenesis` 变更：开场旁白从「opening 非空才插」改为**永远插入**
  （opening 优先，否则确定性合成句）；同事务插入状态行（pending, attempts=0）。
- 新增 `commitFirstNight(scope, recordId, pack)`：单事务——按名字解析同行者
  participant；依次追加场景旁白（environment/story/fact 三段 presentation）、
  同行者 utterance.committed（action+dialogue 段）、钩子 narration.committed
  （payload.metadata.hook=true）；推进 record_heads / worldlines 头；状态行转
  ready（degraded 时同路径）。幂等：状态非 pending 直接跳过。

### 3.5 触发时机与调度

- `POST /api/world/generate {draft}`：事务提交并返回 ids 后 fire-and-forget
  `scheduleFirstNight(recordId)`（与 scheduleInterjection/设定结晶同型的
  void 后台任务，失败只留日志）。
- 单飞守卫：模块级 in-flight Set 按 recordId 去重；attempts 自增，≥3 仍失败
  强制降级包收尾。
- 懒重试：loadRecord 成功后检查状态行，pending 且未在途 → 补跑一次。

### 3.6 前端与 i18n

- Envelope 扩展：`firstNight: { state, hookContent } | null`、
  `openingSuggestions: string[]`（loadRecord 路径携带，与回合 suggestions
  互补：composer 优先用回合 suggestions，空则用 openingSuggestions）。
- realm-client：state=pending 期间显示 `.first-night-status` 状态条
  （i18n 三语），每 2.5s 轮询 reload，最长 90s；ready/degraded 消失并呈现
  新事件与提案。首屏本身已有确定性旁白，绝不空白转圈。
- SceneInspector：state 非空且 hookContent 非空时渲染「钩子」卡。
- i18n：`ui.firstNight.pending/ready/degraded`、`ui.inspector.hook` 三语；
  world-style 模板新增 `first-night.*`（四风格×三语）。

### 3.7 fail-closed 矩阵

| 故障 | 行为 |
| --- | --- |
| 模型未配置/调用失败/超时 | 落确定性降级包：场景旁白必有；钩子视草稿 tension/objective 而定；state=degraded |
| 模型返回非法 JSON/字段不合格 | normalize 返回 null → 同上降级 |
| 异步任务进程中断 | 下次打开记录懒重试（attempts<3），超限强制降级 |
| 状态行缺失（旧世界） | firstNight=null，前端零变化 |
| 轮询超时（90s） | 状态条消失，玩家仍可正常游玩；内容稍后到达时下次刷新生效 |

## 四、验收标准（Playwright 可断言）

tests/gui/t1-first-night.spec.ts（真实模型回合，不 mock）：

1. **T1-1 场景即时生成**：司卷对谈 ≥2 轮 → 提案卡 → 落笔入界；落笔响应快速
   进入记录页；时间线首屏非空（开场旁白可见）；场景卡地点非空且与提案一致；
   轮询内出现场景定格旁白事件（environment/story/fact 呈现）。
2. **T1-2 角色在场**：提案含同行者时，初夜完成后时间线出现以同行者名为
   speaker 的角色事件（event-character）；阵容卡列出同行者。
3. **T1-3 钩子事件开场**：时间线出现钩子事件；composer 上方出现 ≥2 条
   开场提案（.suggestion-slip），文案不是泛化「观察四周」；点选提案填入
   输入框，提交后真实模型回合正常落库（时间线增长）。
4. **T1-4 fail-closed（集成层）**：tests/postgres-first-night.test.ts 用失败
   网关断言降级包落库（场景旁白存在、state=degraded、不抛错）。

回归门槛：`npm test` 全绿；N1 用例中「时间线为空」断言随行为变更同步更新
（逐步引导世界现在也有确定性开场旁白）。

## 五、实施与验证纪律

- commit 拆分：规范文档 → 能力点1（场景即时生成）→ 能力点2（角色在场）→
  能力点3（钩子事件开场）→ GUI 用例收口；每个能力点完成后立即 GUI 验证。
- GUI 验证：HOST_BIND=192.168.31.238，真实模型，不 mock。
- 每次 GUI/集成写入后执行 scripts/clean-gui-test-data.sql（新增表
  record_first_nights 由记录级联删除覆盖，清理脚本无需新增规则——records
  删除即 CASCADE；核对后在报告记录计数）。
- STATUS.md 三段式追加；迭代日志写入 EXPERIENCE-ITERATION.md。
