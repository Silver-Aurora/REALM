# Tauri 桌面客户端 + Android client-only APK 实施计划（TAURI-CLIENT-APK-PLAN）

- 基线：HEAD `30f80fc`（main，工作树 clean）。本文是实施计划 + 持续更新的完成证据。
- 形态目标：桌面 = Tauri 2 客户端（启动/管理既有本地服务；用户选「打开网页」或「打开客户端」）；Android = client-only APK（不捆绑 PG/Node/服务端/凭据，连接用户配置的服务地址）。三端共用同一份 Web 前端与服务。

## 一、现状 / 目标 / 非目标

### 现状（已审计）
- Web 应用：vinext + React/TS，全部 API 走**同源相对路径 fetch**（`app/realm-client.tsx:188` 等）；登录 = `REALM_ACCESS_TOKEN` 门禁 + `realm_session` cookie（`HttpOnly; SameSite=Lax`，`modules/identity/auth.ts:88-89`）；未设门禁时回落本地单用户 principal（`app/api/auth-context.ts`）。
- 本地服务编排已收敛在 `launcher/realm-launcher.mjs` 的可导入 `startRealm(options)`：单实例锁（wx 原子 + stale 回收）、loopback 端口挑选、child env 白名单（`safeParentEnv`，不透出密钥）、PG start → provision → migrations → seed → app start → health → 可选开浏览器、优雅 stop。注释中的「首发不做 Tauri/Electron」将由本批有意识地改为 Tauri 协同。
- 前端无硬编码 API base URL——「集中处理 origin」的落点因此是：**两端 WebView 都直接加载服务 URL**（同源天然成立），不需要改写前端 fetch。
- 构建门：`npm run typecheck/lint/build/test`；Linux 主机有 cargo 1.98 + cargo-tauri 2.11.4 + webkit2gtk-4.1（2.52.6）/gtk+-3.0/libsoup-3.0，可真实 `cargo check/test/build`。
- 无 `src-tauri`、无 Android scaffold；无 java/adb/ANDROID_HOME（磁盘余量 685G）。

### 目标
1. Tauri 桌面客户端：启动可见状态机（启动中→就绪→失败/重试→退出），用户选择「打开网页」（系统浏览器）或「打开客户端」（WebView 加载同一服务 URL），单实例与优雅关闭语义保留。
2. Android APK：client-only；服务地址配置（协议/host/凭据校验）+ 连接检查 + 登录/错误/重试；绝不捆绑 PG/Node/server/key。
3. 自适应：375/412/768/1024/1440 无横向溢出，键盘+触摸可用，保持纸墨直角。

### 非目标
- 不做签名发布（Windows Authenticode / Apple notarization / Play 签名）；Linux 之外的真机构建验收不宣称完成。
- 不改 Record/Event/RLS/锁序/模型链/worker 语义；不启动 propagation worker。
- 不做公网隧道/公网监听；不做第二套移动 UI 或业务页面 fork。
- 不把 `.env`、数据库、日志、provider key、token、连接串打入任何包。

## 二、桌面启动状态机

```
idle ──start──▶ starting(step: lock → postgres → provision → migrate → seed → server → health)
starting ──ready(url)──▶ ready ──用户二选一──▶ client-mode（WebView 导航到 url）
                              │                └ web-mode（系统浏览器打开 url，Tauri 窗口转「服务运行中」面板）
starting ──error(msg)──▶ failed（可见错误 + 重试按钮 + 退出）
failed ──重试──▶ starting
ready/client/web ──停止或关窗──▶ stopping（SIGTERM → host 优雅停 PG/服务/放锁）──▶ exited
已有实例（锁被占）──▶ failed(kind=already-running，文案明确，给重试)
```

- 进程协议：新增 `launcher/realm-service-host.mjs`（复用 `startRealm`，零重复逻辑），stdout 输出 JSON-lines：`step / ready{url,appPort,pgPort,dataHome} / error{message} / stopped`；SIGTERM → `stop()`。绝不打印连接串/密码。
- Rust controller（`src-tauri/src/lib.rs`）：spawn host → 逐行解析 → `ServiceState`（Mutex）→ `service_state` command + `state-changed` event；命令：`service_start / service_stop / open_in_browser(url, 校验 http(s)+loopback 或 LAN) / host_kind(desktop|android)`。
- Tauri 自身窗口关闭 = 触发 `service_stop` 后退出（可见 stopping，不静默杀进程）。

