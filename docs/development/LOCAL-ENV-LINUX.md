# 界核 / REALM Linux 本机环境说明

> 文档性质：Linux（Arch 系）本机开发环境搭建与运行说明。不含任何密钥；
> 密钥只存在于 Git 忽略的本机配置（`.env.local` 与 `.local/settings/model-provider.json`）。
> 进度以根目录 [`STATUS.md`](../../STATUS.md) 为准。

## 1. 前提

- Node.js ≥ 22.13（本项目实测 Node 25.9）
- PostgreSQL 17 + pgvector
- Linux 发行版需自行提供 PG 17 二进制（Arch 的 `postgresql` 包当前为 18，
  项目脚本默认查找 `/opt/homebrew/opt/postgresql@17/bin`，可用
  `REALM_POSTGRES_BIN` 覆盖；推荐直接用下方 Docker 方案，无需系统 PG 二进制）。

## 2. 数据库（推荐 Docker）

```bash
docker run -d --name realm-pg17 --restart unless-stopped \
  --network host \
  -e POSTGRES_HOST_AUTH_METHOD=trust \
  pgvector/pgvector:pg17 \
  -c port=55432 -c listen_addresses=127.0.0.1
```

**必须 `--network host` + 显式回环监听**：项目的迁移/运行脚本会校验
`inet_server_addr()` 必须为 `127.0.0.1`/`::1`（拒绝外部数据库）。
docker 桥接端口映射会让容器内看到的连接源地址变成网桥地址（172.x），
触发 "Refusing to migrate a PostgreSQL server outside loopback" 拒绝；
host 网络模式下容器直接监听宿主回环，校验通过。

初始化：

```bash
psql postgresql://postgres@127.0.0.1:55432/postgres -c "CREATE DATABASE realm_dev;"
cd <项目根目录>
cp .env.example .env.local
# 编辑 .env.local：DATABASE_URL 使用 postgresql://postgres@127.0.0.1:55432/realm_dev
# （迁移/初始化管理连接用 postgres 超级用户；REALM_RUNTIME_DATABASE_URL 保持 realm_runtime 最小权限）
npm run db:postgres:migrate
npm run db:postgres:seed
```

迁移会创建 `realm_runtime`/`realm_control` 角色并应用 0001–0008，
权威表 30 张 + 迁移账本，pgvector 可用。

## 3. 模型配置（LAN 本地推理）

当前默认供应商是本机 LM Studio（OpenAI-compatible），无需 API Key。
复制 `.env.example` 到 `.env.local` 后默认值即可用：

```bash
REALM_MODEL_PROVIDER=lmstudio
REALM_MODEL_ID=unsloth/gemma-4-12b-it-qat
REALM_MODEL_BASE_URL=http://127.0.0.1:1234/v1   # 必须含 /v1；可改为自己的 LAN 私有地址
REALM_MODEL_API_KEY=   # 可留空；留空时请求不带 Authorization 头
```

- LM Studio 端点遵循 local 策略：仅允许 loopback 与 RFC1918/ULA/link-local
  私有地址及 localhost 主机名（路径钉定 `/v1`）；公网地址会被
  `MODEL_ENDPOINT_NOT_ALLOWED` 拒绝（SSRF 防护，本机受保护语义）。
- 网关对 max_tokens 强制 1024 下限（Gemma 推理会先吃 token 预算，
  小预算会得到空 content）；不发送供应商专属 thinking 字段。
- 覆盖默认（模型/端点/超时/thinking 标记）可在设置页操作或写
  `.local/settings/model-provider.json`（`.local/` 已被 Git 忽略，权限 600）。

### 3.1 ComfyUI 图像生成设置（operator 专属）

- ComfyUI 设置页/API 是全局 operator 能力：在 `.env.local` 配置
  `REALM_OPERATOR_PRINCIPALS=principal_xxx,principal_yyy`（逗号分隔的
  精确 principal ID；空缺 = 无 operator，所有 `/api/settings/comfyui`
  请求 fail-closed 403）。principal ID 可用浏览器 devtools 从
  `realm_session` cookie 的载荷段（`.` 前第一段）读出，或查 accounts 表。
- ComfyUI endpoint 只允许 loopback 与 RFC1918/IPv6 ULA 地址（link-local/
  公网拒绝）；hostname 在连接时解析全集校验并 pin 到已验证地址；
  redirect 不跟随；详见 `docs/development/IMAGE-GENERATION-STAGE-1-2.md`
  §3.1。

## 4. 启动开发服务

```bash
npx vinext dev --hostname 127.0.0.1
```

**不要跑 `npm run dev`**：其 `db:postgres:bootstrap` 会先执行
`db:postgres:start`，而脚本的 `start` 在检测到 55432 已有外部
PostgreSQL（如 Docker 容器）时按设计拒绝并报错。数据库已由
容器/系统 PG 提供时，直接 `npx vinext dev` 即可。

服务默认监听 `http://127.0.0.1:9999`（仅回环）。

## 5. 验证与测试

```bash
npm run typecheck
npm test                    # 既有自动化（typecheck + core + contracts + build + render）
npm run lint
npm run db:postgres:status  # 容器方案下会报找不到 pg_ctl（预期），用 docker ps 检查容器

# GUI 交互测试（需要 dev server 已启动或允许 Playwright 自动拉起）
npx playwright test                     # Chromium + WebKit 双引擎全量
npx playwright test --project=chromium  # 仅 Chromium（主测）
npx playwright test --project=webkit    # 仅 WebKit（Safari 引擎近似）
```

GUI 测试详见 [`GUI-TEST-PLAN.md`](./GUI-TEST-PLAN.md) 与
[`GUI-TEST-REPORT.md`](./GUI-TEST-REPORT.md)。

## 6. WebKit（Safari 引擎近似）运行补丁

Playwright 官方不支持 Arch Linux 的 WebKit。本机实测解决方式：

1. 从 Ubuntu 24.04 官方仓库提取 WebKit 运行所需动态库
   （`libxml2.so.2` / `libicu74` / `libflite` 等）到 `~/.cache/webkit-deps`
2. 软链进 `~/.cache/ms-playwright/webkit-*/minibrowser-{gtk,wpe}/sys/lib`

这是**本机浏览器运行环境补丁**，不属于仓库内容；在受支持系统
（macOS / Ubuntu）上无需此步骤。补丁细节见
`tests/gui/README.md` 的 "WebKit 环境说明"。

## 7. 停止数据库

```bash
docker rm -f realm-pg17   # 数据在容器内，删除即清空；如需保留数据请先 dump
```
