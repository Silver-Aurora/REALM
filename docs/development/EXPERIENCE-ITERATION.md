# REALM 体验迭代总纲（批次 T 系列）

> 启动：2026-08-20 01:31 +08:00 · 截止：~~当日 08:00~~ → **当日 20:00（Lyle 08:10 追加 12 小时）**
> 模式：设计计划 → 开发验证 → 思考改善 → 设计计划（循环，每轮一条 Codex goal）
> 每轮的设计思路与进度都记录在本文档「迭代日志」章节；单轮细节规范由 Codex 落到 docs/development/ 下独立文档。

## 一、现状基线（批次 S 完成后）

已具备：司卷对谈创世（LLM 引导）、世界记忆与默认入口、观察者席位与姿态切换、添角色即入阵容、卷首旁白、文风系统（4 风格）、i18n 三语、酒馆卡/世界书导入、行动卡动态生长、下一步对话提案。

## 二、体验缺陷盘点（待逐轮消化）

1. **创世之后无局可玩**（最致命）：落笔入界后，新世界的头几分钟仍是空时间线 + 泛化行动卡。玩家花 10 分钟创世，进场却没有「可玩密度」——场景、角色、钩子事件缺位。
2. **角色在场感弱**：添了角色只是进名单，角色不会主动说话、不会对环境产生反应、彼此无关系。
3. **观察者体验空白**：切到观察者后看什么、世界由谁推进、观察者的乐趣点未设计。
4. **世界管理缺位**：多世界创建后无浏览/删除/归档体验，世界卡信息密度低。
5. **长期反馈不可见**：canon/世界线/记忆机制存在但玩家感知不到「世界因我而变」。
6. **导入的角色卡不参局**：批次 R 的导入卡在库里躺着，与游玩闭环断开。
7. **demo 与自建世界体验断层**：demo 有塞娜/弥洛的活内容，自建世界只有旁白开场。
8. **规则系统是 demo 写死的**：全库唯一规则包 createLocalRulePack 硬编码 1 技能（careful_observation）+1 资产（signal_lantern）+1 姿态（guarded_watch），判定数值写死（modifier 2/target 12），还专门为 demo 硬编码 letter_seal 蜡封桥段。自建世界用任何技能/资产/姿态都直接 FatalTurnError。
9. **引擎工具全部未实现**：M2 设计要求的 resolve_uncertainty/consume_resource/apply_effect 三个引擎工具在代码中零命中（只有 ACTOR_TOOL_SPECS 声明）。
10. **骰子无随机性**：deterministicInteger(callId, 20) 由 callId 哈希决定——同一行动重放永远同结果，无真随机、无概率表达，且 M2 设计的 d20/2d6/百分骰/骰池/抽牌只实现了 d20 一种。
11. **死代码与假接口**：memory snapshot/delta（DB 0 条）、CharacterMemoryService 的 summarize/relationships 等接口写了但游玩闭环未接；skill_definitions/character_skills/effect_definitions 表存在但被 SQL 硬编码 key 锁死（WHERE skill.skill_key='careful_observation' 等三处）。
12. **DM 审查无降级路径**（T1 验收 O3 实锤）：DM 审查模型连续三次否决候选（goalSatisfied=false）后回合直接裸挂（DM_OUTPUT_REJECTED），响应无输出无提案，玩家侧体验为「世界沉默」。三次否决后应有确定性降级（如：接受第三次候选 / 落保守旁白），而非让回合失败。
13. **世界治理引擎零接线**（第二轮设计文档审计）：M5 的 modules/propagation（传播引擎+worker）与 modules/worldline 全部五个模块（conflict-detection/merge/branching/semantic-conflict/canon 的服务侧消费）**在全库无任何调用方**（排除自身与测试）；propagation_jobs/propagation_exposures/canon_proposals/canon_revisions 全部 0 行。M5 §7 明确「传播 Worker 与离线调度第二批补齐」——属设计内暂缓，但第一批承诺的冲突检测/传播曝光也未接线。
14. **知识图谱零数据**：world_entities/world_claims/world_relations 全 0 行——运行时没有任何事实流入图谱；world_articles 仅创世/酒馆导入写入。M5 §6 验收标准 1「实体→Claim→Article 引用链完整」实际未达成。场景晶化（scene-crystallization）已接线但产出未入图谱。
15. **canon 不回读**（玩法风险#4 的实锤）：回合生成只消费 buildSceneCanon（场景快照 brief，model-powered.ts:92），canon_revisions/canon_proposals 数据（0 行）从不进入任何生成 prompt——世界观漂移无约束。API 层 app/api/canon/route.ts 存在但无产生方也无消费方。
16. **M4 工具暂停未接线**：modules/streaming/tool-pause.ts 的 generateWithToolPause 零调用方（M4 §6 承诺的工具调用暂停与续写）。注：parallel-candidates 已修正——raceCandidates 有真实接线（非死代码）。

## 三、玩法设计风险

1. **创建成本 > 游玩收益**：创世流程越精致，玩家对进场体验的期待越高，落差越大。
2. **AI 单点依赖**：模型沉默/跑偏时体验崩塌（现有静默自愈只保底，不救场）。
3. **自演叙事疲劳**：无外部刺激的单人叙事难以持续超过数回合，需要节奏钩子。
4. **canon 无约束力**：后续生成若不回读世界设定，世界观会漂移。
5. **行动卡泛化**：新世界的场景快照浅，行动提案容易退化成「观察四周」式空话。

## 四、迭代主轴（按优先级排布，实际执行按每轮结束时的判断滚动调整）

- **T1 世界初夜**：创世落笔后的第一轮可玩密度——场景即时生成、角色在场、钩子事件开场。
- **T2 记忆管线 sync_turn + prefetch**（Lyle 指定，参考 Hermes 记忆系统；**现状实锤未实现**——全源码 grep sync_turn/prefetch 零命中）。
  - **Hermes 参照**（/home/lyle/.hermes/hermes-agent）：`agent/memory_provider.py` 定义 ABC——`prefetch(query)` 在每次 API 调用前注入召回文本（实现应快，用后台线程+缓存结果）；`sync_turn(user, assistant)` 在每回合后持久化，**必须非阻塞**（提交后台队列）；`queue_prefetch` 回合结束后为下一回合排队后台召回，结果由下一回合 prefetch 消费。`agent/memory_manager.py:695-730` 用单写者后台线程串行化写入（turn N 先于 N+1 落库）；`prefetch_all`（:525）在独立线程跑 provider.prefetch，失败不阻塞其他 provider。
  - **REALM 现状（修正：管线存在但为同步惰性版，非缺失）**：①摄取已接入回合写入管线（runtime-repository.ts:718 落 observations，DB 实测 169 条）；②萃取是**惰性萃取**——藏在 recallAuthorized 内部（memory-repository.ts:148-206），每次召回在同一事务里先物化新 observations 再排序，萃取粘在注入里、全在关键路径；③注入已通（model-powered.ts:462/511）；④snapshot/delta 接口存在但 DB 0 条，游玩闭环未接（死代码）；⑤萃取为纯机械搬运无 LLM 提炼（113 conclusions=90 explicit 原样搬运 + 23 API 手动 summary）。
  - **实施顺序（先拆后移）**：①第一步把萃取从 recall 事务里拆出——回合提交完成后异步物化 observations→conclusions（sync_turn 形态，后台单写者串行，失败不阻塞回合），recall 只做排序查询；②第二步召回并行化——回合开始时与模型调用并行发起召回（prefetch），请求体组装时结果已就绪，移除 model-powered.ts:442/491 的串行 await；③请求体注入位置不变：角色 messages 的「当前时间点可用的相关记忆」段（:462/:511）。
  - 验收：回合响应时间不再包含萃取+召回串行延迟；sync_turn 失败不影响回合提交；召回内容真实进入请求体；幂等性保持（重复萃取不产生重复 conclusions）。
- **T3 角色在场**：角色主动发声与环境反应（回合内角色行为投影）。
- **T4 规则系统真化（Lyle 指定）**：把 demo 写死的规则包换成数据驱动——技能/资产/姿态定义来自世界数据（创世与导入生成），判定参数由角色能力与世界规则推导而非硬编码；同时落地 M2 设计的三个引擎工具（resolve_uncertainty/consume_resource/apply_effect）与状态账本闭环（character_effects 的 apply/expire、资产余额真实扣减）。这是 T6 导入卡参局的前置——导入的技能卡要能真正参局。
- **T5 骰子真随机与多骰系**：替换 deterministicInteger 的哈希伪随机为真随机源；实现 M2 设计的 d20/2d6/百分骰/骰池/抽牌，判定结果进 Delivery Projection 让玩家可见。
- **T6 导入卡参局**：酒馆角色卡进入游玩闭环（依赖 T4 规则系统真化）。
- **T7 观察者之眼看世界**：观察者模式有内容可看（世界自演/旁白推进）。
- **T8 世界管理台**：世界浏览/删除/归档 + 世界卡信息密度。
- **T9 canon 回读**：世界设定对后续生成的约束注入。**第二轮审计实锤（缺陷#15）**：回合生成只消费 buildSceneCanon 场景快照，canon_revisions/canon_proposals 从不进生成 prompt，app/api/canon 无产生方也无消费方——需同时补 canon 产生侧（场景晶化产出晋升）与消费侧（回读注入），否则只有半条链。
- **T10 死代码与假接口清理（持续审计）**：Lyle 指令「迭代过程中逐步核对设计文档并审计偷懒实现」。已实锤清单：①memory snapshot/delta、CharacterMemoryService 未接接口；②action-state.ts 三处硬编码 key（T4 正在解）；③**世界治理引擎零接线**——modules/propagation 与 modules/worldline 五模块全库无调用方，propagation/canon 四表 0 行（M5 §7 部分属设计内暂缓，需区分「第二批计划内」与「第一批承诺未兑现」）；④**知识图谱零数据**——world_entities/world_claims/world_relations 0 行，场景晶化产出未入图谱；⑤**M4 tool-pause 未接线**（parallel-candidates 已核实有接线，非死代码）。每轮验收时对照 M1-M5 设计文档核对该批次实现，发现新的偷懒/假实现追加第二章缺陷清单并排入后续轮次。
- （滚动区：每轮结束时把新发现的缺陷追加到第二章并排序）

## 五、验证纪律（2026-08-20 08:40 Lyle 指令：全量回归太耗时，从 T4 起范围化）

**每批验证 = npm test 全链（必跑，约 2min）+ 范围化 GUI 回归**，不再默认全量 GUI（146+ 项约 1h）：

1. **范围圈定规则**：跑「本批新增 spec + 本批改动文件的直接关联 spec」。GUI spec 与模块映射：
   - orchestration/local-record-service（回合管线）→ b-record, c-actions, d-visibility, g-realtime, o-action-suggestions
   - memory → e-memory, t2-memory-pipeline
   - 创世/引导（library-service/genesis）→ l-genesis, n-guided-genesis, s-onboarding, t1-first-night
   - actions/rules → c-actions, o-action-suggestions
   - 导入 → r-tavern-import；风格/i18n → p-world-style, q-i18n
   - 基础冒烟（每批必带）：a-library, b-record
2. **何时仍跑全量**：①一批结束时的里程碑轮（每 2-3 批一次）；②改动触及共享管线且关联面无法圈定（如 envelope 投影结构变更）；③Iris 验收时发现范围回归与 Codex 报告不符。
3. **失败定性不变**：失败逐一复跑，偶发须注明复跑证据；Q1 类数据敏感用例复跑前先执行清理脚本。
4. T3 例外：任务书已含全量要求，中途不干预，按原计划跑完。

## 六、迭代日志

（每轮验收后由 Iris 追加：轮次、主题、commit 链、验证证据、新发现、下一轮决策）

### T0 · 启动（2026-08-20 01:31）

- 总纲落地，基线为批次 S 完成态（6e170db..42c9b55）。
- 第一轮选定 **T1 世界初夜**：理由——「创建成本 > 游玩收益」是当前最大落差，且 T2/T3 的在地内容都依赖世界进场先有密度。

### T1 · 世界初夜（2026-08-20 01:43 派单 → GOAL DONE）

- **commit 链**：3fe5f4b（规范）→ f830f9b（能力点一·场景即时生成）→ 5cc6738（能力点二·角色在场）→ eb1cb63（能力点三·钩子事件开场）→ 1175681/305aedf（回归断言同步与超时）→ 0bd0197（STATUS+迭代日志收口）。共 12 commit 领先 origin/main（含 Iris 审计 docs commit）。
- **设计思路**（Codex 规范 T1-FIRST-NIGHT.md）：根因=创世只写结构不写叙事、角色只进名单、钩子机制不存在、行动提案素材浅。方案=「事务内保底 + 事务外增密」——落笔事务内零模型调用插入确定性开场旁白 + record_first_nights pending 状态行（迁移 0019，RLS+CASCADE）；事务外单飞模型调用生成初夜包（场景定格三段/同行者发声/钩子+2-3 提案），normalize 严格规整，失败 fail-closed 落确定性降级包，懒重试自愈（attempts<3，超限强制 degraded）。旧世界无状态行→firstNight=null 完全向后兼容。
- **验证证据**：npm test 全链绿（Iris 独立复跑，含 PG 26=21+5 初夜测试）；Codex 自报 GUI T1 组 3 用例真实模型通过、全量 146 项 140 过 6 复验转绿。**Iris 全量独立复跑：143/146 过（58.5min）**——3 失败逐一复跑定性：B5 复跑过（偶发）；Q1 双端复跑仍挂→按 Codex 线索清 DB（accounts 18→1）后复跑过（测试账号污染演示世界语言投票，实锤）；**O3 干净 DB 下连续 3 次复跑仍挂——非偶发，根因 DM 审查模型对候选连续三次 goalSatisfied=false 否决（journalctl 实锤），T1 未触碰 DM 审查路径（git diff 核实：local-record-service 仅加 envelope 投影，model-powered.ts:345-407 零改动），定性为模型侧持续否决，坐实玩法风险#2「AI 单点依赖」，列入审计发现待后续轮次加审查降级路径**。
- **新发现**：①见第二章缺陷 8-11（规则系统 demo 写死、引擎工具未实现、骰子伪随机、死代码清单）——T1 的开场提案虽补位行动卡，但行动卡能力目录仍被硬编码 key 锁死，自建世界行动卡仍只有「观察四周」，须 T4 规则真化根治；②行为变更需全文检索同步旧断言——本轮「时间线为空」断言更新了 N1 却漏了 L1（补修 1175681）；③Q1 对同一运行会话内新建测试账号的演示世界成员关系敏感，跑 i18n 组前应先执行清理脚本；④C4 两次真实模型回合贴 240s 全局超时，对模型时延敏感（放宽至 600s）。
- **下一轮决策**：T2 记忆管线 sync_turn + prefetch，按「先拆后移」实施（设计要点已实锤落档 3ff309f、现状修正 fa79aa8）。

### T2 · 记忆管线 sync_turn + prefetch（2026-08-20 05:51 派单 → GOAL DONE）

- **commit 链**：2d05531（规范）→ b6e22dc（第一步·拆萃取 sync_turn）→ cea4885（第二步·并行预取 prefetch）→ 本提交（测试收口：GUI T2 组 + STATUS/迭代日志；Iris 独立验证日志 28779f5 并入本条）。共 4 commit 领先 origin/main。
- **设计思路**（Codex 规范 T2-MEMORY-PIPELINE.md，对照 Hermes memory_provider.py 的 prefetch/sync_turn/queue_prefetch 语义与 memory_manager.py 单写者串行化）：根因=萃取粘在 recallAuthorized 同一事务里（每次召回先物化新 observations 再排序），萃取粘在注入里、全在关键路径，召回事务兼具写职责。方案两步走：①sync_turn——把 observations→conclusions 物化拆为 extractAuthorized（集合式物化，唯一约束 + ON CONFLICT DO NOTHING 幂等），回合提交完成后 fire-and-forget 调度（单飞守卫按 (workspaceId, recordId) 去重，在途重入只挂 re-arm 补跑，失败只留日志），玩家回合与插话回合同型触发；recallAuthorized 事务退化为 READ ONLY 纯排序查询。②prefetch——createMemoryPrefetchHub 按 recordId 键控、会话标识隔离：回合开始（可见性裁决/DM 规划之前）为各在场 AI 角色并行发起召回，请求体组装时同步消费就绪结果，pending/failed/无会话 fail-closed 返回 ""（行为等价「当前没有额外召回记忆」，不阻塞模型调用），replay 不预取，finally 统一 end 且仅清仍属当前的会话（插话与下一回合并发互不误删）。注入位置与格式不变（model-powered propose/react 的「当前时间点可用的相关记忆」段）。
- **验证证据（Iris 独立，28779f5）**：npm test 全链绿——Core 134（+10）/契约 39/前端 16/应用 16/PG 29（+3 记忆管线集成）/渲染 3；代码层审计：recallAuthorized 确认 READ ONLY 无回写，新增 modules/memory/pipeline.ts（95 行）+ postgres-memory-pipeline.test.ts（467 行）+ memory-prefetch-application.test.ts（364 行）。
- **验证证据（Codex 收口）**：npm test 全链绿复证（134 Core 含管线 10 项：单飞/re-arm/幂等/并行/fail-closed/会话隔离；16 应用含预取时序 begin→plan→end、失败清理、插话独立会话；29 PG 集成含记忆管线 3 项：重复触发零重复 conclusions、READ ONLY 纯查询不回写、萃取失败不阻塞回合提交；构建 + 3 渲染）。GUI T2-1 真实模型两次独立运行均过（单项 44.9s / 全量回归内 35.8s）。全量 GUI 回归 148 项（chromium 74 + webkit 74）单次运行 144 过（1.1h），4 失败逐一复跑全部转绿，均非代码回归：G2 chromium 复跑 40.1s 过（全量运行超 240s 全局预算）、O3 chromium 复跑 30.7s 过（真实模型回合时序）、Q1 webkit 复跑 1.8s 过（测试数据敏感，复跑前已执行清理脚本）、T1-3 webkit 复跑 2.0min 过（模型时延致 180s 等待超时）。清理脚本两轮执行回基线：世界=2/账号=1/first_nights=0，残留测试实体 0。
- **新发现**：①记忆卡 /api/memory 只在挂载/深度切换/整理摘要后读取（memoryVersion 仅由 summarize 递增），后台萃取落库后页面不自动刷新——GUI 断言以直链重进重读兜底；若后续要让玩家「看见」记忆生长，需 UI 侧增补重读机制（候选体验项）。②插话回合的 prefetch begin 在 scheduleInterjection 首个 await 前同步发生——玩家会话在玩家 end 之前即被插话会话替换，end 的会话标识隔离是正确性必需而非防御冗余（应用层测试实锤时序）。③extractAuthorized 为确定性物化（关键词 + 词法嵌入，无模型调用），sync_turn 开销毫秒级、失败面仅 DB 故障——与 Hermes 的后台写线程同形态但更轻。④无新增缺陷；O3 的 DM 审查降级路径（缺陷#12）顺延后续轮次。
- **截止说明**（Iris）：用户指令「迭代到 08:00」。08:00 时 T2 代码主体已落地且单元/集成全绿，Codex 于 08:01 GOAL DONE 善终收口（5 commit、7737s）。**Iris 最终独立验收（GOAL DONE 后）**：probe/check_result 双确认 completed；npm test 独立复跑全绿；T2 组 GUI 独立复跑 2/2 全过（chromium+webkit，1.7min）；DB 清回基线 worlds=2/accounts=1/first_nights=0；验收通过，推送。
- **下一轮决策**：按迭代主轴进入 T3 角色在场（角色主动发声与环境反应）；T2 遗留的 UI 侧记忆刷新与缺陷 #12 DM 审查降级路径一并候选。

