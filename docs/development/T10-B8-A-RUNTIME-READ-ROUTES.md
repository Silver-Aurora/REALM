# T10-B8-A · 残余 owner-pool 读路由统一下沉

> 批次 T10-B8-A（docs/development/EXPERIENCE-ITERATION.md T10-B7 收口遗留：files/[id]、settings/language、auth/me 三条 owner-pool 读/轻写路径）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=74d16b6；不改迁移 0001–0023、不新增迁移（授权面已齐，证据见 §二）、不改 LM Studio/T6–T10-B7；不接 propagation/semantic-conflict/图谱刷新/D1 删除。

## 一、逐路由现状与读写面（读代码确认）

| 路由 | 现状 | 读/写面 | 门禁 |
|---|---|---|---|
| `GET /api/files/[id]` | owner pool（DATABASE_URL，每请求建池） | world_files SELECT + memberships EXISTS 成员校验；file id 形态校验；immutable cache | 会话 principal 须为文件所在世界成员（404/403） |
| `POST /api/settings/language` | owner pool（每请求建池） | accounts.ui_language 列级 UPDATE（normalizeUiLanguage 规整 zh-CN/en/ja） | 会话 principal |
| `GET /api/auth/me` | owner pool（每请求建池，best-effort） | accounts SELECT（displayName/ui_language 反查） | gate 关闭→本地单用户应答；开启→session principal；反查失败不阻塞 |

三条都不从客户端接收 workspace/principal（principal 只来自会话解析）。

## 二、授权面实查（realm_dev 当次实测，无新增迁移）

- `has_table_privilege('realm_runtime','world_files','SELECT') = true`（0016）；
- `has_column_privilege('realm_runtime','accounts','ui_language','UPDATE') = true`（0015）；
- accounts 列级 SELECT 含 display_name/ui_language/principal_id（0012/0015）。
- 结论：**本批零迁移**；任何授权缺口才允许新增最小迁移——本次实测无缺口。

## 三、统一下沉契约

- 三路由统一 `getSharedRuntimePool(process.env.REALM_RUNTIME_DATABASE_URL)`；
  不再引用 `process.env.DATABASE_URL` / `createLocalPostgresPool` / owner pool；
  共享池进程级复用、测试经 `endSharedRuntimePools()` 回收。
- 缺 runtime URL：files/[id] 与 settings/language → 503 安全文案
  （`LOCAL_RUNTIME_NOT_INITIALIZED`）；auth/me → 既有多档语义：gate 关闭照常
  本地应答（不触 DB）；gate 开启且缺 runtime URL → 身份应答保留
  principalId，displayName/uiLanguage 回落 null/默认（best-effort 语义不变，
  不因可选账户读取失败泄漏数据库错误或 token）。
- 错误语义不变：files 404 形态/403 越权、immutable cache header、id 形态
  校验；language 非法值回落默认中文；auth/me 401 门禁。
- RLS：所有读取经 `withWorkspaceTransaction` workspace scope；runtime 角色
  只读 world_files/accounts 限定列，不扩大跨世界访问。
- 日志/响应不回显连接串、错误 message、token。

## 四、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 缺 REALM_RUNTIME_DATABASE_URL（files/language） | 503 安全文案 |
| F2 | auth/me 缺 runtime URL 或账户读取失败 | gate 语义不变，best-effort 回落 |
| F3 | 非成员读文件 | 403 照旧；未知 id 404 |
| F4 | 语言非法值 | 回落 zh-CN（既有） |
| F5 | 客户端伪造 workspace/principal | 无入口（服务端常量 + 会话解析） |

## 五、验收标准

1. 静态契约（api-core-wiring 扩展）：三路由只用 runtime URL + 共享池，
   零 owner 引用。
2. 临时 PG（库名 realm_t10b8_*，t.after 强制 DROP）：文件成员隔离 403/404 +
   响应头 immutable；语言写回 accounts.ui_language；auth/me gate 开/关两态 +
   反查成功/失败回落。
3. auth-session/既有 wiring 回归；typecheck/受影响 eslint/文档布局/
   git diff --check exit 0；不跑全量/GUI。
4. 开发库基线不变；本批临时库全清。

## 六、不在本批范围

library/world-generate/import（T10-B7 已收口）、propagation、
semantic-conflict、图谱刷新、D1 删除、全局授权文档（POSTGRESQL-RUNTIME-
CONTRACT 增补）。

## 七、交付步骤

1. 本规范（单独 commit）；2. 三路由下沉 + 契约 + 测试（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
