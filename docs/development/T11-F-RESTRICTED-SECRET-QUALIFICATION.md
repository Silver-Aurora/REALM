# T11-F · restricted / secret qualification design

> 批次：T11-F（T11-C deferred 资格闸门的产品与数据契约）
> 立项：2026-08-22 · 状态：决策已定，待 T11-G 实现
> 前置：T11-B public propagation 已落地；T11-D semantic review 操作面已收口；T11-E 已用真实独立进程验收 public Job。

## 一、决策摘要

本批只冻结 restricted/secret 的资格模型，不开放生产非 public 传播。后续实现必须先满足本规范的 schema、授权和负向回归；任一项缺失都保持 fail-closed。

| 决策项 | 选择 | 不能采用的替代 |
|---|---|---|
| 安全分类粒度 | `CanonRevision` 级 | Claim 级混合分类、Packet 运行时临时改级 |
| 受众主体 | worldline-scoped `character_continuity` | account/principal 直接授权、record 内短命 `character_instance` |
| 受众保存 | immutable audience snapshot | 每次读取实时重算、从 propagation node clearance 推断 |
| 读取授权 | principal 在当前 Record/stance 中实际控制 audience continuity；owner 仅作为控制面审阅权限 | membership role 自动授予角色秘密、observer 默认可读 |
| secret 传播渠道 | 第一阶段仅 `private_letter` | official_bulletin、market_rumor、隐式降级为 restricted |
| 缺失授权时 | 拒绝入队/拒绝读取 | 隐式 public、空 audience 放行、返回“暂无数据”伪装成功 |

## 二、为什么是 CanonRevision 级

Canon merge 是一条不可变 Revision、一个 root Packet 和一个 Campaign 的原子边界。当前传播 root packet 携带整组 `claimIds`，引擎没有按 Claim 拆分 Campaign 的能力。

因此一条 Revision 必须只有一个 `securityClass`：

- `public`：兼容现有 T11-B 显式 `propagate: "public"` 路径；不要求 audience snapshot；
- `restricted`：必须存在一个非空、已校验的 audience snapshot；允许多个角色主体；
- `secret`：必须存在一个非空、已校验的 audience snapshot；首阶段只允许 private_letter 路线；
- 一条 Revision 中不得混合不同安全级 Claim。若产品未来需要混合，必须先拆成多个 Revision/Campaign，不能在 Worker 中临时切分。

`world_claims` 继续保持事实本体；安全分类不写回 Claim，不修改旧 Claim 的 append-only 语义。`CanonRevision` 是传播安全分类的唯一正史承载点，`information_campaigns.security_class` 只作为不可变派生快照。

## 三、受众主体与快照

### 3.1 主体选择

受众使用 `character_continuities.id`：它在同一 worldline 内跨 Record 保持稳定，能够覆盖角色实例重建、继承和多 Record 连续性。`character_instance_id` 只用于读取时解析当前 Record/stance，不写入 CanonRevision 的长期授权主体。

新表建议：`canon_revision_audiences`

- 复合主键：`workspace_id, world_id, worldline_id, revision_id, continuity_id`；
- 复合外键指向同一 worldline 的 `canon_revisions` 与 `character_continuities`；
- append-only trigger，禁止 UPDATE/DELETE；
- workspace/world/worldline 全量 RLS；
- `public` Revision 不插 audience 行；`restricted/secret` 至少一行；
- 快照在 merge 同一事务中写入，之后角色控制权变化不会扩大历史 Canon 的 audience 集合。

### 3.2 propagation node 映射

现有 `propagation_nodes.clearance` 只表达“节点最高可接收的安全等级”，不表达“哪个玩家角色可以读取 Exposure”。两者必须分离。

新表建议：`propagation_node_audiences`

- 复合主键：`workspace_id, world_id, worldline_id, node_key, continuity_id`；
- node 必须是同一 scope 的 `propagation_nodes`，continuity 必须是同一 worldline 的 `character_continuities`；
- public topology 可不配置映射；restricted/secret 的有效 Exposure 读取必须经过映射；
- `private_letter` 的 route recipient 必须指向唯一目标 node，且该 node 对应唯一 continuity；零映射或多映射一律拒绝，不猜 recipient。

这两张表的职责不同：

- `canon_revision_audiences`：这条 Canon 允许哪些 continuity 接收；
- `propagation_node_audiences`：某个社会节点代表哪些 continuity；
- `propagation_nodes.clearance`：节点在传播网络中能承接的最高 security class。