### T3 · 角色在场（2026-08-20 派单 → GOAL DONE）

- **commit 链**：8d5a19a（规范）→ 72618b3（触发机制）→ 2cd051d（事件落库与预算）→ e87e71f（GUI 真实模型复现的 principal 透传修复）→ 本次 fix（在场回合版本冲突重读重试）→ 本次 feat（测试收口：PG 集成 + GUI T3 组 + STATUS/迭代日志）。
- **设计思路**（Codex 规范 T3-CHARACTER-PRESENCE.md）：根因=角色发声只有 propose/react 两条路径且全部由玩家回合触发，角色之间无互动、对环境变化无反应、彼此无关系表达。方案=回合提交后的自治在场分流：点名回合走插话（M4 既有路径），未点名回合走在场评估——确定性抽取三类叙事素材（环境事件/同侪行动/未响应钩子），素材全空确定性沉默不调模型；候选过滤排除本回合已激活/已发声角色并复用 TurnControl 冷却；模型门禁裁决「谁因何开口」（fail-closed 沉默）；presence react 分支生成 action+dialogue（可附关系表达），消费 T2 prefetch 记忆注入，关系落库走 CharacterMemoryService.relationships/recordRelationship 既有接口；utterance.committed 经既有事件形态落库（payload presence 标记三值 environment/peer/hook），delivery 投影透出、前端时间线 data-presence 锚点（观察者同见）。预算硬约束：PRESENCE_MAX_PER_TURN=1、HARD_CAP=2；失败矩阵全部 fail-closed，绝不阻塞玩家回合主流程。
- **验证证据**：npm test 全链绿（Core 143、契约 39、应用 25 含在场 9、前端契约 16、PG 30 含在场落库与预算 1 项、构建 + 渲染 3）。GUI 全量回归 150 项（chromium 75 + webkit 75）单次运行 145 过（1.2h）；5 失败逐一复跑定性：B5/O3/G2/Q1 复跑全部转绿（偶发：乐观发送时序、真实模型回合时序、断线回补超预算、测试数据敏感——Q1 复跑前已执行清理脚本）；**T3-1 webkit 复跑再次失败，非偶发**——journalctl 实锤 presence turn did not complete: RECORD_VERSION_CONFLICT，DB 事件序列实锤 玩家回合 v1 → 场景晶化 system.correction.committed v2（+4s）→ 在场提交（+9s）撞冲突 fail-closed 沉默；修复（版本冲突重读快照重试一次：不重新过门禁、预算仍为 1、重试仍冲突维持沉默）+ 应用层确定性用例后，T3-1 webkit/chromium 复跑双双通过（各 1.1min）。修复后范围化回归（a/b/c/d/g/o 6 组 × 双引擎 54 项）52 过，2 失败均为 O3：信封提案取自本回合 DM 旁白结构化输出，该时段模型未附提案，复跑定性偶发（webkit 复跑过、chromium 二次复跑 16.6s 过）。清理脚本回基线 worlds=2/accounts=1/first_nights=0。
- **新发现**：①自治路径的快照加载必须透真实 principal——e87e71f 之前非演示 principal 的记录 membership 解析回落演示身份 → notInitialized → 静默（M4 插话同病同修）；②回合管线之外存在异步落库写者（场景晶化 system.correction.committed，回合后 ~2–5s 落库），任何「回合后自治回合」都会与其竞态——T3 以版本冲突重读重试一次收口，后续批次的自治写者（如 T7 观察者内容供给）应直接复用该模式而非各自处理；③PRESENCE_RESPONSE_INVALID（门禁结构化输出解析失败）在全量回归中真实出现一次并正确 fail-closed 沉默——门禁确定性规整是必需的而非防御冗余；④record_version 按提交批次递增、ordinal 按事件致密（PG 断言据此修正）；restricted 回合走提案→确认两步、replay 指纹含 visibility 故 restricted 重放必冲突（应用层测试改 public 回合验 replay）；⑤O3 的信封提案断言对模型输出敏感（DM 旁白结构化输出偶尔不附 suggestions），自 T2 起多次偶发——候选体验项：提案缺失时是否要有确定性兜底素材。
- **下一轮决策**：按迭代主轴与验证纪律（T4 起范围化回归）进入下一批；T3 遗留候选——观察者模式在场内容可见性核验（T7 供给）、UI 侧记忆刷新与缺陷 #12 DM 审查降级路径、O3 提案兜底素材一并顺延。

### T3 · 交付自查补记（2026-08-20 交接后复核）

- **缺口**：交接摘要与 STATUS 声称「eslint 干净」，复核实测 `npm run lint` 有 3 个 error——均为 T3 之前遗留（批次 S guided-genesis-chat.tsx 两处：phase 死状态、渲染期写 draftRef；T2 postgres-memory-pipeline.test.ts 一处未用变量），非 T3 引入，但声称失实即须收口。
- **修复**（6c083c3）：phase 去绑定（`const [, setPhase]`，setter 调用与重渲染语义不变，零行为变更）；draftRef 写入由渲染期移入 useEffect（所有读取点均为 commit 后的事件/回调，语义等价）；测试死代码移除（import 仍被他测使用）。lint 复验 0 error 0 warning，npm test 全链复验绿（Core 143/契约 39/应用 25/前端契约 16/PG 30/渲染 3）。
- **范围化回归**（创世/引导映射 l-genesis/n-guided-genesis/s-onboarding/t1-first-night + 基础冒烟 a-library/b-record，× 双引擎共 50 项）：48 过（21.0min）；2 失败逐一复跑定性——S2 chromium=开场请求 3 连阵发沉默（已知偶发类，复跑 23.7s 过）；S4 webkit=首轮对谈阵发沉默，首次复跑未先执行清理脚本（违反 Q1 纪律，过程失序归因）撞残留世界「雾灯纪」子串（hasText 为子串匹配），清理后二次复跑 1.3min 过。
- **新发现**：①「eslint 干净」类声称必须以当次 `npm run lint` 实测输出为准；②S4 `.library-world filter hasText` 子串匹配对库内残留/并发世界敏感——与 O3 提案兜底一并记为测试健壮性候选（精确匹配或清理前置）；③Q1 纪律「数据敏感用例复跑前先清理」在 S4 上再次验证必要。
- **Iris 最终独立验收（GOAL DONE 后）**：8 commit 链完整（8d5a19a→72618b3→2cd051d→e87e71f→8471be1→625b4ee→6c083c3→9120919），工作区干净；npm test 独立复跑全绿（143/39/25/16/30/3，与 Codex 报告数字吻合）；DB 基线 worlds=2/accounts=1/first_nights=0；**T3 组 GUI 独立复跑 2/2 全过（chromium+webkit，2.6min，真实模型）**；STATUS 15:00 三段式在位。验收通过，推送。

### T4 · 规则系统真化（2026-08-20 派单 → 收口）

- **主题**：把 demo 写死的规则系统换成数据驱动——技能/资产/姿态定义来自世界数据（创世装配/酒馆导入按文风生成），判定参数由定义 metadata 推导，落地 M2 设计的三个引擎工具与状态账本闭环。这是 T6 导入卡参局的硬前置。
- **commit 链**：3169633（规范）→ dcad4c8（解锁与数据驱动）→ 6f1ad2f（引擎工具）→ 03e0553（账本闭环）→ 35517d0（测试收口）。规范单独 commit，每步独立 commit，无 git add -A。
- **设计思路**（Codex 规范 T4-RULE-REALIZATION.md）：根因=规则层把「演示世界的一套定义」误当成「规则引擎本身」，定义/判定参数/能力目录/模型提示词四处复制同一份硬编码，PostgreSQL 账本表反而成了没人读的摆设。方案=「定义即数据，引擎只认数据」——世界数据（创世/导入生成 skill/asset/effect_definitions + 记录级 character_skills/character_assets 授权）→ 数据驱动规则包按定义裁决 use_skill/use_asset/take_stance → 三引擎工具收拢 modules/actions/engine.ts（resolve_uncertainty 纯判定 / consume_resource 账本扣减 / apply_effect 状态 apply-remove），DM 不代填任何规则值；失败矩阵 13 类全部 fail-closed 绝不静默改判。判定参数写 metadata.check（骰系字段预留 d20/2d6/percentile/pool/draw，T4 只解算 d20，未知骰系 RULE_DECISION_INVALID）。基础定义数量受控防泛滥：固定 2 技能（keen_insight 带 check / steady_hand 自动成功）+ 1 资产（travel_kit consumable）+ 1 姿态（watchful_guard），按四类文风各一套确定性文案，无模型调用，ON CONFLICT DO NOTHING 幂等。
- **验证证据**：npm test 全链绿（类型检查、Core 149 含新增 rule-realization 6 项、契约 39、应用 25、前端契约 16、PG 集成 33 含新增 postgres-rule-realization 3 项、构建 + 渲染 3）；eslint 本次实测 0 error。GUI T4 组真实模型 2/2 过（T4-1 自建世界 keen_insight 检定技能回合落库时间线可见 16.1s、T4-2 旅行工具包「剩余 2 次」→ 使用 →「剩余 1 次」12.8s，资产扣减经 consume_resource 真实走账本）。范围化 GUI 回归（总纲第五章纪律）t4 2 + c-actions 4 + o-action-suggestions 4 + l-genesis 2 + n-guided-genesis 3 + t1-first-night 3 + 冒烟 a-library 6 + b-record 7，合计 31 项全部一次通过，零失败。演示世界零回退：C1/C3 既有断言不改一次通过，realm_dev demo 定义 metadata 实锤在库（check d20+2/目标 12 + letter_seal targets 全文）。清理脚本两轮执行回基线 worlds=2/accounts=1/残留测试世界 0。
- **新发现**：①能力目录/规则包/选择解析/模型提示词四处硬编码同一份 demo 数据——「解锁」必须四处同步去硬编码，只改 SQL 仍会被规则包 FatalTurnError 卡死；②定义即数据后，演示桥段（蜡封）逐字落 metadata 并由种子幂等 upsert，存量库重跑种子即对齐，避免行为回退；③资产 affordance 描述「剩余 n 次」直接投影 character_assets.quantity，是账本闭环的确定性 GUI 锚点（不依赖模型叙事），GUI 扣减断言据此稳定；④引擎工具只在发布事务内由已授权 Receipt 驱动，consume_resource/apply_effect 与源事件/Receipt/Head 同事务原子提交，重放幂等不重复扣减/不重复施加（PG 集成确定性覆盖）。
- **下一轮决策**：T4 是 T6 导入卡参局的硬前置——导入的技能/资产卡现在能经 ensureWorldBaseRuleDefinitions/授权与数据驱动规则包真正参局。后续按迭代主轴推进；T5 多骰系（2d6/percentile/pool/draw 真实解算 + metadata.check.system 分流）、T6 导入卡参局一并顺延。
- **Iris 独立验收（2026-08-20 ~17:30）**：probe goal NULL 干净退场；commit 链 5 个与报告一致、工作区干净；代码层核对——action-state.ts 三处硬编码 key（careful_observation/signal_lantern/guarded_watch）残留清零、engine.ts 三引擎工具落地、deterministicInteger 收敛 engine.ts 单点；npm test 独立复跑全链绿（exit 0）；GUI T4 组独立复跑 **4/4**（chromium+webkit 真实模型，1.6min）；dev server 重启后 DB 基线 worlds=2/accounts=1/first_nights=0。**框架交接决策（Lyle 2026-08-20 拍板）**：T5 起执行框架由 Codex(qwen3.7-max) 换为 kimi-code(K3) via kimi-kite——同项目同类型任务横向对比实验；交接包与 T5 任务书已备（含项目纪律平移、环境操作要点、已知坑清单、Codex T4 参照数据）。

### T5 · 骰子真随机与多骰系（2026-08-20 派单 → 收口）

- **主题**：deterministicInteger 哈希查表换密码学安全真随机；M2 五种骰系（d20/2d6/百分骰/骰池/抽牌）全部数据驱动落地；判定骰点进 Delivery Projection 让玩家可见。框架对比实验首个非 Codex 批次（kimi-code K3 via kimi-kite 执行）。
- **commit 链**：b4ca9bb（规范）→ aed2491（真随机源与统一结构）→ eabe351（多骰系）→ 3a85a6c（投影与 UI）→ 31ca94d（测试收口）。规范单独 commit，每步独立 commit，无 git add -A。
- **设计思路**（规范 T5-DICE-RANDOMNESS.md）：根因=seed 哈希查表不是骰子（同行动重放同结果）、骰系只有 d20、骰点落库但不透出。方案=「掷骰一次物化，重放只读库」——随机源换 node:crypto CSPRNG 并留 randomInt 注入点（测试确定性序列）；掷骰只在裁决时刻发生，MechanicDetail 随 Receipt 与 action.transaction.committed 同事务原子落库；重放幂等三层共存（进程内 callId 指纹去重 / 回合 completed 短路 / 跨进程读已落库 Receipt），未提交候选的骰点随候选废弃不违反幂等。五骰系契约全部由 metadata.check.system 分流（不新增硬编码分支）：d20/2d6 目标值比大小、percentile 成功率语义（roll≤target+modifier 夹取）、pool N 颗数成功数、draw 不放回抽 1 张；暴击规则按系（d20 天然 20/1、2d6 双 6/双 1、percentile ≤5/≥96、pool 全成功/零成功）。统一 MechanicDetail（rolls/modifier/target/total/success+critical/fumble，draw 附 skillKey/drawnCard/deckRemaining）。抽牌不放回走 provider.listDrawnCards 扣本记录已物化抽牌（PG 查 action_receipts；内存组合无历史=完整牌堆），抽空 DECK_EXHAUSTED fail-closed。玩家可见性：delivery 投影放开带骰点的判定事件（M2 第一批「不透出」规则按任务书显式开启，无骰点自动行动仍留控制面），既有事件形态不加新事件表（T3 先例），前端只渲染投影值不模拟骰点。
- **验证证据**：npm test 全链（当次实测）：typecheck 干净、Core 160（+11 骰系语义）、契约 39、应用 25、前端契约 16、PG 36（+3：五骰系语义+物化+重放 duplicate/重读同结果、抽牌不放回耗尽、大样本分布非恒定）；eslint 当次实测 0 error 0 warning；验收方独立复跑全链数字一致（含渲染 3）。GUI T5 组真实模型 2/2（chromium 32.2s + webkit 22.3s：自建世界检定→骰点行可见→刷新重进逐字一致）。范围化回归（总纲第五章）6 spec × 双引擎 50 项 47 过（17.3min），O3 chromium 复跑 16.9s 过（提案断言对模型输出敏感，T2 起已知偶发类）。演示世界零回退：C1/C3 既有断言不改通过。清理回基线 worlds=2/accounts=1/first_nights=0/action_receipts=0（回归期 C 组 demo 检定落账 2 行手动清除，无账本副作用）。
- **新发现**：①delivery 投影的 EVENTS_SQL 把 action.transaction.committed 整体排除在玩家时间线外（M2 第一批规则）——「骰点可见」必须先改这条 SQL 过滤（带 mechanic 才放行），否则投影层万无一失也到不了玩家；②GUI「刷新重读」断言不能裸 reload——应用按 last-opened 恢复会回到 demo 记录，需刷新后重进目标记录（T2 直链重进先例）；③信封事件跨回合累计，断言骰点要取最新一条（findLast）；④幂等重放必须原样复用整条命令对象（writeToken 也要同），否则 IDEMPOTENCY_CONFLICT；⑤O3 提案缺失偶发第三次出现（T2/T3/T5），提案兜底素材候选体验项再次顺延；⑥环境存在并发活动（他 agent 的 demo 回合与 DB 清理），测试核对 DB 时要预期记录会被并发清理。
- **下一轮决策**：按迭代主轴进入 T6 导入卡参局（T4/T5 双前置已就绪：导入卡数据驱动参局 + 多骰系可挂在导入技能定义上）；顺延候选——缺陷 #12 DM 审查降级路径、UI 侧记忆刷新、O3 提案兜底素材、T7 观察者供给。

### T6 · 导入卡参局（2026-08-20 派单 → 2026-08-21 收口）

- **主题**：酒馆导入角色卡进入游玩闭环——授权闭环（卡技能 + 基础技能 + 基础资产，幂等）与回合内真实参局（use_skill 走数据驱动规则包 + T5 骰点投影）。消解缺陷 #6。
- **commit 链**：0c569d5（规范）→ 57c3714（授权闭环）→ 5fe6734（挂入通道）→ 0fa301d（测试收口）。规范单独 commit，每步独立 commit，无 git add -A。
- **设计思路**（规范 T6-TAVERN-IMPORT-PLAY.md）：调查实锤——进阵容两条路径（新建记录装配/既有记录挂入）T4 已授权基础技能，但资产未授权 AI 实例、酒馆卡无结构化技能槽位（parser 不读 data.extensions）、导入卡无挂入既有记录的 UI 入口。方案——卡携带技能来源定案为 V2 标准扩展点 `extensions.realm_skills`（数组 {skillKey,title,description,check?}）：parser 透传、导入服务逐条校验（check 按 T5 契约 parseCheckSpecification 预验）、非法跳过+warnings 显式报告、合法落 skill_definitions（realm.imported.v1，ON CONFLICT 幂等）、profile.realm_skill_keys 记卡→技能关联（无新表新迁移）；授权统一收口 grantCharacterCardSkills + grantBaseAssetToInstance（quantity 参数化，AI 实例 1 / 玩家 2 不变），装配与挂入两路径同调用；挂入通道 attach-character 命令 + 角色行「入阵容」按钮。参局零新代码——T4 持有技能提示词/规则包 + T5 骰点投影直接承接。
- **验证证据**：npm test 全链（当次实测）Core 161（+1）/契约 39/应用 25/前端契约 16/PG 38（+2）/构建+渲染 3；eslint 当次实测 0 error 0 warning。PG 集成：授权闭环幂等（卡技能 2+基础 2+资产 quantity 1、玩家 2 不变、重复挂入行数不变、跨世界定义/不存在记录 WORLD_NOT_FOUND）、卡技能 2d6 推导与 resolver MechanicDetail、未定义 SKILL_NOT_AVAILABLE、非持有不进持有列表。GUI T6 组 2/2（chromium 4.2m 一次过、webkit 12s 前置秒挂无现场→复跑 58.3s 过，定性偶发）。范围化回归 23 项 × 双引擎 46 项：块 2（t4+t5+a+b）32 项一次全过；块 1（t6+r+c）12 过 2 失败——C1 webkit 复跑 1.1s 过（批量运行页面加载时序偶发）、T6-1 chromium 复跑 41.6s 过（模型技能选择波动）。清理回基线 worlds=2/accounts=1/first_nights=0/action_receipts=0。
- **新发现**：①插话与在场回合是固定计划 actionBudgetPerCharacter=0——「点名让 AI 角色用技能」在结构上不可能，角色用技能只在普通回合 DM 激活且预算 ≥1 时发生（GUI 引导措辞必须服务于 DM 激活）；②角色 propose 提示词只含 displayName+场景 canon+工具清单，卡面 personality/description 不进 propose——措辞引导只能靠玩家请求文本与技能枚举标题/描述对齐（「稳定操作」这类泛化描述会抢走请求，点技能名+对齐描述后通过）；③卡面强指令（「每次行动都必须用技能」）会让模型一次调多个工具撞 CHARACTER_ACTION_INVALID 超预算——引导要软；④整批次回归单进程跑 46 项曾被宿主回收（exit -1 无输出）——分块 + 输出落盘是稳妥形态；⑤T6-1 类「模型自由选择技能」用例天生波动，断言锚定「带检定技能（d20/2d6）」而非具体技能名是稳定写法。
- **下一轮决策**：按迭代主轴进入 T7 观察者之眼看世界；顺延候选——缺陷 #12 DM 审查降级路径（T6 回归期 DM_OUTPUT_REJECTED 仍多次出现）、UI 侧记忆刷新、O3 提案兜底素材。
- **Iris 独立验收（2026-08-20 ~18:35）**：commit 链 6 个（含收口 docs f1f50c3）与报告一致、工作区干净；npm test 独立复跑全绿且与 kimi 声称数字逐项吻合（Core 160/契约 39/应用 25/前端 16/PG 36/渲染 3）——自验诚实度无失实；代码层核对 CSPRNG（node:crypto randomInt）/五骰系 case 分支/投影 parseDiceMechanic/前端 data-dice-system 锚点全部落地；GUI t5-dice 独立复跑 2/2（28.8s）；DB 基线 2/1/0/0。**框架对比实验结论（kimi-code K3 vs Codex qwen3.7-max，同类引擎层改造）**：①速度 kimi 快约 2.4 倍（主体 44min vs 1h48m）；②规范/commit 纪律/自验诚实度两者同级；③自抓真设计点能力两者同级（kimi 抓投影 SQL 过滤，codex T4 抓基建坑）；④结构差异=回合边界：kimi 的后台任务通知可自动唤醒续跑 turn（生产实锤 turn 16/18 均自动触发，闭环完整），但 kite 把首个 prompt.completed 判成终态导致卡片提前标完成（kite 侧缺陷，已派单修复）——对比实验中两次"疑似中断"实为 kite 观测模型与 kimi 回合模型的错位，非 kimi 丢任务。

