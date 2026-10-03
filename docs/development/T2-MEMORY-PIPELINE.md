# T2 · 记忆管线：sync_turn 异步萃取 + prefetch 并行预取

> 批次 T2（docs/development/EXPERIENCE-ITERATION.md 迭代主轴第二项，Lyle 指定，参照 Hermes 记忆系统）
> 立项：2026-08-20 +08:00 · 状态：规范定稿，实施中
> 一句话目标：把萃取从召回事务里拆出来，回合后异步落库（sync_turn），回合开始并行预取（prefetch），关键路径上不再有串行记忆等待。

## 一、现状调研（读代码确认）

### 1.1 摄取（已通）

回合提交事务内写入 observations：`database/postgres/runtime-repository.ts:718`
`INSERT INTO observations`，随正式事件逐条落库（observer_character_instance_id、
occurred/available_from 世界坐标、fidelity、metadata）。DB 实测已有 200+ 条。
摄取无需改动。

### 1.2 萃取（惰性，粘在召回里——本批根因）

萃取不是独立环节，而是藏在召回事务内部：
`database/postgres/memory-repository.ts` 的 `recallAuthorized`（约 :71-300）在
**同一个 workspace 事务**里依次做三件事：

1. 解析消费者 scope（character_instances × worldlines 取 continuity 与世界头）；
2. **物化**：SELECT 该 continuity 截至世界头的全部 observations，对每一行现场计算
   `extractMemoryKeywords` + `lexicalEmbedding`，逐行 `INSERT INTO memory_conclusions
   ... ON CONFLICT (workspace_id, observer_continuity_id, source_record_id,
   source_observation_id) DO NOTHING`（约 :148-206）；
3. 排序：对 memory_conclusions 做 keyword/vector/fidelity/recency 混合打分，LIMIT 返回。

后果：

- **每次召回都先全量扫描+逐行写**，observations 越多召回越慢（O(全部历史观察)）；
- **写挂在读的关键路径上**：模型回合里每个在场角色的 propose/react 都各触发一次
  `recallMemory`（`modules/orchestration/model-powered.ts:442/:491` 串行 await），
  N 个角色就是 N 次「全量物化事务 + 排序」串在模型调用之前；
- **读写职责耦合**：召回失败会连同物化一起回滚；无法单独重试萃取，也无法把萃取
  挪出关键路径——萃取粘在注入里。

### 1.3 注入（已通）

`modules/orchestration/model-powered.ts:462/:511`：角色 propose/react 的 user 消息里
拼「当前时间点可用的相关记忆：…」。注入位置与文案本批不变。

### 1.4 回合管线时序（现状）

```
玩家输入 → submitMessage
  → 可见性裁决（模型调用）
  → executeTurn：DM 规划（模型）→ 角色 propose（模型，前置串行 recall×N）
    → 行动解析 → 角色 react（模型，前置串行 recall×N）→ 旁白（模型）→ 校验
  → commitRelease（事务：events + observations 落库）
  → 响应                                          ← 萃取从未独立发生，全靠下一次召回补
```

### 1.5 snapshot/delta 死代码

memory snapshot/delta（DB 0 条）本批不接，留 T10 审计项处理。

## 二、Hermes 参照（/opt/hermes/hermes-agent，只读）

- `agent/memory_provider.py`：
  - `prefetch(query)`——每次 API 调用前注入召回文本；**实现必须快**（后台线程跑真召回，
    这里只返回已缓存结果）；
  - `sync_turn(user, assistant)`——回合后持久化；**必须非阻塞**（提交后台队列）；
  - `queue_prefetch(query)`——回合结束时为下一回合排队后台召回，结果由下一回合
    prefetch 消费。
- `agent/memory_manager.py`：
  - `sync_all`（:675-730）——回合完成路径**不内联** provider.sync_turn，派发到后台
    单写者 worker 串行执行（turn N 先于 N+1 落库）；provider 抛错只 `logger.warning`，
    绝不影响主流程（注释记录了 Hindsight daemon 阻塞 298s 的事故）；
  - `prefetch_all`（:525）——逐个 provider 收集 prefetch 文本，单个 provider 失败
    不阻塞其他 provider。

## 三、设计方案

### 3.1 总体形态：先拆后移

```
回合 N 提交完成 ──► sync_turn：fire-and-forget 后台萃取（单飞按 workspace+record 去重）
                     │  失败只留日志，不回灌回合
                     ▼
             observations ──► memory_conclusions（幂等 ON CONFLICT DO NOTHING）

回合 N+1 开始（玩家输入到达）──► prefetch：并行发起各在场角色召回（后台 promise）
                     │  与可见性裁决/DM 规划等模型调用重叠
                     ▼
        请求体组装（propose/react）──► 只消费「已就绪」结果；未就绪/失败 fail-closed
```

