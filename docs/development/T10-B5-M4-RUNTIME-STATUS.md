# T10-B5 · M4 Tool-Pause / Parallel-Candidates 运行态围栏

> 批次 T10-B5（docs/development/EXPERIENCE-ITERATION.md T10 拆批，第二章缺陷 #16）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=97cf2ed；本批是 contract-only/deferred 标记批——不接线、不删除、不改行为。

## 一、逐文件调用图与 gateway 能力证据（读代码确认）

| 模块 | 生产调用方 | 测试调用方 | 结论 |
|---|---|---|---|
| `modules/streaming/tool-pause.ts`（generateWithToolPause） | **零**（全库检索） | tests/m4-streaming-interruption.test.ts | 承诺但不可接线（见二） |
| `modules/orchestration/parallel-candidates.ts`（raceCandidates） | **零**（全库检索） | tests/m4-streaming-interruption.test.ts | 承诺但不可接线（见二） |

- gateway 能力实锤：活动网关（`modules/inference/openai-compatible-gateway.ts`）
  的 `ModelGateway` 契约只有完整文本 `chat`/`streamChat`（文本 chunk），
  **没有可消费的 tool-call 事件流**——tool_calls 只在完整响应里返回
  （`ModelChatResponse.toolCalls`）。tool-pause 需要「流式中途的工具暂停
  事件」，当前供应商响应形态不提供；强行接线只能是假实现。
- parallel-candidates 需要「同一 Record 写边界内多个候选生成器」的业务
  语义（多模型/多采样竞争）；当前单模型配置下没有真实业务调用点，
  伪造调用点违反「不顺手接线无业务语义路径」。
- 迭代总纲旧摘要曾写「raceCandidates 有真实接线（非死代码）」——
  与当前代码不符，本批以 T10-B5 日志纠正（不改写历史条目，追加更正）。

## 二、本批结论：contract-only-deferred（不是完成）

两模块保留，头部追加机器可读运行态标记 `@runtime-status contract-only-deferred`
与原因，函数行为零改动。未来接线前置条件（写入模块头与本文档）：

1. **gateway tool-call 事件契约**：供应商/适配层提供流式 tool-call 事件
   （或等价的结构化中途信号），而不是仅完整响应；
2. **工具白名单与权限**：暂停恢复期间只允许只读/裁决内工具，状态变更工具
   仍走账本闭环；
3. **Record 单写入与 preview 边界**：暂停内容不进入正式历史（M4 preview
   session 语义）；
4. **取消/Abort 语义**：任一阶段取消使整段生成不提交；
5. **parallel-candidates 额外前置**：真实业务场景需要并发候选（如多模型
   配置或采样竞争），且胜者仍走单写入路径。

## 三、围栏（可执行，非口号）

新增 focused 静态测试 `tests/m4-runtime-status.test.ts`：
1. 两模块文件存在且头部含 `@runtime-status contract-only-deferred`；
2. 活动生产代码（`app/`、`modules/application/`）零导入两模块
   （静态扫描，防未来悄悄接线或悄悄删除）；
3. 既有行为测试 `tests/m4-streaming-interruption.test.ts` 全过（保留）。

文档：M4-STREAMING-INTERRUPTION.md 追加运行态小节（不改写历史契约）。

## 四、失败矩阵

| # | 场景 | 形态 |
|---|---|---|
| F1 | 模块被删除/标记丢失 | 静态测试失败 |
| F2 | 生产代码悄悄导入两模块 | 静态测试失败 |
| F3 | 头部标记改写为已接线 | 静态测试失败（标记内容断言） |
| F4 | 既有行为回归 | m4-streaming-interruption 全量过 |

## 五、验收标准

- `tests/m4-streaming-interruption.test.ts` 原始 exit 0；
- 新 `tests/m4-runtime-status.test.ts` exit 0；
- typecheck/受影响 eslint/文档布局/git diff --check exit 0；
- 不跑全量/GUI；开发库计数基线不变（本批零 DB 写入）。

## 六、交付步骤

1. 本规范（单独 commit）；2. 模块头标记 + M4 文档追加 + 围栏测试（独立
   commit）；3. STATUS/EXPERIENCE-ITERATION 收口（独立 commit）。
