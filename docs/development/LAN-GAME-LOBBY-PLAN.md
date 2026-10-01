# 局域网游戏大厅 v1（LAN-GAME-LOBBY-PLAN）

- 基线：以当前 `main` 与 `STATUS.md` 最新记录为准。本文档是调研结论 + 实施计划 + 验收记录。
- 范围：同一局域网内，多个玩家用浏览器/客户端连接同一台 REALM 主机服务，在「游戏大厅」看到开放的房间并加入。**只做 LAN**；不做公网 Relay、WebRTC、跨主机发现、mDNS、Internet 穿透。

## 一、现状调研结论（可复用资产）

- **身份/会话**：`modules/identity/auth.ts` —— `REALM_ACCESS_TOKEN` 门禁 + HMAC 会话 cookie（`realm_session`，HttpOnly+SameSite=Lax）；principal 由 displayName 哈希派生（`principalIdForDisplayName`），登录即建 `accounts` 行（`0012`，昵称唯一）。**不同 displayName = 同一 LAN 服务上的不同玩家**，零新机制复用。principal 只能来自服务端 session（`resolveRequestPrincipal`），客户端不能伪造 userId。
- **LAN 暴露**：`launcher/realm-launcher.mjs` 已有显式 `lanBind` opt-in（显式 IPv4 + 强制访问令牌门禁，fail-closed，零 0.0.0.0 默认）。大厅不新增任何监听/暴露面。
- **实时链路**：图谱 SSE（`app/api/world-knowledge/events/route.ts`）已验证「pg_notify 唤醒 + 客户端权威回读 + Last-Event-ID 心跳重连」模式；大厅复用同一模式但不建账本（大厅状态是小集合，失效即重读全量列表，无需游标重放）。不新增 WebSocket，不重写 record SSE 或 server 启动方式。
- **World/Story/Record 边界**：大厅房间**不是**世界/故事/记录；未绑定房间只是「约局元数据」（房名/房主/成员/容量/密码/状态），不触碰 World/Story/Record。绑定世界是房主的显式分享，加入时只授予既有 World 的 player membership，不创建或修改 Story/Record。已有世界的 owner/member/RLS 语义零改动。UI 明确标注这是「约局大厅」阶段，进入房间不等于进入共享 Record。
- **数据落地**：workspace 级两张小表 + 既有 `realm_workspace_isolation` RLS 模式 + realm_runtime 最小授权；密码用 node:crypto `scrypt`（现成依赖）+ 随机 salt + `timingSafeEqual` 校验，绝不存明文。

## 二、数据模型（migration 0045，新增独立文件，不改旧迁移）

- `lobby_rooms`：`(workspace_id, id)` PK；`name`（1–40 字符）、`status ∈ (open, closed)`、`capacity` int 2–8（默认 4）、`password_hash` text（可空=公开；scrypt 格式串）、`host_principal_id`、`created_at/updated_at`。
- `lobby_room_members`：`(workspace_id, room_id, principal_id)` PK；`role ∈ (host, player)`、`joined_at`、`left_at`（软离开，可回归）；FK 级联到 rooms。
- 两张表都 `ENABLE/FORCE RLS` + `realm_workspace_isolation`；`realm_runtime` 仅 `SELECT, INSERT, UPDATE`（无 DELETE——离开是 `left_at` 置位）。
- 重启策略：房间是长期状态行（服务器重启后保留）；0047 起新房间带服务端 host lease，房主离线后由权威读/写或大厅 SSE tick lazy reap 为 closed；0046 前 lease=NULL 的遗留房间保持永不过期。

## 三、API 与服务层

- `GET /api/lobby`：房间列表（仅 open/closed 状态 + 白名单字段：id/name/hostDisplayName/memberCount/capacity/hasPassword/viewerRole/status），不含 password_hash、不含世界正文。
- `POST /api/lobby`：命令面 `{kind: create-room | join-room | leave-room | close-room | heartbeat, ...}`；密码只在 join 的 body `password` 字段（绝不进 URL/日志/错误消息）。heartbeat 只续本人 host 的 open 房间，过期不复活。
- `GET /api/lobby/events`：SSE——连接即发 `snapshot` 帧；业务写同事务 `pg_notify('lobby_changed', workspace_id)` 唤醒 → 客户端收到 `changed` 帧后权威重读 `GET /api/lobby`；25s 传输心跳，并在 tick 触发 lazy reap；断线由浏览器 EventSource 自动重连（重连即收到新 snapshot）。刷新/重连后状态永远来自服务端。
- 服务层 `modules/application/lobby-service.ts`：错误码 `ROOM_NOT_FOUND/ROOM_CLOSED/ROOM_FULL/BAD_PASSWORD/NOT_HOST/NOT_MEMBER/INVALID_COMMAND`；加入幂等（已是在册成员返回 ok）；容量判定与加入在同一事务对房间行 `FOR UPDATE`（不超卖）；关闭/离开幂等；成员详情只允许在册成员读取。
- 房主显示名：JOIN accounts 读 display_name；账号行缺失时 fail-closed 为「旅人」占位，不暴露 principal id。