三者缺一不可，不能用其中任意一张替代另外两张。

## 四、读取授权矩阵

Exposure/Packet 读取必须落在服务端授权查询中，不能只靠前端隐藏：

| 请求主体 | public | restricted/secret |
|---|---:|---:|
| world owner 的控制面审阅请求 | 可读 | 可读，但必须标记为控制面视图，不等于角色已知 |
| 当前 Record 中由 principal 控制、且 continuity 在 revision audience 内的角色 | 可读 | 可读 |
| 当前 Record 中 principal 控制角色，但 continuity 不在 audience 内 | 可读 | 拒绝/过滤 |
| observer 且没有对应角色控制关系 | 可读 public | 拒绝/过滤 |
| 未加入 world 的 principal | 拒绝 | 拒绝 |

`owner` 的例外只适用于明确的控制面审阅 API；普通 Character Delivery Projection 仍必须按角色 perspective 过滤，不能因为 owner 身份把秘密塞入角色 Prompt。

读取授权必须通过现有 `participants.principal_id → character_instances.continuity_id` 关系解析当前 Record/stance，并同时检查 active 状态、workspace/world/worldline scope。禁止用 display name、node key 或 account display name 作为授权键。

## 五、传播资格闸门

### 5.1 Canon merge

T11-G 才实现以下入口，T11-F 不改生产行为：

1. merge 请求显式声明安全分类；
2. public 继续要求现有 attest；
3. restricted/secret 必须同时提供 audience continuity 列表；
4. 列表必须去重、非空、属于当前 worldline 且状态有效；
5. Revision、audience snapshot、Campaign、root Packet、Job 必须同事务提交；
6. 任何一项失败，整个 merge 回滚，不能先写 Canon 再静默跳过传播。

当前客户端只支持 `propagate: "public"`。在 T11-G 接线完成前，任何 `restricted`/`secret` 请求必须返回结构化 `PROPAGATION_SECURITY_UNAVAILABLE`，不能被当成“不传播但照常 merge”吞掉。

### 5.2 Worker / engine

Worker 不负责决定受众，只消费已冻结的 immutable Job input：

- Job 必须携带 `securityClass`、revision id 和 audience snapshot digest；
- Worker 只能按 topology clearance 和 route channel 传播；
- Worker 不得实时查询角色权限，也不得扩大 audience；
- security class 与 audience digest 不匹配时，Job 永久失败，不重试；
- secret 的第一阶段只允许 private_letter，其他 channel 属永久拓扑/资格错误。

### 5.3 Exposure 读取

`propagation_exposures` 仍是传播事实，不直接等于角色可见知识。读取 API 必须 join：

```text
Exposure
  → Campaign / canon_revision_id
  → canon_revision_audiences
  → current controlled character continuity
```

join 任一环节缺失都过滤；不得返回空壳 Exposure，也不得 fallback 到 public。

## 六、迁移与实现顺序（T11-G）

1. `0026_canon_security_audience.sql`：Revision security class + immutable audience snapshot + RLS/least privilege；
2. `0027_propagation_node_audiences.sql`：node→continuity 映射、复合外键、private_letter 唯一目标约束；
3. 领域类型与 merge qualification：显式分类、同 Revision 单一分类、非 public audience 校验；
4. Canon repository/propagation enqueue：同事务写 snapshot 与 immutable Job input；
5. Exposure 读取授权：控制面和 Character perspective 分开；
6. 负向回归：跨 workspace/worldline、空 audience、重复 audience、失效 continuity、node clearance 不足、secret 非 private_letter、无授权读取；
7. public 回归与 T11-E 真实 public harness 保持通过。

每一步都必须先写失败测试再实现。迁移 runner、RLS、runtime role grants 和开发库基线单独验收。

## 七、明确不做

- 本批不新增 migration；
- 本批不让任何生产路径产生 restricted/secret Campaign；
- 不把现有 record `visibility_policies` 直接当作 Canon audience；
- 不把 membership owner/player/observer 直接当作情报受众；
- 不让 frontend 自己决定安全分类或 audience；
- 不把 security class 变化做成 UPDATE；变化必须新建 Revision/Campaign；
- 不做 secret 的 official_bulletin/market_rumor；
- 不推送远程仓库、不执行开发库 DROP。

## 八、资格结论

T11-F 的产品与数据决策已冻结。T11-G 可以在上述边界内实现迁移和 fail-closed qualification；在 T11-G 的数据库、读取授权和负向回归全部通过前，restricted/secret 仍保持不可传播。
