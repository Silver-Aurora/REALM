# 世界文风系统实施规范

> 文档性质：实施规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 视觉约束一律以 [`../design/UI-DESIGN.md`](../design/UI-DESIGN.md)（纸墨纪事）为准。

## 1. 定位

文风（World Style）是世界级设定：每个世界独立选择，影响**所有 AI 生成文本**与**世界内固定文案**。界面层文案（登录页、按钮、面板标题）不在本批范围——那是批次 Q 的 i18n 范围。

## 2. 枚举与存储

- 内部 key（稳定不变）：`modern`（现代）/ `classical`（古风）/ `western_fantasy`（西幻）/ `anime`（二次元）。
- 存储：`worlds.settings.style`（jsonb 键，不改表结构，无需新迁移；0014 已授 realm_runtime `UPDATE worlds.settings`）。
- 缺省规则：缺失/未知 style 键一律 fail-closed 到 `modern`；demo 世界（烬海诸国）由 demo-seed 显式写入 `classical`。
- 显示名与全部文案按 key 化模板组织（`worldStyleText(key, style, params?)`），当前仅中文（`zh-CN`）；结构上等价于 `TEMPLATES[key][language][style]`，批次 Q 只加语言维度，不改 key。

## 3. 风格描述数据（prompt 注入用）

每种风格一组结构化约束（非单个名词），字段固定：

| 字段 | 含义 |
|---|---|
| `label` | 中文显示名 |
| `tone` | 语调约束（一句话） |
| `diction` | 用词约束（避免/偏好） |
| `imagery` | 意象约束（与环境描写相关） |
| `sample` | 一句示范语（供模型锚定） |

注入方式：回合管线（DM/旁白/角色/复核/可见性评估）、设定结晶提取与裁决、司卷问答代笔候选、下一步对话提案的系统 prompt 统一附加 `「文风要求」+ describeWorldStyle(style)` 文本块。

## 4. 世界内固定文案 · 模板 key 清单（× 4 风格）

| key | 用途 | 参数 |
|---|---|---|
| `guided.step.world-name.question` | 司卷问答·世界之名提问 | — |
| `guided.step.era.question` | 纪元基调提问 | — |
| `guided.step.style.question` | 文风步提问 | — |
| `guided.step.summary.question` | 世界底色提问 | — |
| `guided.step.story.question` | 故事开篇提问 | — |
| `guided.step.player-role.question` | 你的身份提问 | — |
| `guided.step.companions.question` | 同行之人提问 | — |
| `guided.step.scene.question` | 初始场景提问 | — |
| `guided.step.review.question` | 合卷总览提问 | — |
| `guided.style.modern` 等 4 键 | 文风步选项显示名 | — |
| `action.observe.description.scenic` | 观察四周·有场景 | `{location, weather}` |
| `action.observe.description.blank` | 观察四周·留白兜底 | — |
| `action.observe.suggested.scenic` | 观察四周建议语·有场景 | `{location}` |
| `action.observe.suggested.blank` | 观察四周建议语·留白 | — |
| `scene.crystallize.location` 等 5 键 | 场景定格事件分句（地点/天气/局势/时间/目标） | `{value}` |
| `scene.crystallize.prefix` | 场景定格事件前缀（「场景定格 —」） | — |
| `scene.crystallize.speaker` | 结晶事件说话者名（「界核」） | — |
| `timeline.empty.title` | 空记录卷首标题（「长卷初展，诸事未定」） | — |
| `timeline.empty.hint` | 空记录卷首副行 | — |

断言要求：每个 key 四风格齐备、无空串（Core 契约测试强制）。

## 5. 司卷问答文风步

- 位置：纪元基调之后、世界底色之前。理由：文风先于底色与后续步骤落定，其后所有 AI 代笔候选与提问语才能立即按所选风格产出；放在纪元之后保持「先时空、后笔调」的叙述顺序。
- 四个纸签选项（显示名走 `guided.style.*` 模板），点选即定并浮现已定之卷条目；可留白跳过（跳过 = 缺省 modern，不显示条目）。

## 6. 世界设置入口

- 世界库面板每个世界卡提供「文风」切换（四枚纸签），点击即保存（`POST /api/library` 扩展 `kind: "world-style"`），保存即生效：回合管线每回合从运行时世界快照读设定，无缓存失效问题。

## 7. fail-closed 矩阵

| 环节 | 失败形态 | 行为 |
|---|---|---|
| 风格解析 | 缺失/未知 style 键 | 一律按 modern |
| 模板查找 | key 缺失或语言缺失 | 回退 zh-CN → modern；仍缺则抛出（契约测试兜底不允许发生） |
| 文风步 | 未点选 | 留白跳过，不写 style 键 |
| 切换保存 | 网络/服务端失败 | notice 提示，UI 保持原风格 |

## 8. 验收标准

1. Core 契约：枚举校验与未知兜底、模板 key × 4 风格全覆盖无空串、风格描述数据字段完整。
2. PG 集成：style 写入/读取、demo seed=classical、缺省 modern。
3. 模型探针：modern 与 anime 各一轮真实回合，语调符合风格，摘录进交付报告。
4. GUI P 组：新世界默认 modern 无古风串场、文风步点选与跳过、设置改风格后行动卡随之变化、demo 世界保持 classical；既有组全量回归。
