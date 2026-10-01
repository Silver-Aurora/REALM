# T10-B23-A · T10 整体复盘与 T11 主题规划（候选清单耗尽后的文档收口）

> 批次 T10-B23-A（docs/development/EXPERIENCE-ITERATION.md T10 拆批末端）。
> 立项：2026-08-21 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=815e614；本批只做事实型复盘与 T11 立项边界——不改 app/modules/database/migrations/package/lock，不接线 propagation，不加图谱轮询/SSE，不写 DB，不跑全量/GUI/PG。
> 本文只声称「文档化了 T10 事实与 T11 闸门」，不声称 T11 任何功能完成。

## 一、T10 完成矩阵（从 git log 与 STATUS 原文回读，2026-08-21）

系列起点：T10-A「DM 审查降级路径」（缺陷 #12，a3522f0 测试与收口）——
模型复核三连否决/不可用时的确定性接受与审计标记，完成。

| 批次 | 主题与结论 | commit 链（真实短 SHA） | 最小可核验证据 | 改动面 |
|---|---|---|---|---|
| B1 | 治理可达性审计 + merge 路由作用域/审计主体修复，**完成** | 60b577c→42e0723→72eedc3 | route PG 测试 exit 0；验证期实锤授权缺口 | 活动代码 + **新迁移 0022** |
| B2 | 因果冲突检测 causal 只读预览接线，**完成** | e694cf0→8920c6e→6abe690 | route PG 测试 exit 0；m5 12/12 零回退 | 活动代码，零迁移 |
| B3 | 路由 500 脱敏可观测性，**完成** | d4541ba→25e5f96→0b2d2e7 | 断库回归 5/5 exit 0；响应体逐字节不变 | 活动代码（新增共享助手） |
| B4 | memory snapshot/delta 生产入口 + D1 退役围栏，**完成** | 9667d38→4dc904a→29f2efd | route PG 测试 exit 0；api-core-wiring 全路由围栏 | 活动代码，零迁移 |
| B5 | M4 tool-pause/parallel-candidates 定性 contract-only-deferred，**评审完成（deferred）** | 04e7d4b→3014676→4a983b3 | 围栏 2/2 + m4 11/11 exit 0 | 模块头注释 + 测试，零行为改动 |
| B6 | Library 非管理命令权限矩阵 + 分池加固，**完成** | b6f2063→fa89ff5→2b23bfa | postgres-library-permissions exit 0；11/11 回归 | 活动代码，零迁移 |
| B7 | 迁移 0023 最小授权 + owner-pool 例外下沉，**完成** | d48319f→3f885af→74d16b6 | 双临时库 pre/post 实锤 exit 0；library 圈 18/18 | 活动代码 + **新迁移 0023** |
| B8-A | 残余 owner-pool 读路由（files/language/auth-me）下沉，**完成** | 6d4f9ac→0a38ca2→f184ed1 | postgres-runtime-read-routes exit 0；授权面实查零缺口 | 活动代码，零迁移 |
| B9-A | M5 propagation/semantic-conflict 定性 contract-only-deferred，**评审完成（deferred）** | 19fe444→ab83771→2c14ade | 围栏 3/3 + M5 单测 12/12 exit 0 | 仅测试，零生产改动 |
| B10-A | 图谱面板手动刷新闭环，**完成** | 0cd2a55→803f45e→d500083 | K7 Chromium/WebKit exit 0 | 前端组件 + GUI 测试 |
| B11-A | D1 退役可执行评审：「不能直接删除」结论，**评审完成** | b2488b5→54bf033→75d3fad | audit 5/5 exit 0 | 仅 docs/测试 |
| B12-A | delivery projection 纯类型提取 + 2 生产 5 测试改道，**完成** | 2e0fff3→2331c68→feb44ab | audit 6/6 + 应用套件 37/37 exit 0 | 类型宿主 + import 改道，零行为改动 |
| B13-A | D1 pair 删除资格评审：零外部引用实锤，**评审完成** | 233e837→522e7d2→ba8aeb2 | review 5/5 exit 0 | 仅 docs/测试 |
| B14-A | pair（record-store + story-record-repository）删除执行，**完成** | 22bddad→cced318→b3baa6d | 两 audit 6/6+5/5 exit 0；DB 基线不变 | **删除两文件**（用户确认） |
| B15-A | drizzle/ archive 决策：保留原位只落档，**决策完成** | 99065ac→79abc33→83ddcdd | archive review 3/3 exit 0 | 仅 docs/测试 |
| B16-A | local-only generic 与 D1 archive 契约解耦，**完成** | 6c8af80→94444d2→77f2858 | local-only 1/1 + archive 4/4 exit 0 | 仅测试契约迁移 |
| B17-A | tooling 退役资格评审与执行方案，**评审完成** | 790b253→ec81161→83e5485 | review 5/5 + 五测试链 18/18 exit 0 | 仅 docs/测试 |
| B18-A | D1/Drizzle tooling 退役执行（删 db 三件套、drizzle 归档 docs/archive/d1-drizzle/、script/依赖移除），**完成** | 5637285→9c9a5c2→adc5ec1→f153b66→dc998e3 | 六测试链 20/20 exit 0；lock 一致 exit 0 | **删除/移动文件 + package/lock**（用户授权） |
| B19-A | PG T 系列授权 provenance 集中记录，**完成** | dccf9ad→1b9f9f4→fd963e6 | schema 契约 24/24 exit 0 | 仅 docs/契约测试 |
| B20-A | 最终回归窗口，**通过** | 2201e5a（仅收口） | typecheck/Core 177/contracts 46+33+16+54/build 全 exit 0 | 零文件改动 |
| B21-A | propagation/semantic-conflict 接线资格复核：三前置逐项 FAIL，**仍 contract-only-deferred** | f4298c6→b332a80→329fab7 | 围栏 4/4 + M5 12/12 exit 0 | 仅 docs/测试 |
| B22-A | 图谱自动刷新评审：无 graph-specific invalidation contract，**维持手动刷新（deferred）** | a201768→eccd4f1→815e614 | review fence 5/5 exit 0 | 仅 docs/测试 |

