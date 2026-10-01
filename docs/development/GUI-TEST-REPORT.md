# 界核 / REALM GUI 交互测试与修复报告

> **历史快照**：本报告只记录 2026-08-14 的 A1–J4 共 41 项测试；旧 41/41 不是当前状态。2026-09-27 修复前全量基线为 124 项、69 passed / 55 failed；修复后有 17 项确定性 focused GUI 在 Chromium、WebKit 各通过，WebKit z2 邀请流程另 1/1 通过。完整套件仍待重跑，详见私有 `.hermes/reviews/promotion-readiness.md`。

> 执行时间：2026-08-14（Asia/Shanghai）。
> 测试计划：[`GUI-TEST-PLAN.md`](./GUI-TEST-PLAN.md)（用例 A1–J4）。
> 测试实现：Playwright 1.62.1，`tests/gui/`，运行说明见 `tests/gui/README.md`。
> 浏览器：Chromium（主测）+ WebKit（WPE WebKit 2.53.3，Safari 引擎近似）。
> 模型：真实 DeepSeek（deepseek-v4-flash，thinking: enabled）。

## 1. 总体结果

| 项目 | 通过 | 失败 | 跳过 |
|---|---:|---:|---:|
| Chromium（41 用例） | 41 | 0 | 0 |
| WebKit（41 用例） | 41 | 0 | 0 |

最终全量回归：`npx playwright test --project=chromium` 41 passed（14.0m）；
`npx playwright test --project=webkit` 41 passed（12.0m）。
既有自动化：`npm test`（typecheck + core + contracts + build + render）与 `npm run lint` 全部通过。

## 2. 逐用例结果

### A. 世界库

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| A1 初始加载 | 通过 | 通过 | 修复 Bug#3 后通过（世界状态渲染） |
| A2 创建世界 | 通过 | 通过 | 唯一名称隔离；自动获得原初世界线 |
| A3 创建故事 | 通过 | 通过 | |
| A4 创建记录并进入 | 通过 | 通过 | 修复 Bug#4 后断言 cast=玩家+两名 AI |
| A5 世界线分支 | 通过 | 通过 | UI 已有分支入口，显示原初/分支标签 |
| A6 timeline 标签 | 通过 | 通过 | Retrospection 标签可见，普通记录不带 |

### B. 记录页交互

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| B1 时间线加载 | 通过 | 通过 | |
| B2 语义段渲染 | 通过 | 通过 | 五色值、无底色/边框/内边距、正文占满 |
| B3 自然输入提交 | 通过 | 通过 | 修复 Bug#1/#6/#7 后 201 稳定 |
| B4 括号动作+台词 | 通过 | 通过 | 修复 Bug#2（拆分器） |
| B5 乐观发送与清理 | 通过 | 通过 | 在途禁用态用 1s 轮询窗断言（在途文案变为“处理中”，按 type=submit 定位） |
| B6 失败恢复 | 通过 | 通过 | 路由中断模拟失败：草稿移除、输入恢复、无幽灵事件、可重试 |
| B7 语义流式呈现 | 通过 | 通过 | 观察到中间短文本态，最终完整 |

### C. 行动面板

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| C1 能力投影 | 通过 | 通过 | 技能/物品/姿态/场景四层（演示记录） |
| C2 选择行动→草稿预填 | 通过 | 通过 | |
| C3 提交行动 | 通过 | 通过 | 提交后选择态清理；演示记录同一行动可多次提交，断言最新一条 |
| C4 过期行动处理 | 通过 | 通过 | 修复 Bug#5（409 后跳出记录） |

### D. 可见性确认

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| D1 密谋预判 | 通过 | 通过 | 428 + 确认卡片，未确认不落库 |
| D2 仅目标可听 | 通过 | 通过 | 201 + 受限可见性标签，卡片消失 |
| D3 公开发送 | 通过 | 通过 | 201 + 公开 |
| D4 取消确认 | 通过 | 通过 | 草稿保留可编辑 |

### E. 场景检查器

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| E1 角色记忆卡片 | 通过 | 通过 | 前置真实回合产生观察后内容非空 |
| E2 三档深度切换 | 通过 | 通过 | 真实接口重读、按钮态正确、375px 不溢出 |
| E3 关系线索 | 通过 | 通过 | |
| E4 整理摘要 | 通过 | 通过 | 入口与进行态反馈存在；模型结果不作为断言 |

### F. 模型设置页

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| F1 基本信息 | 通过 | 通过 | DeepSeek / api.deepseek.com / 密钥掩码 |
| F2 模型发现 | 通过 | 通过 | flash + pro |
| F3 模型选择与 think | 通过 | 通过 | flash 选中、thinking=enabled |
| F4 连接测试 | 通过 | 通过 | 修复 Bug#1 后真实端点成功 |
| F5 数据边界说明 | 通过 | 通过 | |

### G. 实时同步

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| G1 SSE 实时事件 | 通过 | 通过 | 双会话，无需刷新 |
| G2 断线回补 | 通过 | 通过 | 事件流端点中断→正在重连→恢复回补 |

