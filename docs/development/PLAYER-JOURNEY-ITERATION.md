# REALM 玩家路径设计迭代（冻结版 · 本轮实现范围）

- 基线：HEAD=a5194ee。本文只冻结**本轮已实现**的路径设计，不改写历史文档。
- 范围：登录/返回 → 入口 → 创世 → 首回合 → 持续游玩 → 探索 → 管理；桌面与窄屏。
- 不在范围：API 协议、World/Story/Record 语义、权限/RLS/worldline、模型调用、桌面打包、Settings 页 i18n 改造（单列 skipped）。

## 0. 状态图（路径视角）

```
登录/返回
   │
   ▼
入口屏（WorldOnboarding）
   │ primary: AI 助手对话        secondary: 分步引导
   ▼                              ▼
GenesisChat ──fallback──▶ GuidedGenesis ──back/exit──▶ 入口屏
   │                              │
   └──── 草稿/提案确认 ────────────┘
                 │ 创建成功（唯一回 Record 路径）
                 ▼
新 Record（首回合）→ 持续游玩（composer 主回路）
                 │
                 ├─ 探索：World → Story → Record（浏览层，随时回当前 Record）
                 └─ 管理：Library overlay（创建/导入/管理，显式回 Record 动作）
```

## 1. 页面责任边界（冻结）

| 层 | 组件 | 唯一责任 | 明确不属于 |
|---|---|---|---|
| 入口 | `WorldOnboarding` | 一个 primary 创世入口 + 一个 secondary 创世入口 + 「继续/进入已有世界」列表 | 管理工具、世界浏览 |
| 创世 | `GuidedGenesisChat` / `GuidedGenesis` | 全屏 shell（dialog 语义 + 返回/退出）+ 草稿/提案编辑 + 底部唯一确认动作 | 世界管理、导入 |
| 首回合 | Record main（heading/scene/timeline/composer） | 读卷首与场景 → 直接输入第一次行动 | 管理操作、内部状态展示 |
| 持续游玩 | `MessageComposer` + `PendingSubmission` + `PreviewCards` | 输入 → 可选行动 → 送出；等待反馈单一化；失败保稿重试 | 多通道状态噪声 |
| 探索 | `WorldNavigation` + `WorldView` + `StoryView` | World→Story→Record 层级浏览与进入；每个视图可回当前 Record | 创建/导入/删除/归档 |
| 管理 | `LibraryPanel`（overlay） | 创建（三种方式）、导入、角色/世界管理、记录删除 | 浏览主路径（已归探索层） |

## 2. 主/次动作（冻结）

- 入口屏：`is-primary` = AI 助手对话；secondary = 分步引导；已有世界为第三区（继续/进入）。**不新增**任何管理入口。
- 创世 shell：底部唯一确认（创建世界）；GenesisChat 的 fallback = 显式降级到分步引导；GuidedGenesis 的 back/exit = 显式返回；创建成功只走 `onOpenRecord` 一条回 Record 路径（loading 屏即"正在进入"反馈）。
- Record：第一主动作 = 送出行动（composer submit）；Action 面板与下一步提案为辅助；复制/Canon 为标题区次级管理动作（保留但不抢视线）。
- 探索视图：World/Story 视图内每条记录可进入；当前 Record 有三条等价回路——面包屑 record crumb（桌面）、左侧 nav 当前项、Library 的「回到当前记录」（本轮新增，见 §4）。
- Library：头部 = 「回到当前记录」（primary，有当前 Record 时）+ 关闭；内容 = 管理分组；删除/归档/姿态切换维持显式确认，不上移。

## 3. 状态层契约（冻结）

- 每个异步动作只在一处表达状态：notice（操作结果）、sync-state（连接态）、自演卡（会话态）、按钮内文案（动作进行中）。**禁止**同一事件同时出现在 notice + 状态卡 + 按钮三处。
- 等待反馈：发送中 = `PendingSubmission` 暂存卡 + 可能到达的 `PreviewCards` 流式预览；不加第三条等待提示。
- 错误：保留原文与选择（composer 恢复），提示为一句业务结果 + 一个恢复动作（重试/关闭）；**禁止** token、内部 ID、SQL、provider URL、ACL、堆栈出现在普通用户界面。
- 空态：结构性消隐 + 一句可理解说明，不写 Demo 假内容。

