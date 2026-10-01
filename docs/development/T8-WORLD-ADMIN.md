# T8 · 世界管理台：浏览 / 归档 / 删除与世界卡信息密度

> 批次 T8（docs/development/EXPERIENCE-ITERATION.md 迭代主轴第八项，对应第二章缺陷 #4「世界管理缺位」）。
> 上位依赖：批次 S 观察者/姿态与账号记忆（./WORLD-ONBOARDING.md）、T7 自演会话（./T7-OBSERVATION-VISION.md）、M1 不可变事件契约（./M1-CORRECTNESS-FOUNDATION.md）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中

## 一、现状实锤（审计结论，勿重复调查）

1. **没有任何删除/归档入口**：`LibraryCreateCommand` 八种 kind 无 delete/archive；`worlds.status` CHECK 自 0001 起就含 `'archived'`，但全库无写入方（唯一 archive 写入是 worldline 合并后记录置 archived）——「归档」是 schema 预留、零接线的假状态。
2. **物理删除被架构硬性约束**：events 等 11 张表挂 append-only 触发器（BEFORE UPDATE OR DELETE 拒绝），realm_runtime 对 events 连 DELETE 授权都没有；events 无外键指向 records（删 records 不会带走事件），且 events→command_inbox 等引用为 NO ACTION——有事件的世界无法物理删除（触发器拒绝 + 引用阻断）。清理脚本能删是因为它以属主身份 DISABLE TRIGGER。
3. **权限裸奔**：library 写命令只校验登录，任何 principal 可对任何世界发 world-style/stance/attach 等命令；`role='owner'` 现成可读（membership 行），但没有任何写命令消费它。
4. **世界卡信息密度低**：快照只带 name/era/style/summary/status/membershipRole + 角色/世界线/故事列表；排序恒 created_at ASC；无记录数、角色数、最近活动、归档态呈现。
5. **删除当前打开世界的兜底已在**：`accounts.last_record_id` FK ON DELETE SET NULL（0018）+ `openDefaultRecord` 失败回落 onboarding；`last_world_id` 无外键（悬空文本无害，删除时顺手清）。
6. **进行中写者无防护**：删除/归档若撞上活动自演会话（running/stopping）或进行中回合，写者会因 FK/行缺失失败；自演懒恢复依赖账本行存在。

## 二、核心设计

### 2.1 归档（主交付，零表结构变更）

- `LibraryCreateCommand` 新增 `{ kind: "world-archive"; worldId; archived: boolean }`。
- **owner 门禁（本批引入，仅作用于管理命令）**：`player_world_memberships.role='owner'` 才允许归档/删除，否则 `WORLD_NOT_OWNED`（403 形态）。这是 library 服务首个权限门禁，只加在管理命令上，既有命令行为零变化（遗留权限加固另案）。
- 归档语义 = 封存：
  - worlds.status 'active' ⟷ 'archived' 翻转（幂等：重复归档 rowCount 0 也返回成功现状）；
  - 归档世界**不可开新局**：story/record/character/branch/import/attach-character 命令 fail-closed `WORLD_ARCHIVED`；
  - 归档世界**既有记录只读**：`submitMessage` 与 `startSelfPlay` 拒绝（`WORLD_ARCHIVED`）；读取/投影/SSE 不变（回看自由）；
  - 实现：`RecordRuntimeScope` 增 `worldStatus`（record-scope SQL join worlds 带出），写路径在 submitMessage/startSelfPlay 入口检查；library 命令侧在 dispatch 前查世界状态。
- 活动自演会话防护：归档前检查该世界无 running/stopping 会话（否则 `WORLD_SELF_PLAY_ACTIVE`，先停止再归档）；归档世界 startSelfPlay 本身已被只读门禁拦住。
- 前端：世界卡归档/恢复按钮 + 归档徽标；列表排序——active 在前（按最近活动 desc），archived 沉底独立分组。

### 2.2 删除（受限物理删除，fail-closed）

- `LibraryCreateCommand` 新增 `{ kind: "delete-world"; worldId }`。
- **仅零记录世界可删**（实测收窄，原拟「零事件」不成立：记录装配即写 character_skills/assets 账本行，经 NO ACTION 引用网（character_skills→skill_definitions、participants/command_inbox→memberships）与世界分支交错，append-only 触发器与 realm_runtime 授权边界使运行时物理清除不可行）：有记录的世界 → `WORLD_NOT_EMPTY`（引导归档）。
（原「零事件」判据已按上条实测收窄为零记录；空世界 ⇒ events/action_receipts/observations/账本行全空，CASCADE 单分支无触发面。）
- 删除 = 单事务 `DELETE FROM worlds WHERE …`（ON DELETE CASCADE 带走 worldlines→stories→records→参与者/席位/definitions/files/articles/初夜与自演账本等；级联动作以表属主权限执行，不要求 realm_runtime 持子表 DELETE）；`accounts.last_record_id` 由 FK 置空、`last_world_id` 悬空文本顺手清（UPDATE accounts SET last_world_id=NULL）。
- 同 2.1 的活动会话防护 + owner 门禁；删除当前打开世界后前端回 onboarding（既有兜底）。
- 新迁移 **0021_world_admin.sql**：`GRANT UPDATE (status, updated_at) ON worlds TO realm_runtime`（归档翻转；0014 只授了 settings 列）+ `GRANT DELETE ON worlds TO realm_runtime`（受限删除）。新建理由：世界管理台是全新写能力，既有两份 worlds 授权（0001/0014）不覆盖 status 列与 DELETE。
- **Record 级删除（新增）**：`LibraryCreateCommand` 支持 `{ kind: "delete-record"; worldId; recordId }`。这是 owner-only 的隐藏归档：`records.status` 置为 `archived`，Library 与运行时 scope 不再展示/打开它；append-only Events、worldline 游标、观察和审计引用保留。重复请求幂等，`accounts.last_record_id` 清空。
- Record 删除前检查目标 Record 没有 running/stopping 自演会话，也没有 accepted/planning/drafting/validating/releasing/retryable 的活动 Turn；否则分别返回 `RECORD_SELF_PLAY_ACTIVE` / `RECORD_TURN_ACTIVE`。
- 新迁移 **0037_record_archive_grant.sql**：仅授予 `realm_runtime` 更新 `records.status/updated_at`，不授予物理 DELETE；物理删除事件仍被架构保留边界拒绝。
### 2.3 浏览与信息密度

