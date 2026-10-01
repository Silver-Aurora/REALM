# 图像生成自动模式（I2I 前最后一批基础设施）

基线：HEAD `576f831`（0048 台账 + 手动 dispatch 落库已验收）。本批：账号级
自动模式偏好 + 持久化请求队列 + 独立 worker + 触发接线 + 前端控制。
不碰 I2I/ControlNet/IP-Adapter/角色一致性本体。

## 1. 偏好归属与模式语义

**账号级**（`accounts.scene_image_mode`，migration 0049）：每个 principal
自己决定是否为自己的游玩消耗 GPU；observer 无玩家回合（不触发）但可读/
可改自己的偏好。默认 `off`（打开页面零 GPU 消耗；手动按钮不受影响）。
不进 localStorage/worlds.settings。

- `off`：不自动排队（0048 手动路径不变）。
- `scene_change`：仅当设定结晶 `store.applyDelta` 真正写出且 delta 含
  场景字段（location/weather/tension/objective/displayTime 非空）时排队；
  source_event_id = 晶化事件 id（稳定证据，幂等键）。
- `every_turn`：每个成功 committed 且非 replay 的玩家回合排队一次；
  source_event_id = 回合提交的幂等键/事件 id；回合失败/取消/visibility
  未确认不排队；self-play 拍不算（文档化：不触发，除非未来单列模式）。
- 触发方读取的是**触发回合所属 principal** 的账号偏好，绝不影响他人。

## 2. 请求与生成的关系（队列不丢、生成不吞）

- `scene_image_requests`（migration 0050）是持久化**意图**台账：
  `(workspace_id,id)` PK + world/record/scene FK + `trigger_kind`
  （scene_change|every_turn）+ `source_event_id` + status
  （queued/leased/completed/failed）+ generation_id（→0048 台账）+
  attempts/leased_at/lease_expires_at/last_error/created_at/updated_at。
  幂等：`(workspace_id, trigger_kind, source_event_id)` 唯一——重复提交/
  回放/重试不重复排队。
- 0048 的「同一 Record 一条 active 生成」唯一索引仍然成立：worker 对同一
  Record **串行**消费请求（按 created_at 顺序，逐条 claim→执行→complete）；
  请求多条排队不丢失，生成台账复用/新建沿用 0048 active 语义（一条
  running 时后续请求排在 queued，active 完成后下一条接管）。envelope 的
  latest ready 始终指向最新完成图——每条请求按执行时刻的最新 scope 生成。
- route/worker 进程退出不丢请求：queued/leased 行在库；worker 启动时
  stale 恢复（lease 过期 queued 化）。

## 3. 状态机与失败分类

`queued → leased → completed | failed`；lease 默认 120s，stale（lease 过期
未更新）由 worker 启动/周期回收为 queued。attempts 上限 3：
- 临时错误（网络/连接/超时）→ failed→queued 重试（attempts 递增）；
- 永久错误（路径/magic/校验/schema 类）→ failed 终态，last_error 只写
  安全分类码（绝不写 provider body/路径/URL/key/prompt_id）。

worker 内执行沿用 0048 路径：prepare（composeScenePrompt，公开授权 scope
+ public focus，不含 private/restricted 原文）→ /prompt → 有界 history
轮询 → /view 校验 → world_files+台账同事务 ready。

## 4. Worker（参考 propagation worker）

- `modules/application/scene-image-worker-runtime.ts` +
  `scripts/scene-image-worker.mjs`；专用连接持 advisory lock
  （`realm_scene_image_worker`）单实例，抢不到即退出；
  `REALM_SCENE_IMAGE_WORKSPACES` 显式清单（缺省 demo workspace）；
  loopback runtime DB（createLocalPostgresPool 自带回环校验）；
  realm_runtime 最小权限；启动健康检查（0048/0050 表存在性）+ stale 恢复；
  idle backoff 有上限；SIGTERM/SIGINT 干净退出。
- `scripts/systemd/realm-scene-image-worker.service` 只作模板提交，
  **不安装/不启用**（等 Iris 验收后由运维决定）。

## 4a. 跨平台宿主（supervisor，T10 后续批）