### 3.2 第一步·拆萃取（sync_turn 形态）

**仓储层**（database/postgres/memory-repository.ts）：

- 新增 `extractAuthorized({workspaceId, worldId, worldlineId, recordId})`：
  一个事务内做**集合式**物化——SELECT 该 record 下尚未物化的 observations
  （JOIN character_instances 取 observer continuity；`NOT EXISTS` 排除已有
  conclusion 的 (continuity, observation) 对），逐行计算 keywords/embedding 后
  `INSERT ... ON CONFLICT DO NOTHING`。返回 `{ materialized }`。
  - 幂等性完全沿用现有唯一约束 `(workspace_id, observer_continuity_id,
    source_record_id, source_observation_id)` + DO NOTHING，重复触发零新增；
  - memoryId 哈希、keywords、embedding、metadata 与现有物化逐字段一致，
    新旧数据可互相消费。
- `recallAuthorized` 退化为**纯排序查询**：删除 observations SELECT 与 INSERT 循环，
  只留消费者 scope 解析 + 混合打分排序；事务使用 `BEGIN READ ONLY`，从机制上
  杜绝召回回写。

**应用层**（modules/application/local-record-service.ts + modules/memory/pipeline.ts）：

- 新增 `createMemorySyncScheduler({ extract })`：
  - `schedule(scope)`：fire-and-forget；单飞守卫按 `(workspaceId, recordId)` 去重——
    同一 record 已有萃取在途时，新触发只挂 pending（脏标记），在途结束后自动补跑
    一次（re-arm），保证不丢新写入也不并发重复跑；
  - 萃取抛错只 `console.warn` 留日志，绝不向回合传播；
  - `idle()`：等待在途+pending 清空，仅测试断言使用。
- 触发点（回合提交完成后，与 scheduleInterjection/scheduleSceneCrystallization 同型）：
  - 玩家回合：`submitMessage` 提交成功且非 replay 时 `void schedule(...)`；
  - 插话回合：`scheduleInterjection` 内 executeTurn 完成后同样触发
    （插话同样写 observations）。
- 萃取范围按 record 集合式覆盖该回合所有 observer continuity（含玩家），
  不需要逐角色枚举；第一夜等不写 observations 的管线不触发。

**时序语义（最终一致）**：回合 N 提交的新观察，在萃取完成前发起的召回读不到——
与 Hermes「sync_turn 后台落库、下一回合消费」同义。回合提交正确性不依赖萃取。

### 3.3 第二步·并行预取（prefetch）

**预取枢纽**（modules/memory/pipeline.ts 的 `createMemoryPrefetchHub`，内存态、按 recordId 键控）：

- `begin(input)`：回合开始时为每个在场 AI 角色并行发起召回 promise，返回
  会话句柄；同一 record 再次 begin 整体替换旧会话（旧会话的迟到回写被会话
  标识隔离，绝不串入新会话）；
- `consume(recordId, characterInstanceId)`：**同步**返回已就绪结果；
  pending / failed / 无会话一律返回 `""`（fail-closed，绝不等待、绝不抛错）；
- `end(handle)`：回合退出路径清理会话——仅当该会话仍是 record 当前会话时
  移除（插话与下一回合并发时互不误删）。

**回合时序接线**（local-record-service.ts）：

- `submitMessage`：`resolveRuntimeScope` 之后、可见性裁决（首个模型调用）之前
  `memoryPrefetch.begin(...)`——召回与可见性裁决/DM 规划模型调用并行重叠；
  executeTurn 结束（提交成功/失败/异常）各路径 `end(...)`；
- `scheduleInterjection`：插话 executeTurn 前后同样 begin/end（插话 react 也消费召回）；
- 请求体组装消费：`createPostgresLocalRecordService` 的 `orchestratorFactory` 里
  `recallMemory(character)` 直接返回 `Promise.resolve(hub.consume(...))`——
  model-powered.ts:442/:491 的 await 保留但不再挂数据库/模型工作；注入位置
  （:462/:511「当前时间点可用的相关记忆」段）与格式（`- content` 逐行）不变。

**fail-closed 矩阵见第四章。**

### 3.4 依赖注入形态

`createLocalRecordService` 新增两个可选依赖（缺省不启用，内存/单元测试零变化）：