纪律提醒：「有契约/有单测/有 repository 导出」不等于「生产接线完成」——
propagation/semantic-conflict（B9-A/B21-A）与 M4 两模块（B5）是
contract-only-deferred，图谱自动刷新（B22-A）是 deferred。

## 二、T10 结果分类

**1. 已完成且当前可声称**：merge 路由作用域修复（B1）、causal 冲突只读
预览（B2）、500 脱敏可观测性（B3）、memory snapshot/delta 入口（B4）、
Library 权限矩阵与分池（B6）、0023 最小授权与 owner-pool 清零（B7/B8-A）、
图谱手动刷新闭环（B10-A）、D1/Drizzle tooling 全链退役（B11–B18，
D1 原型历史仅存 docs/archive/d1-drizzle/ 与 Git 历史）、PG 授权
provenance 落档（B19-A）、回归窗口通过（B20-A）。

**2. 资格/评审完成但实现 deferred**：propagation/semantic-conflict
（三前置：enqueue 入口/worker 启动方/模型预算与降级，B21-A 逐项 FAIL）、
图谱自动刷新（无失效契约；轮询缺产品决策、SSE 缺事件源，B22-A）、
M4 tool-pause/parallel-candidates（B5）。

**3. 不能从现有证据推出的结论**：不得声称「项目全部功能完成」「传播/
语义评估已接线」「图谱已自动刷新」「全量测试已覆盖 GUI/渲染」——
B20-A 回归窗口明确未跑 GUI 与 rendered-html；render/GUI 覆盖以各
体验批次自身的 focused GUI（如 K1–K7）为准。

回归窗口已跑范围（B20-A 实录）：typecheck、test:core 177/177、
test:contracts 全链（静态 46 + application 33 + frontend 16 +
PG runtime 54）、build。明确未跑：GUI（Playwright）、rendered-html、
npm test 全量。

## 三、T11 主题候选与闸门（只规划，不选产品方向，当前不应自动开工）

### T11-A · 图谱 invalidation contract（实现自动刷新）

