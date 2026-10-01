# REALM Prompt System v2

全部 REALM 生产 system prompt 的统一契约。目标：prompt 格式化、精简、English-only；动态资料走 user context；自然语言链有 anti-AI 约束；结构化输出有容错与恰好一次修复。

## 一、分层

- **system message = 静态 English 策略**：身份、权限边界、输出契约、voice 规则、文风（`stylePromptBlock`）、变量化语言规则（`outputLanguageRule`）。绝不包含动态世界资料、动态角色名、中文说明文字。
- **user message = 动态 context block**：`contextBlock("[Label]", value)` + `composeContext(...)`。世界/场景/角色/公开事实/权限/玩家输入全部从这里进入；动态值保持原始语言，不做机器翻译；标签为英文。
- **canon 权威语义**：canon 原文（动态）进 user 的 `[World and scene]` 块；system 的静态规则 `CANON_BINDING_RULE` 声明其约束力。
- **角色身份**：`[Character]` context block（characterInstanceId/displayName 来自调用时数据）；system 只说「the character identified in the [Character] context block」。

## 二、Prompt Kit（`modules/inference/prompt-kit.ts`）

- `PROMPT_KIT_VERSION = "prompt-kit/v1"`：kit 自身的稳定版本常量。注意：各链已有的 ledger 版本常量（`scene-crystallization/v1`、`semantic-conflict-v1`）是**证据台账 schema 版本**，本次内容迁移不 bump，避免打断既有审计行。
- English-only 静态片段：身份/规则行由各链组合；共享片段 `CONTEXT_MATERIAL_RULE`（防注入）、`CANON_BINDING_RULE`、`NO_UNSUPPORTED_FACTS_RULE`、`NO_RULE_MECHANICS_RULE`、`CHARACTER_AUTHORITY_RULE`（model-powered 内部常量）。
- `NATURAL_VOICE_RULES`：anti-AI 规则，**只**注入自然语言生成链（Narrator、角色 dialogue/reaction、Dynamic Discovery、First Night、Genesis draft/chat/suggestions）。分类器/门禁器（visibility、presence gate、DM plan、DM review、scene extraction/adjudication、semantic conflict、连接探针）只用 `CONCISE_RATIONALE_RULE` 或自带的短理由约束。
- `jsonOutputInstruction(fields)`：schema 形状、长度上限、枚举值全部来自调用方传入的真实常量（与 normalizer 同源），prompt 文本不再维护第二份数字。
- `outputLanguageRule(configuredLanguage?)`：`Write in the language of the latest player input.`；无输入时用世界配置语言（可用时传入实际值，不可用时 fail-closed 不臆造）。first-night 锚定既有卷首旁白 + 配置语言。旧版四处语言规则（枚举中英日 / 枚举中英 / 锚定+默认简体中文 / 无规则）全部统一到这一变量化规则。

## 三、English 文风描述

`modules/style/world-style.ts` 新增 `WORLD_STYLE_PROMPT_PROFILES` + `stylePromptBlock(style)`（English，无 sample 示范句）。中文 `describeWorldStyle`/`WORLD_STYLE_PROFILES`/`TEMPLATES` 是 UI 与世界内文案，保持不动。

## 四、容错与恰好一次 repair（`modules/inference/structured-output.ts`）

状态机：`request → tolerantParse → normalize → (invalid ? 恰好一次 repair : 返回) → 调用方既有 fail-closed`。