## 三、运行时边界

- **桌面（dev）**：Tauri app spawn 系统 `node launcher/realm-service-host.mjs`（cwd=repo 根）；app 目录=repo，数据目录=launcher 既有规则（Linux `~/.local/share/REALM`，可被 `REALM_DATA_HOME` 覆盖）。
- **桌面（prod bundle）**：资源布局 `resources/launcher/*.mjs + resources/scripts/*.mjs + resources/app/**`（生产构建产物）+ bundled Node sidecar（后续批次的打包白名单沿用 `installer/windows/inventory.json` 模式）；本批先把 resource 协议与 dev/prod 路径解析做实，Linux release binary 可跑 dev-equivalent 路径（appDir=repo 或 REALM_HOME 指定目录）。
- **Android**：APK 内只有 Tauri shell + 配置页；无 Node/PG/server；服务地址存 WebView localStorage（非敏感）；登录凭据交给既有 REALM 登录页，不落地、不入日志。
- 前端 origin：两端 WebView 均加载 `http(s)://host/`，同源相对 fetch 不变；**不集中改写前端 API**（无硬编码 base URL 可改）。新增一个集中的 origin 校验函数（Rust + JS 双端同规则）：仅 `http/https`、非空 host、禁止 userinfo/query/fragment、禁止 file:/tauri: scheme。

## 四、LAN 访问安全方案（显式、可关、仅可信内网）

- 默认保持 loopback：launcher child env 恒 `HOST_BIND=127.0.0.1`。
- LAN 模式为**显式 opt-in**：`realm-launcher.mjs --lan <具体网卡 IP>`（或 `REALM_LAN_BIND`）；只允许用户显式给出的 IPv4 地址，**不硬编码 0.0.0.0**（用户显式传 0.0.0.0 时接受但打印醒目警告文案）。
- LAN 模式强制门禁：`REALM_ACCESS_TOKEN` 必须已设置且非空，否则 fail-closed（拒绝启动并给出明确错误）；会话 cookie 已是 `HttpOnly; SameSite=Lax`（跨站 POST 不携带 cookie，CSRF 面天然受限）；路由不输出任何 CORS 允许头（跨域读默认被浏览器拦）。新增静态契约测试钉住：无 `Access-Control-Allow-Origin: *`、cookie 属性不变、LAN 缺 token 拒绝启动。
- 不把 LAN URL/监听地址写进包或日志的敏感段；文档只给占位示例 `[REDACTED-LAN-IP]`。

## 五、依赖 / ABI / 构建与签名策略

- 桌面：Tauri 2.11；Linux 构建 `cargo tauri build`（debug+release 二进制；bundle 的 deb/AppImage 视系统依赖而定，非目标）；Windows/macOS 打包走后续批次（既有 installer 流程不受本批影响）。
- Android：`cargo tauri android init`（gen 目录入库白名单评审后决定——gen 为生成物，默认**不入库**，CI/本地按需再生成）；目标 ABI `aarch64`（arm64 优先），minSdk 按 tauri 默认（24+）；签名：debug APK（不伪造发布签名；发布 keystore 属用户资产，文档说明）。
- Android toolchain 审计：JDK（Temurin 17 tarball，GPLv2+CE）、cmdline-tools（dl.google.com 官方 zip）、sdkmanager 装 platform-34/build-tools/platform-tools/NDK（Android SDK License）；全部 user-local（`~/.local/`），不用 sudo；任何一步失败如实标 blocked，不伪造 APK。
- 许可证/NOTICE：沿用 `installer/windows` 的清单范式，Tauri 依赖经 `cargo tree` 审计。

## 六、验证矩阵（可执行）

