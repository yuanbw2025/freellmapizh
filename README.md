<div align="center">

# FreeLLMAPI 中文版 / FreeLLMAPI Chinese Edition

**一个 OpenAI 兼容入口，聚合多个免费/低门槛 LLM Provider，自动路由、自动 fallback、统一密钥管理。**

**One OpenAI-compatible endpoint for multiple free or low-barrier LLM providers, with automatic routing, fallback, and unified key management.**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)

![Fallback chain with per-provider token budget](repo-assets/fallback-chain.png)

</div>

---

## 中文说明

### 项目定位

`freellmapizh` 是一个自托管的 OpenAI 兼容 LLM API 聚合代理。

它把多个模型平台的免费层、试用层或低门槛 API 入口统一到一个本地服务里，对外提供：

- `POST /v1/chat/completions`
- `GET /v1/models`

你可以把 OpenAI SDK、LangChain、LlamaIndex、Continue、Hermes、opencode 或其他 OpenAI-compatible 客户端的 `base_url` 指向这个服务，让请求自动在可用模型和可用 key 之间路由。

这个项目适合：

- 个人实验
- 学习 OpenAI-compatible API 代理如何实现
- 聚合多个免费/试用模型做低成本测试
- 给本地工具、Agent、IDE 插件提供一个统一模型入口
- 研究 fallback、rate limit、key health、provider adapter 的工程实现

不适合：

- 正式生产服务
- 对外多用户转售
- 稳定 SLA 场景
- 绕过上游平台服务条款

### 这次 README 审查结论

原英文 README 的主体说明是有价值的，但和当前代码相比不够全面，主要问题如下：

1. **Provider 列表不完整。**
   README 仍写成 12 个 provider，但当前代码和 UI 已包含 Google、Groq、Cerebras、SambaNova、NVIDIA、Mistral、OpenRouter、GitHub Models、Cohere、Cloudflare、Zhipu、Ollama、Kilo、Pollinations、LLM7、HuggingFace 等 16 个平台类型。

2. **“Vision / multimodal inputs not supported” 表述需要更精确。**
   当前代码会接受 OpenAI 的多模态 content array envelope，并把文本块提取出来；非文本块会被丢弃或转为文本处理。因此准确说法是：**接口能接收部分多模态 envelope，但实际代理仍是文本聊天，不提供真正视觉理解能力。**

3. **路由行为比 README 描述更强。**
   当前代码不仅做基础 fallback，还包含：
   - `auto` 虚拟模型
   - 显式模型 pinning
   - sticky session
   - 每次 429 后的动态 penalty
   - per-key round-robin
   - 最多 20 次 retry

4. **Dashboard 功能说明可以更细。**
   当前前端包含 Keys、Fallback、Playground、Analytics 四个主页面，支持统一 API key、provider key 健康检查、fallback 拖拽排序、token budget、请求统计和错误分布。

5. **部署边界需要更强调。**
   代码本身是单用户本地代理设计，没有多租户账号体系。README 应更明确地提醒不要把它直接暴露给公网给多人共用。

6. **上游来源需要保留但不应让人误解。**
   本仓库是 `yuanbw2025/freellmapizh`，不是原始 `tashfeenahmed/freellmapi`。中文 README 应说明这是中文整理和后续自用开发版本，同时保留原项目许可与署名。

### 核心能力

- **OpenAI-compatible API**
  - `POST /v1/chat/completions`
  - `GET /v1/models`
  - 支持 OpenAI SDK 和常见兼容客户端。

- **自动路由**
  - `model: "auto"` 或省略 `model` 时，系统自动选择当前可用模型。
  - 也可以显式指定某个模型 ID。
  - 如果指定模型不存在或被禁用，会返回 `model_not_found`。

- **Fallback 链**
  - 模型按 fallback priority 排序。
  - 某个 provider/key 遇到 429、超时、5xx、模型下线等可重试错误时，自动跳到下一个可用模型。
  - 单次请求最多尝试 20 次。

- **动态降权**
  - 某个模型频繁 429 后会被临时 penalty，下沉到 fallback 链后面。
  - 成功请求会逐步降低 penalty。
  - penalty 会随时间衰减。

- **Per-key 限流追踪**
  - 按 `(platform, model, key)` 追踪 RPM、RPD、TPM、TPD。
  - 路由时跳过已超限的 key。
  - 429 后会给对应 model+key 设置短 cooldown。

- **Sticky Session**
  - 多轮对话会尽量保持同一个模型 30 分钟。
  - 避免中途换模型导致上下文风格和推理能力突变。

- **统一 API Key**
  - 客户端只需要使用一个 `freellmapi-...` bearer token。
  - 上游 provider key 不暴露给调用方。

- **Provider Key 加密保存**
  - API key 使用 AES-256-GCM 加密后写入 SQLite。
  - 请求前在内存中解密。

