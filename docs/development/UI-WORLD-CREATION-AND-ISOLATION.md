# 创世体验革新与世界数据隔离实施规范

> 文档性质：实施规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 视觉约束一律以 [`../design/UI-DESIGN.md`](../design/UI-DESIGN.md)（纸墨纪事）为准。

## 1. 背景与问题

身份鉴权与多世界创建上线后，新建世界暴露三类问题：

1. **鬼影角色**：`assembleDefaultRecord` 对任何新世界都硬编码注入「洛川 / 塞娜 / 弥洛」，新世界阵容被演示世界角色污染；人类玩家名不随登录账号变化。
2. **跨世界文字污染**：前端 `normalizeRecordProjection` 在切换记录时用旧投影兜底，旧世界的世界名 / 故事名 / 场景文字会渗入新世界；场景空字段展示「未记录」「平稳」「地点未定」等死板占位文字。
3. **导航树静态**：交付投影不返回 stories / records 列表，`WorldNavigation` 只能靠前端拼凑，拿不到动态树。

## 2. 启笔铸界（自然语言一键创世）

### 2.1 交互流程

1. 世界库面板顶部为主推入口「启笔铸界」：大面积宣纸质感文本框，接受任意构思段落或灵感提示。
2. 提交后调用 `POST /api/world/generate`（`{ prompt }`）生成分段草稿，在前端以「纸墨手稿」卡片展开预览，所有字段可微调。
3. 用户点击「落笔入界」：`POST /api/world/generate`（`{ draft }`）在**单个事务**内原子创建全部实体，前端直接打开新记录。
4. 既有手动表单保留为「工笔细琢」折叠抽屉，默认收起。

### 2.2 生成契约（WorldGenesisDraft）

```json
{
  "world": { "name": "", "era": "", "summary": "" },
  "story": { "title": "", "premise": "" },
  "record": { "title": "" },
  "playerRole": "",
  "companions": [{ "name": "", "role": "", "summary": "" }],
  "scene": { "location": "", "weather": "", "tension": "", "objective": "" }
}
```

- 模型通道：复用 `modules/inference`（LAN LM Studio OpenAI-compatible，`responseFormat: "json_object"`），由模型设置服务提供 gateway；凭证只存在于 `.env.local` / `.local/`，不落库不进日志。
- **mock 降级**：模型未配置、超时或返回非法 JSON 时，使用确定性本地生成器从 prompt 提炼草稿，接口仍返回 200 并以 `source: "fallback"` 标注（模型成功时为 `"model"`）。
- 人类玩家不入 draft：名称在创建时绑定当前会话 principal 对应 accounts 的 `displayName`；`playerRole` 只描述其在本世界的角色定位。
- 同伴 1~2 名，必须契合该世界观；禁止复用演示世界角色名。

### 2.3 原子创建（createGenesis）

`LibraryService.createGenesis(scope, draft)` 在一个 `withWorkspaceTransaction` 内依次写入：

1. worlds（settings 携带 `era / displayTime / weather / tension`）；
2. worldlines（原初世界线）；
3. player_world_memberships（owner，ON CONFLICT DO NOTHING）；
4. character_definitions（同伴阵容）；
5. stories（开幕故事）→ records + record_heads（初始记录）；
6. `assembleDefaultRecord`：玩家（human，principal 绑定）+ 同伴（ai）+ 初始场景（location / objective）+ public 可见性策略。

任何一步失败整体回滚；返回 `{ worldId, storyId, recordId }`。

## 3. 数据隔离与鬼影清除

### 3.1 assembleDefaultRecord 重构

- 人类角色（controller `human`）名称解析顺序：
  1. `accounts` 表中当前 `scope.principalId` 的 `display_name`；
  2. 本世界种子提供的人类角色定义（`char_def_player`，数据驱动，供演示世界延续自身设定）；
  3. 中性名「旅人」。
- 角色定位同理：优先草稿 / 种子定义，缺省为空（UI 空态消隐）。
- 世界无自定义角色定义时**不再**注入任何默认 AI 角色；阵容 = 玩家 + 本世界自定义角色。
- 初始场景 `location` / `objective` 缺省为空串，禁止写入「尚未命名地点」。

### 3.2 运行时玩家解析（record-scope）

`RecordRuntimeScopeRepository.resolve` 的玩家席位解析改为：精确匹配 `participant.principal_id` → 人类 controller → 首名角色；删除按角色名兜底的逻辑。

### 3.3 交付投影动态树

`delivery-projection` 在投影中真实查询并返回：

- `stories`：当前 worldline 下的全部故事（id / title / status）；
- `records`：当前故事下的全部记录（id / title / status / worldTime）。

status 与 worldTime 属动态知识，遵从 `can_view_dynamic_knowledge` 门禁（无权限时置空，与既有字段一致）；标题属于导航元数据，始终可见。

### 3.4 前端 fallback 隔离

- `normalizeRecordProjection` 仅当 `fallback.id` 与本次载荷的记录 id 一致时才允许继承（同记录的乐观草稿场景）；跨记录一律不得继承任何字符串或列表。
- 缺失字段的默认值一律为空串，由 UI 结构性消隐；前端不再持有「未命名世界」「未记录」「平稳」等占位文字。

## 4. 空态消隐与动态生长

- 场景检查器：世界时间 / 天气 / 局势 / 当前目标为空时整行不渲染；地点未定时标题显示所属故事名，再缺省为「——」。
- 阵容为空时整个 Cast 卡片消隐，不展示空态说明文字。
- 空记录时间线（0 事件）：卷首意象「长卷初展，诸事未定」，副行引导落笔，视觉聚焦底部输入框。
- 场景数据随推演事件写入后自然显现（字段存在即渲染），无需额外占位。

## 5. 测试要求

- Core 契约：草稿规整（非法 JSON / 缺字段 / 同伴超量截断）与本地降级生成器。
- 前端契约：跨记录 fallback 不继承；空场景字段归空；动态知识关闭时导航状态置空。
- PG 集成：新世界记录阵容无演示角色；accounts displayName 绑定；createGenesis 原子性（世界 / 故事 / 记录 / 阵容 / 场景一次成型）；投影 stories / records 列表。
- GUI：自然语言创世全流程、新世界数据隔离断言、场景空态留白断言；既有用例同步更新空态文案断言。

## 6. 明确不做

- 多世界并行生成、生成历史与版本对比。
- 玩家多角色阵容（仍一席人类玩家）。
- 创世过程的流式预览（手稿一次性返回）。