| 项 | 命令/方式 | 通过判据 |
| --- | --- | --- |
| 计划审计 | 本文件 + review | 覆盖全部必读点 |
| 前端门 | `npm run typecheck/lint/build` | exit 0 |
| 契约 | 新增 `tests/tauri-desktop-contract.test.mjs` + LAN 安全契约 | exit 0 |
| Rust | `cargo check/test`（src-tauri） | exit 0 |
| 服务 smoke | 一次性 REALM_DATA_HOME + host 直接运行 | JSON 协议 step→ready→GET / 200→stopped，exit 0 |
| 桌面 smoke | `cargo tauri dev` 无头不可行 → host 级 smoke + Rust 单测替代；WebView 分支用构建产物人工/契约核验 | 如实记录 |
| 自适应 | Playwright Chromium 375/412/768/1024/1440 | 主路径 scrollWidth≤innerWidth+1，键盘/触摸交互可用 |
| Android | `cargo tauri android build --apk`（若工具链）+ APK 内容审计（无 .so server/无 PG/无 node） | 否则 blocked + 原因 |
| 敏感扫描 | grep token/key/连接串形态 + `git diff --check` | 零命中 |
| 全量质量门 | `npm test` | exit 0 |

## 七、主机限制（诚实声明）

- 无 macOS/Windows 真机：桌面验收仅 Linux；不宣称跨平台已验证。
- Android 真机缺席：WebView 触摸边界只能用构建产物 + 桌面 Chromium 近似 + 静态 manifest 审计；真机 BACK/手势列为 Limited。
- Tauri dev 的 GUI 窗口在无显示服务器环境下不能自动弹窗验证：桌面 smoke 以 host 进程级 + Rust 单测 + 前端契约覆盖，WebView 内行为由加载同一 URL 的 Playwright（浏览器）等价验证并注明差异。

## 八、逐项验收条件（完成时逐项打钩并附证据）

1. [ ] `docs/development/TAURI-CLIENT-APK-PLAN.md` 入库（本文件）。
2. [ ] `launcher/realm-service-host.mjs` + host smoke（真实 PG 起停、health、优雅停止）。
3. [ ] `src-tauri/` 真实 Tauri 2 工程：commands、状态机、选择界面（启动中/就绪/失败/重试/已有实例/退出可见）。
4. [ ] origin 校验集中实现 + 双端一致测试。
5. [ ] LAN 模式：显式 IP、token 强制、警告文案、契约测试。
6. [ ] 自适应修复 + Playwright 五档 viewport。
7. [ ] `cargo check/test` 通过；Linux `cargo tauri build` 产物或硬原因。
8. [ ] Android scaffold +（工具链允许则）debug APK + 内容审计，否则 blocked 文档。
9. [ ] 敏感扫描零命中、`git diff --check`、`npm test` 全绿。
10. [ ] STATUS.md 真实更新；分主题 commit；无 push。

## 九、完成证据（实施中持续更新）

实施完成（2026-09-15）：

1. [x] 计划入库：`6769691`。
2. [x] `launcher/realm-service-host.mjs` + host smoke：`tests/tauri-service-host.test.mjs` 1/1 exit 0（真实 PG17 起停、帧序列 lock→…→ready、GET / 200+SSR 标题、双实例 already-running exit 1、SIGTERM→stopped exit 0、帧零敏感形态）。smoke 暴露并修复 3 个既有 launcher bug（local-postgres 未传 start / initdb 超级用户 / psql -U）。
3. [x] `src-tauri/` 真实 Tauri 2 工程：6 commands + 状态机 + 选择界面（启动中/就绪二选/失败重试/已有实例/停止退出）。`cargo test --lib` 3/3；`cargo tauri build --debug --no-bundle` exit 0（`target/debug/realm-desktop` 190MB，ldd 零缺失，headless 于 GTK init 停止属预期）。
4. [x] origin 校验集中实现（Rust `normalize_server_url` 单测 2 例组 + UI 经 invoke 复用）。
5. [x] LAN 模式：显式 IPv4、token 强制（缺失 fail-closed 零副作用）、0.0.0.0 警告、`tests/tauri-desktop-contract.test.mjs` 5/5（含 CORS 通配扫描与 cookie 属性钉住），已注册 `test:contracts`。
6. [x] 自适应：修复 ≤720px 面包屑隐藏；`tests/gui/w-adaptive.spec.ts` 375/412/768/1024/1440 + 登录 6/6 exit 0（隔离 server + 一次性 PG）。
7. [x] Linux `cargo tauri build`（debug）通过；release/bundle 未做（非目标）。
8. [x] Android：工具链 user-local 实装（JDK 17.0.20.1 / cmdline-tools / platform-34 / NDK 27.0.12077973）→ `cargo +stable tauri android build --apk --target aarch64` exit 0 → `app-universal-release-unsigned.apk`（9,476,579 B，sha256 `646614e403927ca8b3241e1332a2ab54503ae2779d47055f8fc6c284bf082f73`）；审计：仅 INTERNET 权限、arm64-v8a、minSdk 24/targetSdk 36、无 Node/PG/server/凭据。
9. [x] `npm test` exit 0（typecheck/core 347/contracts 含 tauri-desktop-contract 与 postgres-runtime 144/build/render）；lint exit 0；`git diff --check` exit 0；敏感扫描 0 命中。
10. [x] STATUS.md 已更新（2026-09-15 00:55 +08:00）；commit 分主题；未 push。