### T6 · 复核补记（2026-08-21 06:45 +08:00，kimi 接管主线后的重新验收，仅验收未开发）

- **d98022b 事实落档**：酒馆来源徽标只认 `source_format === "tavern"`，导入实际存 `sillytavern_character_card`/`tavern-*`，徽标恒「原生」；修正为前缀族识别。该修正在 T6 收口（908586f）之后落 HEAD，01:01 STATUS 条目未覆盖，补记于此。
- **重新验收证据**：tavern-import 4/4、postgres-tavern-import-play 2/2、typecheck/lint 干净；npm test 全链 282 项全过 exit 0；GUI t6 chromium 35.3s 一次过 / webkit 首跑模型侧失败（零事件+安全校验警告条，artifact 留存）→ 清理后同引擎复跑 45.7s 过（定性模型偶发）；DB 实查 cartography 定义/授权/2d6 receipt 全链在库；清理回基线 2/1/0/0。
- **环境干扰实录**：验收窗口存在另一会话的同库并发 GUI 活动（同款 t6 spec 一次 + t4/t5/a/b 32 项回归，非验收方启动，未干预）；最终基线计数在该进程结束后实测。

### T7 · 观察者之眼看世界（2026-08-21 派单 → 收口，kimi 接管主线首批）

- **主题**：观察者模式有真实内容可看——世界自演/旁白推进闭环。消解缺陷 #3。
- **commit 链**：1767788（T6 复核补记）→ cf74066（规范）→ 9720f60（会话账本 0020）→ 5b6d5d1（自演回合管线）→ e7665a8（前端自演卡）→ 本次（测试收口 + STATUS/迭代日志）。规范单独 commit，每步独立 commit，无 git add -A。
- **设计思路**（规范 T7-OBSERVATION-VISION.md）：根因=观察者席位/投影完备但无「无人输入的推进器」（自治回合只有插话/presence 且都挂玩家回合后）。方案=显式触发的自演会话：`record_self_play_sessions` 账本行（五态+部分唯一索引+RLS，T1 状态行先例）+ 单飞拍循环（T2 调度形态）+ 每拍一次完整 executeTurn 自治回合（payload.selfPlay 变体，T3 presence/interjection 先例）——planner 真实 DM 模型规划后确定性规整（旁白恒开/预算恒 1/空激活兜底），drafter/validator 与玩家回合同路径（保留模型复核），releaseBuilder 无玩家事件、事件附 selfPlay 拍号标记透出投影与前端锚点；版本冲突重读重试一次原样复用 T3 模式；拍后 sync_turn+场景晶化同型触发；取消=拍边界收束 cancelled（在途拍跑完如实计账）；stale 懒恢复防进程崩溃失控。
- **验证证据**：npm test 全链绿（Core 166/契约 40/应用 32/前端 16/PG 40/渲染 3，exit 0；eslint 0 error）。GUI t7 真实模型四次独立运行全过（chromium 4.6m/26.9s、webkit 5.2m/39.6s）。范围化回归 29 spec 项 × 双引擎 58 项：chromium 28/28 一次全过；webkit 27 过 + B5 复跑 47.8s 过（乐观发送时序偶发，T1 起已知类，复跑前已清理）。清理回基线 2/1/0/0 + record_self_play_sessions=0。
- **新发现**：①自建世界（批次 S 鬼影清除后无默认 AI 角色）的自演拍=纯旁白推进——「角色行动」乐趣点依赖世界有 AI 角色，观察者世界的角色密度是体验前提（T8/T10 候选：创世默认同伴或引导添角色）；②自演拍失败族实测为 DM_OUTPUT_INCOMPLETE（结构校验拒绝：模型输出的语义段拼接/角色回应不完整），与玩家回合 422 同族——fail-closed 落 failed 且前端可再演的形态被真实验证（8 次会话 3 次首拍失败后重试 completed）；③DM_OUTPUT_REJECTED/DM_OUTPUT_INCOMPLETE 在自演场景被放大（每拍都是一次完整模型链），缺陷 #12 降级路径从「体验优化」升级为「自演可靠性前置」，纳入 T10 安全收口；④事件 payload 顶层即 mapLocalFormalEvent 的 metadata 对象（presence/selfPlay 标记直挂顶层，非 metadata 子键）——写测试断言时按 presence 先例对齐；⑤T7 顺带补登 docs 索引 T4–T6（三批规范未登记的旧账）。
- **下一轮决策**：按主轴进入 T8 世界管理台（规范 9779797 已定稿：归档=封存+owner 门禁+受限物理删除+信息密度）；T9 规范 94a4093 同步定稿。顺延候选——缺陷 #12 降级（T10 收口）、O3 提案兜底、UI 记忆刷新。

### T8 · 世界管理台（2026-08-21 派单 → 收口）

- **主题**：世界浏览/归档/删除 + 世界卡信息密度。消解缺陷 #4。注意：本批期间工作区发生过一次外部隔离事件——旧 session 误跑出的 T8 半成品（迁移/测试文件）被宿主隔离到 /tmp，本批全部文件按当前真实代码重新撰写，未恢复隔离 patch。
- **commit 链**：9779797（规范）→ 0d1091c（后端：0021 + 命令 + 门禁 + scope.worldStatus + PG 测试）→ cec2915（前端世界卡 + i18n）→ 本次（GUI/回归/清理 + STATUS/迭代日志）。无 git add -A。
- **设计思路**（规范 T8-WORLD-ADMIN.md）：归档=封存（status 翻转幂等 + 开新局/导入/挂角色/回合/自演全拒 + 读取自由），删除=受限物理删除。两个实测修正落进实现与规范：①删除判据由「零事件」收窄为「零记录」——记录装配即写 character_skills/assets 账本行，character_skills→skill_definitions 与 participants/command_inbox→memberships 两组 NO ACTION 引用与世界分支交错（跨分支级联不排序），append-only 触发器与 realm_runtime 授权边界使「有记录世界的运行时物理删除」不可行，WORLD_NOT_EMPTY 引导归档；②观察者创世 membership role='observer' 不是 owner——owner 门禁下观察者姿态世界主不能归档/删除，管理命令以 role='owner' 为准（创建者默认 owner）。
- **验证证据**：PG 集成 3/3（exit 0；含 realm_runtime 受限角色真实归档/删除授权实锤、级联清零七表、demo 有记录拒删、活动自演防护、幂等）；schema 契约 21/21（exit 0）；typecheck/eslint 0 error；i18n+前端契约 14/14。GUI t8 组双引擎各一次全过（chromium 2/2 5.1s、webkit 2/2 3.4s）。范围化回归 chromium 27/27（块 1 t8+a+s+b 19 项 5.3min、块 2 l+n+t1 8 项 5.1min）+ webkit 跨引擎面 10/10（3.0min），零失败无需复跑。清理回基线 worlds=2/accounts=1/first_nights=0/action_receipts=0/self_play=0/files=0/articles=0。
- **新发现**：①「schema 预留状态 ≠ 可接线」——worlds.status 的 archived 自 0001 存在但零写入方；接线时才暴露权限裸奔（library 写命令此前无任何 owner 校验），T8 只给管理命令加了门禁，其余命令的权限加固是遗留项；②跨分支 CASCADE 与 NO ACTION 引用网使「运行时物理删除非空世界」架构性不可行（realm_runtime 无触发器权限是 M1 有意设计）——归档是唯一诚实的封存语义，删除只能是空世界回收；③worldstyle/stance 等既有命令对归档世界仍放行（文风/姿态切换不生产内容，属合理留白）；④删除当前打开世界的回退链（last_record_id FK 置空 + openDefaultRecord 回落 onboarding）批次 S 已备，本批只接线。
- **下一轮决策**：按主轴进入 T9 canon 回读（规范 94a4093 已定稿：产生侧=晶化入图谱+API 去 demo 硬编码，消费侧=brief.canon 正史注入）；顺延候选——缺陷 #12 降级（T10）、library 非管理命令权限加固、O3 提案兜底。

### T9 · Canon 回读（2026-08-21 派单 → 收口）

- **主题**：canon 产生侧（晶化入图谱 + API 去 demo 硬编码）与消费侧（正史注入回合生成）双端闭环。消解缺陷 #15，部分消解 #13/#14。
- **commit 链**：94a4093（规范，T7 期先行）→ 4857873（产生侧）→ 4254565（消费侧）→ 本次（GUI/回归/清理/文档收口）。无 git add -A。
- **设计思路**（规范 T9-CANON-READBACK.md）：产生侧=晶化 approved delta 落库后独立 best-effort 入图（不碰晶化主事务；世界本体实体确定性 id + 白名单谓词 record 级 Claim，M5 层级门禁不打扰用户）；API=显式 worldId + membership + 当前 active 世界线解析；消费侧=record-scope 读 story_canon 以上（supersede 最新、上限 12、截断、fail-closed 空串）经 brief.canon 注入 buildSceneCanon——五调用点零改动。正史只约束生成，模型输出仍走既有校验/复核链，零新增写库路径。
- **验证证据**：PG 3/3 + Core 4/4 + 受影响回归 24 项全过（逐条 exit 0）；GUI t9 双引擎终态过（chromium 26.6s/webkit 18.5s）；范围回归 k-graph 6/6×双引擎、m+c 6/6、a+b 13/13。失败定性实录：t9 spec 两次规格化修正（表单双 select 严格模式、close 按钮 aria-label≠文本）+ 回合步骤当日模型阵发 422（DM_OUTPUT_INCOMPLETE/REJECTED 混合）——双探针实锤与 canon 无关（无 canon 201、有 canon 201），T6 措辞先例换中性措辞 + 重试上限 4 后双引擎转绿；chromium 一次失败还暴露我自己换措辞后未同步断言（界碑→我环顾四周），修复后过。
- **新发现**：①清理脚本在 canon 真实接线后整体失效——canon_revisions→canon_proposals 删除顺序撞 NO ACTION，psql 无 ON_ERROR_STOP 时事务静默回滚且 exit 0 假绿（当天多轮"已清理"实际未清）；已修顺序 + entity_world_* 清除并复验；②canon/图谱路由此前每请求新建 Pool 从不回收（拆库即未捕获异常）——共享池 + endSharedRuntimePools 修复；③「正史文本进 prompt 会诱导旁白 fact 段复述正史」撞 narratorAuthority 结构校验——回合措辞引导在 canon 世界更敏感，T6 措辞对齐纪律同样适用于玩家侧引导文案；④图谱面板数据仅挂载时加载，提案后需重开面板才可见（现状可接受，候选体验项）；⑤record-scope 的 canon 读取在 playerActor 解析之后进行——空席位（无 narrator/character 席位）记录仍 fail-closed 不装配。
- **下一轮决策**：按主轴进入 T10 死代码与假接口清理 + 持续设计审计（审计事实已在 T7 期盘点：propagation/worldline 零接线、tool-pause/parallel-candidates 未接线、memory snapshot/delta 零调用、D1 遗留链、缺陷 #12 降级）；顺延——缺陷 #12 降级路径（今日 DM_OUTPUT_* 阵发再次实锤其必要性）、图谱面板刷新、空阵容纯旁白回合的结构校验韧性。
- **Iris 独立更正（2026-08-21 11:18:20 +08:00）**：`c645c6b` 去掉 T9 GUI spec 的 retry 循环后，单次 Chromium 为 exit 1/HTTP 422（`DM_OUTPUT_INCOMPLETE`），单次 WebKit 为 exit 0/HTTP 201；两次 cleanup exit 0，基线净。无 canon/有 story canon probe 均 201，故将问题定性为缺陷 #12 的模型稳定性，不把 `9d3199d` 的 retry 绿写成单次双引擎全绿；T9 生产闭环与必要 Core/PG/API/关联回归证据保留。

### T10-A · DM 审查降级路径（2026-08-21 派单 → 收口）

- **主题**：缺陷 #12——模型 DM 复核三连否决后回合裸挂（DM_OUTPUT_REJECTED）改确定性降级。T10 拆批第一批。
- **commit 链**：21d72e7（规范）→ c1ec7be（实现）→ 本次（测试 + STATUS/迭代日志）。无 git add -A。
- **设计思路**：候选在复核前已过全部确定性硬门（权限/回执/预算/旁白权限/结构完整），模型复核自 M2 起定位就是软性兼容检查——三连否决说明复核通道不稳定而非候选违规。选定「确定性接受 + 审计标记」（validation body 可选 review 字段 + 日志），否决「保守旁白替换」方案（会引入第二套降级叙事模板且丢角色回应）。三类结果区分：结构失败照旧 fail-closed 不进复核；复核否决/非法/不可用分别 triple-veto/invalid-json/review-unavailable 记账。降级有界：复核上限 3 不变，降级零额外模型调用。
- **验证证据**：model-inference 13/13 exit 0（+4）；self-play-application 8/8 exit 0（+1，覆盖 payload.selfPlay→orchestrator.validate 共享路径）；m2-orchestration/core-runtime/runtime-turn/presence-application 33/33 exit 0；typecheck/eslint/文档布局 exit 0。未跑 GUI/PG 写入（纯内存/Core 覆盖足够，任务书允许）。
- **新发现**：①FatalTurnError 的 code 不进 message——assert.rejects 正则匹配 message 会假失败，断言错误码要用谓词形式；②self-play 应用层测试复现了 T3 版本冲突重读日志（内存夹具下拍循环与快照的既有时序），重试路径按设计工作；③DM_OUTPUT_REJECTED 字符串在 postgres-self-play 测试中作为 lastError 样例残留（非管线断言，无害）。
- **下一轮决策（T10-B 候选排序）**：①propagation/worldline 生产接线审计（modules/propagation、worldline 五模块零调用方——设计内暂缓 vs 承诺未接线的区分落档）；②memory snapshot/delta 零调用与 D1 遗留链（modules/story-record、db/story-record-repository、drizzle/）清除决策；③tool-pause/parallel-candidates 接线或标记暂缓；④library 非管理命令权限加固（T8 遗留）；⑤图谱面板挂载后才加载的刷新体验。每项需独立规范与回归证据，不为数字删占位模块。

### T10-B1 · 世界治理可达性审计 + Merge 作用域修复（2026-08-21 派单 → 收口）

- **主题**：缺陷 #13 第一步——以当前调用图逐项重估世界治理模块可达性，纠正过时「全库零调用」说法；修复 /api/worldline/merge 的作用域与审计主体。
- **commit 链**：60b577c（规范）→ 42e0723（路由修复 + 迁移 0022 + route PG 测试）→ 本次（STATUS/迭代日志）。无 git add -A。
- **设计思路**：可达性分四档落档——生产可达（canon/branching/merge）、承诺但缺 runtime 接线（conflict-detection 的 detectCausalConflicts，merge 自带 plan 内冲突逻辑不用它）、M5 明确第二批暂缓（semantic-conflict §9、propagation §7，保留不删不伪装完成）。路由修复复用 T9 资产（resolveWorldScopeForMember + getSharedRuntimePool）：worldId 显式必填、membership 404 不泄露、operator 恒为已解析 principal。
- **验证证据**：route PG 测试 exit 0（临时库拆库不污染开发库）；m5-batch2 12/12、postgres-m5 两组 6/6、schema 契约 22/22、typecheck/eslint/diff --check/文档布局全部 exit 0；开发库基线 2/1/0/0 + 图谱/canon/合并台账全 0。
- **新发现**：①真实合并写路径在 realm_runtime 下 permission denied（0004 收回 worldlines/stories/records INSERT，M5 batch2 测试全用 owner 池）——「测试用 owner 池」会系统性掩盖生产受限角色的授权缺口，后续 PG 测试默认应过 realm_runtime 重定向验证（T7/T8/T9 已有此先例）；②merge 的 worldlineExists 已按 worldId 过滤，无跨世界线缺口；③route 层吞错返 500 无日志——调试期被迫复现于 scratch，后续 route 错误路径应留本机日志（候选小项）。
- **下一轮决策（T10-B2 候选）**：①conflict-detection 接线或降级为 merge 内部件（需独立规范裁决）；②propagation/semantic-conflict 暂缓标记写入模块头注释（防再次被误报为零接线）；③route 层 500 无日志的小修；④memory snapshot/delta 与 D1 遗留链、library 非管理命令权限加固顺延。

### T10-B2 · 确定性因果冲突检测生产接线（2026-08-21 派单 → 收口）

- **主题**：M5 §4 承诺的三层因果检测接入 /api/worldline/conflict（causal 只读预览分支）。缺陷 #13「承诺未接线」项收口其一。
- **commit 链**：e694cf0（规范）→ 8920c6e（接线 + route PG 测试）→ 本次（收口）。无 git add -A。
- **设计思路**：同路由双分支——legacy（classifyWorldlineConflict 纯游标分类）零改动；causal 分支以 mode="causal" 显式分流，worldId + membership + active 世界线经 T9/T10-B1 资产解析，claims/edges 只读库（客户端无注入面），changeSet 严格白名单解析，detectCausalConflicts 纯函数零改动，响应带 algorithm=realm-causal-conflict/v1 与 scope 元信息，零写库零自动 branch。
- **验证证据**：route PG exit 0（legacy 不回归/400/404/hard 冲突/依赖冲突/DB 独有 claim 证明/台账零写）；m5 核心 12/12 exit 0；typecheck/eslint/diff --check/文档布局全 exit 0；开发库基线不变。
- **新发现**：①T10-B1 的 scope/共享池资产直接复用无摩擦，路由接线成本已在 T9/T10-B1 付清；②terminate/supersede 的 targetClaimId 必填校验是输入面唯一非显然约束（M5 测试夹具已示范）；③路由 500 仍无日志（同 T10-B1 发现，留小项）。
- **下一轮决策（T10-B3 候选）**：①propagation/semantic-conflict 暂缓标记写入模块头注释；②memory snapshot/delta 零调用与 D1 遗留链（modules/story-record、db/story-record-repository、drizzle/）清除决策；③tool-pause/parallel-candidates 接线或暂缓；④library 非管理命令权限加固；⑤route 层 500 日志小修。