## 四、安全边界

- 门禁：所有大厅路由经 `resolveRequestPrincipal`；门禁关闭的纯 loopback 模式回落单用户 principal（与全站一致）。
- 密码：scrypt(`N=16384,r=8,p=1`) + 16B 随机 salt；存储格式 `scrypt:16384:8:1:<saltB64>:<hashB64>`；校验用 `timingSafeEqual`；错误提示只说「密码不正确」；hash 绝不出现在任何 API 响应/日志。
- 列表白名单字段；创建/加入只接受服务端 session principal；房名校验 1–40 字符 trim；密码最长 80 字符；容量 2–8；成员详情必须先入房。
- LAN 暴露仍由既有 `lanBind` 显式开启 + 访问令牌门禁承担；本批不改任何监听默认值。

## 五、UI

- header 新增「大厅」入口（世界库旁）；大厅 overlay（直角纸墨、light/night 双主题、三语 i18n、375/412 无横向溢出、键盘可达）。
- 房间卡：房名/房主显示名/成员数 n/cap/锁标记/状态/加入（密码房内联密码框）/离开/房主关闭。
- 创建表单：房名必填、密码可选、人数（2–8，默认 4）。
- 同步：EventSource 驱动权威重读 + 断开提示；不伪造实时。

## 六、明确留到公网阶段

跨主机房间、公网 Relay/WebRTC、mDNS 自动发现、房间内聊天/语音、房间直接绑定共享 Record 或共享写入、房间转让与 Last-Seen 展示。

## 七、验收矩阵

- PG 服务测试 `tests/postgres-lobby.test.ts`：scratch 库全链迁移 + 双 principal——创建/列表白名单/密码校验（对错）/重复加入幂等/容量拒绝/关闭后拒绝加入/非房主不能关闭/离开/字段白名单（响应无 password_hash）/host lease 心跳、过期回收、并发与重启。
- 契约：`tests/lobby-contract.test.mjs`（路由鉴权接线、字段白名单、UI 入口、i18n 三语、无密码进 URL/日志的静态钉住）。
- GUI `tests/gui/y-lobby.spec.ts`：隔离 server + 临时 token + 双浏览器上下文双身份——创建→对端可见→密码加入→双方成员计数更新→错误密码/离开/关闭→375/412 无溢出；`tests/gui/z-lobby-lease.spec.ts` 覆盖在线续租与房主离线后的观察者关闭。
- 围栏对齐：PG migration 计数 44→47（0045 大厅表、0046 World 绑定列、0047 host lease 列；后两者不新增领域表）。
- 门：typecheck / lint / focused / 全量 npm test / `git diff --check`。

## 八、房主在线租约（0047，本批落地）

- **语义**：open 房间创建即带服务端时钟租约（默认 90s，TTL 见
  `lobby-service.ts` 的 `LOBBY_LEASE_TTL_MS`；隔离/测试环境可经
  `REALM_LOBBY_LEASE_MS` 注入短租约）。房主客户端在线期间按服务端
  meta 告知的频率（TTL/3）心跳续租（`POST /api/lobby {kind:"heartbeat"}`，
  幂等、只续本人 host 的 open 房间、过期不复活）。
- **回收（lazy reap）**：过期 open 房间在下一次权威读（list）/写（join）/
  大厅 SSE heartbeat tick 时同事务置 closed 并 pg_notify 唤醒观察者。
  **没有请求或连接时没有任何后台动作**——这是明确的触发边界，不声称
  瞬时后台执行。服务器重启后租约是持久化绝对时点，过期房间读回仍被
  回收，不复活。
- **边界**：0046 前的遗留房间 lease=NULL 永不自动关闭（语义不动）；
  已获得的世界 player membership 不因房间过期回收；客户端本地时间
  不参与权威判定。
