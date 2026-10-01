# Article Qualification Migration（0041）设计冻结

来源：`/tmp/realm-swm-next-plan-v10.md` §5（Iris 最终审计通过）。本文档是该 migration 的落稿设计；0041 内容经 runner checksum 台账入库后不可再改（改内容 = checksum 不一致 fail-closed），设计冻结必须先于入库。

## 文件与 runner 语义

- 文件固定：`database/postgres/migrations/0041_article_qualification_and_import_entries.sql`（当前最新为 0040，0041 是下一个排序文件，只追加不改旧）。
- 现有 runner 对每个 `.sql` 文件执行独立 **per-file transaction**：`BEGIN` → 执行 SQL → 写 `realm_schema_migrations` 台账 → `COMMIT`，失败 `ROLLBACK`（`scripts/postgres-migrate.mjs:110-145`），前有 session advisory lock，台账行 `FOR UPDATE` 比对 sha256 checksum，不一致 fail-closed。
- 因此 0041 **不包含任何 BEGIN/COMMIT/ROLLBACK**，事务完全由 runner 包裹。
- 0041 内容顺序：`CREATE EXTENSION IF NOT EXISTS pgcrypto;` 先于两表；两表 + 复合 FK + 全部具名 CHECK + append-only 函数/触发器 ×2 + RLS/policy ×2 + grants ×2 + 唯一键全部在同一文件。
- 回滚/兼容：0041 应用失败由 runner 对该文件整体 ROLLBACK；未应用 0041 的库上 feature 读取 fail-closed（两表不存在 → lore 空注入、GET 全部按 pending_review 处理，不接 prompt）；不改旧 `world_articles` 历史数据；共享 `realm_dev` 永不应用 0041（未来部署是另行审批的发布动作），常驻服务不重启。

## 表：`article_qualifications`

Append-only 资格事件账本。latest-state = 每 article `seq` 最大者（应用侧行锁保证 seq 唯一，`id` 字典序双保险 tiebreak）。

- 主键 `(workspace_id, id)`；唯一键 `(workspace_id, world_id, worldline_id, article_id, seq)`。
- 复合 FK `aq_article_fk` → `world_articles (workspace_id, world_id, worldline_id, id)` ON DELETE CASCADE；另有 `aq_workspace_fk` → workspaces。
- CHECK：
  - `aq_provenance_check`：`provenance_kind IN ('canon_generated', 'tavern_import', 'manual', 'owner_attest')`；
  - `aq_status_check`：`status IN ('pending_review', 'qualified_public', 'rejected', 'revoked')`；
  - `aq_qualified_provenance_check`：`qualified_public` 只能来自 `owner_attest`（当前唯一公共资格来源；canon_generated 当前生产 writer = none，仅预留）；
  - `aq_identity_check`：`pending_review` 必须无 attested_by/attested_at，其余状态必须有；
  - `aq_cursor_check`：`available_from_tick/available_from_ordinal >= 0`；
  - `aq_hash_check`：content_hash 非空白。

## 表：`article_import_entries`

Tavern 导入 source identity 持久化，file_exact 唯一模式，schema 级锁死：

- 唯一键 `(workspace_id, world_id, worldline_id, source_kind, source_namespace, stable_entry_identity)`；
- `aie_namespace_check`：`source_namespace = bundle_content_hash`（整文件 sha256 hex，仅精确同文件识别，不承担跨上传身份）；
- `aie_identity_check`：`stable_entry_identity = entry_ordinal::text`；
- `aie_hash_check`：两个 hash 列必须匹配 `^[0-9a-f]{64}$`；
- `aie_source_kind_check`：当前唯一 `'tavern_worldbook'`；`aie_identity_kind_check`：当前唯一 `'file_exact'`；`aie_ordinal_check`：非负；
- `entry_uid` 仅 bundle 内信息列，可为空，不进唯一键；keys/filename 永不作身份；
- 复合 FK `aie_article_fk` → world_articles；另有 `aie_workspace_fk` → workspaces。

## 三 hash 分离（不得混用；content hash 不是安全授权）