### T10-B3 · Worldline 路由 500 脱敏可观测性（2026-08-21 派单 → 收口）

- **主题**：T10-B1/B2 共同发现「route 层 500 无日志」收口——内部失败保留 500 契约 + 脱敏结构化诊断。
- **commit 链**：d4541ba（规范）→ 25e5f96（助手 + 两路由接线 + 断库回归）→ 本次（收口）。无 git add -A。
- **设计思路**：共享助手 logRouteInternalError 只输出白名单字段（route/stage/errorType/code），errorType 构造名白名单规整、code 仅 errno/SQLSTATE 形态带出；error.message/stack 一律不进日志（断库错误的 message 含地址）；业务 404/输入 400 不伪报 internal。不改业务响应契约。
- **验证证据**：route 测试 5/5 exit 0（含 2 项新断库回归：HTTP 不变 + 捕获日志恰一行白名单字段 + 严禁字段逐项缺席断言）；typecheck/eslint/diff-check/文档布局 exit 0；开发库基线不变。
- **新发现**：①pg 连接失败是 name=error 的普通 Error + code=ECONNREFUSED——errorType 规整为 Error、code 带出是足够的定位面；②console.error 捕获测试要在 finally 恢复，否则污染同进程其他测试输出；③断库测试用 127.0.0.1:1 无需临时库，拆库纪律仍由共享池 endSharedRuntimePools 承担。
- **下一轮决策（T10-B4 候选）**：①propagation/semantic-conflict 暂缓标记写入模块头注释；②memory snapshot/delta 与 D1 遗留链清除决策；③tool-pause/parallel-candidates 接线或暂缓；④library 非管理命令权限加固；⑤图谱面板刷新体验。

### T10-B4 · Memory Snapshot/Delta 生产入口 + D1 退役围栏（2026-08-21 派单 → 收口）

- **主题**：缺陷 #11 的 snapshot/delta 零接线收口（受限生产 API）+ D1 链保留决策与静态围栏。
- **commit 链**：9667d38（规范）→ 4dc904a（API + 共享池最小修复 + 围栏 + route PG 测试）→ 本次（收口）。无 git add -A。
- **设计思路**：snapshot/delta 是只读缓存契约（M3 §2.2/§2.3），生产入口=受限只读/创建，客户端只可传 recordId/kind，scope 经 record-scope 解析；D1 链不删——迁移参考与类型宿主（record-store 仍承载 ProjectionEvent/RecordProjection 类型），删除前置条件（零生产引用+替代锚点+独立评审）入规范，围栏用静态扫描全部活动路由证明零导入。
- **验证证据**：route PG exit 0（伪造不生效/epoch stale/触发器拒绝等全矩阵）；memory 核心 11/11 + PG 6/6；api-core-wiring 6/6；typecheck/eslint/diff-check/文档布局全 exit 0；基线不变。
- **新发现**：①/api/memory 存在与 canon/图谱同款的每请求建池不回收——第三批同型泄漏（T9 canon、T10-B4 memory），共享池应成为路由默认约定（候选：统一 route 池助手推广）；②delta 增量以世界游标为界——测试里新增结论前必须推进 worldline 头（appendAuthorized 取当前游标）；③update 操作有 revision shape CHECK 约束必须带 supersedesMemoryId；④D1 record-store 的类型宿主身份意味着「删除」需先迁移类型出处，不是纯删文件。
- **下一轮决策（T10-B5 候选）**：①tool-pause/parallel-candidates 接线或暂缓标记；②library 非管理命令权限加固；③图谱面板刷新体验；④propagation/semantic-conflict 模块头暂缓标记；⑤路由共享池约定推广（其余每请求建池点普查）。

### T10-B5 · M4 Tool-Pause / Parallel-Candidates 运行态围栏（2026-08-21 派单 → 收口）

- **主题**：缺陷 #16 收口——tool-pause/parallel-candidates 定性 contract-only-deferred + 机器可读标记 + 静态围栏。
- **commit 链**：04e7d4b（规范）→ 3014676（围栏 + M4 文档追加 + 测试）→ 本次（收口）。无 git add -A。
- **设计思路**：不接线的硬理由=gateway 能力——活动 OpenAI-compatible 网关只在完整响应返回 tool_calls，没有流式 tool-call 事件可消费；parallel-candidates 在单模型配置下无真实业务调用点。与其伪造调用点，不如把「为什么现在不能接」写成可执行围栏。未来接线前置条件五项入规范与模块头。
- **验证证据**：围栏 2/2 exit 0；m4-streaming-interruption 11/11 exit 0；api-core-wiring 6/6 exit 0；typecheck/eslint/文档布局/diff --check 全 exit 0；零 DB 写入，基线不变。
- **新发现**：①旧摘要「raceCandidates 已接线」是错误记忆（可能源自 M4 批次时的一次短暂接线后回退），本批以追加更正收口——声称接线状态必须以当次 grep 为准；②运行态标记用注释内 `@runtime-status` 键即可被静态测试锚定，是零依赖的防漂移形态。
- **下一轮决策（T10-B6 候选）**：①library 非管理命令权限加固（T8 遗留）；②路由共享池约定推广（其余每请求建池点普查）；③propagation/semantic-conflict 模块头暂缓标记（同 B5 形态）；④图谱面板刷新体验；⑤D1 实际删除评审（前置条件见 T10-B4 规范）。

### T10-B6 · Library 非管理命令权限与 Runtime Pool 加固（2026-08-21 派单 → 收口）

- **主题**：T8 遗留「其余命令权限另案」收口 + library 路由 owner pool 问题。顺带实锤并修复 T9 潜伏缺陷。
- **commit 链**：b6f2063（规范）→ fa89ff5（实现 + 围栏 + 测试）→ 本次（收口）。无 git add -A。
- **设计思路**：权限矩阵以「非成员 404 不泄露 / 成员 observer 只读 403 / owner-only 403」为骨架；分池契约按 realm_runtime 授权面实查切割——runtime 池承载 list 与 style/story/branch/archive/delete，owner 例外枚举（world/character/record/attach/stance）因 worlds/character_definitions/participants/record_heads INSERT 与 memberships/participants UPDATE 未授权且本批不新增迁移，暂留 owner pool 并以静态契约锚定不再扩大。
- **验证证据**：permissions PG exit 0；library 圈回归 11/11 exit 0；i18n/契约/auth 18/18 exit 0；typecheck/eslint/diff-check/文档布局 exit 0；基线不变。
- **新发现**：①T9 canon 查询放 scope 主事务是潜伏缺陷——PG 失败语句中止整个事务，JS try/catch 救不回来；辅助读取必须独立事务（本批已修）；②姿态往返后 owner 变 player（S 语义：stance 只在 observer↔player 间切换）——测试编排要注意角色漂移；③既有 owner-pool 夹具与新策略的冲突用「对齐夹具状态」而非删断言解决；④list 过滤后 membershipRole 的假 player 回落实际不可达。
- **下一轮决策（T10-B7 候选）**：①owner 例外命令的最小授权补齐迁移评审（worlds/character_definitions/participants/record_heads 列级 INSERT + memberships/participants 受限 UPDATE）；②propagation/semantic-conflict 模块头暂缓标记；③图谱面板刷新；④D1 实际删除评审（前置条件见 T10-B4）。

### T10-B7 · Library owner-pool 例外的最小授权迁移评审与下沉（2026-08-21 派单 → 收口）

- **主题**：T10-B6 owner 例外枚举的迁移评审与下沉——0023 最小授权，library/generate/import 全量 realm_runtime。
- **commit 链**：d48319f（规范）→ 3f885af（迁移 + 下沉 + 契约 + 测试）→ 本次（收口）。无 git add -A。
- **设计思路**：逐表逐列矩阵从真实 SQL 抽取（不规范想象授权）；INSERT 表级同 0022 型、UPDATE 列级守 0004 以来惯例；RLS/触发器/业务门禁不由 grant 替代。owner 例外清空而非保留「为了方便」的 fallback。
- **验证证据**：双库实测 exit 0（迁移前 permission denied 证据 + 迁移后全命令 + RLS 隔离 + 路由无需 DATABASE_URL）；library 圈 18/18、契约/文档/typecheck/eslint/diff-check 全 exit 0；基线不变。
- **新发现**：①测试编排的角色漂移（姿态往返后 owner→player）第二次咬人——凡涉及 owner 命令的测试，姿态操作放最后；②创世含开幕记录 → 非空不可删（T8 语义自洽）；③node:test 的 t.after 按注册顺序执行——维护连接必须最后注册回收；④files/[id]、settings/language、auth/me 仍用 owner pool 读（授权面够但属独立路由），留统一收尾。
- **下一轮决策（T10-B8 候选）**：①残余 owner pool 读路由（files/[id]、settings/language、auth/me）下沉收尾；②propagation/semantic-conflict 模块头暂缓标记；③图谱面板刷新；④D1 实际删除评审（T10-B4 前置条件）；⑤回合管线与 library 的授权面文档化（POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段）。

### T10-B8-A · 残余 owner-pool 读路由统一下沉（2026-08-21 派单 → 收口）

- **主题**：T10-B7 遗留的 files/[id]、settings/language、auth/me 三条 owner-pool 路径下沉共享池。
- **commit 链**：6d4f9ac（规范）→ 0a38ca2（实现 + 契约 + 测试）→ 本次（收口）。无 git add -A。
- **设计思路**：授权面先实查再动手——三张表所需权限 0012/0015/0016 已授，故零迁移纯下沉；auth/me 的 best-effort 反查语义（缺 DB 不阻塞身份应答）原样保留，只换池来源。文件 id 形态校验（hex-only）与语言非法值真实写回是测试编写期撞上的两个既有行为，按事实写进断言。
- **验证证据**：route PG exit 0（门禁/响应头/写回/gate 两态全覆盖）；auth-session 4/4、wiring 7/7、typecheck/eslint/文档/diff-check 全 exit 0；基线不变；本批临时库 0 残留。
- **新发现**：①world_files.id 是 hex-only 形态（file_[a-f0-9]+），测试数据 id 含非 hex 字符会被 404——形态校验真实生效；②语言非法值回落不是只读规整而是真实写回 zh-CN——测试顺序必须把「反读验证」放在「回退写」之前；③owner-pool 至此清零（app/api 全路由共享池），连接生命周期纪律统一。
- **下一轮决策（T10-B9 候选）**：①propagation/semantic-conflict 模块头暂缓标记；②图谱面板刷新体验；③D1 实际删除评审（T10-B4 前置条件）；④POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化。

### T10-B9-A · M5 propagation / semantic-conflict 运行态围栏（2026-08-21 派单 → 收口）

- **主题**：缺陷 #13 的 M5 暂缓模块定性收口——propagation（引擎 + worker）与 semantic-conflict（模型评估器 + needsSemanticReview）定性 contract-only-deferred，静态围栏防漂移。M5 §7/§9 设计内第二批暂缓的如实落档，不宣称生产接入。
- **commit 链**：19fe444（规范）→ ab83771（围栏测试）→ 本次（收口）。无 git add -A。
- **设计思路**：与 T10-B5（M4 围栏）同型但有两点不同——①本批模块头不加 `@runtime-status` 标记（任务边界：不改模块文件，行为与内容零改动），定性只落在规范与围栏测试；②围栏用 .mjs 静态扫描而非 .ts 单测形态。围栏的扫描范围刻意限定 app/ 与 modules/application/ 生产源码：tests/m5-* 合法调用被测模块、模块自身是定义点，全仓禁字符串会误伤这两类。正向锚定（模块存在 + 导出不变）防「删模块冒充未接线」。
- **验证证据**：围栏 3/3 exit 0；m5-world-governance 7/7 + m5-batch2 5/5 exit 0（行为零回退）；api-core-wiring + 文档布局 9/9 exit 0；typecheck/eslint/diff --check 全 exit 0。零 DB 写入，基线不变，realm-dev.service 无需重启。
- **新发现**：①上一会话 retry 卡住时规范初稿已落盘但未提交——跨 session 续作先 git status 核对工作树再动手，未提交规范文件可以直接接着用；②T10-B5 型的「模块头标记」与「不改模块文件」两种围栏形态并存，选型取决于批次边界是否允许触碰被围栏模块本身。
- **下一轮决策（T10-B10 候选）**：①图谱面板刷新体验；②D1 实际删除评审（T10-B4 前置条件）；③POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；④propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二，未满足前任何接线属假实现）。

### T10-B10-A · 知识图谱面板刷新闭环（2026-08-21 派单 → 收口）

- **主题**：T10-B3 起多轮候选「图谱面板刷新体验」收口——只做刷新体验：显式「刷新图谱」入口 + 加载状态 + 失败重试 + 竞态/卸载防护，不碰 API/鉴权/DB/迁移/inference 与 POST 编辑语义。
- **commit 链**：0cd2a55（规范）→ 803f45e（实现+K7）→ 本次（收口）。无 git add -A。
- **设计思路**：初次加载与手动刷新统一进同一条 load()——loading + loadedOnce 状态机区分「正在加载图谱…」/「正在刷新…」文案；双端点（world-knowledge + canon）Promise.all 整体成败，任一失败保留旧数据不出撕裂态；每次 load 递增序号仅最新响应落 state（防旧覆盖新），mountedRef 防卸载 setState；非 2xx 显式 throw 进 notice（此前会静默 json() 出空快照冒充「没有数据」）；选中实体存在保留/消失才清空。按钮置 .graph-actions 末位——CSS sibling 选择器让首按钮为主色实心，末位追加避免改变「新建实体」既有主次样式。
- **验证证据**：K7 Chromium exit 0、WebKit 补跑 exit 0（focused 单用例，未跑完整 GUI）；typecheck/受影响 eslint/api-core-wiring 7/7/文档布局 2/2/git diff --check 全 exit 0；未跑全量。种子数据落开发库 demo 世界（同 K1–K6 惯例，clean-gui-test-data.sql 可清），零临时库对象，realm-dev.service 保持 active。
- **新发现**：①图谱节点标签超 6 字截断（`name.slice(0, 6)…`），K1–K6 的前缀 hasText 匹配因此成立，但 uniqueName 全名匹配会假阴性——且时间戳 base36 前导字符变化极慢，「名称前 6 字」在同窗口期重跑会撞残留实体；K7 改用节点 `<g role="button">` 的 aria-label（带完整名称）精确匹配，既绕开截断又重跑安全；②realm-dev.service 以 HOST_BIND=192.168.31.238 绑定而非回环，GUI focused 跑法需带 HOST_BIND 环境变量（playwright.config 已支持，curl 127.0.0.1 探测会误判服务不可用）。
- **下一轮决策（T10-B11 候选）**：①D1 实际删除评审（T10-B4 前置条件）；②POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；③propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）；④图谱自动刷新（轮询/SSE）评审——本批明确只做手动刷新。

### T10-B11-A · D1 退役可执行评审（2026-08-21 派单 → 收口）

- **主题**：T10-B4 留下的「D1 实际删除评审」前置条件收口——只审计/围栏不删除：逐文件分类实锤、record-store 混合兼容边界论证、删除批准闸门。核心结论：「D1 零运行时调用」不能偷换成「D1 可删除」——类型引用也是引用。
- **commit 链**：b2488b5（规范）→ 54bf033（audit 围栏）→ 本次（收口）。无 git add -A。
- **设计思路**：分类不靠印象靠逐文件 grep 实锤——record-store.ts 生产 2 处（local-record-service、delivery-projection）+ 测试 5 处全部 `import type`，编译期擦除所以 D1 runtime 不进生产 bundle，但这个安全完全依赖 import 形态；story-record-repository 零外部引用是唯一的 safe-to-delete 候选，仍走闸门（删除会牵动 T10-B4 围栏措辞与 local-only 契约）；drizzle tooling 的最后引用方是 local-only-boundary.test.mjs 的故意锚点，退役必须先迁移契约。audit 与 T10-B4 围栏互补而非重复：B4 只扫 route 文件，本批扫全量生产/工具/测试源码并把 specifier 解析到仓库内真实路径（兼容 @/ 别名与相对路径），record-store 的已知消费方做成精确集合正向锚定——新增生产引用方必须显式过评审改清单。
- **验证证据**：audit 5/5 exit 0；local-only 1/1、wiring 7/7、docs 2/2、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；未跑全量/GUI/PG；零 DB 写入、零删除，realm-dev.service 保持 active。
- **新发现**：①`db/schema 2.ts` 这种本地杂散文件真实存在（tsconfig exclude + .gitignore 在册）——评审面以 git 追踪文件为准，exclude/gitignore 清单也是边界审计的一部分；②story-record-repository 虽零外部引用，但它同时是 modules/story-record/public 的唯一 repository 实现——「零引用文件」与「domain 契约的孤儿实现」要分开定性，后者影响 core-runtime.test 的语义归属；③多行 import 与 `@/` 别名是静态 import 审计的两个真实坑，只按行 grep `from "db/` 会漏判。
- **下一轮决策（T10-B12 候选）**：①extract portable types（record-store 的 RecordProjection/ProjectionEvent 迁至中立地，2 生产 + 5 测试引用改道——T10-B11-A §三）；②POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；③propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）；④图谱自动刷新（轮询/SSE）评审。

### T10-B12-A · 提取 D1/PG 共享 delivery projection 纯类型边界（2026-08-21 派单 → 收口）