- **UI**：房主房间显示托管提示（离线自动关闭说明）；客人经现有 SSE
  失效 + 权威重读看到关闭；过期房间加入入口消失。

## 2026-09-18 独立验收与测试更正（00:52:54 +08:00）

- Kimi 完成事件 `kite-f251cfb14cdc` 未直接作为验收证据。独立复核发现租约测试原先的“重启”只重建 service、复用旧连接池；父侧修正提交 `ddc0255`：先保留过期但仍 open 的房间，关闭旧连接池，再用新连接池通过权威读触发 lazy reap，并显式断言 heartbeat/reap 并发结果均成功。
- 修正后最终验证：`npm run typecheck`、`npm run lint`、`git diff --check` 通过；大厅契约 6/6；大厅 PG 套件 5 连跑均 4/4；最终 `npm run test:postgres-runtime` 在一次性 loopback scratch PG17 中 148/148；`npm test` exit 0，Build complete，render 3/3。
- 真实 Chromium 双身份 GUI：短租约在线续租→房主页面关闭→客人经 SSE 看到自动关闭 1/1；大厅主回归 3/3；均使用隔离 server、scratch PG 和临时 fixture token。scratch 容器/数据库、服务、端口和 Playwright 产物均已清理。
- 中途一次全链运行出现大厅并发测试连接被 PostgreSQL administrator command 终止（147/148），未采纳为通过；随后同一隔离集群大厅 5 连跑和最终全量 PG 均通过，未复现。该次异常不改变完成结论，但保留在记录中。
- 当前已完成服务端 host lease/heartbeat、服务端时钟权威、lazy reap、SSE tick、重启不复活和前端托管提示；无连接时不执行后台回收。Last-Seen 展示、房间转让、公网 Relay/WebRTC/mDNS、房间聊天、共享 Record 联机、真实跨设备 LAN 和真机平台仍未完成。
- 本批提交尚未 push。

## 九、安全邀请链路（本批落地）

- **链接形态**：`{当前 Web origin}/?lobby=<不透明 roomId>`——只含 roomId，
  不含密码/REALM_ACCESS_TOKEN/session/principal/连接串/provider 信息；
  密码房链接不绕过显式密码加入（加入路径校验不变）。
- **授权边界**：invite query 只是导航提示（读取后从 URL 剥离）——可见性/
  可加入性仍由服务端 session + 房间状态裁决；邀请落地绝不自动加入，
  失效/关闭/不存在给稳定提示，不泄露成员详情。
- **分享 UX**：Web Share（可用时）→ clipboard → 手动复制行三级 fallback；
  用户取消（AbortError）静默不算失败；权限拒绝/非安全上下文落手动复制；
  反馈逐房间可见。
- **入口**：onboarding 与主 session 共用同一 renderLobbyOverlay，深链在
  两条路径都生效。
- **验证**：契约 `tests/lobby-contract.test.mjs` 7/7；GUI
  `tests/gui/z2-lobby-invite.spec.ts` 1/1（双身份、clipboard 断言、失效/
  关闭/密码房深链）；大厅既有 y/z 套件回归 3/3 + 1/1。

## 2026-09-18 邀请链路独立验收与并发修正（01:43:09 +08:00）

- Kimi 完成事件 `kite-2193bfb1f1a8` 未直接作为验收证据。独立 GUI 复跑确认邀请链接只含当前 origin 与不透明 roomId；深链打开大厅、定位目标且不自动加入；密码房仍需显式密码；关闭/不存在目标均稳定 fail-closed。
- 独立边界测试真实通过 2/2：恶意 `roomId` 不触发 pageerror；clipboard 权限失败时显示手动复制 fallback。复核发现原实现把不可信 roomId 插入 `querySelector`，父侧改用固定属性查询 + dataset 精确匹配；从邀请目标进入世界或重新打开普通大厅时清理旧高亮，修正提交 `e9f2550`。
- 全量回归曾真实复现大厅并发容量超卖（2 个成功加入）；根因为 `FOR UPDATE` 与成员计数在同一 READ COMMITTED 语句中，等待锁后相关子查询可能使用旧快照。`e9f2550` 拆为锁房间与锁后计数两条命令；前序完整 PG 套件复跑通过，最终 `npm run test:postgres-runtime` 148/148，`npm test` exit 0。
- 最终 Chromium 验收：邀请/大厅/租约共 5/5；静态契约 7/7；typecheck、lint、diff check、Build complete、render 3/3 全部通过。另有大厅 focused PG 套件 5 连跑 4/4；临时 scratch 数据库、容器、服务、端口、Playwright 输出均已清理。
- 当前完成安全邀请导航与分享 fallback，但二维码、邀请有效期/次数限制、Web Share 系统弹窗、WebKit/真机、两台物理设备跨 LAN、公网 Relay/WebRTC/mDNS、共享 Record 联机仍未完成。
- 本批提交尚未 push。