## 4. 本轮已实现改动（与契约一一对应）

1. **Library 显式回 Record**：头部新增「回到当前记录」primary 按钮（`ui.library.returnToRecord`，仅在有当前 Record 时渲染）；panel `aria-label` 去硬编码中文；h2 语义从"开始新的故事"改为"创建 · 导入 · 管理"（`ui.library.heading` 新值）——管理层职责在头部即讲清。
2. **GuidedGenesisChat 固定文案入 i18n**：提案卡字段 label（世界名称/时代背景/世界概述/开场故事/故事简介/同伴/开场场景）与同伴/场景 placeholder 全部迁入 `ui.genesisChat.field.*` / `ui.genesisChat.ph.*`（三语），消除破坏语言切换的硬编码中文。
3. **探索回 Record**：WorldView / StoryView header 新增显眼的「进入/回到当前记录」primary action，使用既有 `currentRecordId`，不新增 API、不改数据来源。
4. **移动上下文**：≤720px 不再整体隐藏 `.record-section`；仅隐藏非当前项，并将当前 Record 作为跨两列的可见导航项，避免用户离开故事视图后失去定位。
5. **主动作视觉**：探索视图的回 Record 按钮使用墨黑 primary 样式，Graph/管理保持 secondary。

## 5. skipped + reason（冻结）

- **Settings 页 i18n 改造**：设置页约 30 处硬编码中文 label，迁移面大且与本次玩家主线弱相关——单列后续批次，不在本轮混合。
- **ESC 关闭创世弹层**：GuidedGenesis 草稿存于组件内 state，ESC 误触会丢全部已填内容；保留显式 ×/back 路径。
- **自演卡位置/详细度**：现状单行紧凑且是唯一起拍入口，移动会隐藏功能；不改。
- **Record 标题区复制/Canon 按钮迁往 Library**：迁移会打断既有 T12 验收契约（位置被 GUI 断言锁定），保留为次级动作。

## 6. 移动/窄屏契约（冻结）

- ≤720px：面包屑与 sync-state 隐藏；左侧 nav 折叠为当前 story + 当前 record 两项（非当前项隐藏，record-section 跨两列显示）——定位由 nav 当前项与 record heading eyebrow「当前记录」共同承担；不新增 drawer。
- composer 不遮最后一条事件：timeline-scroll 底部留白 + composer 非 fixed 覆盖（既有布局）；弹层（guided/library）已有 `role="dialog" + aria-modal`。
- 不使用颜色单独表达状态（现状：文本 + 图标 + 文案三重）。
- 保持纸墨直角、低饱和语义色；不引入圆角胶囊/玻璃拟态/原生风格下拉。

## 7. 验证（本轮）

- 结构契约：入口唯一 primary、Library 回 Record 按钮存在条件、genesis-chat 字段无硬编码中文、i18n 新 key 三语完整、overlay dialog 语义（既有）。
- 受影响前端测试 + lint + typecheck + 完整 npm test。
- GUI（隔离 fixture）：已完成真实浏览器闭环——Chromium 分步创世与对谈创世、WebKit 分步创世；均真实经过登录、入口、创建 API、Record、World/Story、Library、Settings 与 375px 窄屏回路。无 page error、无 4xx/5xx 资源、无横向溢出；模型只在 Genesis Chat 用浏览器层确定性响应替代，`POST /api/world/generate`、`GET /api/record`、`GET /api/library` 仍走真实隔离服务。截图已在像素层复核后清理。
- GUI 初跑发现 WebKit 窄屏事件标题允许 CJK 逐字换行，已通过 `.speaker-block` flex 与标题单行省略规则修复，并在 Chromium/WebKit 375px 复跑通过。