```ts
memoryPrefetch?: {
  begin(input: {
    recordId: string;
    workspaceId: string;
    worldId: string;
    worldlineId: string;
    playerText: string;
    characters: readonly { characterInstanceId: string }[];
  }): MemoryPrefetchHandle;   // 会话句柄：end 只清理仍属当前的会话
  consume(recordId: string, characterInstanceId: string): string;
  end(handle: MemoryPrefetchHandle): void;
};
memorySync?: {
  schedule(input: { workspaceId: string; worldId: string; worldlineId: string; recordId: string }): void;
  idle(): Promise<void>;
};
```

真实 PG 装配（`createDefaultLocalRecordService`）用
`createPostgresCharacterMemoryRepository.extractAuthorized` 构造 scheduler、
用 `memory.recall` 构造 prefetch 召回回调；`CharacterMemoryRepository` 接口
新增 `extractAuthorized`（测试 stub 同步补齐）。

## 四、失败矩阵

| # | 失败点 | 行为 | 对回合的影响 |
| --- | --- | --- | --- |
| F1 | sync_turn 萃取事务抛错（DB 不可用/约束异常） | console.warn 留日志；单飞位释放 | 无：回合已提交；缺失的 conclusions 由后续回合萃取补（re-arm/下次触发） |
| F2 | sync_turn 单飞重入（同 record 连续回合） | 后来者挂 pending，在途结束后补跑一次 | 无 |
| F3 | prefetch 召回 promise 抛错 | 该角色条目标记 failed + warn | 无：consume 返回 ""，角色无额外记忆，模型调用照常 |
| F4 | prefetch 未就绪（召回慢于请求体组装） | consume 返回 ""（不等待） | 无：该角色本轮无额外记忆；不阻塞模型调用 |
| F5 | 无 prefetch 会话（未装配/旧路径） | consume 返回 "" | 无：行为等价「当前没有额外召回记忆」 |
| F6 | recallAuthorized 纯查询下无任何 conclusions | 返回 [] | 无：注入空记忆文案，与现状空召回一致 |
| F7 | extractAuthorized 对空 record / 无 observations | materialized=0，静默返回 | 无 |

**不变量**：回合提交（commitRelease）与回合响应链路上不存在任何萃取/召回的串行
等待；萃取与预取的任何失败都不改变回合的提交结果与响应码。

## 五、验收标准

### 5.1 PG 集成（tests/postgres-memory-pipeline.test.ts，真实 PG）

1. 异步萃取落库：record 提交后触发 extractAuthorized，observations 全量物化为
   memory_conclusions（数量相等、内容一致、continuity 正确）；
2. 幂等：同一 record 连续触发两次 extractAuthorized，conclusions 总数不变、
   无重复 (continuity, observation) 对；
3. recall 纯查询不回写：只插 observations 不萃取，recallAuthorized 返回 [] 且
   memory_conclusions 计数不变；事务为 READ ONLY；
4. 萃取失败不影响回合提交：应用层注入抛错的 extract，submitMessage 仍 committed。

### 5.2 应用层（node:test）

5. 单飞守卫：并发 schedule 同 record 只跑一次萃取；在途期间的新触发以 re-arm 补跑；
6. 预取并行发起：begin 后各角色召回并行在途（不等前一个完成）；
7. fail-closed：pending 时 consume 得 ""；failed 时得 ""；ready 后得格式化文本；
   会话替换后旧会话迟到结果不串入新会话。

### 5.3 GUI 真实模型（tests/gui/t2-memory-pipeline.spec.ts，不新增 mock）

8. 真实模型回合后，后台萃取使记忆卡 projection 出现该回合内容（沿用 E1 断言形态：
   记忆卡 representation 非空/含回合要素，轮询等待异步萃取）；
9. 第二轮真实模型回合正常提交（prefetch 消费路径不阻塞、不改变响应形态）。

### 5.4 回归

10. `npm test` 全绿（含既有 PG 集成）；全量 GUI 回归通过（HOST_BIND=192.0.2.10，
    真实模型不许 mock）；旧断言中依赖「召回即物化」的（如
    postgres-local-record-application.test.ts 的 conclusions==observations 断言、
    GUI E1 的非空断言）同步改为「萃取后断言」，全文检索核对无遗漏。

## 六、实施与验证纪律

- commit 拆细：规范（本文档）→ 拆萃取 → 并行预取 → 测试收口；每步独立 commit，
  禁 `git add -A`。
- 每次 GUI/集成写入后执行 `scripts/clean-gui-test-data.sql` 并核对计数
  （世界/账号/first_nights 回基线 2/1/0）。
- STATUS.md 三段式追加（时间戳 + 数字 + 验证证据）。
- 迭代日志写入 EXPERIENCE-ITERATION.md（设计思路、验证证据、新发现、下一轮决策）。
- 行为变更涉及的旧断言全文检索同步（T1 教训：漏 L1）。
