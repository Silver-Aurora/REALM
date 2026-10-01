# 图像生成 Stage 3：ComfyUI 输出落库 + Record 前端背景

基线：HEAD `f03eada`。闭环：显式点击生成 → POST /prompt → 有界轮询 history →
/view 下载校验 → world_files 落库 + 生成台账 → envelope 透出 fileId →
前端经 `/api/files/<id>` 换背景。不自动生成、不接 I2I/ControlNet。

## 1. 存储设计（migration 0048，不改历史迁移）

- `world_files.kind` CHECK 经 `DROP CONSTRAINT … / ADD CONSTRAINT` 扩展为
  `('character_avatar', 'scene_background')`（bytea 语义、world FK、
  既有 SELECT/INSERT grant 不变；文件字节绝不进文件系统或 worlds.settings）。
- 新表 `scene_image_generations`（生成台账 + Record 绑定）：
  `(workspace_id, id)` PK；`world_id`/`record_id`/`scene_id`/`file_id`（可空，
  ready 才有）/`status`（queued/running/ready/failed）/`prompt_id`/
  `error_code`（仅安全分类码）/created_at/updated_at。
  FK：(workspace_id)→workspaces、(workspace_id,world_id)→worlds、
  (workspace_id,record_id)→records、(workspace_id,file_id)→world_files。
  FORCE RLS + workspace 隔离 policy；`GRANT SELECT, INSERT, UPDATE` 给
  realm_runtime（状态推进需要 UPDATE；无 DELETE——台账不可删）。
  索引：`(workspace_id, record_id, status, created_at DESC)`（active 复用 +
  latest ready 查询）。workspace 清理随 workspaces/worlds/records 级联删除。
- 契约：`tests/postgres-schema-contract.test.mjs` 的 scopedTables/RLS 清单
  登记新表（测试文件的显式登记即评审闸门）。

## 2. 生成状态机与并发

`queued`（已接受未跑）→`running`→`ready`/`failed`。本批实现：
queue accepted（POST /prompt 返回 prompt_id）≠ ready；route 在**有界预算内**
同步轮询 `/history/<prompt_id>`（默认 ≤90s、1s 间隔、可注入时钟），
terminal 才下载。失败（history error/超时/下载失败/校验失败）→ failed +
error_code（安全分类），**不留可见半成品**（world_files 与 ready 状态同一
事务写入）。active 复用由事务级 advisory lock + `scene_image_generations_one_active_idx` 双重保护：并发点击在 provider queue 前串行 claim，同一 Record 最多一条 queued/running；非成员/未知 Record 在 active 查询和 provider 调用前由真实 scope 解析 fail-closed。
无后台 worker：route 关闭后轮询停止，活跃生成保持 running，下次点击
（或打开页面后的显式操作）继续轮询同一 prompt——文档标注为已知边界。

## 3. 下载与安全

- history 输出只取 SaveImage 节点的 `{filename, subfolder, type:"output"}`；
  filename 必须 basename（拒绝 `/`、`\\`、`..`），subfolder 仅允许
  `^[a-z0-9_-]+(/[a-z0-9_-]+)*$`i 或空；`/view` 参数 URL 编码。
- 上限 20MB；magic sniff PNG/JPEG/WebP（RIFF…WEBP）；content_type 白名单
  image/png|image/jpeg|image/webp；sha256 落库。
- 客户端只见 `/api/files/<fileId>`（既有会话+membership+403/404+immutable
  缓存路径）；provider URL/路径/prompt/body/key 不进 envelope/客户端/日志。

## 4. Delivery 与前端

- envelope 新增可选 `sceneImage: { fileId, fileUrl, status } | null`
  （latest ready；非成员/未知 Record 走既有 403/404；旧数据 null）。
  record-types normalize + recordEnvelopesEqual 兼容。
- Record header 加「生成场景图/重新生成」按钮（membership owner/player
  可点；观察者禁用）。点击 → POST dispatch:true → running：按钮转
  role=status 轮询中；ready：刷新 envelope 并以
  `style={{ "--record-scene-image": "url(/api/files/<id>)" }}` 覆盖
  `.record-scene-bg` 的 token（遮罩/装裱层不变）；failed：安全固定文案 +
  可重试。无图时完全沿用既有静态背景。不显示 prompt/远端路径/prompt_id。
- 三语 i18n；无圆角/玻璃/新视觉体系。

## 5. 旧行为兼容与边界

- `prepareSceneImageWorkflow` 不变；`dispatch:true` 语义升级为
  「queue + 落库 ready」，返回多 `status`/`fileId`/`fileUrl`；上一批的
  queue-only `dispatched:true` 应答被新语义取代（边界：旧调用方仅测试）。
- 缺本地 ComfyUI/未启用 → COMFYUI_DISABLED 409（不变）。
- 未实现：后台 worker/断线续传 UI、I2I、角色一致性、输出图管理界面。