## 十、LAN advertised origin 独立验收与首帧修正（2026-09-18 02:28:24 +08:00）

- Kimi 完成事件 `kite-9d80a02ab289` 未直接作为验收证据。实现采用显式 `REALM_ADVERTISED_ORIGIN`，服务端校验后经大厅 meta 下发；不读取 Host/X-Forwarded-Host，不扫描网卡、不猜私网 IP，不改变默认 loopback 或 LAN bind 门禁。
- 父侧复核补上面板可见的“邀请链接来源”，并把 origin 诊断延迟到大厅首次权威 GET 完成，消除显式配置下的错误首帧闪烁；非法配置不再与 loopback 警告重复显示。路由同时只解析一次 origin 配置。
- 真实隔离 GUI：合法显式 origin 1/1、无配置 loopback 回退 1/1、非法配置 `null + invalid=true` 1/1；合法 origin 的复制链接使用配置来源，非法配置回退当前地址，均无 pageerror。契约 origin 3/3 + lobby 7/7，最终 `npm test` exit 0，lint/typecheck/diff check 通过。
- 这些 GUI 仍是 Chromium + scratch PG 的单机证据；`192.0.2.10` 只用于文档地址形态，不代表真实跨设备可达。Android 客户端、WebKit、两台物理设备 LAN、公网 Relay/WebRTC/mDNS、二维码和邀请有效期仍未完成。
- 本批父侧修正尚未 push。

## 十、advertised origin 的桌面/launcher 配置入口（本批落地）

- **配置语义**：`REALM_ADVERTISED_ORIGIN`（环境变量）与 launcher
  `--advertised-origin <origin>` 同源校验（单一实现
  `launcher/advertised-origin.mjs`，TS 入口只 re-export）。优先级：
  显式 CLI/option > 环境变量 > `--lan` 具体 IPv4 推导
  `http://<bind>:<appPort>` > null。`0.0.0.0` 不生成可分享地址并给出
  醒目提示；非法值在任何副作用（锁/日志/子进程）之前 fail-closed，
  错误不回显可能内嵌凭据的原值。
- **Tauri**：`service_start` 接受可选 advertisedOrigin（Rust 侧同规则
  严格校验，先于 spawn）；桌面选择界面有「LAN 分享设置」配置行
  （localStorage 持久化 + 保存并重启服务）；ready 状态显示「LAN 分享
  地址已配置 / 仅本机可用」。Android client-only 无服务端 launcher
  接线，属后续批次。
- **证据**：launcher 单测 4/4；desktop 契约 8/8；Rust 6/6；service-host
  真实路径 GUI（生产构建 + 完整 launcher 链）1/1。

## 十一、桌面 LAN bind 独立复核与生命周期修正（2026-09-18 03:43:12 +08:00）

- 父侧源码复核发现 Tauri 原实现只传 `advertisedOrigin`，未传既有 `lanBind`；若桌面服务仍绑定 loopback，界面显示的 LAN 分享地址会误导用户。已补齐 Tauri `lanBind` 配置、Rust IPv4 校验、`--lan` 参数传递，以及只有显式 LAN bind 才透传 `REALM_ACCESS_TOKEN` 的门禁。
- Tauri 新增 `validate_advertised_origin` / `validate_lan_bind` 命令；校验与缺失 token 检查先于 spawn，UI 持久化 `realm.lanBind` 与 `realm.advertisedOrigin`，保存后重启服务。默认 loopback、`0.0.0.0` 不推导分享地址、Android client-only 边界保持不变。
- 真实 service-host 参数 smoke 使用临时 PG17/端口验证：ready URL、显式 advertised origin、HTTP 200/SSR、协议无敏感形态、SIGTERM 优雅停止均通过；首次 smoke 暴露 dev-server wrapper 未转发 SIGTERM、残留 Vinext 端口，父侧修正 `scripts/dev-server.mjs` 并复跑确认临时端口无残留。
- 独立验证：Rust `cargo test --lib` 7/7；launcher/origin/Tauri/lobby 契约 22/22；typecheck、lint、diff check、生产构建和 service-host 参数 smoke 全部通过。原 service-host 固定候选端口测试在本机长期服务占用时曾 skip/失败，未冒充通过；使用显式隔离端口完成等价真实路径验证。
- 当前仍未有 Tauri 显示服务器窗口点击证据、macOS/Windows 真机、Android 真机、两台物理设备 LAN 或公网 Relay/WebRTC/mDNS 证据；Kimi 报告的 192.0.2.10 仅为 TEST-NET 形态校验。
- 本批父侧修正尚未 push。