- **主题**：T10-B11-A 删除批准闸门第 2 条执行——delivery ProjectionEvent/RecordProjection 从 db/record-store.ts 提取为纯类型单一来源，2 生产 + 5 测试引用改道；只提取不删除，D1 旧链保留。
- **commit 链**：2e0fff3（规范）→ 2331c68（类型提取+改道+audit）→ 本次（收口）。无 git add -A。
- **设计思路**：两套同名 RecordProjection 的分离是红线——Core（story-record/public.ts，persistence-neutral 回合契约）不动，delivery（交付/UI 投影）迁入 modules/application/legacy-record-projection-types.ts，文件名记录来源、文件头写死「非 Core projection」。纯度定义落到可围栏的形态：零值导入（值导入即生产 runtime 边）+ 仅 import type 依赖 SemanticSegment/MechanicDetail。record-store 不删而是改 import type + re-export 兼容桥——story-record-repository 等 legacy 消费方零改动，单一类型来源与「不删旧链」同时成立。audit 同步为「提取后」形态：record-store 引用方从「生产 import type 允许」收紧为「仅 db/ 内部」，新模块生产消费方精确锚定两个已知文件（新增消费方必须过评审改清单）。
- **验证证据**：audit 6/6 exit 0；5 应用套件 37/37 exit 0；local-only 1/1、wiring 7/7、docs 2/2、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；未跑全量/GUI/PG；零 DB 写入、零删除，realm-dev.service 保持 active。
- **新发现**：①正则解析 import 语句有两个真实幻影源——块注释里的「import」字样（` * 不得 import db/…`）会被 \b(import) 匹配并吞掉后续真实语句的 type 标记，锚定行首（^[ \t]*import）后消失；纯度断言也必须查解析出的语句而非原始源码，否则规范注释里的模块名（story-record/public）会自摆乌龙——这恰好再次验证「不做全仓字符串禁用」的纪律；②相对路径深度是改道的高频错点（database/postgres → modules 需要 ../../），typecheck 的 implicit-any 级联（一处模块解析失败带出一串下游文件报错）是这类错误的典型信号，先看 TS2307 根因而非逐个修 any；③B11 时发现的「delivery-projection 反向依赖 modules/application」方向问题在本批落地为纯类型层——import type 编译期擦除，模块方向约束对纯类型层不形成 runtime 环。
- **下一轮决策（T10-B13 候选）**：①db/story-record-repository 与 record-store 删除评审（B11 闸门：archive 决策 + local-only 契约迁移先行）；②drizzle/ 迁移 archive 决策落档；③POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；④propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）。

### T10-B13-A · D1 pair 删除资格评审（2026-08-21 派单 → 收口）

- **主题**：B11 闸门继续推进——对第一对删除候选（record-store + story-record-repository）的资格评审：零外部引用实锤、孤儿实现与 Core 契约区分、pair 删除批准闸门。只评审不删除。
- **commit 链**：233e837（规范）→ 522e7d2（review audit）→ 本次（收口）。无 git add -A。
- **设计思路**：「有资格删」的判据做成可围栏的事实而非印象——pair 的引用方集合恰为 db/ 内部互引，扫描面必须含 db/ 自身（B11 audit 不扫 db/ 是因为只断言「引用方 ∈ 允许列表」，不扫描即隐式放行；pair 评审要读 pair 自己的 import 语句锚内部链，就必须把 db/ 放进扫描面、用路径前缀排除来定义「外部」）。Core StoryRecordRepository 的 retain 论证落到一个此前未写明的事实：Core 自带 createInMemoryStoryRecordRepository，pair 只是唯一 D1 持久化适配器而非唯一实现，core-runtime 单测走 in-memory——删除 pair 对 Core 契约与测试零影响。B11 §八的统一前置（archive+local-only）按影响面细化：pair 删除不触碰 db/index/drizzle/script，故 pair 批不含 tooling 前置——避免过度阻塞，也避免搭车。
- **验证证据**：review audit 5/5 exit 0；B11 audit 6/6、local-only 1/1、wiring 7/7、docs 2/2、core-runtime 7/7、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；未跑全量/GUI/PG；零 DB 写入、零删除，realm-dev.service 保持 active。
- **新发现**：①「扫描面排除目录」与「断言里允许目录」是两种语义——前者让该目录的引用关系对围栏隐形（B11 对 db/ 内部链其实无感知），后者才是真锚定；写引用关系审计时先想清楚要不要读被审计文件自己的 import；②B12 提取后 record-store 的出边仍有一条 runtime 依赖（semantic-segments 的 fallbackCompositeSemanticSegments）——pair 删除批的规范要记得把这条边写进影响面（presentation 模块不随 pair 受影响，只是 record-store 不再消费它）；③删除评审的闸门要按「影响面」切分前置，统一前置会把不相关的决策（drizzle archive）绑死在第一对文件上。
- **下一轮决策（T10-B14 候选）**：①pair 删除批（前置：删除批规范 + B11/B13 audit 锚点同步方案 + 派单方独立确认，见 T10-B13-A §七）；②drizzle/ archive 决策落档（tooling 批前置）；③POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；④propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）。

### T10-B14-A · 执行 D1 pair 删除（2026-08-21 派单（含用户独立确认）→ 收口）

- **主题**：B13 资格评审 + 用户独立确认后的执行批——git rm record-store + story-record-repository 两文件；pair 删除而非 D1/Drizzle 全退役，deferred 五件套与 tooling 不搭车。
- **commit 链**：22bddad（规范）→ cced318（删除+audit 围栏）→ 本次（收口）。无 git add -A。
- **设计思路**：批准链完整留痕——B11 分类 → B12 类型提取 → B13 资格实锤 → 用户确认 → 本批执行，每一步都有独立规范与围栏。删除后 audit 的关键形态转换：「存在性正断言」翻为「不存在断言 + 零引用断言」，零引用必须按语句中的 specifier 形态匹配（resolve 不到已删文件也要命中），扫描面含 db/ 自身防内部残留；db/index 删除后失去全部源码引用方但按 B13 影响面分界保留——其自身链（./schema + drizzle-orm/d1 + inactive 文案）做成锚定防搭车改动。B13 测试文件整体改写为「删除后纪事」并在文件头注明历史形态见 Git 记录，B11/B13 规范文档一字不改（历史事实不静默改写）。
- **验证证据**：两 audit 6/6+5/5、local-only 1/1、wiring 7/7、docs 2/2、core-runtime 7/7、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；staged diff 精确核对仅 pair 删除 + 两 audit 修改；开发库基线实测不变（2/1/0/0/0/0），realm-dev.service 保持 active；未跑全量/GUI/PG。
- **新发现**：①开发库表名与记忆不符——first_nights 实为 record_first_nights，基线核对要先查 pg_tables 再下结论，口头表名清单会过时；②删除批的 audit 同步里「resolve 不到的引用」是新盲区——文件删了 resolveSpecifier 返回 null，纯 resolve 路径的断言会静默放行残留 import，必须退到 specifier 形态匹配；③record_first_nights 类基线核对顺带证明：纯源码删除批对开发库零影响可以用一组固定计数器（worlds/accounts/record_first_nights/action_receipts/memory_snapshots/world_entities）快速证伪。
- **下一轮决策（T10-B15 候选）**：①drizzle/ archive 决策落档（tooling 批前置）；②local-only 契约迁移方案（script/inactive 文案新锚点）；③db/index/schema/d1-types + drizzle/tooling 退役批（前置①②完成后独立评审）；④POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；⑤propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）。

### T10-B15-A · Drizzle/D1 历史迁移 archive 决策（2026-08-21 派单 → 收口）

- **主题**：tooling 退役批前置决策①——drizzle/ archive 决策落档：保留原位、只写决策，两套迁移系统边界（D1/SQLite 历史 vs PostgreSQL 权威）实锤，未来整体 move 的前置写死。不移动不删除不改写。
- **commit 链**：99065ac（规范）→ 79abc33（archive review 围栏）→ 本次（收口）。无 git add -A。
- **设计思路**：「archive」定性与「物理移动」分离——三 SQL + meta + config + script 是完整 provenance 证据链，原位零风险（B11–B14 audit 链已证无活动消费方），拆散才毁链；决策落档 + 围栏实锚比急着移动更符合最小动作原则。围栏的反伪证据纪律：用存在性/关键头部/journal tags 实锚证明「内容未被改写」，不生成 hash 伪证据（hash 只能证明「等于我记录的值」，不能证明「这个值是对的」）；边界检测用签名特征交叉（PG 目录零 statement-breakpoint、drizzle SQL 零 PG 专有构造）而非路径名单。
- **验证证据**：archive review 3/3 exit 0；B11 audit 6/6、B13 audit 5/5、local-only 1/1、docs 2/2、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；未跑全量/GUI/PG；零 DB 写入、零文件改动，realm-dev.service 保持 active。
- **新发现**：①drizzle `_journal.json` 的 entries tags 与 SQL 文件名一一对应、version 7 dialect sqlite——journal+snapshot+SQL 三件套互证是比单文件存在性更强的 provenance 锚点，archive 类围栏优先找这种「自洽结构」；②PG migrations 0001–0023 与 drizzle 0000–0002 编号体系相邻易混（都有 0001/0002），边界断言必须双向前置（PG 目录零 sqlite 签名 + drizzle 零 PG 构造），单向前置防不住「复制错方向」；③statement-breakpoint 是 drizzle-kit sqlite 产物的稳定签名，可做为两套系统的判别特征——比 dialect 字段更贴近 SQL 文本本身。
- **下一轮决策（T10-B16 候选）**：①local-only 契约迁移方案（script/inactive 文案新锚点——tooling 退役批最后前置）；②db/index/schema/d1-types + drizzle/tooling 退役批（前置①完成后独立评审+确认）；③POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；④propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）。

### T10-B16-A · 迁移 local-only 契约锚点（2026-08-21 派单 → 收口）

- **主题**：tooling 退役批最后前置——generic local-only 边界与 D1 archive 契约解耦：local-only 只守活动 runtime 事实，legacy script/inactive 文案迁移为 archive review 的指定锚点。不是退役批，历史文件与 script/依赖零改动。
- **commit 链**：6c8af80（规范）→ 94444d2（解耦+锚点+B15 状态更新）→ 本次（收口）。无 git add -A。
- **设计思路**：契约解耦的本质是「测试文件的责任边界」而非断言数量增减——local-only 删掉的是三条 D1 历史链断言 + db/index.ts 读取（该文件是永久 throw 的 legacy 入口，通用边界为它背书本身就是错位），generic 断言（Sites/Worker/回环/启动路径）一条不少；archive review 把已有断言独立成命名用例并注明所有权——「指定锚点」的价值在于未来 tooling 退役批的锚点迁移面是显式枚举的（archive review 专门用例 + B11/B13 同族断言），不用全仓找谁还断言了 inactive 文案。B15 文档用追加标注节更新状态而非改写——历史决策文档的「追加不改写」纪律与 STATUS 同型。
- **验证证据**：local-only 1/1、archive review 4/4、B11+B13+docs 13/13、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；git diff 名单核对仅三个目标文件；零 DB 写入，realm-dev.service 保持 active；未跑全量/GUI/PG。
- **新发现**：①「契约迁移」要先画责任边界表再动手——本次解耦后每条契约恰好一个指定承担者（generic→local-only、历史链→archive review、退役面→B11/B13），同族重复是回归网不是耦合，跨族依赖才是；②db/index.ts 上的 cloudflare:workers 负断言是「看起来有用实则错位」的典型——它守的是一个永久 throw 的 legacy 文件，活动 runtime 的 DB 通路是 database/postgres，通用边界删掉它反而更诚实；③「指定锚点 + 所有权注释」比「哪里都断言一遍」更适合做迁移面管理——后者让退役批必须全仓考古。
- **下一轮决策（T10-B17 候选）**：①D1/Drizzle tooling 退役批（前置已全部核销：独立规范 + 独立确认；范围 db/index/schema/d1-types 删除、drizzle/ 整体 move、script/依赖移除、锚点迁移）；②POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；③propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）；④图谱自动刷新（轮询/SSE）评审。

### T10-B17-A · D1/Drizzle tooling 退役资格评审与执行方案（2026-08-21 派单 → 收口）

- **主题**：tooling 退役批的资格评审与执行方案落档——前置（B14 pair 删除、B15 archive 决策、B16 契约解耦）全部核销后，逐文件实锤退役资格，把未来执行批的删除/move 清单、锚点迁移顺序、回滚路径与用户确认闸门写死。只评审不执行。
- **commit 链**：790b253（规范）→ ec81161（review 围栏）→ 本次（收口）。无 git add -A。
- **设计思路**：资格评审的价值在「把执行批的决策点前移到评审批」——move 单元里 drizzle.config.ts 的处理（本体删除 + provenance 入 archive README，而非 move 一个指向已删 schema 的活配置）是本次评审才真正想清楚的决策；锚点迁移面经 B16 收窄后已可显式枚举（archive review 指定锚点 + B11/B13 同族断言，local-only 不动），执行批不需要再考古。围栏沿用双轨扫描纪律：resolve 轨判归属、specifier 形态轨兜 resolve 失败的悬挂引用——B14 已证明纯 resolution 会在文件消失后静默放行。
- **验证证据**：新 review 5/5 exit 0；五测试链 18/18 exit 0；typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；diff 名单核对仅新增测试文件；零 DB 写入，realm-dev.service 保持 active；未跑全量/GUI/PG。
- **新发现**：①「config 引用」与「import 引用」是两种边——db/schema.ts 最后的指向是 drizzle.config.ts 里的配置字符串，import 解析抓不到，资格审计要为配置字符串单独断言，否则退役后会留下指向不存在路径的 config（B15 §四.1 的禁令正是为此）；②api-core-wiring 的 T10-B4 路由围栏正则（record-store|story-record-repository|db/schema|drizzle）在退役后依然有效——负断言形态与文件存废无关，评审时区分「需要迁移的锚点」与「天然免疫的负断言」可以少做无用功；③执行方案的写法是「清单 + 闸门」而非「步骤」——A–F 清单把每个动作的边界、影响面、回滚写死，用户确认闸门逐条对应清单，执行批照单勾选即可，减少执行时再决策。
- **下一轮决策（T10-B18 候选）**：①D1/Drizzle tooling 退役执行批（前置：用户逐条确认 T10-B17-A §四闸门）；②POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；③propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）；④图谱自动刷新（轮询/SSE）评审。

### T10-B18-A · 执行 D1/Drizzle tooling 退役批（2026-08-21 派单（含用户明确授权）→ 收口）

- **主题**：B11–B17 评审链的执行批——db 三件套删除、drizzle 整体归档 docs/archive/d1-drizzle/、legacy tooling（script + drizzle-orm/drizzle-kit）移除、四测试 post-retirement 迁移。完成后 D1/Drizzle tooling 正式退役。
- **commit 链**：5637285（规范）→ 9c9a5c2（A+B）→ adc5ec1（C）→ f153b66（D）→ 本次（收口）。无 git add -A。
- **设计思路**：执行批几乎零临场决策——B17 已把清单/闸门/回滚写死，本批照单执行：停止条件先复核（B17 围栏 exit 0）再动手；git mv 被 git 识别为 100% rename 是「内容零改写」的机器证据，比人工 diff 复述更强；config provenance 入 archive README 时明确写「不要据此重建配置」，防止归档被误当活文档。中间 commit（A/B/C 后 D 前）旧锚点测试暂红在规范 §八预先声明为批次内预期——多 commit 批次的「每 commit 全绿」让位于「主题拆分清晰」，但最终态零 stale 红测。
- **验证证据**：六测试链 20/20、wiring 7/7（负断言天然免疫实锤）、core-runtime 7/7、typecheck/四文件 eslint/diff --check/npm install --package-lock-only 全 exit 0（原始退出码）；保留项 diff 名单（83e5485..HEAD 九类路径）为空；DB 计数 2/1/0/0/0/0 不变；realm-dev.service 保持 active；未跑 GUI/PG/全量。
- **新发现**：①编辑 package.json 长行脚本区是高危操作——本次误把 test:core 整行当目标删（old_string 含了相邻行），靠 git diff 为空才发现两次编辑相互抵消；教训：动 package.json 前后各看一次完整 git diff，不要只看编辑工具的成功回执；②npm uninstall 的级联清理比预期大（lock -1603 行，drizzle-kit 拖了一整套 esbuild 平台二进制依赖）——顶层依赖的真实体积要看 lock 级联不是看 dependencies 列表；③退役批的扫描面要随目录消失同步收缩（pair-deletion-review 的 SCAN_DIRS 含 db/ 在 db/ 删除后直接 scandir 崩溃）——「目录不存在」本身要成为 collect 的合法状态或从扫描面移除，这是 B14「resolve-null 盲区」的姊妹坑。
- **下一轮决策（T10-B19 候选）**：①POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权段文档化；②propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）；③图谱自动刷新（轮询/SSE）评审；④退役后全量回归窗口（typecheck+test:core+test:contracts+build，确认退役链对全量套件零影响）。

### T10-B19-A · POSTGRESQL-RUNTIME-CONTRACT 增补 T 系列授权 provenance（2026-08-21 派单 → 收口）

- **主题**：T10 拆批候选项①——把 T 系列体验批次的 runtime grant 来源集中进架构契约文档（此前分散在各批 STATUS/规范），追加不改写，静态契约守住。只文档/契约，不改 SQL。
- **commit 链**：dccf9ad（规范）→ 1b9f9f4（章节+围栏）→ 本次（收口）。无 git add -A。
- **设计思路**：provenance 表的全部 grant surface 回读真实 SQL 逐条抄录（0014 的 11 列列级 INSERT、0023 的 11 表 INSERT+两列级 UPDATE 等），批次映射从 STATUS 原文回查（0013=DEPLOY-AUTH 登录即加入、0014=设定结晶、0015=i18n、0016=酒馆导入、0017/0018=批次 S、0019=T1、0020=T7、0021=T8、0022=T10-B1、0023=T10-B7）——不按记忆写。章节用英文与架构文档语体一致（开发规范用中文、架构契约为英文的既有分工）。围栏的反接线断言做成逐行形态：章节里任何提及 propagation/semantic-conflict 的行必须同含 contract-only-deferred，比全文关键词共现更贴语义。
- **验证证据**：schema 契约 24/24 exit 0（新增 1 项 provenance 断言）；docs 2/2、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；diff 名单仅四文件、migration 零改动；零 DB 写入，realm-dev.service 保持 active；未跑 GUI/PG/全量。
- **新发现**：①全库 GRANT ALL 扫描发现唯一实例在 0006→realm_control——「T 系列无全特权授权」的结论必须限定范围说，否则与 0006 矛盾；授权审计的结论措辞要自带作用域；②0018 是零 grant 的纯 FK 语义修复迁移——provenance 表用「(no grants)」如实记录比硬凑 grant surface 更诚实，迁移的「授权面」可以是空的；③架构契约文档是英文而批次规范是中文——provenance 表里的批次锚点（DEPLOY-AUTH、T10-B7 等）充当双语言文档间的稳定引用键，这也是围栏直接拿它们做锚点的原因。
- **下一轮决策（T10-B20 候选）**：①退役后全量回归窗口（typecheck+test:core+test:contracts+build，确认 B11–B19 链对全量套件零影响）；②propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二）；③图谱自动刷新（轮询/SSE）评审。

### T10-B20-A · T10-B11–B19 退役链后的最终回归窗口（2026-08-21 派单 → 收口）

- **主题**：退役链闭环后的纯验证批——typecheck/test:core/test:contracts（含 PG runtime）/build 四条命令真实 exit 0，证明 B11–B19（D1 评审→类型提取→pair 删除→archive 决策→契约解耦→tooling 退役→provenance 落档）对既有套件零影响。只验证不修代码。
- **commit 链**：本次（收口记录）。验证过程零文件改动。
- **验证证据**：typecheck exit 0；test:core 177/177 exit 0；test:contracts exit 0（静态 46 + application 33 + frontend 16 + PG runtime 54，全链路含 realm_runtime 受限角色实跑）；build exit 0（vinext 五阶段）；git porcelain clean、fd963e6..HEAD 为空、DB 2/1/0/0/0/0、realm-dev active。
- **新发现**：①T10-B7 时期留下 6 个测试临时库（realm_t10b7_pre/post_* × 3 对）——t.after 强制 DROP 只覆盖正常退出的运行，被杀进程/异常会话的运行会漏清；「回归窗口收口」把临时对象盘点列为固定动作后此类残留才浮出水面，命名约定（realm_t<批次>_<label>_<uuid>）是让残留可安全识别清理的关键设计；②回归窗口的命令拆分（不跑 npm test 而拆四条）让「rendered-html/GUI 不在本轮范围」成为显式决策而非隐含；③退役链后的 PG runtime 全绿顺带证明一件事：B18 删除 db/ 与 drizzle 依赖后，27 个 PG 集成文件的迁移链（0001–0023 纯净 PG）与 drizzle 零耦合——两套迁移系统的边界在运行态也成立，不只是静态扫描成立。
- **下一轮决策（T10-B21 候选）**：①propagation/semantic-conflict 真实接线评审（前置条件见 T10-B9-A 规范 §二——enqueue 入口/worker 启动方/模型预算与降级，未满足前任何接线属假实现）；②图谱自动刷新（轮询/SSE）评审；③T10 系列整体复盘与 T11 主题规划。

