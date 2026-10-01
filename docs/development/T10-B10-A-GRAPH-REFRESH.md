# T10-B10-A · 知识图谱面板刷新闭环

> 批次 T10-B10-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批；T10-B3 起多轮候选「图谱面板刷新体验」收口）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=2c14ade；本批只做刷新体验——不改 API、鉴权、数据库、迁移、inference，不改 POST 编辑语义与实体/Claim/关系/Article 数据契约。

## 一、当前缺口（2026-08-21 代码复核）

- `app/components/knowledge-graph-panel.tsx` 的 `load()` 仅两个触发点：初次
  effect（setTimeout 0）与 POST 编辑成功后回读。外部数据（其他客户端、API
  直连、回读管线写入）产生后，用户没有任何显式入口让新数据进入视图，只能
  关闭重开面板。
- `load()` 无加载状态：请求中无可见反馈，失败只有一行 notice，无明确重试
  路径（重开面板不算）。
- `load()` 无竞态/卸载保护：旧响应可覆盖新响应；卸载后 setState 无防护；
  非 2xx 响应不检查，`json()` 静默产出空快照冒充「没有数据」。
- GUI 围栏 K1–K6 覆盖加载/分色/下钻/编辑立即反映/审核/设计语言，缺
  「外部数据产生后点击刷新可见」的契约用例。
- 两个 GET（`/api/world-knowledge`、`/api/canon`）已显式携带 worldId 与
  `cache: "no-store"`（T9 收口事实），本批保持不动。

## 二、刷新状态机

- 状态：`loading`（布尔）+ `loadedOnce`（布尔，区分初次加载与手动刷新文案）。
  初值 `loading=true`（初次 effect 立即触发），`loadedOnce=false`。
- 初次加载：effect → `load()`；状态文案「正在加载图谱…」（`role="status"`），
  「刷新图谱」按钮禁用；初次尝试结束（成功或失败）置 `loadedOnce=true`。
- 手动刷新：点击「刷新图谱」→ `load()`；按钮禁用、文案「刷新中…」，状态
  文案「正在刷新…」；成功后更新 snapshot 与 proposals；失败后 notice
  「暂时无法读取世界知识。」，按钮恢复可点——重试路径即再次点击。
- POST 编辑进行中（`busy`）刷新按钮同样禁用；POST 成功后沿用既有 `load()`
  回读，语义不变。

## 三、双端点一致性

- 一次刷新以 `Promise.all` 同时 GET `/api/world-knowledge` 与 `/api/canon`，
  显式 worldId 与 `cache: "no-store"` 保持。
- 任一端点失败（网络错误或 非 2xx）整体视为失败：保留旧 snapshot 与旧
  proposals，不做部分更新——不允许出现「图谱新、canon 旧」的撕裂态。

## 四、worldId/权限边界

- worldId 来自组件 props，显式 `encodeURIComponent` 进 query；不引入跨世界
  读取，不缓存跨世界数据。
- 权限判定留在服务端（会话/token 门禁不变）；本批不触碰任何 API 路由、
  鉴权逻辑与数据库授权。

## 五、竞态/卸载语义

- 每次 `load()` 递增请求序号，仅最新序号的响应允许落 state——旧请求不得
  覆盖新请求（含快速连续触发与 POST 回读交叠）。
- `mountedRef` 在 effect cleanup 置 false，任何异步分支在卸载后不再
  setState。
- 选中实体：刷新成功后若 `selectedId` 仍存在于新快照则保留；已消失才清空
  （回「点击左侧节点」空态），不扩大到其它 UI 状态。
- 失败可诊断：非 2xx 显式 `throw` 进 notice 分支，不再静默 `json()` 出空
  快照；notice 文案与既有失败提示一致，不泄露内部细节。

## 六、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 初次加载失败 | notice「暂时无法读取世界知识。」+ 按钮可点重试 |
| F2 | 手动刷新失败 | 同 F1；旧 snapshot/proposals 保留不清空 |
| F3 | 快速连续点击/与 POST 回读交叠 | loading/busy 期间按钮禁用 + 序号防旧响应覆盖 |
| F4 | 请求中组件卸载 | mountedRef 阻断 setState，无泄漏告警 |
| F5 | 刷新后选中实体消失 | 清空 selectedId 回空态；仍存在则保留 |

## 七、验收标准

- 新增 focused GUI K7（外部 API 造实体 → 打开图谱确认不可见 → 点击
  「刷新图谱」→ 新节点可见）Chromium exit 0；WebKit 若快速可跑另记，
  否则只记 Chromium。
- 受影响 `npx eslint`、`npm run typecheck`、
  `tests/api-core-wiring.test.mjs`、`tests/documentation-layout.test.mjs`、
  `git diff --check` 原始 exit 0；不跑全量、不跑完整 GUI 套件。
- 收口前工作树 clean、无临时文件/数据库对象；realm-dev.service 保持
  active。

## 八、范围外清单（本批不做）

POST 编辑语义与回读时机、实体/Claim/关系/Article 数据契约、任何
API/DB/migration/inference 行为、图谱布局算法与节点渲染、Canon 审核交互、
自动轮询/SSE 推送刷新、其它图谱/Canon 已知问题、WebKit 专项。

## 九、交付步骤

1. 本规范 + docs 索引（单独 commit）；2. 面板刷新实现 + K7（独立 commit）；
3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
