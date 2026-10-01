# REALM Transfer 迁移与 provisioning（v37：0042/0043 + realm_transfer）

> 规范基线：`/tmp/realm-world-export-parallel-write-plan-v37.md`（sha256
> 0880bc17…1f72）§C/§E.2/§F。本文档只描述落地后的运维事实；设计与审计
> 推演以计划为准。

## 角色与信任边界

- `realm_transfer`：cluster-global LOGIN 角色，`.realm` 导出/导入的唯一
  写入面 = 13 个受控 SECURITY DEFINER 函数的 EXECUTE；对全部表**无任何
  直接 INSERT/UPDATE/DELETE**（仅 42 内容表 + 5 ledger 的 SELECT）。
- **信任边界（不可降级）**：realm_transfer 凭据本身是信任边界——凭据
  失守即越界。DB 函数不做 principal 认证；`realm_current_workspace_id()`
  只是 caller 可设置的一致性等式 GUC。principal 认证在 route 会话层
  （session cookie → 验证后 principal 调 `realm_import_job_create` 一次 →
  `job.operator_principal` 落库并被不可变列守卫冻结）。
- 角色生命周期：**只能由 provisioning 创建/管理 LOGIN 与凭据**；
  migration 永不触碰 LOGIN/NOLOGIN/PASSWORD（0042 全文关键词零命中的
  静态锚点由测试锁定）。

## provisioning（provision → migrate 强制顺序）

```bash
REALM_TRANSFER_PASSWORD=<env-only> npm run db:postgres:migrate # 先 provision：
node scripts/provision-realm-transfer.mjs   # 在执行 0042 之前运行一次
```

`scripts/provision-realm-transfer.mjs` 执行序（§E.2）：
1. 预检：`shared_preload_libraries` 含审计库（pgaudit/pg_stat_statements）
   或扩展已存在 → 整项 no-go；
2. session gate：`log_statement='none'`、`log_min_duration_statement=-1`、
   `log_min_error_statement='PANIC'`（任一失败即中止，绝不执行密码 DDL）；
3. 同事务紧邻：角色缺席 `CREATE ROLE … LOGIN <负向属性>`，已存在
   `ALTER ROLE … WITH LOGIN <负向属性>`（NOLOGIN 残留收敛回 LOGIN），
   密码 `ALTER ROLE … PASSWORD '<escaped>'`（escapeLiteral 纯函数 `'`→`''`；
   PG 工具语句无 bind 参数）；失败整体回滚非零退出；
4. finally 恢复三项 GUC（session 级 SET 随连接销毁）。

**威胁模型（受信单操作者窗口例外）**：`ALTER ROLE … PASSWORD '<literal>'`
执行窗口内同实例其他 superuser 可经 `pg_stat_activity` 读到 query 文本——
本地单操作者、实例空闲时执行；部署策略不接受该窗口则整项 no-go。

0042 在角色缺席或 NOLOGIN 残留时 `TRANSFER_NOT_PROVISIONED` fail-closed
（无顺序无关性）；双向 `pg_auth_members` 异常同样拒绝。

## 0042/0043 语义

- **0042** `realm_transfer_and_import_jobs`：编码前置（UTF8 断言）→
  角色块 → 0028 capability 中间态硬化（fail-closed body +
  `search_path = pg_catalog, public, pg_temp` + 四主体 REVOKE EXECUTE）→
  五 ledger 表（jobs/events/bootstrap/content_log/pack_tables）→
  守卫函数×4 + helper×4 + 受控函数×13 → owner gate → RLS 五表
  ENABLE+FORCE → REVOKE schema CREATE/库 TEMPORARY（四主体）→
  表级+列级 lockdown（`ON TABLE public.%I`）→ GRANT SELECT/EXECUTE →
  提交前 ACL gate（闭包/PUBLIC grantee=0/23 签名四层检查/default ACL）。
- **0043** `propagation_node_audience_archived_guard`：capability 最终
  body（0028 全部检查逐字保留 + `worlds FOR KEY SHARE` + active 断言）
  + 恢复 realm_runtime EXECUTE + 目标表 mutation 闭合 + 提交前 ACL gate
  （与 0042 同构 + 终态形态断言）。
- 两文件均不含事务控制语句（runner per-file transaction 包裹）；
  forward-only；0028 文件与台账行零改动。

## drain/admission（C4，0042→0043 窗口）

- capability 调用方（route `app/api/propagation/node-audiences/route.ts`
  与 CLI）在调用事务内先 `SELECT pg_advisory_xact_lock(7200043)`——锁随
  事务持有至 commit/rollback，覆盖 active/idle-in-transaction/驱动改写。
- migration runner 应用 0043 时先以 `lock_timeout='30s'` 持
  `pg_advisory_lock(7200043)`——**阻塞即在途 → no-go 不执行**；持锁期间
  新 admission 阻塞；0043 提交后解锁，被阻塞调用以新 catalog 继续。
- runner 持锁期间崩溃 → session 级锁随连接释放（无残留）。
- PG 默认 `max_prepared_transactions=0`——prepared 事务态在本部署不可达
  （query-text 探针盲区天然不存在）。
- 中间态（0042 已提交/0043 未提交）capability 完全不可调用：临时 body
  RAISE + EXECUTE 撤销双重兜底。

## 读回与失败恢复

- 发布闸门 §C ⑨⑩ 不变：0042 失败 → 事务回滚，读回旧 body/path/ACL 后
  才允许重启 realm-dev；0043 失败 → 中间态保持，**禁止启动 realm-dev**，
  诊断后重跑 0043；0043 终态读回须全量（body/runtime EXECUTE/23 函数
  ACL/表 ACL/owner/闭包/default ACL）。
- 应用与重启：`npm run db:postgres:migrate`；只重启 `realm-dev.service`；
  `realm-propagation-worker.service` 不重启不修改；bind/systemd 零改动。

## 测试锚点

- `scripts/test-postgres-runtime-with-scratch.mjs`（`test:postgres-runtime`
  唯一入口）：启动一次性 host-network loopback PG17 scratch cluster
  （动态端口；真实 runner 的 `inet_server_addr` loopback 检查通过——不用
  published-port），只在集群内 provision realm_runtime/realm_control/
  realm_transfer（绝不写共享 realm_dev 或长期实例），realm_dev 建库 +
  0001–0043 全链 + seed demo 后以注入环境变量运行全部 PG 套件，
  结束销毁容器（SIGINT/SIGTERM 同步兜底）。依赖 docker；无 docker 即
  fail（不回退共享库）。
- `tests/helpers/v37-test-cluster.mjs`：scratch cluster 共享 helper。
- `tests/postgres-realm-transfer-migration.test.mjs`：全链真实 runner、
  角色五态（独立 scratch 容器）、读回断言全集、权限边界、中间态、
  受控函数链、TEMP shadow、admission/drain、双侧 digest、静态扫描、
  provisioning 五态、gate 负例矩阵、application_name 可观测。
- `tests/article-qualification-design.test.mjs` v37 静态契约段。
