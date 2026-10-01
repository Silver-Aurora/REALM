# 实施规范：部署形态与身份鉴权

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 视觉规范：[`../design/UI-DESIGN.md`](../design/UI-DESIGN.md)。
> 开发边界：本项目保持本机与内部局域网使用；本批次不改变这一定位，只把端口/绑定/鉴权变成显式可配置。

## 1. 端口与绑定

- 服务端口统一为 **9999**：`scripts/dev-server.mjs` 是唯一启动入口，读取 `PORT`（默认 9999）与 `HOST_BIND`（默认 `127.0.0.1`），拼装 `vinext dev|start --hostname --port`。
- `vite.config.ts` 的 `host` 同样读 `process.env.HOST_BIND ?? "127.0.0.1"`。
- 默认回环是安全基线；只有显式设置 `HOST_BIND=0.0.0.0` 或具体内网 IP 才放开监听。
- package.json 的 `dev` / `start` 脚本改为调用该入口；文档（README、LOCAL-ENV-LINUX、GUI-TEST-PLAN）、playwright.config.ts、tests/gui 内 baseURL、systemd 用户服务单元全部同步到 9999。
- `tests/local-only-boundary.test.mjs` 继续守卫边界：断言默认回环值存在、env 可覆盖、无外部托管痕迹。

## 2. 数据模型（迁移 0012_accounts.sql）

```text
accounts
  principal_id  text PK        -- principal_<sha256(display_name) 前 18 位>
  display_name  text UNIQUE    -- 昵称即身份，无注册审批
  created_at    timestamptz
```

RLS 与 `realm_runtime` 授权同既有表。不存密码、不存 token、不存任何凭据。

## 3. 鉴权流程（2026-09 账户名+可选密码改版）

- 门禁开关：`REALM_RUNTIME_DATABASE_URL` 存在 → 启用（账户登录需要 accounts 表）；不存在 → 全部回落既有单用户行为（`LOCAL_RECORD_SCOPE.principalId`），纯单机/测试零变化。**`REALM_ACCESS_TOKEN` 已退役**：存在也仅被忽略，绝不再作为凭据或门禁前置。
- 登录：`POST /api/auth/login { displayName, password }`
  - 账户名即既有 display_name（1–40 字符，中文/自然昵称照旧）；
  - 新账户首次登录可写密码（空密码 = 无密码账户）；既有无密码账户仅空密码通过——非空密码拒绝且**绝不**在登录时静默设置/覆盖密码；有密码账户严格校验；
  - 密码哈希：`scrypt$N$r$p$saltHex$hashHex`（Node 内置 crypto.scrypt，每账户随机 salt，timingSafeEqual 校验；0051 迁移，accounts.password_hash 可空，默认 NULL）；
  - 所有登录失败统一 `INVALID_CREDENTIALS`——不泄露账户是否存在/是否设密/hash；
  - 签发 httpOnly cookie `realm_session`：`principalId.expiresAtMs.hmacSha256`，30 天，SameSite=Lax，path=/；签名密钥取 `REALM_SESSION_SECRET`，缺省用安装级 0600 密钥文件（`<dataHome>/secure/session-secret`，一次性生成后稳定），进程重启会话不失效；开发兜底常量仅限非 production（文件系统不可写时告警一次）——`NODE_ENV=production` 下未配置 env 且安装级密钥不可用/创建失败时，会话签名路径抛 `SessionSecretUnavailableError` fail-closed，绝不返回/缓存兜底、不产生可用 cookie；
  - 凭据不落库（只落 hash）、不进日志、不进任何摘要/哈希存证（principal id 是昵称哈希，不含密码材料）。
- 密码管理（已登录）：`POST /api/auth/password`（当前会话 principal）——设置/修改（有密码须校验旧密码）；明确清除需 `confirmClear:true` + 校验当前密码。
- 登出：`POST /api/auth/logout` 清除 cookie。
- 解析：`principalFromRequest(request)` 校验 cookie 签名与过期；所有 API 路由先过 `requireApiPrincipal(request)`——门禁启用且无有效会话 → 401 JSON；否则返回「会话 principal 或本地回落 principal」。
- 页面门禁：服务端组件（`/`、`/settings`）用 `headers()` 读 cookie；无会话 → `redirect("/login")`。`/login` 与 `/api/auth/*` 是仅有的公开入口。
- 服务层贯通：`loadRecord / submitMessage / listCommittedEvents` 接受可选 principalId；`resolveRuntimeScope`、`writeScope`、投影读取一律使用请求 principal，替代硬编码 `LOCAL_RECORD_SCOPE.principalId`。