- `launcher/scene-image-worker-host.mjs` 是唯一 supervisor：spawn worker
  子进程（`process.execPath --experimental-strip-types`，Windows 不经
  cmd shell）、日志隔离到 `<dataHome>/logs/scene-image-worker.log`
  （supervisor 自身只写 pid/exit code/signal，绝不写 env/连接串/
  provider key/prompt_id）、子环境白名单（只透传 PATH/HOME 等非敏感键
  + 显式注入的 REALM_RUNTIME_DATABASE_URL / REALM_SCENE_IMAGE_WORKSPACES /
  REALM_DATA_HOME）、优雅停止（unix SIGTERM / Windows kill，超时强杀，
  幂等）、意外退出传播 onState("unavailable")；worker 空闲时不产生任何
  ComfyUI/GPU 调用。
- Tauri：`launcher/realm-service-host.mjs` 在 Web ready 之后非阻塞启动
  worker；协议新增 `{"type":"worker","state":"starting|unavailable|
  stopped","reason?":"…"}`（Rust 忽略未知帧类型，向后兼容）；优雅停止
  顺序 = 先停 worker → 停 app → 停 PG。worker 失败只经 worker 帧上报，
  绝不伪装成 Web 失败。
- 三平台开发入口（START-REALM-Windows.cmd / .command / -Linux.sh →
  setup-web.*）共用 `scripts/setup-web.mjs`：Web ready 后启动同一
  supervisor，dev server 退出前 finally 停 worker；不各平台复制逻辑。
- 后台模板全是**用户级可选、不默认安装、不含凭据**：systemd 单元
  ExecStart 改指 supervisor；`installer/windows/realm-scene-image-worker-task.xml`
  （Task Scheduler，InteractiveToken/LeastPrivilege，无需管理员）；
  `installer/macos/com.realm.scene-image-worker.plist`（LaunchAgent，
  ~/Library/LaunchAgents）。三者入口同为 supervisor。
- Worker 心跳：`modules/application/scene-image-status.ts`，runtime 拿锁后
  与主循环节流写 `<dataHome>/diagnostics/scene-image-worker.json`（原子写，
  只含 state/workspaces/updatedAt），stop 写 stopping；API 侧按 15s 新鲜度
  判 online/unavailable（stale/missing/invalid），无 DB 迁移。
  `GET /api/settings/scene-image` 附带 `worker` 投影；主界面据此区分
  「后台 Worker 未连接」（mode≠off 且 unavailable）与真正的「排队中」
  （queued 且 online），queued+unavailable 显示「等待后台 Worker」，
  running/failed 文案不变。

## 5. 触发接线（fire-and-forget 但先落库）

- every_turn：local-record-service 回合提交成功且 `!authorization.replay`
  的分支（与 memorySync/晶化同一位置）enqueue；回合失败/取消/visibility
  未确认不到这里。
- scene_change：晶化管线 `store.applyDelta` 成功返回后，delta 含场景字段
  才 enqueue（source_event_id = written.eventId）；delta 拒绝/无场景字段/
  失败不触发；self-play 产生的新 scene 按同一规则触发（文档化）。
- enqueue 是单条 INSERT（持久化先于一切）；触发失败只留日志，绝不阻塞
  回合响应。enqueue 前读触发 principal 的 accounts.scene_image_mode。

## 6. envelope 与主界面

- envelope 新增 `sceneImageJob: null | { status, triggerKind? }`（当前
  Record 最新请求/活跃生成的安全状态；**不透出 prompt_id/provider 路径/
  body/key**）；latest ready 背景仍由 `sceneImage` 提供。
- header 新增纸墨直角 auto 控制（button + 小型 radiogroup popover）：
  显示当前模式，三语，选择即 PUT 持久化 + role=status 反馈；默认 off；
  observer 可读自己的设置但不触发生成（无回合）；手动按钮保留。
- queued/running 时前端沿用既有信封刷新机制（轮询/手动刷新）拿状态；
  ready 后背景经现有遮罩/装裱层显示；failed 显示安全文案可重试；
  不遮挡 timeline；移动/桌面同构。

## 7. 边界（本批不做）

I2I/ControlNet/IP-Adapter/角色一致性；输出图管理 UI；多 Record 并行生成
（同 Record 串行是刻意简化）；systemd 安装启用；公网 dispatch。
