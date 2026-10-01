# T11-C · operational closure + blocked follow-up preflight 规范

> 批次：T11-C（T11-B 实现收口后的运维闭环与阻塞项前置规范；用户任务书直接派单）。
> 立项：2026-08-22 +08:00 · 状态：规范定稿，实施中
> 硬事实基线：HEAD=94ba7fc（T11-B 收口，工作树 clean）。本规范只覆盖运维纪律与 deferred 边界；不改生产行为、不加迁移、不加 UI。

## 一、Worker 安装 / 启停 / 回滚（systemd 用户级）

- 单元源：`scripts/systemd/realm-propagation-worker.service`（仓库模板）。
  安装动作（用户级，不涉及系统 systemd）：
  `cp scripts/systemd/realm-propagation-worker.service ~/.config/systemd/user/`
  → `systemctl --user daemon-reload` → `systemctl --user enable --now`。
- **与 realm-dev 分离**：Worker 是独立 unit（`Restart=on-failure`，
  `RestartSec=5s`），不随 `realm-dev.service` 启动/停止；`scripts/dev-server.mjs`
  不装配 Worker（T11-B 围栏正向锚定）。Worker 崩溃不影响 Web 请求池；
  realm-dev 重启不中断传播消费。
- **凭据边界**：unit 文件零凭据。环境经 `node --env-file-if-exists=.env.local`
  由进程自行装载；`.env.local` 不进 git、不进 unit、不进 journal 输出。
- **停止/回滚**：`systemctl --user disable --now realm-propagation-worker.service`
  + 删除 `~/.config/systemd/user/realm-propagation-worker.service` +
  `daemon-reload`。Worker 停止是天然安全态：队列持久化（pending 行保留），
  无在途丢失——running 遗留由下一次启动的 stale recovery 回收（§三）。
- **模板最小修正**（本批唯一模板变更）：`ExecStart` 的 `/usr/bin/env node`
  改为绝对路径 `/home/lyle/.hermes/node/bin/node`（v22.23.2，与
  realm-dev.service 同源）——user manager 的 PATH 经 mise shims 解析到
  node v25.9.0，与仓库 `engines: >=22.13.0` 的实测运行时不一致。

## 二、环境契约

- `REALM_RUNTIME_DATABASE_URL`：loopback 连接串（`createLocalPostgresPool`
  自带回环校验，非回环即抛），身份为 `realm_runtime` 最小权限角色
  （拓扑表仅 SELECT；队列表经 FORCE RLS + workspace 上下文）。
- `REALM_PROPAGATION_WORKSPACES`：逗号分隔显式 workspace 清单。
  **缺省实际值 = `ws_demo`**（`LOCAL_RECORD_SCOPE.workspaceId`，即
  `database/postgres/demo-seed.ts` 的 `POSTGRES_DEMO_IDS.workspace`）。
  显式清单是刻意设计而非缺陷：FORCE RLS 下 realm_runtime 不读 workspaces
  表，「全库枚举 workspace」是伪实现（T11-B 新发现③）。
- 连接预算：Worker = pool max 4 + 1 条 advisory lock 专用连接 = 上限 5；
  realm-dev 共享池 max 8。两者合计 13 ≪ 本地 PG 上限，无争用设计。

## 三、Worker 运行观察指标（验收证据面）

| 机制 | 真实代码位置 | 观察信号 |
|---|---|---|
| 单例 advisory lock | `propagation-worker-runtime.ts` `pg_try_advisory_lock(hashtext('realm_propagation_worker'))`，专用 client 持锁至进程退出 | journal `advisory lock acquired`；抢不到即抛 `PropagationWorkerLockError` 退出（systemd on-failure 重启不会叠加实例） |
| 启动健康检查 | 同文件 `to_regclass` 三表（propagation_nodes/routes/jobs） | 缺 0025 即启动失败，不消费 |
| stale recovery | 启动逐 workspace `recoverStale` | journal `recovered N stale job(s) for <ws>`（N>0 才记录） |
| 失败分类 | `isTransientPropagationError`：PG 08xxx/57P01/连接类消息=临时，attempts<3 自动 `retry`；`PropagationTopologyError` 及其余=永久 | journal `re-queued after transient failure` / `failed permanently`；重试间隔由队列按 attempts 持久化退避 `LEAST(2^attempts,60)s` |
| idle backoff | 主循环 1s→15s 指数上限，didWork 重置 1s | 空队列时轮询节奏收敛 15s——这是活性信号（无显式心跳，轮询即心跳） |
| 队列级故障 | 主循环 catch 不崩进程 | journal `poll error: ...` + 退避续跑 |

空队列验收判据：service active 且 journal 有 lock acquired、无 poll error
循环；观察窗内 `information_campaigns`/`information_packets`/
`propagation_exposures`/`propagation_jobs` 计数零增长；realm-dev.service
保持 active 互不干扰。

## 四、临时数据库盘点维护契约（realm_t%）

- 命名约定：PG 测试强制拆库的临时库命名为 `realm_t<批次>_<label>`——
  该约定是残留可安全识别的前提（T10 回归窗口新发现①：被杀进程会漏清
  t.after 的 DROP）。
- **只盘点，不盲删**：任何维护动作先列出 `realm_t%` 全量清单（稳定排序），
  由人核对批次归属后再决定清理；禁止通配 DROP、禁止清理命名约定之外的
  未知数据库。
- 工具：`scripts/db-maintenance-inventory.mjs`（本批交付，§五）。
- 清理闸门：仅接受精确 `--name` 或收窄前缀 `--prefix`（必须比裸
  `realm_t` 更长且命中盘点清单）；**拒绝 `realm_dev`、拒绝未知目标、
  拒绝裸前缀**；DROP 属授权动作，未获用户显式授权一律只报告。

