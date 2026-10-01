# M2 迭代计划：语义叙事与语音边界

> 文档性质：实施计划与验收规范，不记录当前完成进度。当前进度只以根目录 [`STATUS.md`](../../STATUS.md) 为准。
> 开发边界：只在本机及可信内部局域网开发、自测和运行，不接入外部语音服务。

## 1. 目标

将一条叙事 Event 从不可区分的正文升级为有序语义片段，使玩家能快速辨认内容成分，并为后续 STT、TTS、语义段流式和角色打断提供稳定边界。

五类基础片段：

| 类型 | 含义 | 默认语音通道 |
|---|---|---|
| `environment` | 光线、天气、声音、空间等可感知环境 | Narrator |
| `story` | 剧情推进、转折和悬念状态 | Narrator |
| `fact` | 已确认且需要稳定记住的关键事实 | Narrator |
| `action` | 角色姿态、移动、操作和可观察行为 | Narrator |
| `dialogue` | 玩家或角色实际说出口的台词 | Speaker |

## 2. 核心约束

1. 语义片段由 DM/Narrator/Character 输出契约明确给出，前端不通过引号或关键词猜测。
2. Event 的纯文本 `content` 继续存在，用于检索、回退和无障碍读取；结构化片段不得隐藏正文之外的信息。
3. 分隔符是数组节点边界，不向正文插入控制字符、私有 Unicode、XML 标签或可见分隔线。
4. 每个片段具有稳定 `id / kind / content / speechMode`，同一 Event 内 ID 唯一。
5. 未识别、畸形或旧版片段整体降级为单片段，不渲染部分可疑结构。
6. 颜色只作用于文字本身；机器可读类型、悬停说明和无障碍名称提供非颜色语义。
7. 语义类型不改变 Event ACL。秘密过滤必须先于片段投影和语音选择。
8. TTS 只消费玩家已获授权的 Delivery Projection；STT 原文同样先成为当前输入，不直接写入正式 Event。

## 3. 契约

```ts
interface SemanticSegment {
  id: string;
  kind: "environment" | "story" | "fact" | "action" | "dialogue";
  content: string;
  speechMode: "narrator" | "speaker" | "none";
}

interface SemanticPresentationV1 {
  schemaVersion: 1;
  segments: SemanticSegment[];
}
```

语音系统未来按片段逐项排队：

```text
authorized Event
→ ordered SemanticSegment[]
→ speechMode 选择旁白声线 / 角色声线 / 跳过播报
→ 每片段独立音频单元
→ 片段边界允许暂停、取消、插话和重新合成
```

## 4. 批次划分

### 批次 A：结构化呈现

- 冻结 Semantic Presentation v1；
- Fake Turn 产出五类片段；
- PostgreSQL Event 原子保存片段元数据；
- Delivery Projection 校验并投影片段；
- 时间线保持原有纸面与排版，只给五类句段施加克制的文字色；
- DOM 保留机器可读的片段类型、语音通道和边界属性；
- 旧 Event 单片段回退。

### 批次 B：DM/Narrator 原生输出

- 将 Semantic Presentation 纳入 DM 输出完整性校验；
- Narrator 只生成可公开感知的环境、剧情和事实片段；
- Character Runner 生成动作与台词，DM 只检查而不代写立场；
- 拒绝内容缺失、片段重复、正文与片段不一致或工具链未闭合的候选。

### 批次 C：本地语音适配

- 定义本地 `SpeechInput / SpeechChunk / SpeechReceipt` 端口；
- 支持按片段播放、暂停、跳过、取消和断线恢复；
- Speaker Voice 只绑定 CharacterInstance，不绑定角色名称字符串；
- STT 转写作为未提交玩家草稿，用户确认后再进入 Command；
- 首个实现只接本机 Provider，不把项目内容发送到外部服务。

### 批次 D：语义流与打断

- 完整语义句/短段到达后再显示和排入 TTS；
- 候选片段与正式 Event 隔离；
- 打断只取消尚未提交的播放/候选，不修改已经提交的事实；
- DM 依据插话预算和强沉默偏置决定是否激活其他角色。

## 5. 批次 A 验收标准

- 五类片段都能通过同一强类型契约端到端到达界面；
- 每类具有独立但低饱和的文字色，不增加片段底色、边框或可见分隔符；
- 正文中不存在为语音而加入的可见或私有分隔字符；
- 语音消费者可仅凭结构化属性确定顺序、边界和发声通道；
- 旧 Event、乐观玩家输入与 SSE 重放均有合法片段；
- 畸形片段不会导致整条 Event 消失；
- API 不暴露候选、Prompt、Manifest 或未经授权片段；
- 类型检查、真实 PostgreSQL、前端契约、构建与渲染回归全部通过。
