# 图像生成 Stage 1+2：视觉 profile / ComfyUI 配置 / 原生 dispatch

基线：HEAD `2e0eeb9`。范围：视觉风格 profile + prompt 注入；ComfyUI 配置存储 +
WebUI 设置卡；原生 REST client + 显式 dispatch seam。不做 I2I/ControlNet/
AnimaLLLite/角色一致性/输出下载/图片入库/前端背景替换/异步 job 轮询。

## 1. 视觉 profile 与叙事文风的关系

- `WorldStyle`（modern/classical/western_fantasy/anime）是**叙事文风**，
  `describeWorldStyle` 的中文行文约束只服务文本生成，绝不当图片 prompt。
- 图像侧另建 `modules/imagine/visual-profiles.ts` 的 `IMAGE_VISUAL_PROFILES`
  （键入同一 WorldStyle）：每个 profile 是英文视觉约束——medium/rendering/
  line/lighting/palette/composition + 风格漂移 negative；未知/缺失
  fail-closed 到 modern。
- profile 不强制角色出镜（场景建立图优先；continuity negative 固定排除
  角色/文字/水印/UI 漂移）。

## 2. Prompt 分层（固定顺序，契约测试钉住）

```
Anima quality base（既有常量）
→ Visual profile（固定视觉层，预算内不可被动态数据挤掉）
→ World → Story → Scene（displayTime 同行）
→ Canon → Lore（双闸门后的既有 scope 内容）
→ Narrative focus（公开事件 ≤3）
negative = 既有固定负面 + profile 漂移负面
```

- 预算：positive 总 1600 字符不变；profile 是固定常量块（自身 ≤400），
  预算丢块顺序不变（focus→lore→canon 先丢），profile 与 base 永不丢。
- 授权边界/控制字符/空字段 fail-closed 全部沿用既有 composer。

## 3. 配置存储与安全边界

- 独立 store `modules/imagine/comfyui-settings.ts`（`comfyui-settings.json`，
  不复用模型 provider 文件）；目录 0700、文件 0600、临时文件 rename 原子写
  （与 local-settings.ts 同一手法）。
- 字段：`enabled`（默认 false）、`baseUrl`、 `requestTimeoutMs`
  （5s–120s）、`workflowId`（默认 `anima-scene-t2i-v0`）、`apiKey`（可选，
  只存服务端，public snapshot 只回显 `apiKeyConfigured`，永不回显明文）。
- baseUrl 校验：仅 http/https；禁内嵌凭据/query/fragment；路径有界；
  **默认值经 `REALM_COMFYUI_BASE_URL` 环境变量覆盖，内置缺省为 loopback
  `http://127.0.0.1:8188`（server-only，绝不进 client bundle）**。
- 8787 的 comfyui-mcp 包装层不是 REALM 的 provider；REALM 只接原生 REST。

## 3.1 Operator 门禁与出站边界（2026-09-30 M7 收口）

- ComfyUI 设置是**全局 operator 能力**：唯一授权来源是服务端环境变量
  `REALM_OPERATOR_PRINCIPALS`（逗号分隔的精确 principal ID 列表；
  空缺/空白 = 无 operator = 全部 fail-closed 403；无通配符）。
  principal 只来自已验证 session / 本地单用户 fallback；request
  body/query 自报与 world membership role=owner 均不构成 operator。
  GET/PUT/POST `/api/settings/comfyui` 同一门禁；403 在任何设置读取/
  网络请求之前返回，响应不含设置/URL/key。
- 出站地址策略（本机/LAN 定位）：只允许 loopback（127/8、::1）与
  RFC1918/ULA（fc00::/7）；link-local（169.254/16、fe80::/10，含云
  metadata 段）、CGNAT、unspecified、multicast、公网一律拒绝。
  IPv4-mapped IPv6 按嵌入 v4 同规则。IP literal 保存期即校验；hostname
  在连接时一次解析 A+AAAA 全集，任一答案越类即在 connect 前拒绝。
- client 重写为 node:http/https + 自定义 lookup pinning（连接只用本次
  已验证解析结果，无 check-connect TOCTOU；HTTPS 保留默认证书校验与
  原始 host SNI）；redirect 显式 fail-closed（3xx = COMFYUI_REJECTED，
  Authorization/body 绝不离开原始目标）；单一总 deadline 覆盖
  DNS/连接/headers/完整 body 消费；JSON 响应上限 1 MiB、图片 20 MiB
  均流式计数（chunked/无 Content-Length 越界即 cancel）。
- WebUI：非 operator 只见固定权限状态（不渲染表单/按钮，不显示设置值）；
  operator 正常保存/测试。launcher `safeParentEnv` 显式透传非敏感的
  `REALM_OPERATOR_PRINCIPALS`。

## 4. Transport / prepare vs dispatch 状态语义

- `modules/imagine/comfyui-client.ts`：`systemStats()`（GET /system_stats，
  连接测试）、`queuePrompt(graph)`（POST /prompt，body `{prompt: graph}`）；
  AbortController 超时；错误映射 COMFYUI_UNREACHABLE/COMFYUI_REJECTED/
  COMFYUI_INVALID_RESPONSE，静态文案，不回显 provider body/URL/凭据。
- scene-image service：`prepareSceneImageWorkflow`（prepare-only，语义不变）
  + `dispatchSceneImage`（prepare → 读配置 → disabled/未配置 fail-closed
  `COMFYUI_DISABLED` → client.queuePrompt → 返回 `{ dispatched: true,
  promptId, workflowId }`）。route：`POST /api/record/scene-image`
  body `{dispatch?: boolean}`，缺省/False = prepare-only（测试安全），
  显式 true 才真实 POST /prompt。dispatch 成功只代表 queue accepted，
  不声称图片已生成。

## 5. WebUI 设置卡

settings 页新增独立「图像生成 / ComfyUI」卡（模型卡下方）：启用开关、
baseUrl、超时、连接测试按钮、保存；`role="status"` 成功/失败提示；
API key 只显示 configured 占位。直角纸墨，无玻璃/圆角；三语 i18n 走
messages-settings.ts；错误为固定安全文案。

## 6. 验收与边界

- focused：profile/patcher/settings store/client/route/service dispatch；
  生产路径测试 = route/service 工厂 + 真实 store（tmp 文件）+ 注入 client
  fake；真实本机 smoke 走 `scripts/comfyui-local-smoke.mjs`
  （GET /system_stats + POST /prompt 真实提交 + /queue 回读 prompt_id；
  缺服务时 fail-closed 非零退出，不进任何测试门）。
- 本批不做：输出文件下载、图片入库、前端背景替换、异步状态轮询、
  I2I/ControlNet/AnimaLLLite/IP-Adapter/角色一致性。