- `qualification_content_hash`（→ `article_qualifications.content_hash`）：UTF-8 `article.id + "\n" + title + "\n" + body` 的 sha256 hex；绑定具体 article row，用于 attestation 比对与 lore read-time DB-side match（pgcrypto `encode(digest(..., 'sha256'), 'hex')`）。
- `entry_content_hash`（→ `article_import_entries.entry_content_hash`）：UTF-8 `normalized_name + "\n" + content` 的 sha256 hex；不含随机 article.id，用于同 identity 的 duplicate 检测。
- `bundle_content_hash`（→ `article_import_entries.bundle_content_hash`，file_exact 下兼任 `source_namespace`）：整文件字节 sha256 hex。

## append-only / RLS / grants（两表同形态）

- plpgsql guard 函数 RAISE EXCEPTION + `BEFORE UPDATE OR DELETE` 触发器 ×2；
- `ENABLE` + `FORCE` ROW LEVEL SECURITY + `realm_workspace_isolation` policy（USING/WITH CHECK 双写 `realm_current_workspace_id()`）；
- realm_runtime 最小授权：`GRANT SELECT, INSERT`（无 UPDATE/DELETE/ALL）。

## 资格并发锁序（同事务，固定顺序防死锁）

1. `SELECT ... FROM worldlines ... FOR UPDATE`——锁 worldline 行并读 `(head_tick, head_ordinal)` 作为 `available_from` tuple；
2. `SELECT ... FROM world_articles ... FOR UPDATE`——article 行锁（同时读出 DB-side 当前 content hash）；
3. `SELECT COALESCE(MAX(seq), 0) + 1 ...`；
4. INSERT 新 event（`aq_` 前缀 id）。

锁顺序固定 worldline → article，绝不反向；撞 UNIQUE 即回滚（兜底断言）。

## prompt 资格谓词（G4 接线；全部 AND，无旁路）

```text
article.worldline_id = 当前 worldline
AND latest_qualification.status = 'qualified_public'
AND latest_qualification.provenance_kind = 'owner_attest'
AND latest_qualification.content_hash
    = encode(digest(article.id || E'\n' || article.title || E'\n' || article.body, 'sha256'), 'hex')
AND (record_head_tick, record_head_ordinal)
    >= (available_from_tick, available_from_ordinal)
AND article.claim_ids = '{}'
AND 预算（≤2 条、body ≤600、title ≤80、渲染块 ≤1200）
```

record head tuple = `(COALESCE(record_heads.last_world_tick, record.start_tick), COALESCE(record_heads.last_world_ordinal, record.start_ordinal))`；不 retroactive；revoke 立即全隐藏。

## import 并发（SAVEPOINT-per-entry，无孤儿 candidate）

整个 bundle 导入是一个事务；逐 entry：`SAVEPOINT entry_step` → 插入 candidate article + pending qualification（seq=1，available_from 默认 0）→ `INSERT INTO article_import_entries ... ON CONFLICT (唯一键全列) DO NOTHING RETURNING id`；RETURNING 为空即冲突 → `ROLLBACK TO SAVEPOINT entry_step` 撤销 candidate（无孤儿）→ 同事务 readback 既有 entry 比对 `entry_content_hash` → 计入 `duplicates[]`；无冲突 `RELEASE SAVEPOINT`。任一真正错误整体 ROLLBACK；并发双导入同一文件最终只有一组 article/entry/qualification。

## revoke / re-attest / duplicate

- revoke：追加 `status='revoked'` event（available_from 沿用前次）；latest=revoked → 任何 cursor 不注入；
- re-attest：追加新 `qualified_public` event（新 seq、新 available_from tuple）；
- duplicate：同 `(articleId, decision, content_hash)` 且 latest 同 status → 幂等返回，不追加 event。

## 能力边界（诚实声明）

- 本阶段只有「精确同一文件」的重复识别；无法识别跨上传更新/content_changed；无 uid 的 entry 报告 `identity_unstable`。
- 无资格行 = `pending_review`（fail-closed）。
- restricted/secret lore 通道不存在；`source_event_ids` legacy 不参与资格；entry 内混合权限本阶段不支持；lore 不进 scene crystallization；模型不参与授权；消费者 opt-in 默认 false。
- pgcrypto 是唯一 hash 路径：扩展不可用即 no-go（G0 已实证可用），无应用层 fallback。
