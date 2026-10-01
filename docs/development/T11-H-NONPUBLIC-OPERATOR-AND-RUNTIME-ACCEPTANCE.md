# T11-H · non-public propagation operator surface and runtime acceptance

> 批次：T11-H
> 立项：2026-08-22 · 状态：独立审查完成
> 前置：T11-G qualification gate、Exposure service authorization、T11-E real public Worker harness 已通过独立审查。

## 一、目标

把 T11-G 已完成的服务端资格能力接到一个真实可用的 owner operator surface，并用真实 Worker 子进程验收 restricted/secret 的已批准路径。完成后：

1. owner 可以在现有 Knowledge Graph / Canon 审核面明确选择 `public`、`restricted` 或 `secret`；
2. non-public 选择必须显式勾选当前 worldline 的 active character continuity audience；
3. UI 只读显示当前 topology 的 secret readiness，不猜测或自动创建 node→continuity 映射；
4. 资格不足时显示服务端结构化错误，不降级 public、不只合并不传播；
5. restricted 和 secret/private_letter 在隔离临时库中由真实 `scripts/propagation-worker.mjs` 消费并闭环；secret 的 market_rumor / official_bulletin 永久失败。

## 二、范围与不变项

### 2.1 允许修改

- `GET /api/canon?view=qualification&worldId=...` 的只读资格预检；
- 现有 `KnowledgeGraphPanel` 的 CanonReviewList operator surface；
- 对应三语 i18n、frontend contract、API/PG focused test；
- owner 维护脚本 `scripts/propagation-node-audiences.mjs`：只允许列出和追加 immutable node→continuity mapping；
- 隔离临时库的 restricted/secret real Worker acceptance harness；
- 本批规范、STATUS、EXPERIENCE-ITERATION、docs README。

### 2.2 明确不做

- 不给 `realm_runtime` 增加 `propagation_node_audiences` INSERT/UPDATE/DELETE；
- 不在 Web runtime 中读取 `DATABASE_URL`，不扩大 owner-pool 例外；
- 不实现 append-only mapping 的删除/修改假象；错误映射只能通过新的治理决策/拓扑版本处理；
- 不扩展 secret 到 official_bulletin 或 market_rumor；
- 不改 `0026/0027`、Worker 生产循环、systemd unit；
- 不向 `realm_dev` 注入 non-public Campaign/Job；
- 不把真实隔离 harness 写成生产吞吐、多副本或网络故障验收。

## 三、operator surface 契约

### 3.1 资格预检

`GET /api/canon?view=qualification&worldId=<world>` 只对 world member 返回：

- `membershipRole`；
- active continuity 选项：稳定 id、角色显示名、当前 worldline scope；
- topology 节点及 clearance；
- secret readiness：是否存在 private_letter route、recipient 是否存在唯一 node mapping；
- 不返回 Claim 内容之外的秘密，不接受客户端指定 workspace/worldline。

非 owner 仍可读取预检信息，但 POST merge 资格仍由现有 owner gate 决定。UI 对非 owner 隐藏 non-public merge 控件，服务端仍是最终授权点。

### 3.2 Canon 决策

- 默认保持现有 public merge 行为；
- owner 选择 restricted/secret 后必须选择至少一个 continuity；
- 决策请求携带 `propagate` 与 `audienceContinuityIds`；
- UI 不构造 `decidedBy` 身份，服务端从会话主体决定授权并只把审计主体写入 Canon；
- 409/400 的服务端错误保持结构化、可翻译、不可静默降级。

### 3.3 Mapping 维护

`scripts/propagation-node-audiences.mjs` 使用 one-shot `DATABASE_URL` owner 通道，接受：

- `list --world <id>`：列出当前 worldline 的 nodes、routes、continuity 与 immutable mappings；
- `add --world <id> --node <node_key> --continuity <continuity_id>`：在事务中校验同 scope、node/continuity 存在且 continuity active 后追加一条映射；重复追加幂等；
- 不提供 delete/update；不接受 `realm_dev` 的隐式清理；输出不含连接串。

脚本只负责治理映射，不绕过 Canon audience snapshot，不直接创建 Campaign/Job。

## 四、真实 Worker acceptance

新 harness 必须：

1. 创建唯一临时数据库并完整迁移 0001–0027；
2. 写入隔离 workspace/world/worldline、active continuity、secret-clearance nodes、private_letter route 和 node audience mapping；
3. 写入一条 restricted Job，spawn 真实 Worker，验证 `status=done`、campaign/packet/exposure 计数和清理；
4. 写入一条 secret/private_letter Job，验证目标 node Exposure 产物闭环；
5. 写入一条 secret/market_rumor Job，验证 Worker 永久 failed、没有完成态或错误 channel 产物；
6. 通过真实 SIGTERM、临时库清理和无孤儿子进程收口；
7. 不触碰 `realm_dev`，不改已安装 systemd 服务。

## 五、验证

- 先写 UI/API/mapping/acceptance 失败回归，再实现；
- focused：operator/API、mapping CLI、T11-H real Worker、既有 T11-G/T11-E；
- 全量：`env -u REALM_ACCESS_TOKEN npm test`、`npm run lint`、`git diff --check`；
- UI 真实点击仍需登录凭据；没有凭据时不猜、不绕过、不把构建通过写成 GUI 通过；
- 最终盘点 `realm_t%`、`realm_dev` 基线、新表计数、服务与进程边界，原始退出码全部记录。

## 六、完成判据

T11-H 只有同时满足以下条件才收口：

- owner operator surface 可明确选择 class/audience，非 owner 与非法输入 fail-closed；
- mapping CLI 的 list/add/幂等/越界拒绝有真实 PG 证据；
- restricted 与 secret/private_letter 真实 Worker 子进程 acceptance 通过；
- secret 非 private_letter 永久失败有真实证据；
- public/T11-G/T11-E 回归保持通过；
- 全量脚本纳入本批关键测试；
- 文档、清理、服务边界和工作树全部收口。
