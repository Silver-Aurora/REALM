# 设定结晶与逻辑一致性裁决实施规范

> 文档性质：实施规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 视觉约束以 [`../design/UI-DESIGN.md`](../design/UI-DESIGN.md) 为准；鉴权约束以 [`DEPLOY-AUTH.md`](./DEPLOY-AUTH.md) 为准。

## 1. 背景

场景状态（地点 / 天气 / 局势 / 当前目标 / 世界时间）此前只在创建世界或记录时写入一次，对话推进不会更新。本批次实现**设定结晶（Scene Crystallization）**：每回合结束后从回合内容提取场景状态增量，经独立的逻辑一致性裁决后才写回世界状态，防止模型幻觉污染设定。

## 2. 两段式管线

### 2.1 提取阶段（Extraction）

- 触发：玩家回合成功提交（非幂等重放）之后，异步执行，永不阻塞回合响应。
- 输入：玩家原话、本回合已提交事件（旁白与角色回应）、当前场景状态。
- 输出（严格 JSON，全部字段可选）：

```json
{
  "displayTime": "世界时间（如「停战纪元17年 · 雾月12日 · 深夜」）",
  "location": "地点",
  "weather": "天气",
  "tension": "局势",
  "objective": "当前目标"
}
```

- 规则：只提取本回合**明确表达或强蕴含**的变化；没有变化输出 `{}`；禁止无中生有、禁止复述既有状态。

#### 2.1.1 同次提取的自然生长（growth）

同一次结构化提取（不新增模型调用）还允许带回两个 bounded 可选数组：

```json
{
  "worldClaims": [{ "entity": "实体名", "entityKind": "geography|history|setting|faction|person|other", "predicate": "短谓词", "value": "事实值" }],
  "characterNotes": [{ "characterInstanceId": "当前名册成员", "note": "一条耐久的人物设定" }]
}
```

- 条数/长度上限见 `GROWTH_LIMITS`（worldClaims ≤3、characterNotes ≤2，字段限长，规整 fail-closed 逐条丢弃）。
- **仅公共回合运行**：restricted/private 回合在提取器之前结构性跳过（私密原文绝不进入提取器）；公开回合的提取输入中，近期对话与摘要只取 `policy_kind = 'public'` 的事件（owner omniscient 视角也不得把受限素材送进 growth 提取）。
- worldClaims 写入 `world_claims` 时恒为 `scope='record'` + `truth_status='record_confirmed'`，带 `source_record_id`/`source_event_id`；实体按 `sha256(worldId|worldlineId|实体名)` 确定性生成（worldline 级隔离，upsert 幂等），claim id 按 `sha256(recordId|entityId|predicate|value)` 确定性生成，`appendClaimsIdempotent`（`ON CONFLICT DO NOTHING`）保证重复调度不重复落账。绝不自动升级 story/world canon。
- characterNotes 追加/合并进当前 Record 的 `character_instances.state.profileNotes`（JSONB 数组，精确去重、上限 8 条；`database/postgres/character-growth-store.ts`），绝不覆盖 `character_definitions.profile` 或玩家显式自述的 `state.profileSummary`；`record-scope` 解析时把 notes 合并进 `profileSummary` 供下一回合 Character Runner 消费。
- growth 写回的世界知识经 `record-scope` 的 `recordKnowledge`（record 级、record_confirmed、本 Record 来源、temporal 过滤）在下一回合读回，作为「当前记录知识/背景」注入 Character Runner/Narrator 上下文，与 `brief.canon`（story_canon 以上）严格分离。
- growth 任一环节失败只留日志，不阻断已提交回合；没有场景 delta 时不产生空 correction 事件（growth 与场景裁决互不阻塞）。

### 2.2 裁决阶段（Adjudication）

提取结果非空时才调用。输入：当前场景/世界状态 + 候选增量 + 近期事件摘要 + 玩家原话。输出严格 JSON：

```json
{
  "approved": true,
  "reason": "裁决理由（审计用）",
  "adjusted": { "…": "可选；与增量同结构的修正值" }
}
```