### T10-B21-A · propagation/semantic-conflict 真实接线资格复核（2026-08-21 派单 → 收口）

- **主题**：T10-B9-A §二接线前置的当前 HEAD 复核——三前置逐项 FAIL（缺失），正式维持 contract-only-deferred；禁止假接线，本批只复核不接线。
- **commit 链**：f4298c6（评审）→ b332a80（围栏扩展）→ 本次（收口）。无 git add -A。
- **设计思路**：复核纪律是「当前 HEAD 重扫，不引用旧批次证据」——B9 的零调用结论在 B11–B20 大批量文件变动后必须重新建立才有效。复核时把「repository 层事实」单独如实记录（public.ts re-export 是导出面不是调用面、evidence store 工厂零生产实例化），避免两种误判：把 repository 存在当成已接线，或把 repository 存在当成必须删除的死代码。三前置的 FAIL 证据都落到具体符号级反例（路由的 enqueue 是 SSE controller.enqueue、唯一 setInterval 是 SSE heartbeat、assessor 签名无预算参数）——缺失证明也要排除同形干扰项才算实锤。围栏扩展延续「负断言锚定缺失状态」形态：未来接线改规范后同步翻转，不留无锚点窗口。
- **验证证据**：围栏 4/4 exit 0；m5 两套件 12/12 exit 0（行为零回退）；docs 2/2、typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；未跑 PG/GUI/全量；零 DB 写入，realm-dev.service 保持 active。
- **新发现**：①「资格复核」的证据要区分三层——符号零调用（app/application）、导出面存在（public.ts re-export）、实例化零发生（工厂无生产调用方）；三层混着说会把「导出面」误判成「接线面」；②缺失证明需要排除同形干扰项才算完整：grep 命中 enqueue/setInterval 时不逐条甄别语义（SSE 原语 vs propagation 语义），缺失结论就是虚的；③B9 围栏的三测试与本批第四测试是两种锚定——「不得接线」与「前置缺失」，前者防悄悄接线，后者防「以前置已满足为由跳规范」，两层都要留。
- **下一轮决策（T10-B22 候选）**：①图谱自动刷新（轮询/SSE）评审；②T10 系列整体复盘与 T11 主题规划；③propagation/semantic-conflict 接线规范（仅当产品决策要求启用传播/语义评估且三前置能一次性满足时立项，否则继续 deferred）。

### T10-B22-A · 图谱自动刷新（轮询/SSE）资格评审（2026-08-21 派单 → 收口）

- **主题**：T10-B10-A §八范围外项的资格评审——失效信号面重扫结论：无 graph-specific invalidation contract，维持手动刷新，自动刷新 deferred。只评审不实现。
- **commit 链**：a201768（评审）→ eccd4f1（围栏）→ 本次（收口）。无 git add -A。
- **设计思路**：自动刷新评审的核心是「失效信号契约是否存在」而非「前端能不能起定时器」——七项候选信号逐项实查（读 API 游标/etag、canon 游标、行级 updated_at、graph SSE route、客户端 EventSource、LISTEN/NOTIFY、outbox 关联）全部落空，结论才有分量。polling 与 SSE 分案评估把缺口定性分清：轮询缺的是产品决策（预算/焦点/退避），SSE 缺的是契约级事件源（写路径零 NOTIFY/outbox）——后者不是包一层 EventSource 能补的。B10 的序号竞态机制被确认为可复用资产（自动响应 vs POST 回读的交叠已覆盖），写进未来前置而非重复设计。
- **验证证据**：围栏 5/5 exit 0；wiring+docs 9/9 exit 0；typecheck/受影响 eslint/git diff --check 全 exit 0（原始退出码）；diff 名单仅三文件；零 DB 写入，realm-dev.service 保持 active；未跑 GUI/PG/全量。
- **新发现**：①「deferred 语义围栏」要小心自指——评审文档里的禁止句（「不得声称自动刷新已具备」）会被自己的负断言命中，禁止性语境必须显式剔除（继 B12 注释幻影、B19 逐行断言之后第三次同型教训：针对文本的断言先想清楚合法语境）；②path 相对化用 slice(projectRoot.length+1) 这类手算偏移是脆的——projectRoot 末尾带不带斜杠取决于构造方式，relative() 永远比手算 slice 安全；③「SSE route 清单恰为两条」是比「无 graph SSE」更强的锚定——枚举式断言能抓住「加了第三条推送通道但没走评审」的漂移，负向关键词匹配抓不住。
- **下一轮决策（T10-B23 候选）**：①T10 系列整体复盘与 T11 主题规划（B10–B22 全部收口，候选清单已耗尽：D1 退役链闭环、propagation/semantic-conflict deferred、图谱自动刷新 deferred）；②图谱自动刷新实现批（仅当产品决策选定轮询预算或 SSE 契约后立项）；③propagation/semantic-conflict 接线规范（同 B21 结论，产品决策驱动）。

### T10-B23-A · T10 整体复盘与 T11 主题规划（2026-08-21 派单 → 收口）

- **主题**：T10 拆批末端的事实型复盘——B1–B22 完成矩阵（真实 commit 链回读）、结果三分类、T11 三候选闸门、工程结论汇编。只文档不开工。
- **commit 链**：6f9eab1（复盘+索引）→ 本次（收口）。无 git add -A。
- **设计思路**：复盘的可信度来自「从 git log/STATUS 原文回读，不从记忆写」——22 批的主题/结论/SHA/证据/改动面全部对照真实历史，早期批次（B1–B8 非本会话完成）的 SHA 用 git cat-file 抽查 25 个全部存在后才定稿。结果三分类是这个系列反复纪律的总纲：「已完成可声称」「评审完成但 deferred」「不能从证据推出」——第三类专门防复盘最常见的夸大（全功能完成/已接线/全覆盖）。T11 闸门沿用 B17 验证有效的「清单+闸门」写法：每个候选含前置决策/允许面/禁止面/最小验收/阻塞条件，并显式写「当前不应自动开工」。
- **验证证据**：docs 2/2、受影响 eslint、git diff --check 全 exit 0（原始退出码）；diff 名单仅两文件；零 DB 写入，realm-dev.service 保持 active；未跑 PG/GUI/全量。
- **新发现**：①复盘文档的 SHA 引用是隐性事实声明——矩阵里几十个短 SHA 任何一个写错都是虚假信息，cat-file 批量抽查应成为复盘类文档的固定验收动作；②「候选清单耗尽」本身是重要的项目状态信号——它意味着后续批次从「执行既定候选」转为「需要产品决策立项」，复盘批的职能就是把这个转折点显式化而不是硬造候选；③T10 的批次形态演化值得记录：功能修复（B1–B4）→ 定性围栏（B5/B9）→ 权限加固（B6–B8）→ 体验闭环（B10）→ 退役链（B11–B18）→ 文档/回归/评审（B19–B22）——后期批次越来越「非功能化」，是候选自然耗尽的内在原因，T11 需要产品输入才能重新形成功能批。
- **下一轮决策**：T10 候选清单已耗尽。T11 开工需产品决策：①T11-A 图谱 invalidation contract（先选 polling 或 SSE，闸门见 T10-B23-A §三）；②T11-B propagation/semantic-conflict enablement（三前置一次性满足才立项）；③T11-C 维护纪律保持（非业务功能）。无产品决策前不自动开工。

### T11-A2 · 知识图谱 graph-specific SSE 自动刷新（2026-08-21 派单（含用户 SSE 方案决策）→ 收口）

- **主题**：T10-B22-A 评审前置的产品决策落地——SSE 方案真实实现：持久化失效账本（0024）+ LISTEN/NOTIFY 唤醒 + world-knowledge/events SSE + 前端 EventSource 自动回读。T10 系列之后的首个功能实现批。
- **commit 链**：d82d628（规范）→ 1276489（迁移+事件源）→ 9ade36e（SSE 路由+PG 测试）→ a8230ee（前端）→ 424c338（围栏翻转+K8）→ 本次（收口）。无 git add -A。
- **设计思路**：架构核心是把「失效信号」做成一等持久化事实——失效记录与业务写同一事务（PG 语义保证 NOTIFY 提交才送达、回滚同灭），NOTIFY 只当唤醒提示，消费方永远按 cursor 重查账本；这样断连/丢通知/初次连接竞态都退化为同一个重放查询，没有第二种恢复路径。写路径打点在 repository 层而非路由层——晶化入图谱（W8）这类后台管线与路由写入共用方法，零漏接；tavern-import 直写是唯一补点。SSE 连接用专用 pg.Client 而非共享池（LISTEN 长连接占池会饿死请求）。前端只把事件当失效信号触发既有 load()——T10-B10 的序号竞态/卸载防护/双端点整体成败全部自然复用，防抖合并防事件风暴。
- **验证证据**：focused PG 13/13、静态链 36/36、单元 25/25、K1–K8 Chromium 8/8、K8 WebKit 1/1、typecheck/eslint/diff --check 全 exit 0（原始退出码）；清理后 DB 基线 2/1/0/0/0/0、graph_invalidation_events=0、realm_t%=0、守卫触发器恢复、realm-dev active；未跑全量/render。
- **新发现**：①既有 PG 测试硬编码迁移清单是新增迁移的隐性耦合点——六个测试文件各持一份 MIGRATIONS 列表且有的刻意停在子集（library-runtime-grants 的 pre/post 靠 slice(0,-1) 语义），加迁移必须逐文件判断「是否走过打点写路径」而非一律追加；②FORCE RLS 对 owner 也生效——清理脚本对 append-only+RLS 账本表必须显式 DISABLE TRIGGER 才能删，清理脚本随新表同步更新应列入建表批的交付清单；③savepoint 组合（withWorkspaceTransaction 传 PoolClient）是测试「回滚不产生事件」的干净手段——外层 COMMIT 空事务，账本计数不变即证原子性；④初次连接竞态的标准解法（先 LISTEN 再补发、通知驱动重查去重）用 Last-Event-ID=最大 cursor 就能在测试里隔离活推送路径，与重放路径分别取证。
- **下一轮决策（T11-B/C 候选）**：①T11-B propagation/semantic-conflict enablement（三前置仍未满足，需产品决策）；②T11-C 维护纪律保持；③图谱 SSE 的观察项：多面板/多世界的连接数上限与心跳节奏在真实使用中的表现，必要时评审连接预算。

### T11-B · propagation / semantic-conflict 前置设计（2026-08-21 22:51 派单 → 前置设计收口）

- **主题**：在 T11-A2 graph-specific SSE 收口后，按三项资格闸门为 T11-B 定稿——Canon 合并触发 Campaign、独立 Worker 进程拓扑、semantic-conflict 模型预算/超时/降级/evidence 边界。此批只设计，不接生产调用。
- **commit 链**：440124f（规范+docs 索引）→ 本次（STATUS/迭代日志收口）。无 git add -A。
- **设计决策**：唯一自动传播动作是成功的 `/api/canon` `decision=merge`；同事务写 promoted Claims、CanonRevision、Campaign、root Packet、pending job，确定性身份保证重放幂等；拒绝/延期/普通图谱写入/场景晶化不触发。首批只允许可证明为 public 的 Canon，restricted/secret 等可见性字段缺失时 fail-closed。Worker 独立于 realm-dev.service，以单例锁、作用域安全领取、stale 恢复、有限重试和 immutable job input 运转；不把 `world_relations` 冒充通信拓扑，未来必须引入显式持久化 PropagationTopologyProvider。语义复审为用户显式请求的独立 route，既有 legacy/causal 确定性 preview 不变；none/hard 不调模型，high-risk 最多一次、8 秒真实可取消超时，失败回落确定性结论，evidence 不进入正史。
- **当前 HEAD 审计结论**：`propagation_jobs.enqueue()` 尚未原子创建 Campaign；`claimNext()` 只按 workspace 领取且返回 job 不含真实 world/worldline；拓扑没有持久化来源；Worker 无生产启动方；semantic assessor 无专属预算/取消边界，evidence 工厂无生产实例化。这些列为未来实现批硬门槛，本批未修改活动代码。
- **验证证据**：documentation-layout 2/2 exit 0；`npm run lint` exit 0；`git diff --check` exit 0；零数据库写入、零迁移变化；realm-dev.service active。实现批尚未开始，propagation/semantic-conflict 仍 contract-only-deferred。
- **下一轮决策**：进入 T11-B 实现前，必须以本规范为任务书，先落拓扑/队列/evidence schema 与 PG contract，再做 Canon 原子入队、独立 Worker、semantic review；若任一作用域、可见性或事务证据缺失，停在部分实现，不声称已接线。

### T11-B · propagation + semantic-conflict enablement 实现批（2026-08-21 派单 → 2026-08-22 收口）

- **主题**：T11-B 前置设计的真实实现——Canon merge 原子入队、显式 PG 拓扑、独立 Worker、semantic review 路由；M5 三模块撤销 contract-only-deferred。T10 之后第二个功能实现批，也是第一个「deferred → 接线」翻转批。
- **commit 链**：25448e0（实现规范）→ df187a6（拓扑 schema 0025）→ cf91919（canon 原子入队）→ bfca386（独立 Worker）→ 66bde82（semantic review）→ b1ee4ed（测试签名修正）→ bbb97d6（围栏翻转）→ 本次（收口）。无 git add -A。
- **设计思路**：三个绑定决策值得记录——①public 证明的形态：没有 visibility 字段时，「merge 请求里的显式 propagate:"public" attest」是唯一不猜语义的证明，缺省 fail-closed 只留正史；②同事务绑定的接口形态：Core 接口用 opaque `client: unknown` hook，PG 实现在 mergeProposal 事务内回调——Core 不引入 pg 类型，原子性不打折；③失效/拓扑/因果三套「图谱相邻但语义不同」的表各归其主：graph_invalidation_events（SSE 失效）、propagation_nodes/routes（通信拓扑）、world_relations（知识关系）——本批全程没有互相冒充。Worker 的 RLS 约束（realm_runtime 不读 workspaces 表）逼出了「显式 workspace 清单」设计，比全库枚举伪实现诚实。
- **验证证据**：PG focused 19/19、单元 54/54、静态链 56/56、typecheck/eslint/diff --check 全 exit 0（原始退出码）；迁移台账 0025 实查；临时库 0；开发库基线 2/1/0/0/0/0 + 新表计数全录；realm-dev active；Worker 模板未安装。
- **新发现**：①引擎默认 packet id（auto_N 逐运行重置）与 Exposure id 派生（node+packet）在多 Campaign 下同撞 workspace 级主键——单 Campaign 单测从未暴露，多世界隔离测试一跑即现；教训：确定性重放的 id 命名空间必须含业务键（Campaign），「逐字节重放一致」与「多实例共存」是两个独立性质；②node:test 的 test() 选项是单对象签名，分开写 skip/timeout 两个对象只在 typecheck 才炸（strip-types 运行时不拦）——PG 测试也要过 typecheck 再算完；③FORCE RLS 下「worker 枚举所有 workspace」是伪需求——队列方法的 withWorkspaceTransaction 已逐 workspace 设上下文，缺的不是 SQL 技巧而是显式清单；④语义复审的 evidence 写失败路径在 assessor 里会触发二次 fallback 记录尝试——第一次写失败大概率第二次也失败并正确抛出，但「模型成功却因落库失败报 unavailable」是刻意选择：证据先于声称。
- **下一轮决策（T11-C 候选）**：①Worker 服务安装验收后的运行观察（连接预算/心跳节奏）；②restricted/secret 传播批次（需 visibility 字段与受众裁决先行）；③T11-C 维护纪律保持；④语义复审的产品入口（前端什么时候暴露「请求语义复审」动作）需产品决策。

### T11-C · operational closure + blocked follow-up preflight（2026-08-22 派单 → 收口）

- **主题**：T11-B 收口后的运维闭环——Worker 用户级真实安装/启用/空队列观察、realm_t% 盘点维护工具与清理闸门、restricted/secret 与 semantic review UI 前置规范（保持 deferred）。
- **commit 链**：4588d8d（规范+索引）→ c46ecb5（systemd 模板最小修正）→ 5c98d87（盘点工具+测试）→ 本次（收口）。无 git add -A。
- **设计思路**：运维批的可信度同样来自「先核环境契约再动手」——安装前实测 user manager PATH（mise shims 解析 node v25.9.0）与 realm-dev 的绝对路径运行时（hermes node v22.23.2），发现模板 /usr/bin/env node 会选错主版本，按任务书「最小模板修正单独提交」落一行修复而非绕过。盘点工具的核心取舍是把闸门判定（planDropTargets）做成不触库的纯函数：危险目标拒绝（realm_dev/系统库/未知/裸前缀）可以单元测试实锤，真实 DROP 路径本批未获授权一行未跑。「只盘点不盲删」落到工具默认 report + 清理必须命中盘点清单两层。deferred 边界的写法沿用「已有/缺失/fail-closed/资格闸门」四段——restricted/secret 的关键事实是引擎 CLEARANCE_RANK 过滤与 DB 枚举早已存在，缺的是 Campaign securityClass 来源（恒 public）与 visibility schema，写清楚后「没实现」与「不能实现」一目了然。
- **验证证据**：focused 单元+静态 48/48 exit 0；PG focused（propagation-worker）1/1 exit 0；typecheck/受影响 eslint/diff --check 全 exit 0（原始退出码）；盘点工具实测两次 exit 0（临时库 0、基线 2/1/0×8+拓扑 3/2+传播四表 0、台账 0025）；Worker active+enabled，journal 仅 Started+advisory lock acquired，约 20 分钟零 poll error；realm-dev 全程 active。
- **新发现**：①systemd user manager 的 PATH 与交互 shell 不同源（mise shims vs hermes 绝对路径）——仓库模板的 env 解析必须按「user manager 实际环境」验证，不能按开发 shell 推断；②空队列观察的活性信号是 idle backoff 收敛（15s 上限）而非显式心跳——journal 静默即健康，「无日志」在这个设计里是正向证据而非缺证据；③PG 测试拆库纪律在 Worker 常驻后仍成立：测试临时库的 advisory lock 按库命名空间隔离，常驻服务持 realm_dev 的锁不妨碍 realm_t11bwk 的测试锁——「Worker 常驻」与「PG 测试可跑」不互斥。
- **下一轮决策**：①restricted/secret 传播批（闸门：visibility schema 决策/受众裁决模型/Exposure 读取授权/产品标记决策，见 T11-C §七）；②semantic review 前端入口批（闸门：入口位置/预算呈现/evidence 展示/i18n key，见 T11-C §八）；③Worker 有真实传播任务后的运行观察（连接预算/退避节奏实绩）；④realm_t% 清理授权后走工具 --drop 精确名称路径。