### H. 响应式 / I. 纸墨设计语言 / J. Safari·WebKit

| 用例 | Chromium | WebKit | 备注 |
|---|---|---|---|
| H1 桌面三栏 | 通过 | 通过 | |
| H2 平板双栏 | 通过 | 通过 | 检查器移至主记录下方 |
| H3 手机纵向 | 通过 | 通过 | 无横向溢出 |
| I1 直角 | 通过 | 通过 | 关键元素 border-radius=0 |
| I2 语义色 | 通过 | 通过 | 墨黑主按钮/朱红轨迹/墨色正文 |
| I3 硬阴影 | 通过 | 通过 | 3px 3px 0，无模糊漂浮 |
| J1 视口高度回退 | 通过 | 通过 | 见 §4 Safari 说明（text-size-adjust 检查方式） |
| J2 滚动与弹层 | 通过 | 通过 | 聚焦无位移，弹层在视口内 |
| J3 窗口 resize | 通过 | 通过 | grid↔flex 正确切换，无残留错位 |
| J4 差异修复回归 | 通过 | 通过 | 未发现 Chromium 正常而 WebKit 异常的窗口/视口 bug；J1–J3 即回归 |

## 3. 发现的 Bug 与修复

### Bug#1 thinking 模式下 max_tokens 被推理 token 耗尽（模型层，阻塞全部回合）

- **现象**：F4 连接测试 502（MODEL_RESPONSE_INVALID）；所有消息提交 422 TURN_FAILED。
- **根因**：DeepSeek thinking 模式下 `max_tokens` 同时覆盖推理与正文。连接探针 `maxTokens: 32`、编排各结构化调用 100–320 的预算被推理全部吃掉，正文为空（`finish_reason: "length"`，usage 显示 completion_tokens == reasoning_tokens）。
- **修复**：`modules/inference/deepseek-gateway.ts` 对 thinking 启用的请求强制 `max_tokens` 下限 8192（chat 与 streamChat 统一）。
- **验证**：探针 200（约 2s）；真实回合 201；F4 双浏览器通过。

### Bug#2 括号动作 + 台词不拆分（B4）

- **现象**：输入“（环顾四周）这里好安静。”提交后整段存为单个 action 语义段，台词未着色。
- **根因**：`fallbackCompositeSemanticSegments` 只按引号恢复台词边界，不识别跑团惯用的括号动作记法。
- **修复**：`modules/presentation/semantic-segments.ts` 兼容解析器识别全/半角括号动作为 action 段；括号存在时玩家输入的裸文本按台词处理；引号内括号不重复切分。新增 2 项单元测试（该文件共 6 项通过）。
- **验证**：B4 双浏览器通过（action 段“环顾四周”+ dialogue 段“这里好安静”）。

### Bug#3 世界库条目缺少状态显示（A1）

- **现象**：世界条目只有名称/纪元/摘要，计划要求的状态未渲染（数据已在快照中）。
- **修复**：`app/components/library-panel.tsx` 世界标题行增加状态小字（mono、苔绿、大写，符合纸墨规范）；`app/globals.css` 配套样式。
- **验证**：A1 双浏览器通过。

### Bug#4 新建记录阵容重复装配玩家定义（A4）

- **现象**：演示世界新建记录 cast 达 5 人，“洛川”出现 3 次（人类玩家 + 2 个 AI 洛川）。
- **根因**：`modules/application/library-service.ts` 的 customCharacters 查询取出世界全部角色定义，包含种子玩家定义 `char_def_player` 与装配自动创建的玩家定义 `char_def_player_<world>`，玩家被当作自定义 AI 角色重复装配。
- **修复**：查询排除上述两类玩家定义 id；新记录阵容保持默认玩家 + 两名 AI 角色。
- **验证**：新建记录 cast=洛川(human)/塞娜(ai)/弥洛(ai)；A4 双浏览器通过。
- **备注**：修复前已创建的旧测试记录保留历史 5 人阵容（历史数据不改写）；`char_def_player_world_ember_coast` 定义行保留但不再被装配引用。

### Bug#5 409 写入冲突恢复后跳出当前记录（C4）

- **现象**：多会话提交产生 409 后，界面自动重载到演示默认记录，用户当前记录上下文丢失。
- **根因**：`app/realm-client.tsx` 冲突恢复调用 `loadRecord()` 未传 recordId，服务端返回默认记录。
- **修复**：冲突时传入当前 `projection.record.id` 重载。
- **验证**：C4 断言冲突提示可见且仍停留在当前记录，改自然输入可继续提交；双浏览器通过。

### Bug#6 模型复核误否决阻塞合法玩家输入（422 主因之一）