- 前置决策（产品）：二选一——轮询（预算/间隔、页面不可见暂停、失败
  退避参数）或 graph-specific SSE（事件源、cursor、topic route）；
- 允许触碰面：knowledge-graph-panel.tsx、（SSE 方案）新增图谱 topic
  route 与写路径事件源、（轮询方案）面板定时器与可见性策略；
- 禁止越界面：record/events SSE 语义、canon/knowledge 读 API 既有
  契约（除非规范明确加 cursor 字段）、B10 手动刷新语义（须保留）；
- 最小验收：K8（外部写入 → 不点刷新 → 节点按策略出现）+ K7 不回归 +
  B22 围栏同步翻转；
- 阻塞条件：产品未选定方案 / 预算与权限复核未成文。

### T11-B · propagation/semantic-conflict enablement

- 前置决策（产品+架构）：①campaign enqueue 的用户动作；②worker
  拓扑/启动方；③模型预算/超时/降级/证据落库策略——三项一次性满足
  才立项（T10-B9-A §二、B21-A 复核结论）；
- 允许触碰面：新 enqueue 路由/动作、dev-server 或独立进程拓扑、
  modules/propagation、modules/worldline/semantic-conflict.ts、
  模型网关配置；
- 禁止越界面：conflict 路由确定性 causal preview 语义（T10-B2）、
  未满足三前置的任何「试接线」；
- 最小验收：三前置机器证据 PASS + m5-runtime-status 围栏同步翻转 +
  PG 集成证据；
- 阻塞条件：任一前置 FAIL/UNKNOWN。

### T11-C · 维护与回归纪律保持（非业务功能）

- 内容：临时数据库盘点清理（realm_t% 命名约定）、focused→最终回归
  节奏、退役类候选的定期资格复核；
- 边界：不得把清理纪律包装成新业务功能；无产品决策不开新功能批。

## 四、可执行工程结论

**验证有效的方法**（T10 全程沉淀，T11 沿用）：

1. 当前 HEAD 重扫——资格/退役结论不引用旧批次证据（B21 示范）；
2. 指定锚点 + 所有权注释——契约恰好一个承担者，迁移面显式可枚举
   （B16 示范）；
3. focused 先行、最终回归兜底——每批 focused 证据 + B20 式四命令
   窗口（typecheck/Core/contracts/build）；
4. 真实原始 exit code——不用管道掩盖，后台任务完整等待；
5. 保留项 diff 名单 + DB/服务/临时库交叉验证——固定计数器
   （2/1/0/0/0/0）、realm-dev 状态、realm_t% 临时库盘点；
6. 规范 → 实现+围栏 → 收口的三段式提交，每批独立验收。

**已固化的坑**（各批新发现汇编，T11 验收时对照）：

- 管道/tail 掩盖 exit code；package.json 长行脚本区误删（编辑前后
  各看完整 diff）；
- git rename 100% 是「内容零改写」的机器证据；
- 目录消失后扫描面要同步收缩（scandir 崩溃）；t.after 只覆盖正常
  退出，临时库需定期盘点；
- 导出面（re-export）≠ 接线面（实例化调用）；同形原语要逐条甄别
  （SSE controller.enqueue vs propagation enqueue）；
- deferred 围栏的自指陷阱：文本断言先剔除禁止性语境；
- resolve-null 盲区：文件删除后纯 resolution 断言静默放行，须
  specifier 形态双轨。

**T11 验收纪律**：先设计闸门（前置决策/允许面/禁止面/最小验收/阻塞
条件），再实现，最后 focused + 需要时最终回归；不跑无必要全量；
不伪造 exit code；不以单测存在推导生产接线。

## 五、本批验收与范围外

- 验收：documentation-layout exit 0；受影响 eslint、git diff --check
  exit 0；diff 名单仅 docs/STATUS/EXPERIENCE；不跑 PG/GUI/全量；
  零 DB 写入；realm-dev.service 保持 active；工作树 clean。
- 范围外：T11 任何实现、产品方向选择、propagation 接线、图谱轮询/
  SSE、活动代码/DB/migration/package/lock 改动、历史批次记录改写。