- **健康检查**
  - 可以手动检查单个 key 或全部 key。
  - 后台每 5 分钟自动检查。
  - key 状态包括 `healthy`、`rate_limited`、`invalid`、`error`、`unknown`。
  - 连续确认无效的 key 会被自动禁用。

- **Dashboard**
  - Keys：管理 provider key 和统一 API key。
  - Fallback：拖拽排序模型 fallback 链，查看 token budget、动态 penalty、key 数量。
  - Playground：通过代理实际发起 chat completion，查看路由结果。
  - Analytics：查看请求量、成功率、token、延迟、provider/model 分布和错误。

- **Tool Calling**
  - 支持 OpenAI 风格 `tools` / `tool_choice`。
  - assistant `tool_calls` 与 tool role follow-up 可以往返代理。
  - Gemini 会转换为 Google function declarations / function response 格式，再转换回 OpenAI 兼容响应。

- **Streaming**
  - `stream: true` 时返回 Server-Sent Events。
  - 非 streaming 返回普通 JSON。
  - 流式响应开始前的错误仍可 fallback；流式开始后的错误会发送 error SSE frame。

### 当前支持的平台

当前代码里注册的平台包括：

| Platform | 说明 |
|---|---|
| Google | Gemini API，使用独立适配器 |
| Groq | OpenAI-compatible |
| Cerebras | OpenAI-compatible |
| SambaNova | OpenAI-compatible |
| NVIDIA NIM | OpenAI-compatible，默认部分模型可能禁用或受 trial 限制 |
| Mistral | OpenAI-compatible |
| OpenRouter | OpenAI-compatible 聚合平台 |
| GitHub Models | OpenAI-compatible |
| Cohere | Cohere compatibility endpoint |
| Cloudflare Workers AI | 使用 `account_id:token` 形式 |
| Zhipu / Z.ai | OpenAI-compatible |
| Ollama Cloud | OpenAI-compatible，部分免费模型延迟较高 |
| Kilo Gateway | OpenAI-compatible 网关 |
| Pollinations | OpenAI-compatible，路径带 `/openai/v1` |
| LLM7 | OpenAI-compatible 聚合入口 |
| HuggingFace Router | HuggingFace Inference Providers router |

注意：平台是否免费、额度多少、是否需要信用卡、模型是否仍可用，都可能随时间变化。项目内置的 catalog 是工程默认值，不是稳定承诺。

### 当前不支持或不完整支持

- **Embeddings**：未实现 `/v1/embeddings`
- **Images**：未实现 `/v1/images/*`
- **Audio / Speech**：未实现 `/v1/audio/*`
- **Moderation**：未实现 `/v1/moderations`
- **Legacy Completions**：未实现 `/v1/completions`
- **真正视觉理解**：可以接收部分 content array，但非文本块不会被作为真实图像输入处理
- **多租户账号体系**：没有用户注册、权限隔离、计费系统
- **生产 SLA**：免费层和试用层不适合生产可靠性承诺

### 项目结构

```text
freellmapizh/
├── client/                  # React + Vite dashboard
│   └── src/pages/           # Keys / Fallback / Playground / Analytics
├── server/                  # Express proxy server
│   └── src/
│       ├── app.ts           # Express app, API routes, static dashboard
│       ├── index.ts         # server entry
│       ├── db/              # SQLite schema, seed, migrations
│       ├── providers/       # provider adapters
│       ├── routes/          # /api/* and /v1/* routes
│       ├── services/        # router, health, ratelimit
│       └── lib/             # encryption, content helpers
├── shared/                  # shared TypeScript types
├── repo-assets/             # README screenshots
├── docs/                    # docs / OG assets
├── .env.example             # environment example
├── package.json             # npm workspaces
└── README.md
```

### 快速开始

要求：

- Node.js 20+
- npm

```bash
git clone https://github.com/yuanbw2025/freellmapizh.git
cd freellmapizh
npm install
cp .env.example .env
```

生成 32 字节加密密钥：

```bash
node -e "console.log('ENCRYPTION_KEY=' + require('crypto').randomBytes(32).toString('hex'))" >> .env
```

启动开发环境：

```bash
npm run dev
```

默认端口：

- Server: `http://localhost:3001`
- Dashboard: `http://localhost:5173`
- Proxy endpoint: `http://localhost:3001/v1/chat/completions`

`ENCRYPTION_KEY` 是启动必需项。只有在 `DEV_MODE=true` 且 `NODE_ENV` 不是 `production` 时，server 才会退回使用数据库里保存的开发密钥；不要用这个开发 fallback 保存真实 provider key。

打开 Dashboard 后：

