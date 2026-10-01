# T11-E · public propagation 真实运行验收 harness

> 批次：T11-E（T11-B Worker 的真实进程入口验收）
> 立项：2026-08-22 · 状态：前置决策已定，待实现
> 前置：T11-B Worker/拓扑已实现；T11-C 已完成 systemd 空队列运行观察；T11-D semantic review 操作面已收口。

## 一、目标

证明 `scripts/propagation-worker.mjs` 这一**真实独立进程入口**，在存在一条合法 public propagation Job 时，能够：

1. 建立 advisory lock；
2. 按显式 workspace 清单领取 Job；
3. 完成 Campaign、Packet、Exposure 持久化并把 Job 标记为 done；
4. 收到 SIGTERM 后干净退出；
5. 退出后临时数据库可强制清理，开发数据库和 systemd 服务不受影响。

现有 `postgres-propagation-worker.test.ts` 已覆盖核心 queue/worker `runOnce`、幂等、作用域和锁，但本批专门补真实入口进程链，不把核心单测当成进程验收。

## 二、明确决策

### 2.1 只测 public

Harness 只创建 `security_class='public'` 的 Campaign、public topology、官方公告/市场传闻合法路径。restricted/secret 仍不产生、不读取、不定义受众语义。

### 2.2 临时数据库隔离

- 每次测试创建 `realm_t11e_rt_<uuid>` 临时库；
- owner 连接执行迁移 0001–0025、demo seed、public topology 和一条 pending Job；
- Worker 子进程使用 `REALM_RUNTIME_DATABASE_URL` 指向该临时库，并通过 `REALM_PROPAGATION_WORKSPACES=ws_demo` 显式授权 workspace；
- 禁止把 Job 写入 `realm_dev`，禁止让已安装的 systemd Worker 消费本测试 Job；
- `t.after` 必须先 SIGTERM/等待子进程退出，再回收连接并 `DROP DATABASE ... WITH (FORCE)`；测试失败也必须清理。

### 2.3 真入口，不复制实现

Harness 必须 spawn 当前仓库的：

```text
node --experimental-strip-types scripts/propagation-worker.mjs
```

不得在测试里直接调用 `createPropagationWorker().runOnce()` 冒充入口验收。Worker 的 stdout 只收集无凭据的状态行，禁止打印连接串或环境完整内容。

### 2.4 可观测完成信号

不得用固定 sleep 作为完成判断。测试必须轮询临时库的 Job 状态，直到：

- `propagation_jobs.status='done'`；
- `information_campaigns=1`；
- `information_packets` 与 `propagation_exposures` 数量符合本 topology 预期（当前 demo topology 预期 3 exposures）；
- Worker 输出同时出现 `advisory lock acquired` 与 `job done for ws_demo`（Job 身份由临时库中的唯一 Job 状态与产物计数闭环确认）。

超时必须非零失败，并在失败信息中只带 Job 状态/安全日志行，不带连接串。

## 三、实现范围

1. 新增独立 PG integration harness 测试，沿用项目 loopback URL 检查、迁移全链、临时库命名和 `t.after` 清理纪律；
2. 必要时抽取无副作用的测试 helper，但不改变 Worker 生产循环；
3. 将 harness 登记到 package 的 focused test script，使用 `--test-concurrency=1`；
4. 补 docs/README 索引与 STATUS/EXPERIENCE-ITERATION 开发事实；
5. 不新增 migration，不修改 systemd unit，不安装/重启/停止现有服务。

## 四、验收

- harness 红灯阶段证明“没有真实入口完成信号”或实现前测试失败；
- harness 绿灯：真实子进程完成一条 public Job，进程 SIGTERM 后 exit 0；
- focused Worker 旧回归保持通过；
- typecheck、受影响 eslint、harness、PG focused、全量 `npm test`、全量 lint、`git diff --check`；
- 测试后 `realm_t%` 为 0；`realm_dev` 的 Campaign/Packet/Exposure/Job/semantic evidence 仍为 0；
- systemd `realm-dev` 与已安装 propagation-worker 均保持 active/running/enabled，既有 Worker journal 无新增 poll error；
- 任何失败/超时都不能留下临时数据库、子进程或凭据输出。

## 五、禁止扩展

- 不把 harness 变成生产诊断 API；
- 不向 realm_dev 注入测试 Job；
- 不执行任何用户未授权的生产清理；临时库 DROP 仅是 harness 自己创建的隔离库清理；
- 不实现 restricted/secret；
- 不改变 public propagation 的业务语义、topology、重试或模型配置；
- 不把 systemd 空队列观察改写成真实任务吞吐，真实吞吐结论只来自本 harness 的临时库事实。