## 五、维护盘点工具契约（scripts/db-maintenance-inventory.mjs）

- 配置来源：`--env-file-if-exists=.env.local` 装载后的
  `DATABASE_URL`（owner 通道，与 migrate/seed 同约定）；输出永不包含
  连接串/密码（连接串只用于建连，不落日志）。
- 默认 `report`：稳定 JSON——①`realm_t%` 临时库清单（名称升序）；
  ②开发库权威基线计数（worlds/accounts/record_first_nights/
  action_receipts/memory_snapshots/world_entities/graph_invalidation_events/
  world_claims/world_articles/canon_proposals + T11-B 新表
  propagation_nodes/propagation_routes/information_campaigns/
  information_packets/propagation_exposures/propagation_jobs/
  semantic_conflict_evaluations）；③迁移台账最大版本。
- `--drop --name <exact>` / `--drop --prefix <realm_t…>`：仅当目标命中
  盘点清单且通过 §四闸门才执行，逐个报告结果；否则非零退出零 DROP。

## 六、回归验收节奏（可执行维护契约）

1. **focused**（开发中）：受影响 PG 测试 `--test-concurrency=1` 串行 →
   受影响单元测试 → 静态链（wiring/local-only/docs/graph fence/D1 族/
   schema 契约/m5-runtime 围栏按改动面选）。
2. **批次收口**：`npm test`（typecheck → test:core → test:contracts →
   build → test:render）→ `npm run lint` 全量 → `git diff --check`。
3. 纪律：全部记录**原始退出码**；失败在批内修复，不为变绿改旧语义；
   PG 多文件永远串行（共享开发库）。

## 七、restricted/secret 传播前置设计（本批只定契约，保持 deferred）

**已有（真实代码/DB）**：
- DB：`propagation_nodes.clearance` CHECK 枚举 `public/restricted/secret`
  （迁移 0025）；`propagation_routes.channel` 枚举
  `official_bulletin/private_letter/market_rumor` 且 private_letter 必填
  recipient；`information_campaigns.security_class` 列。
- 引擎：`modules/propagation/public.ts` `CLEARANCE_RANK` 过滤——
  节点 clearance 等级 ≥ Campaign securityClass 才可接收（等级语义已在
  引擎成立）。

**缺（未实现，禁止假冒）**：
- Campaign securityClass 来源：`canon-propagation.ts` 恒写 `'public'`，
  无任何入口产生 restricted/secret Campaign。
- Claims/CanonRevision 无 visibility 字段——「哪些正史可标 restricted」
  无 schema 承载。
- 受众裁决：谁能接收 restricted/secret 无表无规则；private_letter 的
  recipient 授权判定缺失。
- 治理/UI 入口为零。

**fail-closed 边界**：当前唯一 public 证明 = merge 请求显式
`propagate:"public"` attest（T11-B 绑定）；缺省不建传播任务；不存在任何
代码路径能写出非 public Campaign。

**下一批资格闸门**（全部满足才立项）：①visibility 字段的 schema 决策
（Claim 级还是 revision 级，含迁移授权）；②受众裁决模型（角色/派系
成员关系的持久化来源）；③restricted/secret Exposure 的读取授权矩阵；
④产品决策哪些 Canon 允许非 public 标记。

## 八、semantic review 前端入口前置设计（本批只定契约，保持 deferred）

**已有**：`app/api/worldline/conflict/semantic/route.ts` 独立 POST 路由
（T11-B：显式触发、服务端重读作用域、per-workspace 单飞 409、>64 Claims
400、8s 真实 Abort deadline、evidence 落库失败 503、none/hard 零模型）。
**缺**：前端零调用——`app/` 全量扫描无任何 `conflict/semantic` 引用；
无「请求语义复审」按钮、无 evidence 展示面、无 i18n key。
**fail-closed 边界**：无 UI 入口 = 用户不可触发 = 模型零调用；路由本身
成员门禁/严格解析已 fail-closed，不依赖前端。
**下一批资格闸门**：①产品决策入口位置（conflict preview 面板内还是独立
动作）；②预算/频率的 UI 表达（409 SEMANTIC_REVIEW_BUSY 与 503
UNAVAILABLE 的用户可见形态）；③evidence 展示契约（recommendation 只是
evidence、绝不进正史的界面表达）；④i18n key 集（三语）。未满足前不加
按钮、不造假调用点。

## 九、失败矩阵（运维面增补）

| 场景 | 正确结果 |
|---|---|
| Worker 抢不到 advisory lock | 非零退出，systemd on-failure 退避重启，不叠加实例 |
| 拓扑/队列表缺失（0025 未应用） | 启动健康检查失败，不消费 |
| DB 重启/连接中断 | journal `poll error` + 退避，进程不崩 |
| 临时错误重试达 attempts 上限 | 留 failed，不无限重试 |
| 盘点工具收到 realm_dev/未知/裸前缀清理目标 | 拒绝，非零退出，零 DROP |
| 无凭据环境运行盘点工具 | 报错非零退出，输出零 secret |
| Worker 与 realm-dev 同 unit/同进程 | 禁止——生命周期分离是本批硬边界 |

## 十、交付步骤

1. 本规范 + docs 索引登记；2. systemd 模板最小修正（单独提交）；
3. 用户级安装/enable/journal 验证 + 空队列观察；4. 盘点工具 + 测试；
5. focused 测试/typecheck/eslint/diff --check；6. STATUS 三段式 +
   EXPERIENCE-ITERATION 收口。
