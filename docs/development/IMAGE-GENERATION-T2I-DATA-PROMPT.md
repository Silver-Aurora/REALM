# T2I 真实数据 Prompt 接线方案（IMAGE-GENERATION-T2I-DATA-PROMPT）

基线：HEAD `bdc0657`。范围：只做 T2I 场景建立图的真实数据 prompt 接线 + 可执行
workflow payload 的 server-only 准备 seam；不做 I2I/ControlNet/AnimaLLLite/
角色一致性/风格切换/自动触发/远端 dispatch/DB schema。

## 1. 真实数据来源与调用图（生产入口）

唯一生产数据入口 = 回合管线同一个 scope 仓库：

```
app/api/record/scene-image/route.ts          （本批新增，POST，session 门禁）
  → modules/application/scene-image-service.ts
      prepareSceneImageWorkflow({ workspaceId, principalId, recordId })
  → database/postgres/record-scope.ts
      createPostgresRecordRuntimeScopeRepository(pool).resolve(...)   ← 生产只读事务
        · 主事务：record/world/story/scene 快照（WorldSceneBrief 全部字段、
          displayTime、style）、recentPublicEvents（policy_kind='public'
          且 ≤ Record head cursor 的 committed 事件，≤12 条、每条 ≤400）
        · 独立只读事务 readCanonAndLore：canon（story_canon 以上 +
          security/temporal 双闸门，≤12×160 字）、worldLore（G4
          attested-only 合格 excerpt）、recordKnowledge
  → modules/imagine/scene-prompt.ts        composeScenePrompt(scope)   ← 纯函数
  → modules/imagine/workflow-patch.ts      patchWorkflowGraph(manifest, graph, inputs)
      · manifest = workflows/realm/anima-scene-v0.manifest.json
      · graph    = workflows/realm/anima-scene-t2i-v0.api.json
```

- 不使用单测夹具/导出符号充当数据源；PG 接线测试用 scratch 库 +
  `seedPostgresDemo` + 真实 `createPostgresRecordRuntimeScopeRepository`。
- scope 缺 record/非成员/未初始化 → service 返回结构化错误（沿用
  NOT_FOUND/UNAUTHORIZED 语义），不伪造 prompt。
- canon 读取失败沿用 scope 仓库既有 fail-closed（空串），prompt 对应块省略。

## 2. Prompt 分层与顺序（positive）

固定顺序（测试钉住），缺失字段整块省略（不输出空标签）：

1. **固定质量/Anima 视觉基底**（常量，沿用 graph node 4 的质量/风格前缀：
   `masterpiece, best quality, score_7, …` —— 保留为生产 composer 常量；
   示例场景句 "quiet coastal village…" 从生产路径移除，不再出现）。
2. **World**：worldName / era / summary。
3. **Story**：storyTitle / premise。
4. **Scene**：location / weather / displayTime / tension / objective。
5. **Canon**：brief.canon 行（已是双闸门后的正史）。
6. **World lore**：brief.worldLore 行（attested-only 合格 excerpt）。
7. **当前叙事焦点**：recentPublicEvents 最新 ≤3 条（仅 public、≤cursor，
   每条再截 160）。

negative：固定约束串（与 graph node 5 当前内容一致，提升为 composer 常量）。
不进入 prompt 的字段：player 原始输入、playerText、任何 restricted/private
事件、participant/principal id、内部游标、角色 profile（第一版不强制角色出镜）。

## 3. 安全/边界

- 动态内容一律是**内容块**（labeled `World:` / `Scene:` … 纯文本行），不是
  指令；prompt injection 文本只能作为画面描述被采样器消费。
- 长度预算：每字段截断（worldName/era 80、summary 240、storyTitle 80、
  premise 240、location 80、weather/tension/objective 80、displayTime 60、
  事件 160×3）；positive 总预算 1600 字符（超出从后向前丢块，保基底）。
- 控制字符剥离（保留 `\n` 作为块分隔）；空字段 fail-closed 省略。
- 不写入/回显绝对路径、凭据、内部字段；语言保持世界内容原文
  （不翻译，沿用世界语言约定）。
- seed/width/height/outputPrefix 默认取自 graph 现值（896×1152、
  `realm_anima_scene_v0`），调用方可覆盖 seed；node id 只允许来自 manifest。

## 4. manifest binding 与 provider 边界

- patcher 只按 manifest `bindings` 反查 node/input 注入；业务代码零硬编码
  node id（契约测试扫描保证）。
- 本批交付的是 **payload 准备 seam**：route 返回 `{ workflowId, seed, width,
  height, positivePrompt, negativePrompt, patchedGraph, dispatched: false }`；
  **尚未实现远端 dispatch**（无 ComfyUI client/provider），文档与本文件即
  边界声明。未来 provider 只消费 patchedGraph，不再接触 scope/prompt。
- I2I binding 已在 manifest 中就位，本批不消费。

## 5. 测试矩阵

- `tests/scene-image-prompt.test.ts`：sentinel 全进 positive、顺序稳定、
  缺字段省略、长度/控制字符 fail-closed、negative 稳定、无 demo 示例句。
- `tests/scene-image-workflow.test.ts`：patched graph 的 4/5/6/7/9 节点值与
  语义输入一致；patcher 不从业务代码读 node id；未知 binding 报错。
- `tests/postgres-scene-image.test.ts`（scratch PG）：经真实 scope 仓库 →
  service → composer → patched graph；demo 世界/场景数据真实进入 prompt；
  未初始化/未知 record fail-closed。
- `tests/scene-image-wiring-contract.test.mjs`：route 静态围栏
  （session 门禁、server-only import）、client bundle（app/components 与
  realm-client）不得 import modules/imagine 或 scene-image-service；
  manifest/graph 不被 client 引用。

## 当前实现边界（2026-09-20 修正）

历史段落记录的是第一版完整数据接线，不代表当前人物管线已开启。当前 T2I 只生成**无人物场景建立图**：

- positive 只保留视觉 profile、World name/era、Scene location/weather/displayTime/tension/objective；
- 不再把 Story、长 world summary、Canon/Lore 原文或 recent public events/dialogue 送入图像 prompt；
- positive 明确加入 environment-only / no people / no humanoid figures；negative 同步排除人物、肖像、对白和 speech bubble；
- 人物出镜、角色参考图、角色一致性和 I2I/ControlNet 仍是后续独立管线，不在本批实现范围。
