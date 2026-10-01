# T11-D · semantic review 操作面

> 批次：T11-D（T11-B semantic review route 的显式操作面）
> 立项：2026-08-22 · 状态：前置决策已定，待实现
> 前置：T11-B semantic review route 已存在并通过 PG/route 回归；T11-C 已完成 Worker 运维闭环。

## 一、目标

把已经存在的 `POST /api/worldline/conflict/semantic` 变成一个用户可以明确触发、看懂结果、且不会误写正史的操作面。

本批只处理 semantic review 的**请求与证据展示**：不合并 Canon、不改写 Claim、不创建分支、不改变 Worldline 头，不引入 restricted/secret 传播语义。

## 二、产品决策

### 2.1 入口位置

入口放在现有知识图谱 `KnowledgeGraphPanel` 的实体详情 Claim 列表内。理由：

- Claim 是 semantic review 的事实输入，用户无需跳到另一套世界线工具；
- 操作天然绑定当前 `worldId` 和选中的 Claim；
- 不新增独立导航、页面或隐藏入口。

每条 Claim 展示一个「请求语义复审」动作。没有选中 Claim 时不显示动作。

### 2.2 复审形态

一次只提交一个 change，默认以当前 Claim 作为草稿来源：

- `assert`：保留 subject/predicate，用户编辑候选 object；
- `terminate`：以当前 Claim 为 target，表示在当前世界线位置终止它；
- `supersede`：以当前 Claim 为 target，用户编辑新的 object 作为替代事实。

表单必须明确显示：这是**复审草稿**，结果是 evidence，不会自动合并、不改变正史。

### 2.3 时间游标

前端不让用户手填世界时间。新增只读 context 读取：

`GET /api/worldline/conflict/semantic/context?worldId=...`

服务端完成成员校验后，从当前 `worldlines.head_tick/head_ordinal` 生成 `existingFuture`。本批把复审 change 的 `effectiveCursor` 固定为该当前头游标，避免客户端伪造时间事实。

未来如果支持回溯到任意历史游标，另开批次，不在本批顺手扩展。

### 2.4 调用与预算

- 选择 Claim 不调用模型；打开表单不调用模型；只有点击「请求复审」才 POST；
- `none/hard` 仍由服务端确定性返回，不产生 semantic evidence；
- `high-risk` 才走最多一次模型调用与 evidence 落库；
- 不自动重试。409 `SEMANTIC_REVIEW_BUSY` 提供用户再次点击的明确动作；503 `SEMANTIC_REVIEW_UNAVAILABLE` 只显示稳定提示，不泄露内部错误。

### 2.5 结果展示

展示以下信息：

- 确定性分级：none / bridgeable / hard / high-risk；
- recommendation：merge / branch / reject；
- rationale；
- semantic evidence 的 source：model / fallback（若有）；
- 固定提示：「这是复审证据，不会自动写入正史」。

不展示 prompt、inputDigest、内部 requestId，不提供「直接合并」按钮。

## 三、服务端契约

### 3.1 Context route

`GET /api/worldline/conflict/semantic/context?worldId=<id>`：

- 需要现有 request principal；无 principal 返回 401；
- 缺 runtime 返回 503 `LOCAL_RUNTIME_NOT_INITIALIZED`；
- 非成员/未知世界返回 404 `WORLD_NOT_FOUND`；
- 成功只返回：`ok`、`scope.worldId`、`scope.worldlineId`、`existingFuture`；
- 只读，不写 evidence、不改任何正史。

### 3.2 Semantic POST

继续使用既有 POST route。前端从 context 取得 `existingFuture`，构造：

```json
{
  "worldId": "...",
  "existingFuture": {
    "tick": 123,
    "ordinal": 7,
    "calendarId": "native",
    "display": ""
  },
  "changeSet": {
    "changes": [
      {
        "kind": "supersede",
        "subjectEntityId": "...",
        "predicate": "...",
        "objectValue": "...",
        "targetClaimId": "...",
        "effectiveCursor": { "tick": 123, "ordinal": 7, "calendarId": "native", "display": "" }
      }
    ]
  }
}
```

服务端继续重读 claims/causal edges；客户端传入的 Claim 内容只能作为草稿，不能替代服务端事实。

## 四、前端边界

- `KnowledgeGraphPanel` 新增 `uiLanguage` 入参；本批新增文案全部进入 `modules/i18n` 的 zh-CN/en/ja；不顺手重写既有图谱中文文案；
- 新增独立 `SemanticReviewPanel` 或等价组件，表单、busy、error、result 状态局部管理；
- 选中 Claim 后动作进入面板；关闭/取消不会触发请求；提交成功展示结果并保留原图谱快照；
- SSE 刷新只刷新知识快照，不清除正在展示的复审结果，除非组件卸载；
- 不新增路由导航，不新增模型设置，不新增数据库迁移。

## 五、测试与验收

1. context route：无 principal、无 runtime、未知世界、非成员、成功返回 head cursor；
2. semantic review UI contract：三种 change kind 的 payload 形态、服务端 cursor 来源、显式触发、错误码映射、evidence 不出现内部字段；
3. 既有 semantic route/PG 回归保持通过；
4. typecheck、受影响 eslint、静态接线围栏、`npm test`、全量 lint、`git diff --check`；
5. 若现有 GUI harness 可稳定覆盖该面板，补一个不调用真实模型的 deterministic `hard` 操作验证；无法稳定覆盖时，必须保留可执行的组件/契约测试，不用模型输出冒充 UI 验收；
6. 数据库最终保持 T11-C 基线：`realm_t%` 临时库为 0；semantic evidence 仅允许由显式测试产生，测试后清理为 0；
7. Worker 与 realm-dev 服务边界不变；restricted/secret 传播仍 deferred。

## 六、禁止扩展

- 不实现 restricted/secret visibility 字段、受众模型或 recipient 授权；
- 不把 semantic recommendation 自动写入 Canon/Worldline；
- 不新增 semantic review 的后台队列、自动触发器或轮询；
- 不把模型 evidence 当成正史或事实来源；
- 不把当前头游标方案扩展成完整时间线编辑器。

## 七、提交边界

建议按主题拆分：

1. context route + route contract tests；
2. semantic review panel + i18n + wiring；
3. UI/静态契约测试与文档/STATUS 收口。

每笔提交只包含对应主题；不 push 远程。