- LibrarySnapshot 世界卡增（聚合 SQL，单查询不 N+1）：`storyCount`、`recordCount`、`characterCount`、`lastActiveAt`（max(records.updated_at)，无记录为 null）、`status` 已有。
- 排序：active 按 lastActiveAt desc（null 排后，再按 created_at asc），archived 沉底按 updated_at desc。
- 前端世界卡：状态徽标（active/archived i18n）、三计数一行（角色/故事/记录）、最近活动时间、管理区（归档/恢复、删除按钮——删除按钮恒可见，不可删时服务端回 WORLD_NOT_EMPTY 文案提示先归档）。
- 删除二次确认：前端点击删除 → 浏览器 confirm 语义卡（键入世界名确认从简——本批用一次性确认按钮两段式「再点一次确认删除」，不落新组件库）。

### 2.4 权限与可见性边界

- 管理命令（world-archive/delete-world/delete-record）owner-only；其余命令维持现状（本批不做权限大改）。
- 归档不影响既有成员的读取与投影；observer 姿态与归档正交（归档世界的记录观察者同样只读可看）。
- 世界删除仍是物理删除：知识图谱/canon/传播审计表对该世界的行随 CASCADE/零事件前提不复存在；Record 删除则只隐藏并保留事件与审计引用。worldline_merges/semantic_conflict_evaluations 无 worlds 外键，但零事件世界不可能有这些行（合并/评估都以有事件为前提），无孤儿。

## 三、失败矩阵（fail-closed）

| # | 场景 | 形态 |
|---|---|---|
| F1 | 非 owner 发管理命令 | `WORLD_NOT_OWNED`（403 形态） |
| F2 | 归档世界开新局（story/record/character/branch/import/attach） | `WORLD_ARCHIVED` |
| F3 | 归档世界提交回合/开自演 | `WORLD_ARCHIVED`（写路径入口检查） |
| F4 | 归档/删除撞上活动自演会话 | `WORLD_SELF_PLAY_ACTIVE`（先停止） |
| F5 | 删除有记录世界（含草稿记录） | `WORLD_NOT_EMPTY`（引导归档） |
| F6 | 删除/归档不存在或他 workspace 世界 | `WORLD_NOT_FOUND`（沿用） |
| F7 | 重复归档/恢复 | 幂等返回现状成功 |
| F8 | 删除当前打开世界 | FK 置空 + openDefaultRecord 回落 onboarding（既有） |
| F9 | 删除撞上当前 Record 的自演拍 | `RECORD_SELF_PLAY_ACTIVE`（先停止） |
| F10 | 删除撞上当前 Record 的活动回合 | `RECORD_TURN_ACTIVE`（等待回合结束） |

## 四、验收标准

1. PG 集成：归档翻转幂等 + owner 门禁 + 归档后创建/提交/自演全拒 + 活动会话防护；Record 删除隐藏列表、保留 Events、清空 last_record_id、拒绝活动 Turn/自演且重复请求幂等；零事件世界删除后 worlds/worldlines/stories/records/definitions/memberships 全级联清零、accounts.last_* 复位、可重建同名世界；有事件世界删除被拒；0021/0037 授权真实存在（realm_runtime 重定向验证）。
2. 应用层/Core：快照聚合字段与排序（active 最近活动 desc、archived 沉底）；命令规整 fail-closed。
3. GUI t8 spec 真实链路：创建世界 → 信息密度行可见 → 归档 → 徽标+沉底+开新局被拒 → 恢复 → 删除（两段确认）→ 世界消失且默认入口回落。chromium 主测；前端排序/徽标为跨引擎面，加跑 webkit。
4. 回归圈：t8 + a-library + s-onboarding + l-genesis + n-guided-genesis + t1-first-night + 冒烟 b-record；demo 零回退（A1 断言不动）。
5. 每次 GUI/PG 写入后清理回基线（含 record_self_play_sessions 无残留）。

## 五、交付步骤与 commit 纪律

1. 本规范（单独 commit，含 docs/README.md 索引登记）；
2. 迁移 0021 + library 命令（world-archive/delete-world + owner 门禁 + 活动会话防护）+ scope.worldStatus 写路径只读门禁（PG/应用测试随码）；
3. 快照信息密度与排序 + 前端世界卡（徽标/计数/管理按钮/两段确认）+ i18n（独立 commit）；
4. 测试收口：PG 集成补齐 + GUI t8 spec + 范围化回归 + 清理回基线 + STATUS/迭代日志（独立 commit）。

禁 `git add -A`；每步独立 commit；不 push。
