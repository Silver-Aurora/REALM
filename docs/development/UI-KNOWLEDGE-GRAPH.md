# UI 实施规范：世界知识图谱与 Canon 审核入口

> 文档性质：界面实施契约与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 视觉规范：[`../design/UI-DESIGN.md`](../design/UI-DESIGN.md)（纸墨纪事：直角、语义色、硬阴影、信息克制）。
> 数据契约：[`./M5-WORLD-GOVERNANCE.md`](./M5-WORLD-GOVERNANCE.md)。

## 1. 形态与入口

- 混合形态：左侧图谱总览（节点-边 SVG 视图），右侧详情下钻栏。
- 入口在 World 层级：世界库每个世界条目标题行新增「图谱 · 正史」小按钮，打开全屏覆盖层（沿用 library-overlay 模式），不属于故事或记录级。
- 覆盖层顶栏：世界名 + 关闭按钮；右上角墨色小标签显示当前世界线。

## 2. 页面结构与组件树

```text
.graph-overlay（全屏覆盖层，纸张底）
└── .graph-panel
    ├── header：世界名 / 世界线标签 / 关闭
    ├── .graph-layout
    │   ├── .graph-canvas（SVG 节点-边视图）
    │   │   └── 图例（实体类型 × 颜色，一行小方块）
    │   └── .graph-side
    │       ├── tabs：详情 / 正史审核
    │       ├── 详情 tab
    │       │   ├── 实体卡：名称、类型、摘要（可编辑）
    │       │   ├── Claim 列表（默认最多 5 条，「显示全部」展开）
    │       │   ├── 关系列表（该实体为 subject 的 relation）
    │       │   └── Article 列表（点击展开全文）
    │       └── 正史审核 tab
    │           └── Canon 提案卡：目标层级、理由、关联 Claim/Article、合并/拒绝
    └── .graph-actions（底部操作条）
        └── 新建实体 / 编辑摘要 / 建立关系 / 提交 Claim（直角纸片表单）/ 刷新图谱（批次 T10-B10-A：手动重读图谱+正史，状态与竞态语义见 T10-B10-A 规范）
```

## 3. 图谱渲染（手写 SVG，零新增依赖）

- 布局：确定性环形布局——节点按实体 id 排序后等角分布在圆环上；实体 ≤2 时退化为横排。无物理仿真、无第三方库。
- 节点：直角矩形纸片（非圆形、非圆角），fill 按实体类型取色，1px 墨色边线；选中态加朱红 2px 内框线与硬阴影。
- 边：1px 直线，墨色 40% 描边，中点放 8px mono 谓词标签；不透明度只用于边线文字层级，节点与卡片不使用半透明填充。
- 配色（全部取自既有语义调色板，无新色）：

| 实体类型 | 色值 | 来源 token |
|---|---|---|
| 人物 person | #b94c36 | `--accent` 朱红 |
| 势力 faction | #526b50 | `--semantic-action` |
| 地理 geography | #3f6770 | `--semantic-environment` |
| 历史 history | #9c7438 | `--gold` 金棕 |
| 设定 setting | #70556b | `--semantic-story` |
| 其他 other | #55594f | `--ink-soft` |

- 空态：无实体时显示菱形符号 + 「这个世界还没有结构化知识」说明与新建入口。

## 4. 交互契约

- 点击节点 → 右侧详情显示该实体；再次点击空白处取消选中。
- Claim 列表默认按真值阶梯排序展示前 5 条（story_canon/world_canon 优先），不堆砌全部；「显示全部」展开。
- 编辑四操作均为直角纸片表单（与 library-form 同款）：
  - 新建实体：名称 + 类型 + 摘要；
  - 编辑摘要：仅摘要文本（选中实体后可改）；
  - 建立关系：选中实体为 subject，选择目标实体与谓词；
  - 提交 Claim：选中实体为 subject，谓词 + 值 + scope（story/world）+ 初始 truth_status（mentioned/record_confirmed）。
- 正史审核：pending 提案列表 → 展开详情（目标层级、理由、关联 Claim 标题、Article 摘要）→「合并」/「拒绝」；操作后列表即时刷新。
- 所有写入只走服务端 API（`/api/world-knowledge`、`/api/canon`），前端不直连数据库。

## 5. 服务端接口

`GET /api/world-knowledge` → `{ entities, claims, relations, articles }`（当前默认世界线）。

`POST /api/world-knowledge`：

| action | 载荷 | 校验 |
|---|---|---|
| `upsertEntity` | id?、kind、name、summary | kind 合法、name 非空；无 id 由服务端生成 |
| `appendClaim` | subjectEntityId、predicate、objectValue、scope、truthStatus | 主体存在，scope 合法；新建 Claim 的 truthStatus 仅允许 `mentioned` / `record_confirmed`，更高 Canon 状态必须通过 `/api/canon` owner-gated 决策路径 |
| `appendRelation` | subjectEntityId、objectEntityId、predicate、claimId | 两端实体存在、claim 存在 |

`GET /api/canon`（已有）+ `POST /api/canon { action: "decide" }`（已有）。

## 6. 验收标准

1. 覆盖层从世界库世界条目打开与关闭正常；图谱加载出实体节点与关系边。
2. 不同实体类型节点 fill 色值与规范表一致。
3. 点节点下钻：Claim 列表与关系显示；Article 可展开。
4. 四个编辑操作全部生效并立刻反映在视图（创建实体上新节点、提交 Claim 进详情列表）。
5. Canon 提案列表显示 pending，合并/拒绝后从列表消失。
6. 全部元素直角、无荧光色、无圆角、无半透明卡片；通过既有设计语言检查。
7. 新增 GUI 用例通过；`npm test` 全绿；a-library/b-record 不回归。

## 7. 明确不做

- 不引入 d3 等可视化依赖。
- 不做拖拽布局、缩放平移画布（后续按真实需要再加）。
- 不做 Claim 的 supersede 编辑界面（只读时间线维度）。