### T11-C · 独立审查更正与最终回归（2026-08-22 09:46 +08:00）

- **主题**：T11-C 开发批结束后的独立证据审查。协作器的完成回执只作为线索，最终结论由当前 HEAD 的源码、原始退出码、运行态和数据库盘点交叉确认。
- **审查修复**：发现维护工具精确 `--name` 路径允许裸 `realm_t`，随后发现 `--prefix` 路径对命中项未复用命名约束；按红→绿顺序补回归测试并拆成两笔提交：`7e13335`、`e039a87`。修复仅收紧纯函数闸门，未触发 DROP、未改迁移、未改产品语义。
- **验证证据**：维护工具 11/11、T11-C 静态链 49/49、T11-B 相邻 PG 13/13、全量 `npm test`（177/177 + 46/46 + 33/33 + 16/16 + 54/54 + render 3/3）及 Build complete、typecheck、全量 lint、受影响 eslint、git diff --check 全部原始 exit 0；最终 `realm_t%` 临时库 0，开发库基线与拓扑/传播零计数保持，台账 0025；两个 user service active+enabled，Worker journal 无 poll error。
- **新发现**：①安全选择器的 exact 与 prefix 分支必须共享同一个命名谓词，单测只覆盖「裸前缀」会漏掉「裸精确名」和「畸形前缀命中」；②`*` 形式的后缀正则会把裸命名约定误纳入危险动作，命名协议应要求真实批次段并拒绝空段/悬挂下划线；③协作器声称完成不能替代独立验收，尤其是有副作用的服务安装、数据库清理与提交边界，必须用源码、服务、事件流、数据库和原始退出码交叉核对。
- **状态边界**：未执行任何 `--drop`；Worker 仍只验证空队列常驻，不声称真实传播吞吐；restricted/secret 与 semantic review UI 继续 deferred。

### T11-D · semantic review 操作面实现批（2026-08-22 派单 → 收口）

- **主题**：按 T11-D 规范把既有 `POST /api/worldline/conflict/semantic` 变成用户显式请求、只读证据展示的操作面——图谱 Claim 列表内「请求语义复审」入口、服务端 head 游标 context route、三语 i18n、白名单结果展示。restricted/secret 与自动合并继续 deferred。
- **commit 链**：1a88ece（context route+PG 契约测试，先红后绿）→ 925a069（SemanticReviewPanel+semantic-review-types 纯契约层+i18n 26 key+KnowledgeGraphPanel/realm-client 接线）→ 本次（UI/静态契约测试 12 项+test:frontend-contracts 登记+docs 索引+STATUS/迭代日志收口）。无 git add -A。
- **设计思路**：游标防注入的关键是把 effectiveCursor 的来源从「表单字段」改成「context route 一次性只读下发」——表单打开只读 GET（零模型调用），POST 负载里的 existingFuture/effectiveCursor 原样回带该服务端值，客户端无任何时间输入控件。结果展示走白名单 normalization（normalizeSemanticReviewOutcome 只产出 classification/recommendation/rationale/source 四键），prompt/inputDigest/requestId 在类型层就进不了 JSX；「无 merge 按钮」由静态契约锚定（面板源码不得引用 canon/merge 路由）。结果 state 与图谱快照 state 分离，SSE 的 load() 不触碰 reviewClaimId，Claim 消失时经派生查找自动闭合表单。
- **验证证据**：PG focused 5/5（context 新增 + semantic-route + conflict-route 回归，exit 0）；单元+前端契约 31/31（exit 0）；documentation-layout 2/2（exit 0）；typecheck/全量 lint/build/diff --check 全 exit 0（原始退出码）；realm_t% 临时库=0，realm_dev semantic_conflict_evaluations=0。未跑全量 npm test/GUI，不以中途测试充当最终验收。
- **状态边界**：semantic recommendation 不写入 Canon/Worldline；无后台队列/自动触发/轮询/模型设置改动；restricted/secret 传播继续 deferred；本批为开发侧完成记录，最终独立验收留待后续审查。

### T11-D · 独立审查与收口（2026-08-22 11:19 +08:00）

- **审查修复**：`worldlines.head_tick/head_ordinal` 为 PostgreSQL `bigint`，直接 `Number()` 会在 JS safe integer 边界外静默舍入；补充红灯回归后由 `2be6a3a` fail-closed，超界游标返回脱敏 500。
- **验证证据**：T11-D PG focused 5/5；semantic review/UI/i18n/frontend 契约 26/26；全量 `npm test` 177/177、46/46、33/33、28/28、54/54、render 3/3，Build complete；全量 lint 与 diff check 原始 exit 0；最终临时库 0、开发库基线与传播/semantic evidence 零增长。
- **运行边界**：realm-dev 与 propagation-worker 均 active/running/enabled，Worker journal 无 poll error；没有真实传播任务，不声称传播吞吐。Playwright headless 实际加载页面 HTTP 200、标题正确、pageerror=0，但停在访问令牌墙；未猜 token 或绕过登录，因此不声称完成 Claim 入口手动点击验收。
- **收口**：当前 HEAD=`2be6a3a`，工作树 clean，本地未 push；无新增迁移、无 DROP、无新部署。T11-D 代码与契约验收完成；restricted/secret、自动合并与历史游标编辑继续 deferred。

### T11-E · public propagation 真实运行验收 harness（2026-08-22 派单 → 开发侧收口）

- **主题**：把 T11-B Worker 的验收从「runOnce 单测」升级为「真实独立进程入口」验收——spawn 当前仓库 scripts/propagation-worker.mjs 在隔离临时库 realm_t11e_rt_<uuid> 消费一条合法 public Job，证明 advisory lock → 领取 → Campaign/Packet/Exposure 持久化 → job done → SIGTERM 干净 exit 0 的完整进程链。
- **commit 链**：029e7bf（harness+focused script）→ 本次（docs/STATUS 收口）。无 git add -A。
- **设计思路**：完成信号一律轮询事实而非固定 sleep——临时库 Job status、campaigns/packets/exposures 计数与 stdout 状态行三路交叉；子进程 env 最小化注入（临时库 runtime URL + 显式 workspace 清单 + PATH/HOME），结构上不存在「继承 token 再删除」的窗口；t.after 先 SIGKILL 兜底再 DROP DATABASE ... WITH (FORCE)，失败/超时路径同样零残留。临时库命名空间使已安装 systemd Worker（持 realm_dev 的 advisory lock）与本 harness 天然无争用。
- **验证证据**：harness 1/1 exit 0（子进程 stdout：advisory lock acquired / job done for ws_demo / exit code=0 signal=null）；既有 postgres-propagation-worker 1/1、documentation-layout 2/2、typecheck、受影响 eslint、git diff --check 全 exit 0（原始退出码）；realm_t%=0、realm_dev 传播四表与 semantic evidence 全 0、无孤儿子进程、两个 user service active。未跑全量 npm test/GUI，不以中途结果充当最终验收。
- **新发现**：①propagation_jobs 的完成列是 `status` 而非规范措辞的 `state`——harness 第一跑即被 PG 42703 红灯纠正，规范措辞与真实 schema 的偏差只有真实执行才暴露；②Worker 完成日志是 `job done for ws_demo`（outcome 而非 job id），规范 §2.4 的 `job <id> done` 措辞与实现不符，因不改生产循环，Job 身份改由临时库唯一 Job 的 DB 事实闭环；③vinext 的类型增强把 NodeJS.ProcessEnv.NODE_ENV 标为必填，spawn 最小化 env 字典需经 unknown 断言——这不是本项目代码的问题而是依赖类型与 Node 真实运行时的偏差；④strip-types 运行时不拦 TS 窄化错误（闭包内赋值的 let 被 CFA 窄化为 never），PG 进程类测试必须先过 typecheck 才算数（T11-B 已记录同类教训，再次应验）。
- **状态边界**：本批为开发侧完成记录，最终独立验收待 Iris 审查；restricted/secret 传播继续 deferred；未新增迁移、未改 Worker 生产循环、未动 systemd unit 与在运行服务、未 push。

### T11-E · 独立审查与收口（2026-08-22 11:48 +08:00）

- **审查修正**：真实 harness 执行确认完成列为 `propagation_jobs.status`，日志为 `job done for ws_demo`；规范已由 `6400e20` 修正，生产 Worker 未改。
- **运行证据**：真实 `scripts/propagation-worker.mjs` harness 连续两次 1/1；每次均完成 advisory lock、public Job 消费、campaign=1、packets=3、exposures=3、status=done，并 SIGTERM exit 0。
- **回归证据**：既有 Worker PG 1/1；全量 npm test 177/177、46/46、33/33、28/28、54/54、render 3/3，Build complete；全量 lint、diff check 原始 exit 0。
- **清理/服务**：`realm_t%`=0，realm_dev 传播四表与 semantic evidence=0；无孤儿子进程；realm-dev 与 propagation-worker 均 active/running/enabled，journal 无 poll error。
- **收口**：本批只验证 public propagation，未写入 realm_dev、未新增迁移、未改生产 Worker/systemd、未 push；restricted/secret 继续 deferred。独立核对时 HEAD=`6400e20`、工作树 clean，本地未 push。T11-E 验收完成。

### T11-F · restricted / secret 资格闸门决策（2026-08-22 12:01 +08:00）

- **核心决策**：安全分类挂在不可变 CanonRevision，同 Revision 禁止 Claim 混合分类；受众主体用 worldline-scoped `character_continuity`，不把 account、principal 或短命 character instance 当长期授权键。
- **授权结构**：后续分离 CanonRevision audience snapshot、propagation node→continuity 映射和 node clearance 三种职责；owner 只在控制面审阅 API 例外，普通 Character Projection 仍按 continuity audience 过滤。
- **secret 边界**：首阶段只允许 private_letter；official_bulletin/market_rumor、空 audience、未知 continuity、跨 scope 和未授权读取全部 fail-closed。
- **实现边界**：T11-F 只写决策规范，不新增 migration、不改变 public propagation；T11-G 才实现 0026/0027、merge 资格校验、immutable Job input 和 Exposure 读取授权。规范已登记 docs README，restricted/secret 仍 deferred。

### T11-G · restricted/secret qualification + 授权读取实现批（2026-08-22 派单 → 收口）

- **主题**：T11-F 冻结契约的实现——Revision 级安全分类 + immutable audience 快照 + fail-closed 资格闸门 + 服务端 Exposure 授权读取。T11 系列第三个功能批。
- **commit 链**：af3390e（schema 0026/0027）→ 8214dcd（domain/API/read-path+回归）→ 本次（收口）。无 git add -A。
- **设计思路**：资格校验放在 enqueue 端口的事务内（而非路由或服务前置）——audience 快照必须与 Revision 同一事务，deferred constraint trigger 做提交时双保险（应用校验绕过即回滚）；「unsupported propagate 值」「空晋升 non-public」「缺 audience」全部显式 4xx，没有任何路径把 non-public 请求静默当成 public 或「只 merge」。读取侧把 owner 拆成两个视图：默认 Character 视角按 audience join 过滤（owner 身份也不特权），只有显式 view=control 才给控制面全集且响应带 controlPlane 标记——「审阅可见 ≠ 角色已知」在 API 形态上分开。读取授权键是 participants→instances→continuities 的控制链，不是 display name/node key。
- **验证证据**：PG 13/13（含 T11-E harness 回归）、单元 21/21、npm test 全量 exit 0、npm run lint 全量 exit 0、diff --check exit 0（原始退出码）；台账 0026/0027 实查；临时库 0；开发库基线与新表计数全录；两服务同时刻 active。
- **新发现**：①DEFERRABLE INITIALLY DEFERRED constraint trigger 的测试夹具必须同事务写 Revision+audience——autocommit 下逐条 INSERT 会在每条语句的隐式提交处被触发器拒绝（夹具一度写成分条 pool.query 全挂）；②canon_proposals 的 decision_shape_check（merged 必须带 decided_by/decided_at）与 canon_revisions 无 created_at 列（叫 committed_at）这类既有 CHECK/列名细节，手写 fixture SQL 时必须先读表定义，凭印象写会连环违约；③资格失败回滚的端到端证据链是「409 + 提案仍 pending + 传播三表计数不变」三元组，缺一不可（只断言状态码会漏掉半提交）。
- **下一轮决策（T11-H 候选）**：①non-public 传播的产品化入口（UI/受众管理面，需产品决策）；②secret 非 private_letter 渠道扩展（需新资格模型）；③T11-C 维护纪律保持；④Worker 运行观察（非 public Campaign 的真实消费节奏）。

### T11-G · Iris 独立审查收口（2026-08-22 13:14 +08:00）

- **审查方式**：不采信协作器“全绿”回执；重新读取 migration、domain、Worker 和 Exposure route，补失败回归后再跑全量。正式 `npm test` 已纳入 qualification、Exposure、Worker PG 与 T11-E real-process harness，避免“单独跑过但全量脚本没覆盖”的假绿。
- **四个修正**：①`record.worldline_id` 现在是 scope 解析的强关联；②non-public read 同时要求 node audience mapping、`character_instances.status='present'`、`character_continuities.status='active'`；③secret 的 root packet、route channel 和 Worker 防线统一只接受 `private_letter`；④Job 冻结 audience ID 集合，Worker 重算并校验 digest，API/domain 对混合非法 audience 拒绝而非静默过滤。
- **真实结果**：PG 阶段 58/58（含 T11-G qualification、Exposure route、Worker PG、T11-E harness）；core 178/178、contracts 46/46、application 33/33、frontend 28/28、render 3/3；Build complete；typecheck、lint、diff check 均 exit 0。T11-E 日志仍以真实子进程事实闭环 advisory lock、`job done for ws_demo`、exit 0。
- **运行态**：最终 inventory exit 0，`realm_t%`=0，`realm_dev` 传播运行表与 semantic evidence=0，latest migration=0027；两张 audience 表 FORCE RLS 与 append-only guard 启用；两个 user service active/running/enabled，无孤儿 Worker、无近期 `poll error`。
- **边界**：未做真实 GUI 点击；未向 realm_dev 注入 non-public Job；未扩展 secret 非 private_letter 渠道、UI 受众管理、生产吞吐/多副本/故障/容量验证；未 push。T11-G 通过独立审查，但 restricted/secret 仍不会因资格层完成而自动开放。

### T11-H · 设计冻结（2026-08-22 14:13 +08:00）

- **目标**：把 T11-G 的 non-public 资格能力接到现有 Knowledge Graph / Canon 审核面，并用隔离临时库真实 Worker 子进程验收 restricted 与 secret/private_letter。
- **边界**：Web 只做 class/audience 选择与只读资格预检；node→continuity mapping 继续由 owner one-shot 脚本 list/add/幂等维护，保持 `realm_runtime` 只读和 append-only，不做删除/修改假象。
- **验收**：隔离库迁移 0001–0027；restricted 成功、secret/private_letter 成功、secret/market_rumor 永久失败；真实入口、清理和 full-suite 覆盖均纳入；secret 非 private_letter、生产容量/多副本/网络故障及 realm_dev 注入仍不做。
- **规范**：`docs/development/T11-H-NONPUBLIC-OPERATOR-AND-RUNTIME-ACCEPTANCE.md`。

### T11-H · non-public operator surface + mapping 治理 + 真实 Worker 验收（2026-08-22 派单 → 收口）

- **主题**：T11-G 服务端能力接操作面 + 治理工具 + 真实进程验收三件套。T11 系列第四个功能批。
- **commit 链**：0498588（operator surface）→ 8cef067（mapping CLI）→ 9975ca1（acceptance+注册）→ 本次（收口）。无 git add -A。
- **设计思路**：预检端点把「owner 能看到什么选择项」完全服务端化（continuity 选项/active/拓扑/readiness 都来自成员作用域只读查询），UI 只渲染不构造 id——「不手写 ID」靠数据流保证而非约定；secret 未就绪禁用提交但服务端仍拒（双层防）。审计主体收编到会话 principalId 后，body.decidedBy 成为纯噪音字段——伪造即失效有 PG 证据。mapping CLI 刻意只有 list/add：append-only 治理事实不提供删除假象。acceptance 沿用 T11-E 的真实 spawn 形态但扩到三 Job 终态（2 done+1 failed）轮询，secret 的渠道约束在 Worker 输入守卫（T11-G 审计加固时已含 root 包渠道检查）层永久失败。
- **验证证据**：npm test 全量 exit 0（183+72+33+29+64+build+render）；lint 全量 exit 0；diff --check exit 0；临时库 0；开发库基线与新表计数全录；两服务 active；无孤儿进程。
- **新发现**：①测试注册会漏——T11-A2/B 各有 PG/单测套件从未进 test:postgres-runtime/test:core（focused 过但全量不跑），本批按任务书补齐注册后全量才真实覆盖；「focused 过」不等于「全量在跑」，注册应列入每个测试批的交付清单；②secret 的 Worker 渠道守卫在 T11-G 审计加固时已把 root packet channel 纳入（fixture 的 root 包用 bulletin 会被永久拒绝）——资格约束的完整形态以 Worker 守卫为准，写 acceptance 夹具前先读守卫代码；③预检端点让「非 owner 也可见角色与拓扑摘要」成为有意识的产品选择（读预检 ≠ 有合并资格），服务端 owner gate 仍是最终授权点——读宽写窄的分工要写进规范，避免评审误判为越权。
- **下一轮决策（T11-I 候选）**：①GUI 真实点击验收（需登录凭据与 Playwright 会话）；②non-public 受众管理产品面（映射治理的 UI 化需产品决策）；③T11 系列复盘（A2/B/D/G/H 功能批 + C/E 运维验收批全链）。

## 2026-08-22 15:13:35 +08:00

### T11-H 独立审查收口

- **审查修正**：独立复核后提交 `693c122`：qualification readiness 不再把 inactive route endpoint、错误 recipient 或 inactive continuity mapping 当成 ready；mapping CLI add 的校验与 append 使用同一事务 client；前端 qualification normalizer 对嵌套 payload 逐项 fail-closed。
- **测试注册与结果**：全量 `npm test` 已实际执行本批：core 183、static/contracts 72、application 33、frontend 29、PostgreSQL 64，全部 pass；Build complete、render 3/3。`npm run lint`、`npx tsc --noEmit`、`git diff --check` 全部 exit 0。
- **运行态边界**：隔离临时库真实 Worker harness 覆盖 restricted done、secret/private_letter done、secret/market_rumor permanent failed；最终 `realm_t%`=0，`realm_dev` 传播产物和新 audience 表=0，runtime 对 mapping 只有 SELECT；两个 systemd 服务 active/running/enabled。
- **结论**：T11-H 三个交付项已通过独立审查；真实 GUI 点击、完整受众管理产品、secret 非 `private_letter`、生产压力/多副本/故障验证保持 deferred。资格层完成不改变已批准生产传播范围。
- **下一轮决策（T11-I 候选）**：GUI 真实点击验收（需登录凭据）或先做 T11 系列复盘；受众管理 UI 与 secret 渠道扩展仍需产品决策。

### T11-I · 受众产品面、GUI 与 Worker 验收追加记录（2026-08-22 18:36 +08:00）

