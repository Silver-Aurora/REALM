# M4 补全实施规范：流式 Preview、打断与多人发言权

> 文档性质：实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 前置文档：[ROADMAP M4](../development/ROADMAP.md)、[M2 语义呈现](./M2-SEMANTIC-PRESENTATION.md)、[M2 实施规范](./M2-ORCHESTRATION-AND-TOOLS.md)、[M3 补全规范](./M3-MEMORY-COMPLETION.md)。
> 开发边界：全部状态面位于本机；多模型并行只使用已配置的 LAN LM Studio 并发请求或 Fake Provider，不引入第二个外部供应商；不改动 0001–0009 既有迁移。

## 1. 核心原则

1. **正式历史只来自原子提交**。任何流式内容在 Turn 原子发布前都是 Preview：不落库、不进 SSE 已提交事件流、可被丢弃而不留痕迹。
2. **打断是一等语义**。用户可以在生成进行中显式打断；被打断的 Turn 不得提交任何正式事件；玩家草稿必须原样恢复。
3. **沉默是默认**。自动插话需要同时通过 DM 判断、角色预算、冷却与发言权租约四道门；任一不满足即保持沉默。
4. **并行只发生在候选层**。多个候选可以并行生成，但正式状态永远按 Record 单写入串行提交；落选候选不得触碰仓储。
5. 本批次不改 `modules/context` 编译器与 Turn 状态机骨架；插话复用既有可恢复 Turn 管线，以 payload 变体表达。

## 2. Preview 会话状态机（modules/streaming/preview-session.ts）

```text
streaming ──complete()──▶ committed（返回完整文本，供调用方走正式提交）
   │
   └──abort(reason)─────▶ aborted（内容被丢弃，禁止再读取为正式文本）
```

契约：

- 只有 `streaming` 态接受 `push(chunk)`；进入终态后任何 push/complete/abort 都是非法操作。
- `abort` 后 `readCommitted()` 必须抛错；`complete` 后 `abort` 无效。
- Preview 内容只允许进入：浏览器 Preview 卡片、调用方内存。禁止写入 Event、Observation、memory_conclusions 与任何持久层。
- 支持外部 `AbortSignal` 联动：信号中止等价于 `abort("external")`。

## 3. 发言权与 TurnControlLease（modules/runtime/turn-control.ts）

同一 Record 内的发言权由一个进程内租约管理（多实例协调留给后续批次）：

- `acquire(recordId, holder)`：同一 Record 同一时间只有一个持有者；竞争者在 FIFO 队列等待，`release` 后按序唤醒。
- 角色冷却：`recordInterjection(characterId)` 后在 `cooldownMs` 内 `canInterject(characterId)` 为 false；冷却以角色实例为键，跨 Turn 生效。
- 租约只覆盖发言权；正式写入仍由 Record 单写入锁保证（既有 M1 契约）。

## 4. 自动插话（modules/orchestration/interjection.ts + 服务接线）

四道门按序评估，全部通过才产生插话 Turn：

1. **DM 判断**：规则策略——玩家明确点名/直接对话某角色且该角色本轮未被激活时才考虑；否则沉默（强沉默偏置）。
2. **角色预算**：每次玩家回合后最多 1 个插话角色。
3. **冷却**：目标角色处于冷却期则放弃（不排队等待冷却结束）。
4. **发言权**：`TurnControlLease` 空闲才执行；执行结束立即释放并记录冷却。

插话 Turn 复用既有可恢复管线，以 `player.utterance` 命令的 payload 变体 `interjection` 表达：

- planner：固定计划（仅插话角色、无 Narrator、无行动预算），不调用模型规划；
- drafter：仅生成该角色对触发文本的回应；
- validator：结构校验（恰好一条该角色回应、无行动事务）；
- releaseBuilder：只发布该角色的正式事件，不产生玩家事件；
- 失败即整体不落库（既有原子性保证）。

## 5. 并行候选（modules/orchestration/parallel-candidates.ts）

`raceCandidates(producers, accept)`：

- 所有 producer 并行启动，各自持有独立 AbortController；
- 按完成顺序取第一个通过 `accept` 结构校验的候选为胜者；
- 胜者确定后立即 abort 并丢弃其余候选；落选候选不得写入任何仓储；
- 全部失败时抛出明确错误，由调用方走既有失败路径；
- 正式提交仍按 Record 串行（既有单写入契约），并行不放宽提交纪律。

## 6. 工具调用暂停与续写（modules/streaming/tool-pause.ts）

`generateWithToolPause({ first, runTool, continue_ })` 形式化两阶段生成：

```text
阶段一输出 previewChunks → 检测到工具请求 → tool.pause（暂停产出）
→ runTool 本地裁决 → tool.resolved（结果就绪）
→ 阶段二携带工具结果继续输出 → 汇总返回
```

契约：暂停期间不得向调用方产出新正文；续写输入必须包含工具结果；事件日志顺序为 `chunks → pause → resolved → chunks`；任何阶段被取消则整体返回未提交状态。

## 7. 服务与前端接线

- 服务端 Preview Hub：按 Record 广播 Preview 块（内存，不持久化），`GET /api/record/preview?recordId=`（SSE，事件类型 `preview` / `preview-end`）。
- 取消：`POST /api/record/messages/cancel { recordId, clientMessageId }` 中止对应进行中的 Turn；原提交请求返回“已打断”错误，前端恢复草稿。
- 前端：生成进行中显示直角虚线 Preview 卡片与“打断”按钮； committed/aborted 后卡片消失；保持纸墨设计语言（直角、语义色、硬阴影）。
- 玩家回合成功提交后，服务端按 §4 评估并可能执行一次插话 Turn。

## 8. 验收标准

1. 被打断的未提交内容不污染历史：abort 后 Record 事件数不变、无候选事件残留。
2. 取消后前端草稿与选择状态原样恢复，可直接重试。
3. 自动插话在沉默偏置用例下不发生；点名用例最多插话一次；冷却期内不重复插话。
4. 并行候选只有一个胜者进入提交路径；并发提交同一 Record 不产生双写（沿用既有 409/幂等契约）。
5. 工具暂停事件日志顺序正确，续写携带工具结果。
6. 一对一、一对多、多对一、多对多场景各至少一条自动化回归。
7. `npm test` 全绿；`tests/gui/b-record.spec.ts` 与 `tests/gui/g-realtime.spec.ts`（Chromium）不回归。

## 9. 明确不做

- 多人共控同一角色（ROADMAP 要求显式开启后才生效，本批次不开启）。
- 跨进程/多实例发言权协调。
- STT/TTS 管线本身（语义段已具备隐式语音边界）。

## 10. 运行态补记（2026-08-21，批次 T10-B5；追加不改写）

- §5 并行候选（raceCandidates）与 §6 工具暂停续写（generateWithToolPause）
  当前运行态为 **contract-only-deferred**：契约与本文件设计不变，但全库
  生产调用方为零（活动 gateway 无流式 tool-call 事件；单模型配置无并发
  候选业务场景）。两模块头部带 `@runtime-status contract-only-deferred`
  标记；接线前置条件见 docs/development/T10-B5-M4-RUNTIME-STATUS.md。
- 静态围栏 `tests/m4-runtime-status.test.ts` 持续证明：模块存在、标记在位、
  活动生产代码（app/、modules/application/）零导入。
