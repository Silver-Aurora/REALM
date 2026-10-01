# 多供应商模型设置与 OpenRouter 费率

> 状态：已实现并通过 focused/core 回归
> 范围：REALM 本机模型设置、OpenAI-compatible 网关、OpenRouter Models API

## 1. 目标

REALM 的模型配置不再只有一个供应商文件。每个供应商拥有独立 profile：

- endpoint 与 SSRF 白名单；
- API key（仅服务端本机配置保存）；
- selected model、thinking、timeout；
- 默认输出长度预算 `maxTokens`；调用方显式预算优先于 profile；
- 最近一次模型目录与费率快照。

`activeProviderId` 决定新的回合、创世、可见性判断、场景结晶和语义审查使用哪个 profile。
保存一个 profile 会将它设为 active；未保存的 UI 切换不改变运行时。

## 2. 持久化与迁移

新格式：

```text
.local/settings/model-providers.json  (0600, gitignored)
  schemaVersion: 2
  activeProviderId
  providers.lmstudio
  providers.openrouter
```

旧格式 `.local/settings/model-provider.json` 仍可读取。首次保存新 profile 时迁移为多 profile 文档；旧文件不被覆盖。
缺失的供应商 profile 从注册目录和环境变量生成默认值。

公开 API / 浏览器永远只得到：

- `apiKeyConfigured`；
- `apiKeyHint`（最后四位）；
- 不包含 `apiKey` 字段。

## 3. 供应商边界

### LM Studio

- 默认 `http://127.0.0.1:1234/v1`（loopback）；可改为用户自己的本机/LAN
  私有地址（http/https 均可，路径必须含 `/v1`）；
- 本地服务允许空 key；
- 继续使用既有宽松 `json_schema` 结构化输出适配和 1024 token 最低输出预算；
- thinking 选项不伪装转发。

### OpenRouter

- 只允许 `https://openrouter.ai/api/v1`；
- 必须配置 API key；
- 使用 `/models?output_modalities=text&limit=1000` 拉取文本模型；
- 使用 `/chat/completions`、Bearer 认证、`json_object`、tools；
- 连接探针使用 256 token 上限。OpenRouter 某些 reasoning 模型在 32 token 上限下会返回空正文并以 `length` 结束，HTTP 200 不代表探针成功。

端点验证仍按 provider 注册的 host、协议、端口和 path fail-closed，不接受用户输入任意代理 URL。

## 4. 费率语义

OpenRouter Models API 返回的 `pricing.prompt` / `pricing.completion` 是 USD per token。
REALM 将其保存在 `ModelPricing`，设置页换算为 USD per 1M token：

```text
perMillion = perToken * 1_000_000
```

当输入和输出价格同时为 `0` 时显示“当前免费”，并在模型卡上标记 `FREE`。
这只是最近一次目录响应的价格快照，不承诺永久免费；OpenRouter 的灰测、路由、限额和模型可用性可能变化。

## 5. 输出长度预算

设置页的「返回长度预算 / `max_tokens`」按 profile 保存，可选 512、1024、2048、4096、8192、16384。
默认值为 2048。单次调用显式传入 `maxTokens` 时覆盖 profile；未显式传入时由网关使用 profile 值。
LM Studio 仍保留 1024 token 的适配层最低值，避免本地 reasoning 模型在小预算下返回空正文。

Ox Alpha 的真实复核显示，当前 Stealth provider 拒绝只有 system message 的请求；OpenRouter 网关在
system-only 请求后补一条无业务内容的最小 user turn，保留原始业务 prompt 并使该模型的 DM JSON 路径可达。
目录标记 `structured_outputs=false` 的模型会在设置页显示 `无 JSON`，不能把目录能力标记误当成完整 REALM 兼容性。

## 6. 模型目录浏览

模型卡只渲染当前页，每页 20 项；设置页提供按名称、ID、作者检索，以及“仅看免费”筛选。
分页、检索和筛选都是浏览器侧操作，不会重复请求供应商；只有点击“连接端点并发现模型”才会重新拉取目录与费率快照。

模型卡还展示：

- context length；
- tools 支持；
- structured outputs 支持；
- 当前输入/输出价格。

发现时优先选同时支持 tools 与 structured outputs 的免费模型；若没有，再退到支持 tools 的免费模型，最后才选择目录第一项。

## 7. 官方依据

- OpenRouter Models API：<https://openrouter.ai/docs/api-reference/models/get-models>
- OpenRouter 模型目录与 pricing schema：<https://openrouter.ai/docs/models>
- OpenRouter Chat Completions、tools、structured outputs：<https://openrouter.ai/docs/api-reference/overview>

## 8. 验收

- `tests/model-provider-config.test.ts`：目录、旧格式迁移、多 profile 保存、公开密钥脱敏、费率解析、OpenRouter 请求体；
- `tests/gui/f-settings.spec.ts`：双供应商 UI、免费标记、费率展示、真实连接探针；
- `npm run test:core`；
- `npm run test:application`；
- `npm run test:frontend-contracts`；
- `npm run lint`；
- `npx tsc --noEmit`。