未完成/Blocked：macOS/Windows/Android 真机验收（无设备）；桌面窗口 headless 不可弹窗（只验到 GTK init）；release 签名与商店发布（非本批目标）。

## 十、Iris 独立验收与补充修正（2026-09-15 02:19:26 +08:00）

Kimi 的完成回调仅作为交付线索；本节记录独立源码、构建、运行和产物核对后的终态。

1. 桌面生产 bundle 已补齐：`src-tauri/tauri.*.bundle.conf.json` 使用显式资源 overlay，`scripts/desktop/prepare-tauri-resources.mjs` staging 生产 dist、pruned Node modules、bundled Node、PG17/pgvector；Rust 从 Tauri `resource_dir/realm` 定位资源，并以白名单环境变量启动 host。普通 `cargo test`/Android build 不再依赖 staging，正式入口 `npm run desktop:tauri:build` 在 Linux 实际生成 deb/rpm。
2. launcher 边界已修正：支持 `linux-x64` bundled PG、Linux `LD_LIBRARY_PATH`、数值 IPv4 校验、`--lan` 与 `REALM_LAN_BIND`；LAN 仍要求 `REALM_ACCESS_TOKEN`，默认 loopback。
3. APK 跨域检查已移到 Rust `check_server` command；无 CORS 的 loopback HTTP 401 单测通过（401 被识别为可达）。Android 构建 wrapper 会自动选择已审计的 user-local JDK/SDK；生成项目中的 release cleartext policy 通过 wrapper 固化，UI 明示 HTTP 未加密、HTTPS 优先。
4. 独立验证结果：Rust `cargo test --lib` **5/5**；Tauri/launcher focused suite（含真实 PG host）**25/25**；`npm test` **exit 0**；`npm run lint`、`npm run typecheck`、`git diff --check` **exit 0**；生产资源 host smoke **ready → GET / 200 → SIGTERM → stopped，exit 0**。
5. Linux Tauri release bundle：`npm run desktop:tauri:build` **exit 0**；release deb `136441000` bytes，sha256 `21a152c9153043faca78d83e955719cb2899c98d03882ff67566e2479e8b0905`；release rpm `137121249` bytes，sha256 `60f773d8320b52851acfb0ce61f1d667672e37423223232e96c64b2d6be22a5d`。deb 解包确认包含 launcher、Node、PG17/pgvector、dist 和生产 vinext 依赖，不含 `.env.local`/用户数据。
6. Linux release deb cage GUI：真实窗口显示“服务已就绪”及“打开客户端/打开网页/停止服务并退出”，截图无截断/重叠；Tauri WebView 输入注入工具未能触发按钮 click，因此不把它记为真实窗口导航通过。相同 UI 的 Playwright `375x667` DOM smoke **exit 0**，确认无横向溢出且点击“打开客户端”导航到服务页；这项是逻辑/布局补证，不替代 Android 真机或真实鼠标/触摸验收。
7. Android client-only APK：`npm run desktop:android:build` **exit 0**；`app-universal-release-unsigned.apk` `14164359` bytes，sha256 `89e4f632a212426f4ef7ec915e4cba11274bc6228320ef37db1ac20d492ffd47`，`arm64-v8a`，minSdk 24/targetSdk 36，manifest 含 INTERNET 与 AndroidX 生成的 `DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`，cleartext policy 为 true；ZIP 文件名与内容扫描未发现 Node/PostgreSQL/server/launcher/凭据。unsigned，且当前 `adb devices` 无设备，未宣称真机安装、触摸、系统 BACK 或 LAN 跨设备通过。
8. 本轮补充改动已按主题提交为 `6377c96d3130ee158d1264b1a9558f7fef22473a`；未 push。macOS/Windows 构建与签名发布仍保持 Limited/Blocked。