## 3.1 登录即加入默认世界

- 问题背景：新身份首次登录只创建 accounts 记录，`player_world_memberships` 无任何 membership，投影加载报 `LOCAL_RUNTIME_NOT_INITIALIZED`（"暂时找不到这段记录"）。
- 原方案（用户 2026-08-15 拍板）：登录成功路径自动确保默认世界 membership，最初赋 `role='owner'`。**安全修正（2026-09-28）**：新建 membership 现在默认 `role='player'`，不再把登录当作 operator 授权；workspace/world 常量、`omniscient_player_character=true`、`can_view_dynamic_knowledge=true` 不变。`ON CONFLICT DO NOTHING` 保留现有明确 role，包含旧版本产生的 owner 行；本次不对共享数据批量降权。创建者新建自己的世界仍由 `createGenesis` 赋 owner。
- 实现：`AccountRepository.ensureDefaultWorldMembership(workspaceId, principalId)`，`INSERT ... ON CONFLICT (workspace_id, world_id, principal_id) DO NOTHING`——重复登录幂等，不覆盖既有 membership。
- 权限：迁移 0013 授予 `realm_runtime` 对 `player_world_memberships` 的 INSERT（0004 硬化时该表只有 SELECT）。仍不授 UPDATE/DELETE；omniscience 不可变触发器姿态不变。
- 未来再做"创建新世界"入口；当前所有登录身份自动进入 demo 世界并可见全部 demo 数据。

## 4. 登录页视觉

- 纸墨风格：纸张底 + 直角卡片（1px 深边线、3px 硬阴影）、朱红印章方块、Serif 标题、Sans 表单。
- 字段：账户名 + 密码（可选，password 输入）；一个主动作「进入世界」。
- 禁圆角、禁半透明、禁荧光色；错误提示用浅朱红底 + 深朱红字 + 1px 边线（既有错误语义）。

## 5. 端口/bind 迁移清单

| 文件 | 变更 |
|---|---|
| `scripts/dev-server.mjs` | 新增：env 驱动启动入口 |
| `package.json` | dev/start 改调入口；端口 9999 |
| `vite.config.ts` | host 读 env 默认回环 |
| `playwright.config.ts` | baseURL/webServer → 9999 |
| `tests/gui/*` | baseURL 引用 → 9999 |
| `README.md`、`docs/development/LOCAL-ENV-LINUX.md`、`GUI-TEST-PLAN.md` | 端口与 HOST_BIND/PORT 说明 |
| `~/.config/systemd/user/realm-dev.service` | ExecStart 改入口、PORT=9999（不进 git） |
| `tests/local-only-boundary.test.mjs` | 断言默认回环 + env 覆盖 + 9999 默认 |

## 6. 验收标准

1. 默认启动监听 127.0.0.1:9999；`HOST_BIND=0.0.0.0` 时监听放开；边界守卫测试通过。
2. 无 runtime DB：全部既有单用户行为不变（GUI 全绿即证明）；REALM_ACCESS_TOKEN 残留不影响登录。
3. 配置后：无会话访问 `/` 与 `/settings` → 302 到 /login；API → 401；错误 token → 401；正确 token + 昵称 → 200 + httpOnly cookie；`/api/auth/me` 返回当前 principal。
4. accounts 表查/建幂等（同昵称同 principal）；session 签名篡改/过期被拒绝。
5. 登录页通过设计语言断言（直角、语义色、无半透明/荧光）。
6. `npm test` 全绿；GUI a-library + k-graph 不回归（baseURL 9999）。

## 7. 明确不做

- 多因素/注册审批（账户名即身份，密码可选）。
- 公网安全宣称（当前仅本机/LAN 用途）。
- 角色级权限矩阵（本批次只有门禁 + 身份感知）。
- 会话撤销列表（HMAC 无状态会话；过期即失效）。