- `tolerantParseJsonObject`：容忍 UTF-8 BOM、外围空白、标准 ```json fenced block、JSON 前后简短前言/尾注（只提取唯一完整 object）、trailing comma（字符串感知）。拒绝数组/标量/多 object/截断内容；绝不 eval、绝不执行模型文本。
- `requestStructuredObject`：parse 失败（unparseable）或 normalizer 拒绝（schema）→ 追加一条 English repair user 消息（`buildRepairUserMessage`：错误类别 + 目标 schema + "Return the corrected JSON object only"，**不回显模型原文**）→ 再解析一次 → 仍失败返回 null。provider/network/timeout 错误直接上抛，不伪装成格式 repair。日志只记 code/类别/长度/attempt（脱敏）。
- **分层**：orchestration 的 `modelCall` 不再盲重试 JSON 格式码（原 `RETRYABLE_FORMAT_CODES` 移除 6 个 JSON code），只保留 tools 契约失败（`CHARACTER_ACTION_INVALID`，JSON repair 不适用）的就地重试一次。genesis-chat 既有三级空响应退化（原样 → 关思考 → 备选模型）保留，格式 repair 接在其后。narrator 的越权重写与 DM review 的三轮递进是**语义级**复核，与格式 repair 互不叠加。
- stream preview 保持只读内存广播；repair/partial JSON 不写入 durable event；tools 请求不走 JSON repair。

## 五、迁移清单

| 链 | 文件 | 备注 |
|---|---|---|
| dynamic discovery | modules/orchestration/model-powered.ts | NLG（有 voice）；上限同 `DISCOVERY_FIELD_LIMITS` |
| visibility assessor | 同上 | 分类器；reason ≤ `VISIBILITY_REASON_LIMIT` |
| presence gate | 同上 | 分类器；triggerKind 枚举同 `PRESENCE_TRIGGER_KINDS` |
| DM plan | 同上 | 分类器；`PLAN_GOAL_LIMIT/PLAN_ID_LIMIT` |
| DM candidate review | 同上 | 三轮语义复核保留，每轮内部一次格式 repair |
| character propose | 同上 | tools 链，不走 JSON repair |
| character presence/ordinary/action react | 同上 | NLG；`REACTION_*`/`ACTION_REACT_ACTION_LIMIT` |
| narrator | 同上 | NLG；suggestions 上限同 `NEXT_SUGGESTION_*`；forbiddenNames 为调用时数据 |
| Actor Tool descriptions | 同上 | English；技能列表由持有数据动态生成 |
| scene extraction/adjudication | modules/application/scene-crystallization.ts | 分类器；`FIELD_LIMITS` 同源；`{}` 是合法「无变化」不触发 repair |
| first-night | modules/application/first-night.ts | NLG；system-only 拆为 system policy + user context |
| world-genesis / genesis-chat / genesis-suggestions | modules/application/ | NLG；移除写死排除名（塞娜/弥洛/洛川）；语言规则统一 |
| semantic conflict | modules/worldline/semantic-conflict.ts | 分类器；枚举/上限同源；fallback 语义不变 |
| 连接探针 | modules/application/model-settings-service.ts | English；容错解析 |
| OpenRouter system-only 兜底 user 轮 | modules/inference/openai-compatible-gateway.ts | English |

## 六、移除的旧固定约束

- 三处写死排除名「塞娜/弥洛/洛川」（无数据依据；未来如确需排除须由调用方传 `reservedNames`）。
- 四种互相矛盾的 output-language 规则（含「所有文字使用中文」与「跟随原文」并存的直接矛盾）。
- prompt 文本里与 normalizer 脱节的第二份数字（如 narrator 40 字/2–3 条、extraction 20/40 字 vs 真实 60/120、first-night 60/150 字 vs 真实 120/200）。
- `buildSceneCanon`（动态中文世界资料塞 system role）。

## 七、审计与边界

- `tests/prompt-system-audit.test.ts`：运行时驱动每条链，断言 system English-only、动态资料只在 user、voice 规则只在自然语言链、上限与 normalizer 同源、旧中文惯用语零残留。
- Tavern 用户导入的 `system_prompt`/`post_history_instructions` 落库后无任何读取方（实锤：全仓 grep 仅命中 parser 与落库行），不属于本审计；`tavern-import-service.ts` 无 prompt 构造。
- `isInternalInstructionText` 增加英文元话语过滤（`as an ai`/`language model`/`system prompt`），中文清单不动。
- 保留边界：publicFacts 不作角色对白 grounding、Narrator 不替角色说话、restricted/private 不降级、模型不决定权限/骰点/历史事实、append-only evidence 语义、provider 适配（LM Studio 原生/chat completions、OpenRouter）与 timeout/thinking/stream 语义。
