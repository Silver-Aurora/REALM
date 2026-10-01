# T12 · 世界 / 故事 / Record 体验重构 + 重演状态隔离

> 批次 T12。立项：2026-08-29 · 状态：设计定稿，实施中
> 基线：HEAD=9f0ffe7（feat: ground discoveries and archive records）。
> 本文档是 A/B/C 三项交付的根因实锤与实现决策记录；低风险设计选择在此定稿并注明理由。

## 一、A：重演起点语义（根因 → 决策）

### 1.1 根因实锤（当前 HEAD 代码复核）

`duplicateRecordInTransaction`（modules/application/library-service.ts）当前把「重演」实现为**从源 Record 当前头部复制**：

1. 分叉游标取 `record_heads.last_world_tick/last_world_ordinal`（源 Record 已推进的世界时间）——新 worldline head、新 Record start、新 record_heads 游标全部等于源 Record 的**最新**世界时间；
2. 开场 scene 取源 Record **最新一条** scene 行（`ORDER BY start_tick DESC LIMIT 1`）的 title/location/objective（tension 硬编码为空）——重演 Record 的地点/目标直接继承源 Record 已推进的场景；且 `empty_retrospection` 遮盖（record-scope/delivery-projection 对零事件 retrospection 置空 tension/objective）只覆盖开局前，**第一条事件后遮盖失效，复制的「最新场景」重新显现**；
3. weather/displayTime 由 delivery-projection/record-scope 从 `worlds.settings` 现场读取——scene-crystallization 每次推进都会 `settings || worldPatch` 合并 displayTime/weather（database/postgres/scene-crystallization-repository.ts §worldPatch），旧值无版本，**起点 weather 对既有数据不可恢复**；
4. `worlds.settings->>'tension'` 的 legacy fallback（readSceneTensionFallback）只影响全新装配路径，duplicate 传显式 tension 不经过它——但新 scene 行若无 weather 快照机制，projection 的 weather 永远跟随世界当前态。

### 1.2 「重演起点」语义决策

- **起点游标 = 源 `records.start_tick / start_ordinal`**（每条 Record 的固有起点快照点：primary Record 为 (0,0)，retrospection Record 为其自身分叉起点）。新 worldline `head_tick/head_ordinal`、新 Record `start_tick/start_ordinal`、新 record_heads `last_world_tick/last_ordinal` 全部取该值。
- **开场 scene = 复制源 Record 的首个 scene 行**（`ORDER BY start_tick ASC, start_ordinal ASC, created_at ASC LIMIT 1`）的 title/location/objective/tension。首事件后 `empty_retrospection` 遮盖失效时，显现的是起点场景而非源最新场景。
- **隔离**：新 worldline 使 scenes/events/claims/visibilities 全部按 worldline 作用域隔离；duplicate 不修改 worlds.settings 与源 Record 的任何行；历史 Event/Receipt append-only 不动。
- **weather（兼容语义，决策）**：weather/displayTime 是**世界级推进值**（结晶合并进 worlds.settings，0028 前无逐 scene 快照，起点值不可恢复）。决策：迁移 **0039** 为 `scenes` 增加 `weather` 快照列；今后装配（assembleDefaultRecord）与结晶（scene-crystallization）写入 scene 行时落当时 weather；projection 的 weather 解析改为「scene 快照优先、worlds.settings 回退」。旧 scene 行 weather='' → 回退到世界当前 settings（兼容语义，文档化：0028 前创建的数据无法恢复起点 weather，按当前世界天气显示）。0028 后新建/结晶的 scene 行携带快照，其重演完全隔离。
- **displayTime（决策）**：不加 scene 列。displayTime 是随世界时间推进的世界级时钟标签而非场景状态；重演 Record 的世界时间由归零游标驱动（record_heads），displayTime 标签显示世界级当前值属已知兼容边界，写入本文档而非静默。
- **fail-closed**：duplicate 源 Record 若 `status='archived'` 或不存在 → 404（既有）；起点 scene 缺失（理论上 record 无 scene 行）→ 以空字段装配（与既有 AssembleSceneSeed 缺省一致），不得读 worlds.settings 冒充初始场景。

## 二、B：archived Record 外显（根因 → 修复面）

### 2.1 已核实无泄漏（不再改动）

- Library 树/record_count/last_active_at/story records jsonb（HEAD 已过滤）；
- record-scope（`record.status <> 'archived'`，写入/模型 scope fail-closed）；
- duplicate 源查询（已过滤）；archive 命令已把 `accounts.last_record_id` 置 NULL；
- 前端 `createLibraryItem` 成功后 `loadLibrary()` 权威重读（非本地 splice）；当前 Record 被归档 → `loadRecord("")` 走默认入口/onboarding 回落。

### 2.2 实锤两处服务端漏过滤

1. `delivery-projection.ts` META_SQL 主查询 `WHERE record.workspace_id=$1 AND record.id=$2` **无 status 过滤**——已归档 Record 仍可被 `/api/record`（及 SSE 交付路径）完整打开，违反 fail-closed；
2. 同文件 `loadNavigation` 的 records 导航查询无 status 过滤——左侧 Record 导航（world-navigation 组件读 projection.records）外显已归档 Record。

