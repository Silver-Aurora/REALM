# T10-B19-A · POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权 provenance（只文档/契约，不改 SQL）

> 批次 T10-B19-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批候选①）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=dc998e3；本批只写文档与静态契约——不改 `database/postgres/migrations/*.sql`、活动 TS/TSX、package/lock、archive、DB、服务配置；不新增授权、不执行迁移、不改变 RLS/trigger/role 行为。

## 一、目标与边界

- 在 `docs/architecture/POSTGRESQL-RUNTIME-CONTRACT.md` **追加**「T-series
  authorization provenance」章节（不改写上文历史），把 0013–0023 迁移对
  `realm_runtime` 的授权来源（批次/体验能力）、grant surface 与安全边界
  集中记录——当前文档只有 v1 基线授权面，T 系列分散在各批次 STATUS/规范。
- 完成后只能声称「T 系列授权 provenance 已集中记录并由静态契约守住」，
  不得声称新增权限或改变运行时行为。

## 二、事实基线（2026-08-21 回读真实 SQL，非猜测）

| 迁移 | 批次/能力 | `realm_runtime` grant surface（实读） | 安全边界 |
|---|---|---|---|
| 0013 membership_insert_grant | 部署鉴权（登录即加入默认世界，DEPLOY-AUTH） | INSERT ON player_world_memberships | 不授 UPDATE/DELETE；omniscience 不可变触发器姿态不变 |
| 0014 scene_crystallization_grants | 设定结晶（SCENE-CRYSTALLIZATION） | scenes 11 列列级 INSERT；UPDATE (settings) ON worlds | append-oriented：既有行不可 UPDATE/DELETE（COMMENT 写死）；settings 仅合并写回 |
| 0015 account_ui_language | i18n 用户级界面语言（I18N-CENTRALIZED） | UPDATE (ui_language) ON accounts | 列级 + 三语 CHECK |
| 0016 world_files | 酒馆导入（TAVERN-IMPORT） | SELECT, INSERT ON world_files | ENABLE+FORCE RLS（realm_workspace_isolation）；不可变 blob，无 UPDATE/DELETE |
| 0017 account_last_opened | 批次 S 默认入口「最近打开」记忆 | UPDATE (last_world_id, last_record_id) ON accounts | 列级，仅两列 |
| 0018 account_last_opened_fk_set_null | 批次 S FK 修复 | 无 grant |  targeted SET NULL (last_record_id) 语义修复，零权限变更 |
| 0019 record_first_nights | T1 第一夜 | SELECT, INSERT, UPDATE | ENABLE+FORCE RLS |
| 0020 record_self_play_sessions | T7 世界自演 | SELECT, INSERT, UPDATE | ENABLE+FORCE RLS + 单活跃账本（既有契约断言） |
| 0021 world_admin | T8 世界管理台 | UPDATE (status, updated_at) ON worlds；DELETE ON worlds | 列级 UPDATE；DELETE 由服务层 fail-closed 门禁（零事件世界才可删，demo/有记录拒删）——DB grant ≠ 业务语义放宽 |
| 0022 worldline_merge_grants | T10-B1 merge 作用域修复 | INSERT ON worldlines, stories, records | 只增拓扑行，无 UPDATE/DELETE |
| 0023 library_runtime_grants | T10-B7 owner-pool 例外下沉 | 11 表表级 INSERT + participants(is_active)、player_world_memberships(role) 列级 UPDATE | owner-pool 例外清零后的**最小**授权，非全特权 GRANT；不重复覆盖 0013–0022 历史授权 |

- 唯一 `GRANT ALL PRIVILEGES` 出现在 0006（TO `realm_control`，控制面角色），
  不在 T 系列面内；T 系列对 `realm_runtime` 无全特权授权。
- `realm_control`/maintenance 角色与长驻 Application Service 不混用：
  T 系列授权全部指向 `realm_runtime`；`DATABASE_URL` 仍属一次性维护。
- T10-B7/B8 owner-pool 下沉与 T10-B18 D1/Drizzle 退役均未改动
  PostgreSQL 权威链（迁移 0001–0023 SQL 零改动）。
- propagation/semantic-conflict 仍为 contract-only-deferred（T10-B9-A）：
  授权文档不得暗示其已接线——相关表（0011）的授权事实不构成激活证据。

## 三、「DB grant ≠ 业务语义放宽」三类写清

- `worlds DELETE`（0021）：DB 层仅放行 DELETE 能力；业务层服务门禁
  fail-closed（零事件世界、owner 校验、demo 拒删），RLS 仍逐行生效。
- 列级 UPDATE（0014/0015/0017/0021/0023）：授权精确到列，业务语义
  （settings 合并、语言 CHECK、最近打开记忆、归档状态机、membership
  角色）由应用层与 CHECK/trigger 共同约束。
- append-only 表（scenes 0014、拓扑 0022、ledger 类 0006）：INSERT 放行
  不代表可改写历史——无 UPDATE/DELETE 授权 + 既有触发器/契约断言。

## 四、静态契约围栏（扩展 tests/postgres-schema-contract.test.mjs）

新增 focused 测试，只读真实 migration/doc 内容：

1. 0013–0023 十一个迁移文件存在（沿用既有 migrationPaths 实锚）；
2. 新章节存在且含全部锚点：十一个迁移文件名、批次锚点（T10-B7、
   T10-B1、T8、T7、T1、DEPLOY-AUTH、SCENE-CRYSTALLIZATION、
   TAVERN-IMPORT、I18N-CENTRALIZED、批次 S）、安全边界关键词（RLS、
   column-level、append-only、fail-closed、contract-only-deferred）；
3. 新章节不出现 `GRANT ALL`；0013–0023 SQL 全文不出现 `GRANT ALL`
   （realm_runtime 面无全特权授权）；
4. 新章节提及 propagation/semantic-conflict 的行必须含
   contract-only-deferred（不得写成 active runtime）；
5. 关键 SQL 内容锚点（0021 的 GRANT DELETE ON worlds、0023 的
   列级 UPDATE、0014 的列级 INSERT）在位——证明 SQL 未被本批改写
   （内容锚定，不用 hash 伪证据）。

## 五、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 章节缺失锚点（迁移/批次/边界关键词） | 契约测试失败 |
| F2 | 章节或 T 系列 SQL 出现 GRANT ALL | 契约测试失败 |
| F3 | propagation/semantic-conflict 被写成已接线 | 契约测试失败（逐行 contract-only-deferred 断言） |
| F4 | 本批误改 migration SQL | 内容锚点失败 + git diff 名单核对 |
| F5 | 把文档记录宣称成新增授权 | 验收红线；STATUS 只记 provenance 落档 |

## 六、验收标准（本批）

- 扩展后 postgres-schema-contract、documentation-layout 原始 exit 0；
  typecheck、受影响 eslint、git diff --check exit 0；
  `git diff --name-only dc998e3..HEAD` 仅含本批 docs/test 文件（migration
  diff 为空）；不跑 GUI/PG/全量；零 DB 写入；realm-dev.service 保持
  active；工作树 clean。

## 七、范围外清单（本批不做）

任何 SQL/授权/RLS/trigger/role 改动、活动代码改动、迁移执行、
propagation/semantic-conflict 接线、GUI/PG/全量回归、历史批次文档改写。

## 八、交付步骤

1. 本规范 + docs 索引（独立 commit）；2. 契约文档章节 + 契约测试扩展
   （独立 commit）；3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