- **现象**：普通自然输入以约 30–50% 概率 422（TURN_FAILED / DM_OUTPUT_REJECTED）。
- **根因**：①复核器只看到目标与候选摘要，看不到玩家输入，无法正确判断 goalSatisfied；②温度 0 下同一候选的否决被固化，“第二次独立复核”不独立；③复核输出格式损坏被当作否决。
- **修复**：`validate` 链路新增可选 `playerText` 并传给复核器（`modules/orchestration/public.ts`、`modules/orchestration/model-powered.ts`、`modules/application/local-record-service.ts`）；复核改为连续三次一致否决才拦截（第三次温度 0.4 且明确“候选已通过全部确定性校验，无硬性违规一律放行”）；复核 JSON 损坏不再视为否决。确定性校验与真实拦截能力保留。
- **验证**：实测回合成功率由约 40–50% 升至约 90%（10 次实测 9 次 201）；既有 46 项 core 测试通过。

### Bug#7 thinking 模式流式空正文（422 主因之二）

- **现象**：回合偶发 CHARACTER_RESPONSE_INVALID / DM_PLAN_INVALID，正文为空。
- **根因**：DeepSeek thinking 模式偶发只输出推理、`finish_reason: "stop"` 但正文为空（实测 SSE 47 个数据行、completion_tokens == reasoning_tokens）。
- **修复**：`chatWithStream` 对空正文就地重试流式 3 次，仍空则回退非流式 `chat`；非流式也无正文时抛 `RetryableTurnError` 进入 Turn 级安全重试；结构化格式失败（截断/空内容）在 `modelCall` 就地重试一次，与语义否决区分。
- **验证**：同 Bug#6 实测；服务端保留 `[realm]` 诊断日志（turn 失败码、复核否决标记、空正文回退），仅本机控制面可见。

## 4. Safari / WebKit 专项说明（J 组）

- **结论**：未发现“Chromium 正常而 WebKit 异常”的窗口/视口 bug。既有 `100dvh` → `-webkit-fill-available` 回退链、文本缩放保护与移动端布局在 WPE WebKit 2.53.3 实测生效（J1–J3 通过）。
- **text-size-adjust 检查方式**：WPE 引擎不在 `getComputedStyle`/CSSOM 中暴露 `-webkit-text-size-adjust`（该属性由 iOS Safari 实现）。J1 对该项改用样式表原文检查（任务允许的“CSS 回退链检查”替代）；引擎暴露该属性时仍断言计算值为 `100%`。
- **WebKit 运行环境（Arch Linux 限制与处置）**：本机为 Arch Linux，Playwright 官方不支持；WebKit 运行所需 `libxml2.so.2` / `libicu74` / `libflite` 已从 Ubuntu 24.04 官方仓库提取至 `~/.cache/webkit-deps` 并软链进 `~/.cache/ms-playwright/webkit-*/minibrowser-{gtk,wpe}/sys/lib`。属本机浏览器运行环境补丁，不进入仓库；在 macOS/Ubuntu 上无需此步骤。
- **dev server 会话说明**：测试中途的两次服务不可达为沙箱会话回收后台进程所致（非应用缺陷）；dev server 现以 `setsid` 独立会话常驻。

## 5. 已知残余风险（非 UI bug，按任务约定处置）

- 真实模型回合仍有约 10% 概率被复核一致否决（多为 DM 规划目标与角色激活自相矛盾，复核按设计拦截）。GUI 测试按计划“模型偶发失败不算 UI bug”，以用户视角重试（最多 4–5 次）覆盖。
- 模型耗时 15–60s/回合，全量双浏览器回归约 25–30 分钟。

## 6. 复现方式

```bash
npx playwright test                      # 双浏览器全量
npx playwright test --project=chromium   # 主测
npx playwright test --project=webkit     # Safari 引擎近似
```

详见 `tests/gui/README.md`。

## 2026-09-28 06:42 +08:00 — M1/M2 隔离 Chromium 回归

- 使用 `node scripts/test-gui-with-scratch.mjs --project=chromium tests/gui/m-scene-crystallization.spec.ts`：M1、M2 `2/2 passed`。每次 GUI 写入均使用 disposable PostgreSQL、隔离 Web 服务与 loopback deterministic fake provider。
- M2 首次提交实测返回 `409 WRITE_CONFLICT`；页面显示冲突提示、恢复原草稿并重载 envelope。玩家再次确认提交后返回 `201`，倒退候选触发 `scene-extraction-m2-regression` 与 `scene-adjudication-m2-reject`，观察窗口结束后的场景读回仍为 3 月 2 日，未倒退到 3 月 1 日。OCC token 语义未放宽，也未增加自动重发。
- 回归中发现并修正 fake-provider fixture 匹配历史文本的缺陷：现在按 prompt 的 `[Player's exact words]` 区块选择当前场景 fixture；契约用真实场景上下文覆盖历史场景与当前输入冲突。fake-provider 单测 `11/11`、GUI isolation/scratch config `5/5`。
- 当前树 `npm run typecheck`、`npm run lint`、`npm test` 均 exit `0`；`test:core 405/405`、`test:contracts 5/5`、build 完成、render `3/3`。
- 本轮 scratch 目录和 GUI 容器读回为空；`realm-dev.service` 仍 active，LAN listener 未改。此为 M1/M2 定向验收，不代表全量 Chromium/WebKit GUI、真实 provider smoke 或推广验收通过；其余 findings 未清，NO-GO 保持。
