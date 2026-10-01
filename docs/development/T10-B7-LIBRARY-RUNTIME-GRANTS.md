# T10-B7 · Library owner-pool 例外的最小授权迁移评审与下沉

> 批次 T10-B7（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B6 §三 owner 例外枚举的迁移评审与下沉）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=3e8a1af；旧迁移 0001–0022 不改；本批只允许新增 `0023_library_runtime_grants.sql`；不改 LM Studio/T6–T10-B6。

## 一、逐表逐列需求矩阵（从真实 SQL 抽取，勿重复调查）

owner 例外命令（world/character/record/attach-character/player-stance/
createGenesis/tavern import）的真实写入与所需最小授权：

| 表 | 操作 | 列（真实 SQL 为准） | 证据 |
|---|---|---|---|
| worlds | INSERT | workspace_id, id, name, status, calendar_id, summary, settings | library-service.ts:335/673 |
| character_definitions | INSERT | workspace_id, world_id, id, display_name, source_format, profile | :383/713/931/1568；tavern-import-service.ts:192 |
| character_continuities | INSERT | workspace_id, world_id, worldline_id, definition_id, id, continuity_key, status, born_tick, born_ordinal | :953/1368/1592 |
| character_instances | INSERT | workspace_id, world_id, worldline_id, record_id, continuity_id, id, controller_mode, status, instantiated_tick, instantiated_ordinal, inheritance_cutoff_tick, inheritance_cutoff_ordinal | :975/1383/1618 |
| participants | INSERT | workspace_id, world_id, worldline_id, record_id, id, participant_kind, character_instance_id, principal_id, controller_mode, is_active, speaking_order | :991/1066/1398/1633 |
| participants | UPDATE | is_active（仅姿态开关翻转列） | :1484/1514/1535 |
| record_heads | INSERT | workspace_id, world_id, worldline_id, record_id, record_version, next_record_ordinal, last_world_tick, last_world_ordinal | :639/754 |
| player_world_memberships | UPDATE | role（仅姿态切换列） | :1463 |
| skill_definitions | INSERT | workspace_id, world_id, id, skill_key, title, description, rule_pack_key, metadata | base-rule-definitions.ts:97 |
| asset_definitions | INSERT | 同上 + consumable | base-rule-definitions.ts:117 |
| effect_definitions | INSERT | 同上 + effect_kind | base-rule-definitions.ts:136 |
| character_skills | INSERT | workspace_id, world_id, worldline_id, record_id, character_instance_id, skill_definition_id, acquired_tick, acquired_ordinal | base-rule-definitions.ts:197/283 |
| character_assets | INSERT | 同上（asset_definition_id, quantity） | base-rule-definitions.ts:233 |

已具备（0001–0022 既有，不重复授）：worldlines/stories/records INSERT（0022）、
player_world_memberships INSERT（0013）、scenes/visibility_policies/
visibility_policy_audiences 列级 INSERT（0003/0007/0014）、record_first_nights
（0019）、world_files（0016）、world_articles/world_entities/world_claims
（0010）、events 列级 INSERT（0002）、worlds UPDATE(settings)/UPDATE(status)/
DELETE（0014/0021）。

说明：INSERT 授表级（0022 同型）；UPDATE 授列级（房屋惯例——0004 以来
UPDATE 全为列级）。默认值列可省略与否以真实 SQL 为准——上表列即全部
实写列；表级 INSERT 允许省略的默认列由 schema 默认填充，不额外授权。

## 二、Security invariants（迁移不得削弱）

1. 全部事务仍经 `withWorkspaceTransaction` 的 workspace scope（RLS FORCE 不变）；
2. workspace/world/principal 由服务端守卫解析，客户端不可注入；
3. append-only/immutable 触发器零放宽（本批不触 events/action_receipts 等）；
4. owner/player/observer 业务门禁由 T10-B6 服务层矩阵执行，不由 grant 替代；
5. 严禁 GRANT ALL、整 schema 写、绕 RLS。

## 三、迁移 0023 与下沉接线

- `0023_library_runtime_grants.sql`：上表逐项 GRANT（注释逐条对应本规范 §一）。
- 不可逆说明：GRANT 不可逆地扩大授权面属刻意决策——下沉后 owner pool
  从 library 路由彻底移除；如需回退，应由新迁移显式 REVOKE（不做）。
- 下沉：library 路由删除 owner 例外（GRANT_BLOCKED_OWNER_COMMANDS 清空并
  移除 owner pool 代码）；`/api/world/generate`（createGenesis）与
  `/api/library/import` 切 REALM_RUNTIME_DATABASE_URL + 共享池；缺 runtime
  URL 全部 503 安全文案；不再依赖 DATABASE_URL。
- 不在本批：files/[id]、settings/language、auth/me 的 owner pool（读 accounts/
  world_files，授权面已够但属独立路由，留后续批次统一收尾）。

## 四、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 缺 REALM_RUNTIME_DATABASE_URL | 503（全部已下沉命令） |
| F2 | 非成员/observer/非 owner 业务门禁 | T10-B6 矩阵不变（404/403） |
| F3 | 迁移前授权缺口 | focused 测试记录 has_column_privilege 证据 |
| F4 | 迁移后越权写入（他 workspace） | RLS 拒绝（测试断言） |
| F5 | 既有 T6/T8/T9 约束（归档/活动自演/幂等） | 不回退 |

## 五、验收标准

1. 临时库 0001–0023 后 runtime-only：world/character/story/record/branch/
   attach/player-stance/archive/delete/createGenesis/tavern import 逐项真实成功；
   迁移前后 has_column_privilege/has_table_privilege 对比证据；
   RLS 他 workspace 不可见/不可写；T6/T8/T9 门禁不回退。
2. postgres-library-permissions/library-service/world-admin/tavern-import/
   tavern-import-play/canon-readback/first-night/auth/api-core-wiring 全过；
   schema 契约更新迁移列表 + 0023 断言；typecheck/eslint/文档布局/
   git diff --check exit 0；不跑全量/GUI。
3. 开发库基线不变；临时库 t.after 强制 DROP。

## 六、交付步骤

1. 本规范（单独 commit）；2. 0023 + 路由/服务下沉 + 静态契约更新（独立
   commit）；3. focused 测试 + STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