1. 到 **Keys** 页面添加 provider key。
2. 复制页面顶部的统一 API key。
3. 到 **Fallback** 页面调整模型优先级。
4. 到 **Playground** 页面测试请求。

### 生产构建

```bash
npm run build
npm run start -w server
```

构建后 server 会同时提供：

- API routes
- OpenAI-compatible `/v1/*`
- 已构建的 dashboard 静态页面

默认监听：

```text
http://0.0.0.0:3001
```

### 环境变量

| 变量 | 说明 |
|---|---|
| `ENCRYPTION_KEY` | 必填，64 位 hex 字符串，用于 AES-256-GCM 加密 provider keys |
| `PORT` | server 端口，默认 `3001` |
| `DASHBOARD_ORIGINS` | 额外允许的 dashboard CORS origin，逗号分隔 |
| `VITE_BASE` | client 构建 base path，默认 `/` |

### API 使用示例

#### Python

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:3001/v1",
    api_key="freellmapi-your-unified-key",
)

resp = client.chat.completions.create(
    model="auto",
    messages=[
        {"role": "user", "content": "用一句话解释什么是 fallback router"}
    ],
)

print(resp.choices[0].message.content)
```

#### curl

```bash
curl http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer freellmapi-your-unified-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "auto",
    "messages": [
      { "role": "user", "content": "hi" }
    ]
  }'
```

#### Streaming

```python
stream = client.chat.completions.create(
    model="auto",
    messages=[{"role": "user", "content": "写一首关于 SQLite 的短诗"}],
    stream=True,
)

for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

#### Tool Calling

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get current weather for a city.",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    },
}]

first = client.chat.completions.create(
    model="auto",
    messages=[{"role": "user", "content": "What's the weather in Shanghai?"}],
    tools=tools,
    tool_choice="required",
)

call = first.choices[0].message.tool_calls[0]

final = client.chat.completions.create(
    model="auto",
    messages=[
        {"role": "user", "content": "What's the weather in Shanghai?"},
        first.choices[0].message,
        {"role": "tool", "tool_call_id": call.id, "content": "{\"temp_c\": 26, \"cond\": \"cloudy\"}"},
    ],
    tools=tools,
)

