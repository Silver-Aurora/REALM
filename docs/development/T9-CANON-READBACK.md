# T9 · Canon 回读：世界设定约束后续生成

> 批次 T9（docs/development/EXPERIENCE-ITERATION.md 迭代主轴第九项，对应第二章缺陷 #15「canon 不回读」、#14「知识图谱零数据」的晶化入图部分、玩法设计风险 #4「canon 无约束力」）。
> 上位设计：M5 世界治理（./M5-WORLD-GOVERNANCE.md §2 真值阶梯/Canon 治理）、设定结晶（./SCENE-CRYSTALLIZATION.md）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中

## 一、现状实锤（审计结论，勿重复调查）

1. **消费侧缺失**：回合生成的设定依据只有 `buildSceneCanon(brief, style)`（modules/orchestration/model-powered.ts:109-135）——世界名/纪元/背景/故事/场景快照 + 文风。canon_revisions/canon_proposals/story_canon 以上 claims 从不进入任何生成 prompt；世界观漂移无约束。
2. **产生侧断链**：场景晶化裁决通过的 delta 只写 scenes/worlds.settings/system 事件（database/postgres/scene-crystallization-repository.ts applyDelta），从不流入 world_entities/world_claims——图谱运行时零数据（缺陷 #14）。
3. **API 硬编码 demo 世界**：`app/api/canon/route.ts:19-23` 与 `app/api/world-knowledge/route.ts` 把 scope 写死 world_ember_coast——K 组「图谱 · 正史」UI 对非 demo 世界实际读写错世界（缺陷 #13 的 API 层形态）。
4. **既有资产**：world-knowledge 服务（实体/Claim/真值阶梯晋升/Article）与 canon 服务（提案/合并/Revision）实现完整且有 PG 仓储；claim 按 worldline 作用域隔离——消费侧只需按 scope 读。

## 二、核心设计

### 2.1 产生侧 A：晶化产出入图谱（records→claims）

晶化 `applyDelta` 裁决通过落库时（同事务外的新增步骤，失败只留日志不阻断晶化）：

- 主体实体：每世界一个「世界本体」实体（kind='setting'，name=世界名，确定性 id `entity_world_<worldId 尾>`，upsert 幂等）；
- 每个实际变化的字段落一条 Claim：predicate ∈ `scene.location / scene.objective / world.weather / world.tension / world.displayTime`，objectValue=新值，scope='record'，truthStatus='record_confirmed'，confidence=1，sourceRecordId/sourceEventId=晶化事件，validFrom=晶化游标；
- record 级 Claim 依 M5 层级门禁不产生 canon 提案（不打扰用户）——提案由玩家在图谱 UI 手工发起（既有 K 组链路），或后续批次的 DM 提案器。

### 2.2 产生侧 B：canon/图谱 API 去 demo 硬编码

- `/api/canon`（GET/POST）与 `/api/world-knowledge` 增 `worldId` 参数（必填校验，fail-closed 400）→ scope 解析该世界当前活动世界线（最早 active 世界线，与 library 挂接语义一致）；membership 校验（非成员 404 形态）。
- 图谱覆盖层打开时已知 worldId/worldName，请求带参——demo 世界行为不变（K 组既有断言改带参后语义等价）。

### 2.3 消费侧：正史注入回合生成

- `RecordRuntimeScope.brief` 增 `canon: string`（多行文本，缺省 ""）——record-scope 解析时查询本世界线 `truthStatus IN ('story_canon','world_canon')` 的 Claim（supersede 链最新、按 validFrom/created 确定性排序、上限 12 条、单条 ≤160 字截断），渲染为「- 主语 谓语 值」行；**查询失败 fail-closed 为空串 + 日志，绝不阻塞回合**。
- `buildSceneCanon` 在场景快照之后、文风块之前插入 `正史（不可违背）：\n{canon}` 段——DM 规划/旁白/角色/复核/可见性评估五个既有调用点零改动自动覆盖。
- 版本/世界线一致：canon 读取发生在回合 scope 解析时刻（与回合同一快照语义）；Claim 按 worldline 隔离，分支互不串。
- 权限隔离：canon 是本世界世界线内的公开世界事实（玩家可经图谱 UI 查看同等内容），不含角色私有/秘密受众信息；restricted 回合不多泄——注入文本对所有成员等价。
- 模型输出仍只是候选：正史只约束生成，模型输出永远经既有校验/复核/裁决链，不把模型输出当事实写库（本批零新增写库路径）。

### 2.4 与既有机制的关系

- 不新建表、不改迁移（claim/entity/revision 表 0010 已备）；
- 不触碰真值阶梯语义（晋升仍只经 canon merge 或 promoteClaim）；
- 晶化入图失败不阻断晶化本身（fail-closed 日志）；canon 注入为空时 prompt 零变化（向后兼容）。

## 三、失败矩阵（fail-closed）

| # | 场景 | 形态 |
|---|---|---|
| F1 | canon 查询失败/超时 | brief.canon=""，回合照常（日志） |
| F2 | 晶化入图写库失败 | 晶化落库不回滚，入图跳过（日志） |
| F3 | canon API 缺 worldId/世界不存在 | 400 / 404 形态 |
| F4 | 非成员访问 canon/图谱 API | 404 形态（不泄露存在性） |
| F5 | Claim 形状非法（空值/超长） | 入图跳过该条（日志） |
| F6 | 图谱 UI 对任意世界 | worldId 显式传参，无 demo 串场 |

## 四、验收标准

1. Core/应用：buildSceneCanon 含 canon 段（有/无 canon 两态）；record-scope canon 装配（仅 story_canon 以上、supersede 最新、上限与截断、确定性排序、失败为空）。
2. PG 集成：晶化 approved delta → 实体 upsert 幂等 + Claims 落库（record_confirmed/来源事件/游标正确）；canon/图谱 API 带 worldId 对非 demo 世界读写正确、demo 世界零回退；merge 后 story_canon claim 出现在下一回合 scope 的 brief.canon。
3. GUI t9 spec 真实模型：自建世界 → 图谱 UI 建实体+Claim → 提案+合并 → 提交真实回合不 422 且完成（注入不破坏管线）；demo 世界 K 组既有断言零回退。
4. 回归圈：t9 + k-graph + m-scene-crystallization + c-actions + 冒烟 a-library/b-record。
5. 每次 GUI/PG 写入后清理回基线（含 world_entities/world_claims 无测试残留——清理脚本已覆盖）。

## 五、交付步骤与 commit 纪律

1. 本规范（单独 commit，含索引登记）；
2. 产生侧：晶化入图 + API 去硬编码（PG 测试随码）；
3. 消费侧：record-scope canon 装配 + buildSceneCanon 注入（Core/应用测试随码）；
4. 测试收口：GUI t9 + 范围化回归 + 清理回基线 + STATUS/迭代日志。

禁 `git add -A`；每步独立 commit；不 push。
