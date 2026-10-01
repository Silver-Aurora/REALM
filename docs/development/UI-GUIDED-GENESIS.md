# 司卷问答 · 对话式引导创世实施规范

> 文档性质：实施规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 视觉约束一律以 [`../design/UI-DESIGN.md`](../design/UI-DESIGN.md)（纸墨纪事）为准。

## 1. 背景与定位

「启笔铸界」（自由输入一次性生成）与「工笔细琢」（表单手动录入）都偏硬。本批次新增第三条、也是推荐主入口：**司卷问答**——全屏对话式引导创世。三条入口在世界库面板可自由切换：

| 入口 | 形态 | 定位 |
|---|---|---|
| 司卷问答 | 全屏引导，一次一问 | 推荐主入口 |
| 启笔铸界 | 自由文本 → 纸墨手稿 → 一键落界 | 已有完整构思时 |
| 工笔细琢 | 折叠抽屉内的手动表单 | 精确逐项录入 |

## 2. 引导节奏（当前十步）

问句不在本表硬编码：每步读取 `modules/style/world-style.ts` 的 `guided.step.*.question` 模板，按界面语言与当前 draft.style 选择；标题、输入标签和按钮仍是固定 UI i18n。文风步之前使用默认 modern，选定后后续问句立即切换。

| 步 | 名 | 问句模板 key | 落定内容 | 可留白 |
|---|---|---|---|---|
| 1 | 世界名称 | `guided.step.world-name.question` | world.name | 否（必须有名） |
| 2 | 时代背景 | `guided.step.era.question` | world.era | 是 |
| 3 | 文风 | `guided.step.style.question` | style | 是（跳过 = modern，不写 style） |
| 4 | 世界概述 | `guided.step.summary.question` | world.summary | 是 |
| 5 | 开场故事 | `guided.step.story.question` | story.title + story.premise | 是（缺标题时按创建契约处理） |
| 6 | 你的角色 | `guided.step.player-role.question` | playerRole | 是 |
| 7 | 参与方式 | `guided.step.stance.question` | player stance | 是（缺省扮演角色） |
| 8 | 同伴 | `guided.step.companions.question` | companions（0–2） | 是（可空，空则旁白推进） |
| 9 | 开场场景 | `guided.step.scene.question` | scene.location/weather/tension/objective | 全可留白（不写默认值，留给设定结晶生长） |
| 10 | 确认信息 | `guided.step.review.question` | 总览 +「创建世界」 | — |

每步操作区：纸面输入框 +「跳过」（当前必填步不可跳过）+「下一步」+「AI 代写」（仅支持的步骤）。视觉细节遵循 UI-DESIGN。


## 3. AI 代笔（候选生成契约）

- 端点：`POST /api/world/suggest`（门禁与登录态同既有 API）。
- 请求：`{ step, intent, context }`
  - `step` ∈ `world-name | era | summary | story | player-role | companions | scene`；
  - `intent` 为用户输入的一句意向（可空，≤200 字）；
  - `context` 为已定之卷的结构化快照（WorldGenesisDraft 同构，未定的字段为空串/空数组）。
- 响应：`{ ok: true, suggestions: string[] | CompanionSuggestion[] }`
  - 文本步骤：2–3 条候选字符串（各 ≤ 对应字段限长）；
  - `companions`：`{ name, role, summary }` × 1–2；
  - `scene`：以 4 个字段标签候选返回（`{ location[], weather[], tension[], objective[] }`，各 0–2 条）。
- 模型通道：既有 `modules/inference` 网关，`responseFormat: "json_object"`，thinking 下 max_tokens 下限由网关兜底；prompt 只含已定设定与意向文本，绝不含 token/密钥。
- **fail-closed**：模型错误 / 超时 / 非法 JSON / schema 不符 / 候选为空 → 端点返回 `ok:false`，前端静默退回手动输入（不弹报错、不阻塞引导）。

## 4. 已定之卷（UI 契约）

- 侧栏「已定之卷」：每落定一项浮现一条墨字条目 + 朱印方章，轻量淡入过渡（≤300ms opacity/translate），禁花哨动画。
- 跳过的步骤不显示条目，不出现「未定」「暂无」等占位文字。
- 全程直角卡片、语义色（朱红只用于落定动作与印章）、纸张底色；禁圆角/半透明/荧光。

## 5. 落库管线

- 第 8 步「落笔入界」：把已定之卷组装为 `WorldGenesisDraft`（空字段保持空串，不写默认值），调用既有 `POST /api/world/generate`（draft 分支）单事务原子落库，直接打开新记录。
- 玩家角色名在服务端绑定登录账号 displayName；companions 为空时阵容只有玩家，回合由旁白推进（既有能力）。
- 初始场景留白字段不落默认值，由设定结晶管线随对话自然生长。

## 6. fail-closed 矩阵

| 环节 | 失败形态 | 行为 |
|---|---|---|
| AI 代笔 | 模型/超时/JSON/schema/空候选 | 静默退回手动输入，不阻塞 |
| 步骤确认 | 必填步（世界名）为空 | 「落墨」禁用，提示性文案不退场 |
| 落笔入界 | 落库失败 | 保留已定之卷，notice 提示可重试 |
| 中途离场 | 关闭引导 | 不写任何数据，重开从头开始 |
| 对谈创世失败 | 司卷不可用/创建失败 | 「改用分步引导」移交完整上下文（见下） |

### 6.1 对谈 → 分步 fallback 的上下文保留（2026-10-01）

对谈创世（GuidedGenesisChat）遇到 provider 失败、玩家点「改用分步引导」时，
overlay 切换随带对谈上下文：已提交 turns（角色/正文/次序）、输入框中未发送
文本、当前完整 `WorldGenesisDraft`。分步表单从该 draft 初始化（玩家可修改
字段并继续，创建值反映修改后的 draft）；对谈历史在侧栏恢复面板只读可见；
未发送文本在独立的可编辑输入中，不伪装成历史 turn、不随表单提交；不重复发
最后一条消息、不自动创建、不持久化中途草稿；切换后首焦点仍落在创世字段。

## 7. 验收标准

1. Core 契约：候选生成 schema（各 step、数量/限长截断、非法输入拒绝）、fail-closed 分支、结构化 draft 校验。
2. PG 集成：结构化创世原子落库（含空角色场景）已由既有 genesis 测试覆盖并复跑。
3. GUI（N 组）：逐步引导八步走通、已定之卷动态浮现、留白跳过不出现占位、AI 代笔候选点选、落笔入界进入对局；既有组全部回归通过。
4. `npm test` 全绿；GUI 测试后执行 `scripts/clean-gui-test-data.sql` 清理。