修复：两处补 `AND record.status <> 'archived'`，并补 PG/契约测试锁定（归档后打开 404、导航树不含 archived、Library 树不含、计数正确）。

## 三、C：页面级信息架构

- **顶栏三级面包屑可点击**：世界名→世界视图、故事题→故事视图、Record 题→记录视图；`<button>` + `aria-current`（当前视图）+ focus-visible 样式；不再只是文本。
- **视图状态** `view ∈ {record, world, story}`，URL `?view=` 用 `history.replaceState` 同步——刷新/深链不丢视图（recordId 深链既有）。
- **世界视图**（页面内，非 overlay）：Library snapshot 驱动的真实数据——名称/era/summary/status/角色/世界线/故事/Record 密度/最近活动；复用 T8 管理操作（归档/恢复/删除世界、Record 删除按钮组件）；进入故事（点故事名）与 Record 的明确主动作。
- **故事视图**：标题/premise/所属世界 + 该故事未归档 Record 列表（snapshot 服务端已过滤）；点击 Record 打开；空状态结构性消隐，不写假内容。
- **记录视图** = 现有页面；从世界/故事视图进入 Record 后 breadcrumb 三级上下文同步（envelope 驱动）。
- **世界库 overlay 保留**给创世/导入/手动新建等临时操作（真正的确认/临时操作）；正常导航/浏览在页面内。
- 数据全部来自 `/api/library` snapshot 与 record envelope；无新 API（现有投影足够，理由：世界/故事视图所需字段 Library snapshot 已完整携带）；无硬编码世界/故事/Record 兜底。
- 新增固定文案进 modules/i18n（zh-CN/en/ja 完整）；动态文本不进 i18n。
- 响应式遵循 UI-DESIGN §4（桌面三栏/平板双栏/手机纵向），不引入全局 overflow 隐藏。

## 四、实施与验证计划

1. A：duplicateRecordInTransaction 起点语义改写 + 0039（scenes.weather + realm_runtime INSERT(weather) 列级授权）+ projection weather 快照优先回退 + PG 集成回归（推进源 Record → 重演 → 逐字段断言起点 + 双向隔离）。
2. B：delivery-projection 两处过滤 + PG 回归（归档后打开 fail-closed/导航树/库树/计数）。
3. C：realm-client 视图状态 + 可点面包屑 + WorldView/StoryView 组件（复用 library 管理件）+ i18n + 响应式 CSS + 前端契约/静态测试。
4. 验证：受影响 PG（--test-concurrency=1）、Core/Application/Frontend、tsc、lint、diff --check、npm test 全链；GUI 关键路径（导航切换/归档隐藏/重演起点）如实记录结果与阻塞；收口复核 git status、迁移台账、临时库、服务状态。

## 五、验收修正（独立验收反馈后的冻结决策）

### 5.1 display_time 同为 Record/世界线级隔离状态（0040）

- 初始实现只快照了 weather，displayTime 仍读 `worlds.settings` / 最新 Event——不满足「重演后世界时间一起回起点」。0040 给 `scenes` 增 `display_time text NOT NULL DEFAULT ''` + `GRANT INSERT (display_time) TO realm_runtime`（0014 列级授权模式）。
- duplicate 复制源**首个 scene** 的 location/objective/tension/weather/display_time 五项快照；新 worldline head、record start、scene 起点一致（record_heads/worldlines/records 游标全部为源 `start_tick/start_ordinal`）。
- 读取链一律「scene 快照优先」：record-scope `displayTime`/`brief.weather`、delivery-projection `initial_display_time`/`weather`、结晶事件 `display_time`。**无快照的 retrospection fail-closed 为空**（不得回退 settings 把旧重演拉到当前世界时间/天气）；primary 旧数据保留 settings 兼容回退，按 `timeline_kind` 区分。
- 结晶流转触发条件扩为 location/objective/tension/weather/displayTime 任一变化；新 scene 行未变字段继承**上一 scene 快照**（优先于 settings）。
- worlds.settings 写回评估结论：**保留**。settings 只是世界级「最新已知」与 legacy 回退；有快照的 Record 永远不读它，因此源 Record 推进不会污染重演或其他有快照的 Record；无快照 retrospection 已 fail-closed，不受 settings 影响。

### 5.2 多故事页面级选择

- Library snapshot story 节点增 `premise`（服务端 SQL + normalizer，缺省 fail-closed 空串）——不得用当前 Record 的 premise 冒充其他故事。
- 主区状态 = `view` + `selectedStoryId`，深链 `?view=story&storyId=`（replaceState）；面包屑「故事」入口始终回当前 Record 的故事（清除显式选择）；WorldView 每个故事按钮携带自身 `data-story-id`；StoryView 按被选 story 渲染标题/状态/前提/未归档记录；从被选故事点击记录打开对应 Record 并回记录视图。
- 左侧 WorldNavigation 的 story/record 项从纯文本 `<li>` 改为真实按钮入口（`.nav-entry`），story → 故事视图、record → 打开记录。
- GUI：多故事选择/归档隐藏用例必须走**隔离临时库 + 专属端口**（共享 realm_dev 禁止注入剧情/物理清理）；实测发现 vinext dev 单实例锁（同目录第二 dev server 拒绝启动），故 harness 用 `vinext build + start`（生产模式，:10099）规避，不动 realm-dev.service。