## 十一、Android client-only 邀请粘贴接入（本批落地）

- **输入**：Android 壳的服务地址框同时接受服务根地址与完整大厅邀请
  URL（`http(s)://host[:port]/?lobby=<roomId>`）。新增 Rust
  `validate_invite_url` 单一校验入口（严格白名单：http/https、非空
  host、禁 userinfo/fragment/路径/未知或重复 query；roomId 必须
  `lobby_<24 位小写 hex>`；错误不回显原始输入），返回严格结构
  `{ serverUrl, lobbyRoomId }`；既有 `validate_server_url` 保持不动。
- **连接流程**：`validate_invite_url` → 既有 `check_server(serverUrl)`
  → localStorage 只写服务根地址（roomId/query 绝不落盘）→ 导航
  `serverUrl/?lobby=<roomId>`（一次性导航，由 Web 端既有深链打开大厅、
  定位目标、不自动加入）；无 lobby 的服务地址导航根页。
- **边界**：不做 OS 级 universal link/app link 注册；Android 仍
  client-only（不启动本地服务）；桌面 service_start/LAN bind/
  advertised origin 语义零回归。
- **证据**：cargo test 9/9；tauri-desktop-contract 9/9；z4 浏览器
  mock 壳 3/3（非真机，已标注）；unsigned APK 重建 exit 0 且内容扫描
  无服务端/凭据（仅 INTERNET 权限，arm64-v8a）。

## 十二、Android 邀请接入独立复核与重试修正（2026-09-18 04:54:53 +08:00）

- 父侧复核发现 `?`/`?&` 空 query 会被 Rust 邀请校验器误当成纯服务地址放行，且连接检查失败后的重试会丢失大厅 roomId；已分别改为 fail-closed 和保留 `lobbyRoomId`，并补充 Rust/契约覆盖。
- 独立验证：Rust 9/9、综合 Tauri/Android 契约 9/9、Z4 Chromium mock 壳 3/3、最终 `npm test` exit 0；Z4 第一条实际包含一次合成连接失败后重试，仍落到目标大厅且不自动加入。scratch PG、服务、端口、Playwright 状态和临时 runner 已清理。
- 使用已存在的 user-local JDK17/Android SDK/NDK 真实构建 unsigned APK 成功；实物检查 14,179,487 bytes、arm64-v8a、minSdk 24、target/compile 36、唯一 native library 为客户端库，无 Node/PostgreSQL/.env/凭据文件名。原始构建命令未设置 Java 时的失败未计入通过。
- Android 真机/WebView 触摸与系统 BACK、WebKit、两台物理设备 LAN、公网、OS universal link/app link 仍未验收/实现；浏览器 mock 壳和 unsigned 构建不代表真机发布完成。
- 本批父侧修正尚未 push。

## 十三、最终 GUI 收口独立验收与修正（2026-09-18 06:08:40 +08:00）

- 父侧复核确认 Tauri 壳状态渲染已去除 `innerHTML`；发现 z5 原测试只收集未断言 console error，以及成员列表展开后离开会请求失效成员详情。已补充非预期 console error 断言，并让离开已展开房间只重读列表、清除展开状态。
- 独立 z5 Chromium + scratch PG 1/1；最终选定 GUI 矩阵 17 项在正确环境下全部通过。混合 runner 首次 16/17 的唯一失败是 z3 专用 advertised origin 污染 z2 的无 origin 预期，z2 随后在无 origin scratch server 独立 1/1；不计为产品失败。
- 独立代码门：Rust 9/9；Tauri/launcher/lobby 相关契约 23/23；typecheck、lint、diff check、最终 `npm test` exit 0；桌面 Tauri debug 构建和 Android unsigned APK 重建均成功。APK 实物无 Node/PostgreSQL/.env/凭据文件名。
- 所有 scratch PG、服务、端口、Playwright 状态和临时 runner 已清理。真机/Android WebView、WebKit、两台物理设备 LAN、公网、OS universal link/app link 仍未验收或实现。
- 本批父侧修正尚未 push。