print(final.choices[0].message.content)
```

### 响应头

每个成功响应会包含：

```text
X-Routed-Via: <platform>/<model>
```

发生 fallback 后还会包含：

```text
X-Fallback-Attempts: <number>
```

### 截图

#### Keys

![Keys page](repo-assets/keys.png)

#### Fallback Chain

![Fallback chain](repo-assets/fallback-chain.png)

#### Playground

![Playground page](repo-assets/playground.png)

#### Analytics

![Analytics page](repo-assets/analytics.png)

### 使用边界和风险

- 免费层会变化，模型和额度随时可能调整。
- 免费层不是 SLA，不能按生产基础设施对待。
- 晚些时候高优先级模型可能达到额度，路由会降级到更弱模型。
- 不同 provider 延迟差异很大。
- 不要把 provider key 暴露给前端或第三方。
- 不要把这个代理公开给多人共用。
- 不要拿免费层做付费产品后端。
- 每个上游平台的 ToS 仍然适用。

### 开发

```bash
npm install
npm run dev      # server on :3001, dashboard on :5173, both with HMR
npm test         # server vitest; also runs client tests if the workspace adds them
npm run build    # compile server and dashboard
```

常见开发位置：

- `server/src/providers/`：新增 provider adapter
- `server/src/db/index.ts`：新增模型 catalog 或 migration
- `server/src/services/router.ts`：路由策略
- `server/src/services/ratelimit.ts`：限流账本
- `server/src/services/health.ts`：key 健康检查
- `client/src/pages/`：Dashboard 页面
- `shared/types.ts`：前后端共享类型

### 上游来源

本仓库基于 FreeLLMAPI 项目继续整理和中文化，用于个人学习和后续开发。

原项目 README 中的贡献者、服务条款审查和免责声明有参考价值，但本 README 按当前仓库代码重新整理，避免过时平台列表和能力描述误导使用者。

### License

[MIT](./LICENSE)

---

## English

### What is this?

`freellmapizh` is a self-hosted OpenAI-compatible LLM proxy.

It aggregates multiple free-tier, trial-tier, or low-barrier LLM providers behind one local API:

- `POST /v1/chat/completions`
- `GET /v1/models`

Any OpenAI-compatible client can point its `base_url` to this server and let the router select a working model/key automatically.

This project is best used for:

- personal experimentation
- learning how an OpenAI-compatible proxy works
- low-cost testing across free or trial model providers
- giving local tools, IDE agents, and automation scripts one unified LLM endpoint
- studying fallback routing, rate-limit tracking, key health, and provider adapters

It is not designed for:

- production workloads
- public multi-user resale
- SLA-sensitive systems
- bypassing upstream provider terms

### README completeness review

The previous English README was useful, but no longer fully matched the current codebase:

1. The provider list was incomplete. The code currently includes Google, Groq, Cerebras, SambaNova, NVIDIA, Mistral, OpenRouter, GitHub Models, Cohere, Cloudflare, Zhipu, Ollama, Kilo, Pollinations, LLM7, and HuggingFace.
2. Vision/multimodal support needed a more precise statement. The proxy accepts OpenAI-style content arrays, but non-text blocks are not treated as real image/audio inputs.
3. Routing behavior is richer than previously documented: `auto` model, explicit model pinning, sticky sessions, dynamic 429 penalties, per-key round-robin, cooldowns, and up to 20 retries.
4. Dashboard features needed more detail: Keys, Fallback, Playground, Analytics, unified API key, health checks, token budget, and error views.
5. Deployment boundaries needed to be clearer. This remains a local-first, single-user proxy.

### Core features

- OpenAI-compatible `/v1/chat/completions` and `/v1/models`
- Streaming and non-streaming responses
- OpenAI-style tool calling
- Automatic fallback across providers and keys
- `model: "auto"` virtual model
- Explicit model pinning when a model ID is provided
- Sticky sessions for multi-turn conversations
- Per-key RPM/RPD/TPM/TPD tracking
- Cooldowns and dynamic penalties after rate limits
- Encrypted provider key storage using AES-256-GCM
- One unified `freellmapi-...` API key for clients
- Health checks for provider keys
- React/Vite dashboard with Keys, Fallback, Playground, and Analytics pages
- SQLite storage
- Node.js 20+ runtime

### Supported platforms in the current code

| Platform | Notes |
|---|---|
| Google | Gemini API, custom adapter |
| Groq | OpenAI-compatible |
| Cerebras | OpenAI-compatible |
| SambaNova | OpenAI-compatible |
| NVIDIA NIM | OpenAI-compatible; some models may be trial-limited or disabled |
| Mistral | OpenAI-compatible |
| OpenRouter | OpenAI-compatible aggregator |
| GitHub Models | OpenAI-compatible |
| Cohere | Cohere compatibility endpoint |
| Cloudflare Workers AI | Uses `account_id:token` |
| Zhipu / Z.ai | OpenAI-compatible |
| Ollama Cloud | OpenAI-compatible; some free models can be slow |
| Kilo Gateway | OpenAI-compatible gateway |
| Pollinations | OpenAI-compatible endpoint under `/openai/v1` |
| LLM7 | OpenAI-compatible gateway |
| HuggingFace Router | HuggingFace Inference Providers router |

Provider limits, free tiers, and model availability can change without notice.

### Not supported or only partially supported

- `/v1/embeddings`
- `/v1/images/*`
- `/v1/audio/*`
- `/v1/moderations`
- legacy `/v1/completions`
- real vision understanding
- multi-tenant user accounts
- production SLA

### Quick start

```bash
git clone https://github.com/yuanbw2025/freellmapizh.git
cd freellmapizh
npm install
cp .env.example .env
node -e "console.log('ENCRYPTION_KEY=' + require('crypto').randomBytes(32).toString('hex'))" >> .env
npm run dev
```

Default URLs:

- Server: `http://localhost:3001`
- Dashboard: `http://localhost:5173`
- Proxy endpoint: `http://localhost:3001/v1/chat/completions`

`ENCRYPTION_KEY` is required for startup. The server only falls back to a database-stored development key when `DEV_MODE=true` and `NODE_ENV` is not `production`; do not use that fallback with real provider keys.

### Production build

```bash
npm run build
npm run start -w server
```

The server serves both the API and the built dashboard on port `3001` by default.

### Environment variables

| Variable | Description |
|---|---|
| `ENCRYPTION_KEY` | Required, 64 hex chars, used for AES-256-GCM provider key encryption |
| `PORT` | Server port, default `3001` |
| `DASHBOARD_ORIGINS` | Additional allowed dashboard CORS origins, comma-separated |
| `VITE_BASE` | Client build base path, default `/` |

### API example

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:3001/v1",
    api_key="freellmapi-your-unified-key",
)

resp = client.chat.completions.create(
    model="auto",
    messages=[{"role": "user", "content": "Explain fallback routing in one sentence."}],
)

print(resp.choices[0].message.content)
```

### Development

```bash
npm install
npm run dev
npm test
npm run build
```

Common files:

- `server/src/providers/` for provider adapters
- `server/src/db/index.ts` for model catalog and migrations
- `server/src/services/router.ts` for routing logic
- `server/src/services/ratelimit.ts` for rate-limit tracking
- `server/src/services/health.ts` for key health checks
- `client/src/pages/` for dashboard pages
- `shared/types.ts` for shared frontend/backend types

### License

[MIT](./LICENSE)