- **当前基线**：异步审计读取的 `68b7cf4` 已过时；当前代码基线为 `6012495`，后续受众/API/UI 与 memory refresh 变更均以现场代码和原始测试重新核对。
- **GUI 真实证据**：在获得明确授权后，短暂停止并恢复 `realm-dev.service`，用专属临时数据库与 `10099` 端口启动隔离 dev server。首轮 Chromium 暴露真实缺陷：追加 API 成功后，qualification 刷新 effect 把 `role=status` 成功文案清成 idle。修正后 `tests/gui/t11-i-audience.spec.ts` 两项均通过（owner 追加+重复幂等、malformed qualification 隐藏写面），`T11I_GUI_HARNESS_EXIT=0`；服务恢复 active，临时库残留 0。
- **Worker 真实证据**：扩展 `postgres-propagation-worker-runtime-acceptance.test.ts`，把预置 `running` Job 交给真实入口启动恢复，并在首进程持 advisory lock 后启动第二个真实 `scripts/propagation-worker.mjs`。第二进程以 lock error 退出，首进程记录 stale recovery 后完成 Job 并 SIGTERM exit 0；`T11I_WORKER_MULTIPROCESS_EXIT=0`，T11E/T11H/T11I 临时库前缀均无残留。
- **边界**：owner GUI 是关闭 token gate 的隔离单用户事实，不等于多人 non-owner 授权验收；登录即默认 owner 的现有语义仍需 operator allowlist/invitation/governance role 决策。secret 非 `private_letter`、M4 tool-pause 和生产规模验证保持 deferred。
- **收口状态**：上述是已发生的局部真实证据；完整全量回归、migration inventory、最终文档提交与工作树 clean 尚待本批最后阶段执行，不能提前写成全部收口。

### T11-I · 最终独立回归与 live 边界收口（2026-08-22 19:41 +08:00）

- **全量结果**：`env -u REALM_ACCESS_TOKEN npm test` exit 0：core 183/183、contracts 72/72、application 33/33、frontend 30/30、PostgreSQL 64/64、Build complete、render 3/3；独立 typecheck、lint、documentation-layout、diff check 均 exit 0。0028 增加后两个历史 D1 migration inventory fixture 已同步到 28 条，回归通过。
- **真实 GUI/Worker**：GUI Chromium 2/2 通过；真实双 Worker acceptance 通过，包含 advisory-lock 互斥、stale recovery、clean exit 和临时库清理。没有向 `realm_dev` 注入 Job，常驻 Worker 未改 loop/unit。
- **live schema**：按最小权限决策把 0028 应用到 loopback `realm_dev`；迁移 runner 只跳过 0001–0027、应用 0028，最终台账 28 条。权限事实为 SECURITY DEFINER、固定 search_path、runtime 仅 EXECUTE，mapping 表无 runtime INSERT。
- **运行态**：inventory 报告临时数据库 0；开发库基线 worlds=2/accounts=1、propagation nodes/routes=3/2，传播产物、jobs 和 semantic evidence 全 0；两个 user service active/enabled，Worker 进程仅常驻 systemd 一份。
- **结论**：T11-I 当前可执行交付完成并有独立证据；多人 operator/invitation 授权模型、secret 非 `private_letter`、M4 tool-pause 供应商能力及生产规模验证保持 deferred，不因本批本地证据而扩大产品承诺。

### M4 Canon Claim HTTP 写入边界复核（2026-09-28）

- **复现**：新增真实 PG/runtime API 回归前，player 会话可直接 `POST /api/world-knowledge` 创建 `story_canon`（HTTP 201）；`world_canon` 同属原允许状态集。该入口只验证登录、membership scope 与输入合法性，没有要求通过 Canon 提案/Revision 路径。
- **修正**：Claim HTTP 创建仅允许 `mentioned` 与 `record_confirmed`；不接受 `story_canon`、`world_canon` 或侧态。Canon 提案 merge 流程未改；依 T11-H，所有成员可做 none/public，restricted/secret 仍限 owner。
- **证据**：先运行回归见 RED（`story_canon` 实际 201、期望 400）；修正后 `scripts/test-postgres-runtime-with-scratch.mjs tests/postgres-canon-security-propagation.test.ts` exit 0，内部测试通过，player 的 story/world 高阶直写均 HTTP 400，数据库高阶 Claim count=0。全链 PostgreSQL 0052 在 disposable 集群应用并由 runner 销毁；匹配 scratch 容器查询为空。`npm run typecheck`、目标文件 ESLint、`git diff --check` 均 exit 0；`realm-dev.service` 保持 active。
- **边界与未决**：`0010_world_governance.sql` 授予 `realm_runtime` 对 `world_claims` 的 SELECT/INSERT，RLS/FORCE RLS 只按 workspace 隔离；数据库本身不识别 HTTP principal。当前修复证明 API 服务端信任边界有效，不代表 runtime DB role 能防任意可信服务端代码绕过。若要强化到 DB 级 Canon/操作者约束，需另行设计原子数据库能力，不能只加 owner check 或信任可由 runtime 自设的 GUC。
- **状态**：本地目标代码未提交；整体 REALM 审计与 clean demo 仍 NO-GO。

### M3 Restricted Preview principal/audience 跨身份验收（2026-09-28）

- **发现与修正**：带真实 Postgres scope provider 的授权原先调用完整 runtime-scope 解析，顺带读取 recent public events；此前只用 stub 的 META-only 测试没覆盖此生产组合。新增 principal-bound viewer-id 轻量查询，保留旧 actor 选择规则和 workspace RLS，不再为 SSE 授权装载事件、Canon 或 Lore。另修正无 DB scope provider 的本地 fallback：未知 Record 必须先 fail-closed 为 404。
- **PostgreSQL/HTTP/SSE**：scratch 全迁移库经 `realm_runtime` 读取 membership 与 participant 身份，建立两名真实 principal/角色绑定；匿名 `handlePreviewGet` 返回 401 且 SQL 计数为零，陌生 principal 返回 404；合法 audience 与旁观者都通过 session cookie 打开 SSE，旁观者 query 伪造 principal/viewer 无效。测试注入受控 restricted plan 和 hub 事件：audience 收到 preview chunk 与 `preview-end`，旁观者无事件；授权查询未命中完整 EVENTS_SQL。
- **证据边界**：数据库授权与角色绑定是真实 scratch PostgreSQL/runtime role；HTTP 使用实际 route handler 的 Request/Response stream，hub 事件为 deterministic fixture；Record head 使用 in-memory fixture。未启动浏览器/GUI、未调用真实模型/provider。scratch 由 runner 销毁，测试后容器名查询为空；常驻 `realm-dev.service` 保持 active。
- **验证**：scratch PG 子测试 1/1；`preview-audience-filter` + `preview-cancel-auth` 18/18；`crystallization-singleflight`、`envelope-parallelism`、`scene-image-dispatch` 16/16；`npm run test:core` 405/405；typecheck、目标 ESLint、`git diff --check` exit 0。
- **收口**：M3 restricted-preview principal/audience HTTP/SSE 验收完成；不代表完整 GUI 旅程或其余发布项完成。工作树仍 dirty、未提交；总体 NO-GO。

### M5 provisioning/bootstrap 三缺陷修复（2026-09-29）

- **发现与修正**：① `session-secret.ts` 的既有密钥文件此前只验长度——0644/宽松目录/符号链接都会被接受；现统一为「普通非符号链接 + 0600 + secure 目录 0700（POSIX）」，运行时抛 `InsecureSessionKeyError`、provisioning 返回 null；并发首写在 rename 后回读，收敛到同一 winner 密钥，绝不覆盖。② launcher 生成的 PG URL 指向 realm_dev 而 `local-postgres.mjs` 默认建 realm_local——`launcherPgUrl(user, port, database)` 参数化 + child env 显式注入 `REALM_POSTGRES_DB`（同一来源）。③ `startRealm` 的 dataHome 此前只看 process.env——`resolveStartRealmDataHome` 导出并落实「显式 options.dataHome > options.environment.REALM_DATA_HOME > 平台默认」。
- **证据**：先 RED（auth-session 0644 被接受 + 函数缺失；launcher 契约缺 REALM_POSTGRES_DB/函数），修复后 auth-session 10/10、launcher-bootstrap-env 2/2、M5 焦点 PG（capability provisioning/boundary/actor-proof）11/11、test:core 409/409、test:contracts 504/504（含 scratch PG 171/171、容器销毁）、typecheck/lint/diff-check exit 0。本机 `.local/secure` 由 0755 收敛到 0700（目录权限，非密钥轮换）。
- **证据边界**：scratch 全为一次性 PG17（runner 销毁）；未跑 GUI/真实 provider；全量 npm test 当时仍在执行。待 Iris 独立验收；不代表 clean demo 或整体 GO。
- **收口**：仅 provisioning/launch 链三个缺陷；M5 其余治理面与既有 open 项不变。

### session-secret 内容校验与并发首写原子性修正（2026-09-29）

- **修正前一记录**：2026-09-29 早先批次声称「并发首写经回读收敛」——该说法未经真实竞争证明并被独立反例否定：`renameSync` 本质覆盖，且「权限合规但内容不合格」的既有密钥会被静默消费/替换。
- **修正内容**：既有密钥接受规则收紧为「普通非符号链接 0600 文件 + 0700 目录 + 内容 ≥32 字节且非公知开发值」；provisioning null / runtime `InsecureSessionKeyError`；不覆盖、不 chmod、不轮换、不删除。创建协议改为 `link()` EEXIST 原子 no-replace（候选临时文件 → link 落地；败方回读随附合法 winner；winner 非法则全员 fail-closed），替代原先 rename-after-write。
- **证据**：先 RED（内容校验与毒化 winner 替换两例），修复后 auth-session 13/13 四轮复跑稳定；真实 8 子进程共享 barrier 竞争收敛（1 created + 7 file、哈希一致、0600/0700、零临时残留）；毒化 winner 双方 fail-closed 且文件原样。M5 焦点 PG 11/11；typecheck/lint/diff-check exit 0。
- **边界**：竞争收敛 GREEN 稳定可复跑；崩溃残留的 `.tmp` 文件不参与读取也不阻断后续创建（设计内）。未跑 GUI/真实 provider；待 Iris 独立验收，不代表 M5/REALM GO。

### session-secret 并发回归屏障补强与独立验收（2026-09-29）

- **复审与 RED**：旧 A4 虽启动 8 个真实子进程，但父进程立即创建 go 文件，没有证明全部子进程已经到达屏障；A2 则只是顺序调用，被误标为并发。新增 A4/A5 ready 握手断言后，先运行 A4，旧 helper 缺少 ready marker，按预期 RED。
- **修正与 GREEN**：子进程完成实现模块加载、设置隔离 dataHome 后写入 ready marker；父进程确认所有 A4（8 个）/A5（2 个）参与者 ready 后才统一释放 barrier。A2 名称和注释改为准确描述顺序复用。
- **独立证据**：完整 `tests/auth-session.test.ts` **13/13**；A4 连续 **5 次**每次 1/1、0 skip；临时外部压测以全员 ready barrier 做 **5 轮 × 8 子进程**，每轮恰好 1 created、7 file、结果哈希一致、0600/0700、无 `.tmp`。`npm run typecheck`、`npm run lint`、`git diff --check` 均 exit 0；压测临时文件/目录/进程清零。
- **环境与边界**：仅 unit/进程竞争验收，未触碰数据库、GUI、模型 provider；`realm-dev.service` 保持 active 且 LAN listener 在位。未提交/未推送；整体仍 NO-GO。

## 2026-09-30

### M7 ComfyUI 安全闭环实现（Kimi 实现+自测，Iris independent acceptance pending）

- **主题**：把只读 review 证实的 ComfyUI 缺口一次性收口——operator 门禁（REALM_OPERATOR_PRINCIPALS 唯一来源、fail-closed 默认）、出站地址分类+连接 pinning（node:http 自定义 lookup，拒绝 check-connect TOCTOU 伪修复）、redirect fail-closed、总 deadline 覆盖 body、JSON/图片流式上限、WebUI 403 权限态、launcher 非敏感透传。
- **方法修正**：S3 实现先于新测试写入（违反严格 TDD 次序）；补救是把基线旧 client 忠实副本放 /tmp 复现真实 RED（redirect「Missing expected rejection」、挂起 body 超时等 6/7 失败），再对仓库新实现跑 GREEN。教训：重写型切片必须先落测试文件再动生产文件。
- **关键事实**：项目 Node 22 的 global fetch typings（lib.dom 优先于 undici-types）不含 `dispatcher`，且 undici 不可 import——这决定了 client 只能重写为 node:http/https 而非「fetch + Agent」；`redirect: "error"` 虽可一行修 redirect，但 pinning/lookup 无 fetch 路径可走。
- **越 allowlist 说明**：`tests/comfyui-history.test.ts` 不在书面清单，但 client 构造签名变更使其 typecheck 失败，按「接口变更同步调用方」惯例最小移植到 transport seam（测试语义不变）。
- **blocker**：operator 正向 GUI（保存/测试 fake ComfyUI）需 scratch runner 透传 REALM_OPERATOR_PRINCIPALS；runner 不在本批 allowlist，未改。解锁 = `scripts/test-gui-with-scratch.mjs` 的 appEnv 加一行非敏感透传。
- **证据**：route 4/4、settings 4/4、client 8/8、history 3/3、launcher 3/3、test:core 421/421、test:contracts exit 0（含 PG 177/177）、GUI Chromium F7 1/1 + F 5/5、typecheck/lint/diff-check exit 0；scratch 容器/临时目录/进程清零。

### M7 修正批：lookup all=true + DNS deadline（Kimi 自测，Iris independent acceptance pending）

- 缺陷 1 根因：Node v22 的 `http.request` 始终以 `all:true` 调 custom lookup（官方 dns.lookup 契约），callback 形状必须按 `options.all` 分支；M7 首版只回标量。教训：fake transport 测试里 lookup 由测试以标量形状直接调用，掩盖了真实 transport 的 all=true 行为——pinning 类代码必须有一条「默认生产 transport + 注入 resolver + 真实 loopback server」的端到端回归。
- 缺陷 2 根因：deadline 的 AbortController 只被 transport 段消费，DNS await 段没有竞争。修复模式（raceWithSignal：abort 即 reject、迟到 settle 空转、listener 清理）可复用到任何「外部异步 + 总 deadline」场景。
- 证据：两个缺陷分别 RED（真实 UNREACHABLE / 10s 挂死）→ GREEN（10/10）；相关单测 22/22；typecheck/ESLint/diff-check exit 0；临时 server 均 closeAllConnections 清理。仍待 Iris 独立验收；总体 NO-GO 不变。

### F7 正向 operator GUI 批（Kimi 自测，Iris independent acceptance pending）

- 解锁上批 blocker 的方式不是放开 runner 环境白名单，而是 basename 精确匹配的窄 fixture 注入：只对 `f-comfyui-operator.spec.ts` 目标注入固定 principal；契约测试显式覆盖相似路径（`xf-...`、`.bak`、`.evil`、`--grep` 值）不启用。宿主 operator 配置不透传 scratch。
- 正向 journey 用测试进程内真实 loopback fake ComfyUI（ephemeral、只回假 `/system_stats`），证明「保存 → 刷新读回 → 测试连接成功 → server 实收请求且无 key」全链；普通玩家用新注册账户独立 context 证明 API 三动词 403 零泄漏。两类身份共用同一 scratch app，顺序串行无干扰。
- 证据：contract 3/3、GUI Chromium 2/2、F 回归 5/5、typecheck/ESLint/diff-check exit 0、清理读回为零。生产权限规则零改动；总体 NO-GO 不变。

### F7 修正批：fixture 选择器误中（Kimi 自测，Iris independent acceptance pending）

- 根因：basename 匹配把 option 值（`--grep f-comfyui-operator.spec.ts`）与任意目录前缀（`unrelated/...`）误判为目标 spec。修正为「整参数 === 精确相对目标路径」+ value-taking option 跳过值 + `--opt=value` 整体忽略；真实调用 `--project=chromium tests/gui/f-comfyui-operator.spec.ts`（单/双 token 两种形态）保持启用。
- 教训：CLI 参数匹配不做 option 感知就会误中；此类选择器的契约测试必须覆盖「值位置」「目录前缀」「绝对路径」「裸 basename」四类负例，而不仅是相似文件名。
- 证据：RED（exit 1，`--grep` 用例 actual=principal）→ GREEN（4/4 exit 0）；typecheck/ESLint/diff-check exit 0；无 GUI/DB/网络。Iris 的 F7 两条 Chromium journey（2/2）不受本修正影响；本批只收窄 fixture 注入面。总体 NO-GO 不变。

### F7 二次修正批：skip-list → fail-closed 解析器（Kimi 自测，Iris independent acceptance pending）

- 根因教训：对 CLI 参数做「已知值选项 skip-list」必然 fail-open——Playwright 的值选项面会增长（本次 `--output` 即漏网）。正确形态是反向规则：只有「前驱不是分离式选项 token」的参数才可能是 positional 目标，未知选项一律按可能消费值处理（假阴性换安全）。
- `--` 分隔与 `--opt=value` 单 token 是仅有的两个例外形态；`--project` variadic 下「目标直接作 project 值」属真歧义，fail-closed 并文档化 canonical 形态。
- 测试矩阵直接从本机 CLI help 全量选项生成（26 必需值 + 4 可选值 × 分离/= 两形态），不再只盯最新反例。
- 证据：RED exit 1（`--browser <target>` 误中）→ GREEN 5/5 exit 0；探针三项（--output/canonical/--grep）符合契约；typecheck/ESLint/diff-check exit 0。总体 NO-GO 不变。

### F7 三次修正批：variadic project 消费与未知选项否决（Kimi 自测，Iris independent acceptance pending）

- 两条新教训：①「前驱 token 是不是选项」挡不住 variadic——`--project chromium <target>` 中 target 的前驱是普通值，但真实 CLI 把它当第二个 project 值（本机 `--list` 探针 exit 1 实证）；分离式 variadic 必须消费后续**全部**非选项 token。②前驱规则对未知选项 fail-open：`--future-option value <target>` 里 target 的前驱是普通 token。修正为双保险：已知选项集合外的任何选项直接否决整条命令 + variadic 值消费。
- 错误的「分离式 --project 为正向」断言来自对 variadic 语义的猜测——以真实 CLI 行为（`--list`）为准，不以推测为准。
- 证据：RED exit 1 → GREEN 5/5 exit 0；探针六项全符合契约；typecheck/ESLint/diff-check exit 0；无 GUI/DB/网络。总体 NO-GO 不变。

### GUI 审计收口批：z7 离线化 + F7 自适应（Kimi 自测，Iris independent acceptance pending）

- z7 的离线化关键接缝：ComfyUI 设置在 dispatch 时每请求现读文件，因此 spec 可以直接写 scratch REALM_DATA_HOME 的 comfyui.json 指向进程内 fake server（ephemeral loopback），无需改 runner；ready 图预置走 scratch admin SQL（content-addressed file id 与生产 store 一致），并用 REALM_GUI_SCRATCH + loopback 双护栏防误连共享库。
- F7 自适应模式：用「probe /api/settings/comfyui 是否 403」判定 fixture 是否注入，比读环境更贴近真实授权面；具名 skip 理由写明唯一支持的正向运行命令。
- 证据：两个 RED（z7 exit 1 背景断言失败；F7 --grep 运行 exit 1 正向失败）→ 三个 GREEN（z7 1/1、F7 精确目标 2/2、F7 --grep 1 passed+1 skipped）；门禁全 exit 0；清理读回为零。总体 NO-GO 不变。