裁决规则（写进 prompt 的硬约束）：

1. 时间不能倒退，除非本回合存在显式的回溯叙事设定；
2. 地点变更须与叙述动线相容（相邻、可抵达）；
3. 天气 / 局势突变须有叙述依据；
4. 目标变更须由事件驱动；
5. 拿不准一律 `approved=false`。

`approved=false` 或返回 `adjusted` 时以裁决结果为准；`adjusted` 非法（schema 不符或为空）按失败处理。

### 2.3 写回阶段（Write-back）

仅 `approved=true` 时执行，使用 `adjusted ?? delta`，单事务完成：

1. **场景流转**：`location` / `objective` 有变化时 INSERT 新 `scenes` 行（未变字段继承当前场景，`start_tick/start_ordinal` 接 record head 游标之后）；绝不 UPDATE 既有 scene 行。delivery-projection 按游标自动选中最新场景，读取路径不变。
2. **世界状态**：`displayTime` / `weather` / `tension` 合入 `worlds.settings`（jsonb 合并，只写出现的键）。
3. **时间线留痕**：同时 INSERT 一条 `system.correction.committed` 事件（public 可见性，speaker「界核」，content 为本次结晶的事实摘要），并推进 `record_heads` 与 `worldlines` 游标，与回合提交路径保持同一组不变量。
4. **裁决审计**：`approved=false` 时 INSERT `semantic_conflict_evaluations`（append-only，`prompt_version = scene-crystallization/v1`，`input_digest` 为输入摘要的 SHA-256，`result` 含增量与裁决理由）。

### 2.4 权限（迁移 0014）

`realm_runtime` 在 0004 硬化后只有 SELECT。新增最小授权：

- `INSERT`（列级）ON `scenes`；
- `UPDATE (settings)` ON `worlds`。

`events` / `record_heads` / `worldlines` / `semantic_conflict_evaluations` 的写权限既有迁移已授予，不改动。

## 3. fail-closed 矩阵

| 环节 | 失败形态 | 行为 |
|---|---|---|
| 网关加载 | 未配置 / 密钥缺失 | 跳过，仅日志 |
| 提取 | 模型错误 / 超时 / 非法 JSON / 全空增量 | 跳过，不进入裁决 |
| 裁决 | 模型错误 / 超时 / 非法 JSON / schema 不符 / adjusted 非法 | 跳过，不写回不审计 |
| 裁决拒绝 | `approved=false` | 写审计，不写回 |
| 写回 | SQL / 约束 / 权限失败 | 仅日志，回合不受任何影响 |

铁律：任何环节失败都不更新场景、不抛异常中断回合，只留本机日志。

## 4. 模块边界

- `modules/application/scene-crystallization.ts`：增量与裁决的 schema 规整（纯函数）、提取器 / 裁决器工厂（经 `modules/inference` 网关，`responseFormat: "json_object"`，thinking 下 max_tokens 下限由网关兜底）。
- `database/postgres/scene-crystallization-repository.ts`：写回与审计的事务实现（realm_runtime 连接池）。
- `modules/application/local-record-service.ts`：回合成功后的异步挂接（依赖注入，缺省不启用，便于既有测试零变化）。
- prompt 只含世界/场景/事件文本，绝不含 token、密钥或会话材料。

## 5. 验收标准

1. Core 契约：增量规整（缺字段 / 全空 / 超长截断）、裁决三态（approved / adjusted / rejected）、fail-closed 各分支。
2. PG 集成：写回产生新 scene 行且投影选中它；worlds.settings 合入；结晶事件入时间线且游标推进；拒绝只写审计不落场景；0014 授权对 realm_runtime 生效。
3. GUI（M 组）：对话中给出新地点/天气 → 重载后面板对应字段浮现；矛盾设定（时间倒退）→ 字段不变。
4. `npm test` 全绿；既有 GUI 回归全部通过。

## 6. 明确不做

- 场景状态的版本对比 / 回滚 UI。
- 多场景并行（仍单场景流转）。
- 结晶频率节流与批量合并（后续按使用强度再加）。
