# 大模型 API 协议及兼容性差异

本文梳理 `@earendil-works/pi-ai` 当前支持的大模型 API 协议、线上数据结构、厂商兼容性差异，以及 pi 对这些协议的映射方式。分析对象以 [`KnownApi`](../src/types.ts#L15-L24) 和内置 API 注册表为准，不代表上游厂商提供的全部能力。

## 证据范围

本文使用三类证据：

- 厂商官方 API 文档，用于确认公开的请求字段、响应结构和传输约定。
- pi 的协议适配器，用于确认当前代码实际发送、接收和保留的字段。
- pi 的兼容配置，用于记录 OpenAI-compatible 或 Anthropic-compatible 端点之间已知的行为差异。

第三类证据反映兼容端点的实际行为，不应视为对应厂商的稳定公开规范。OpenAI Codex 使用的 ChatGPT backend 也没有完整的公开 API 规范，本文仅描述 pi 当前实现。

## 协议分层

pi 将 `provider` 和 `api` 作为两个独立维度：

- `provider` 表示服务商或路由平台，例如 `openai`、`anthropic`、`openrouter` 和 `github-copilot`。
- `api` 表示线上协议，例如 `openai-responses`、`anthropic-messages` 和 `google-generative-ai`。

多个 provider 可以复用同一个 API 适配器。一个 provider 也可以按模型使用不同协议。GitHub Copilot、OpenCode 和 Cloudflare AI Gateway 都属于后一种情况。

pi 在协议适配器之外使用统一的内部结构：

- [`Context`](../src/types.ts#L439-L443) 保存系统提示、消息历史和工具定义。
- [`Message`](../src/types.ts#L377-L408) 统一用户消息、助手消息和工具结果。
- 助手内容由文本、推理和工具调用三类有序内容块组成。
- [`AssistantMessageEvent`](../src/types.ts#L445-L465) 统一文本、推理和工具调用的流式生命周期。
- [`Usage`](../src/types.ts#L352-L373) 统一输入、输出、缓存、推理 token 和成本。

这套结构是 agent 调用所需语义的公共子集，不是上游协议的完整并集。音频、引用、grounding、安全评分、computer use 和 provider-native 文件对象等能力可能无法无损进入当前内部类型。

## 协议总览

| pi API | 协议形态 | 请求主体 | 流式响应 | 主要使用方 |
| --- | --- | --- | --- | --- |
| `openai-completions` | OpenAI Chat Completions 及兼容方言 | `messages`、`tools` | `choices[].delta`，通常以 `[DONE]` 结束 | OpenRouter、DeepSeek、xAI、Groq、Together、Cerebras、NVIDIA 等 |
| `openai-responses` | OpenAI Responses | `input` item、`instructions`、`tools` | `response.*`、`output_item.*` 及内容 delta | OpenAI，部分 Copilot、OpenCode 和 Cloudflare 模型 |
| `azure-openai-responses` | Azure OpenAI Responses | 与 OpenAI Responses 接近 | 与 OpenAI Responses 接近 | Azure OpenAI |
| `openai-codex-responses` | ChatGPT Codex Responses | Responses item、`instructions` | Responses 事件，经 SSE 或 WebSocket 传输 | OpenAI Codex |
| `anthropic-messages` | Anthropic Messages 及兼容端点 | `messages`、顶层 `system`、内容块 | `message_*`、`content_block_*` | Anthropic、Fireworks、MiniMax、Kimi 等 |
| `bedrock-converse-stream` | Amazon Bedrock ConverseStream | `modelId`、`messages`、`system`、`toolConfig` | AWS EventStream 内容块事件 | Amazon Bedrock |
| `google-generative-ai` | Gemini GenerateContent | `contents[].parts[]`、`config` | `streamGenerateContent` | Google Gemini API |
| `google-vertex` | Vertex GenerateContent | 与 Gemini Content 协议接近 | GenerateContent stream | Google Vertex AI |
| `mistral-conversations` | Mistral Chat Completions | `messages`、`tools`、Mistral 扩展字段 | data-only SSE，以 `[DONE]` 结束 | Mistral |
| `openrouter-images` | OpenRouter 图像生成扩展 | prompt 或多模态输入 | 当前 pi 实现使用非流式响应 | OpenRouter 图像模型 |

内置文本 API 的注册位置见 [`compat.ts`](../src/compat.ts#L172-L182)，图像 API 使用独立的 `generateImages()` 契约。

## 公共消息语义

各协议的字段名称不同，但 agent 调用依赖的主要语义相近。

| 语义 | OpenAI Chat | Responses | Anthropic | Gemini | Bedrock | Mistral |
| --- | --- | --- | --- | --- | --- | --- |
| 系统指令 | `system` 或 `developer` message | `instructions` 或输入 message | 顶层 `system` | `systemInstruction` | 顶层 `system` | system message |
| 普通消息 | role + `content` | typed input/output item | role + content blocks | role + `parts` | role + content blocks | role + `content` |
| 助手角色 | `assistant` | `assistant` message item | `assistant` | `model` | `assistant` | `assistant` |
| 图片 | `image_url` | `input_image` | base64 image source | `inlineData` 或文件数据 | image block | `image_url` |
| 工具声明 | function + JSON Schema | function tool item | `input_schema` | `functionDeclarations` | `toolSpec` | function tool |
| 工具调用 | `tool_calls` | `function_call` | `tool_use` | `functionCall` | `toolUse` | `toolCalls` |
| 工具结果 | tool role message | `function_call_output` | user 消息中的 `tool_result` | `functionResponse` | `toolResult` | tool role message |
| 推理内容 | 兼容端点扩展字段 | `reasoning` item | `thinking` 或 `redacted_thinking` | thought part | `reasoningContent` 或模型扩展 | thinking content 或 prompt mode |
| 缓存控制 | cache key 或兼容方言 | cache key、retention、response chaining | 内容块上的 `cache_control` | cached content | cache point 或模型扩展 | `prompt_cache_key` 和 affinity |

### 工具调用 ID

工具调用 ID 是跨协议重放时最容易失配的字段：

- OpenAI Responses 可能同时返回 `call_id` 和 output item ID。pi 将两者组合后保存。
- Anthropic 接受的 ID 最多 64 个字符，并限制为字母、数字、下划线和连字符。
- 部分 OpenAI Chat 端点限制 ID 长度或字符集。
- Mistral 要求固定长度的字母数字 ID，pi 使用稳定哈希生成兼容值。
- Gemini function call 可能没有 ID，pi 需要补充本地 ID；部分代理模型又要求请求中显式携带 ID。

[`transformMessages()`](../src/api/transform-messages.ts#L59-L220) 会同步转换工具调用及其结果的 ID，避免两者失去关联。

### 推理内容和签名

推理内容不能统一视为普通文本。部分协议要求在后续轮次中回传 opaque 状态：

- Anthropic 使用 thinking signature，并可能返回只有密文的 `redacted_thinking`。
- OpenAI Responses 使用 reasoning item ID 和 encrypted content。
- Gemini 使用 base64 thought signature。
- 部分 OpenAI-compatible 端点使用 `reasoning_details`。

pi 在公共类型中保留 `textSignature`、`thinkingSignature` 和 `thoughtSignature`。这些值只对相同 provider、API 和模型可靠。切换模型时，[`transformMessages()`](../src/api/transform-messages.ts#L92-L148) 会删除不能重放的密文和签名，并将可读推理降级为文本。

## OpenAI Chat Completions

OpenAI Chat Completions 使用 role/message 结构。标准请求通常包含：

```json
{
  "model": "model-id",
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": "..." }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "tool_name",
        "description": "...",
        "parameters": { "type": "object" }
      }
    }
  ],
  "stream": true
}
```

pi 在 [`buildParams()`](../src/api/openai-completions.ts#L540-L680) 中构造该请求，并解析 `choices[0].delta` 中的文本、推理和工具参数增量。

### 兼容端点差异

"OpenAI-compatible" 只表示请求轮廓接近，不保证字段和行为完全一致。pi 的 [`OpenAICompletionsCompat`](../src/types.ts#L467-L518) 记录以下差异：

- 系统指令使用 `developer` 还是 `system` role。
- token 上限字段使用 `max_tokens` 还是 `max_completion_tokens`。
- 是否支持 `store`、`stream_options.include_usage` 和工具 schema 的 `strict`。
- tool result 是否必须携带工具名称。
- tool result 后是否允许直接出现 user message。
- 推理内容是否必须转成普通文本。
- 是否要求在历史 assistant message 中补充空 `reasoning_content`。
- prompt cache 使用 OpenAI 字段、Anthropic `cache_control` 还是 session-affinity header。

reasoning 请求格式的分歧更大：

| 兼容形式 | 请求字段 |
| --- | --- |
| OpenAI 风格 | `reasoning_effort` |
| OpenRouter | `reasoning: { effort }` |
| DeepSeek | `thinking: { type }`，部分端点另接收 `reasoning_effort` |
| Together | `reasoning: { enabled }`，部分模型另接收 effort |
| z.ai | `thinking: { type: "enabled" | "disabled" }` |
| Qwen-compatible | `enable_thinking` |
| Qwen chat template | `chat_template_kwargs.enable_thinking` |
| 通用 chat template | 可配置的 `chat_template_kwargs` |
| string thinking | 顶层 `thinking: string` |

实际分支见 [`openai-completions.ts`](../src/api/openai-completions.ts#L600-L674)。这些兼容规则属于 pi 的运行经验，不能推导为所有同类端点的统一规范。

## OpenAI Responses

Responses API 使用 typed item，而不是只依赖 role/message。请求核心字段包括：

- `input`：字符串或 input item 数组。
- `instructions`：当前响应的系统级指令。
- `tools`：函数和其他工具定义。
- `reasoning`：推理 effort、summary 等设置。
- `include`：要求返回额外字段，例如 `reasoning.encrypted_content`。
- `previous_response_id`：基于前一响应继续会话。
- `prompt_cache_key`：缓存分桶标识。

工具调用和工具结果分别表示为 `function_call` 和 `function_call_output` item。推理也是独立 item，能够携带 summary、ID 和 encrypted content。

pi 的请求构造见 [`openai-responses.ts`](../src/api/openai-responses.ts#L222-L269)，共享的消息转换与流解析位于 [`openai-responses-shared.ts`](../src/api/openai-responses-shared.ts)。与 Chat Completions 相比，Responses 对推理、工具调用和响应连续性的建模更明确，但 item ID 的重放规则也更严格。

## Azure OpenAI Responses

Azure OpenAI Responses 复用 OpenAI Responses 的消息转换和流解析。主要差异位于部署和认证层：

- 请求中的模型通常对应 Azure deployment name。
- endpoint 由 Azure resource name 或显式 base URL 决定。
- 客户端需要 Azure API version。
- 认证可以使用 API key 或 Microsoft Entra ID；pi 当前适配器使用 API key 路径。
- Azure host 的 URL 需要规范到正确的 `/openai/v1` 基础路径。

相关实现见 [`azure-openai-responses.ts`](../src/api/azure-openai-responses.ts)。因此该 API 更接近 Responses 的部署变体，而不是新的消息协议。

## OpenAI Codex Responses

pi 的 Codex 适配器面向 ChatGPT Codex backend。它沿用 Responses 的 item 和事件语义，但传输与认证不同：

- 从 OAuth token 中读取 ChatGPT account ID。
- 发送 Codex 专用 header。
- 支持 SSE、WebSocket 和带上下文缓存的 WebSocket。
- WebSocket 建连失败时可以回退到 SSE。
- SSE 请求体可以使用 zstd 压缩。
- 复用连接时可通过 `previous_response_id` 只发送新增输入。

请求体见 [`openai-codex-responses.ts`](../src/api/openai-codex-responses.ts#L480-L524)。该 backend 没有完整、稳定的公开 API 规范，不能将当前字段视为第三方可依赖的公共协议。

## Anthropic Messages

Anthropic Messages 原生使用内容块，结构与 pi 的内部 AST 较接近。典型请求为：

```json
{
  "model": "model-id",
  "system": [{ "type": "text", "text": "..." }],
  "messages": [
    {
      "role": "user",
      "content": [{ "type": "text", "text": "..." }]
    }
  ],
  "max_tokens": 4096,
  "stream": true
}
```

主要内容块包括 `text`、`thinking`、`redacted_thinking`、`tool_use` 和 `tool_result`。工具结果位于 user 消息中；连续工具结果通常需要合并到同一个 user turn。

Anthropic 的缓存控制附着在内容块或工具定义上。pi 会根据保留时间生成 `cache_control`，并在系统提示、最后一个工具和最后一个用户内容附近设置缓存断点。thinking 支持两类配置：

- 旧模型使用 `thinking: { type: "enabled", budget_tokens }`。
- adaptive thinking 模型使用 `thinking: { type: "adaptive" }` 和 `output_config.effort`。

请求构造和历史转换见 [`anthropic-messages.ts`](../src/api/anthropic-messages.ts#L901-L1211)。兼容端点可能不支持 eager tool input streaming、工具上的 cache control、长缓存或特定 temperature 组合，相关开关见 [`AnthropicMessagesCompat`](../src/types.ts#L530-L576)。

## Amazon Bedrock ConverseStream

Bedrock ConverseStream 是 AWS 面向多个基础模型的统一接口。pi 发送的主要结构为：

```json
{
  "modelId": "model-or-inference-profile-id",
  "messages": [],
  "system": [],
  "inferenceConfig": {
    "maxTokens": 4096,
    "temperature": 0.2
  },
  "toolConfig": {},
  "additionalModelRequestFields": {},
  "requestMetadata": {}
}
```

官方流事件包括：

- `messageStart`
- `contentBlockStart`
- `contentBlockDelta`
- `contentBlockStop`
- `messageStop`
- `metadata`

通用生成参数位于 `inferenceConfig`，底层模型的特殊参数仍要放入 `additionalModelRequestFields`。因此 Converse 统一了消息和事件格式，但没有消除模型能力差异。

Bedrock 默认使用 AWS credential chain 和 SigV4，也支持 Bedrock bearer token。自定义 header 必须在 Smithy build step 注入，避免破坏签名。pi 的命令构造见 [`bedrock-converse-stream.ts`](../src/api/bedrock-converse-stream.ts#L205-L238)。

## Gemini GenerateContent

Gemini API 使用 `Content` 和 `Part`：

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "..." }]
    }
  ],
  "systemInstruction": {
    "parts": [{ "text": "..." }]
  },
  "tools": [],
  "generationConfig": {}
}
```

assistant role 在该协议中名为 `model`。文本、图片、thought、function call 和 function response 都表示为 part。工具声明使用 `functionDeclarations`，工具结果使用 `functionResponse`。

Gemini reasoning 的配置随模型代际变化：

- Gemini 2.x 主要使用 `thinkingBudget`。
- Gemini 3 系列主要使用 `thinkingLevel`。
- 部分模型不能完全关闭 thinking。pi 会设置最低等级，并不请求返回 thought 内容。

Google 官方协议还定义音频、图像输出、grounding、citation、安全评分和 response schema。pi 当前只映射其中与统一文本 agent 接口相关的部分。消息转换见 [`google-shared.ts`](../src/api/google-shared.ts)，请求构造见 [`google-generative-ai.ts`](../src/api/google-generative-ai.ts#L343-L400)。

## Google Vertex AI

Vertex AI 与 Gemini API 使用接近的 `GenerateContent` 内容模型，pi 复用相同的消息和工具转换。主要差异位于 endpoint 和认证：

- Vertex 请求需要 GCP project 和 location。
- 默认使用 Application Default Credentials。
- 部分 Vertex endpoint 支持 API key。
- 自定义 base URL 需要处理 API version 和 resource scope。

相关实现见 [`google-vertex.ts`](../src/api/google-vertex.ts)。当前 Google AI 和 Vertex 适配器分别维护相似的流解析与 thinking 映射，协议行为接近，但运行环境和认证边界不同。

## Mistral Chat Completions

Mistral 的公开 endpoint 为 `POST /v1/chat/completions`，请求以 `messages` 为核心，结构接近 OpenAI Chat。Mistral 还定义了：

- `prompt_mode`
- `reasoning_effort`
- `prompt_cache_key`
- `tool_choice`
- JSON 和 JSON Schema response format
- 并行工具调用

流式响应使用 data-only SSE，并以 `data: [DONE]` 结束。pi 为 prompt cache 同时设置 `promptCacheKey` 和 `x-affinity`，请求构造见 [`mistral-conversations.ts`](../src/api/mistral-conversations.ts#L220-L268)。

Mistral 工具调用 ID 有固定格式要求。pi 会移除不允许的字符，并在长度不符或发生冲突时生成稳定哈希。

## OpenRouter 图像生成

图像生成使用独立于文本流的内部接口：

```ts
generateImages(model, context, options): Promise<AssistantImages>
```

OpenRouter 官方图像接口支持文本生图、参考图、分辨率、宽高比、输出格式、多图输出，以及部分模型的流式生成。不同上游 endpoint 支持的参数并不一致，调用方应根据模型或 endpoint capability 判断。

pi 当前 `openrouter-images` 适配器使用非流式 Chat Completions 多模态扩展，请求 `modalities: ["image"]` 或 `["image", "text"]`，并从 assistant message 的 `images` 字段解析 data URL。实现见 [`openrouter-images.ts`](../src/api/openrouter-images.ts)。因此 OpenRouter 官方图像能力是 pi 当前映射的上限，不代表这些能力已全部暴露在 `ImagesOptions` 中。

## Provider 与协议映射

当前生成的模型目录包含以下映射：

| API | Provider |
| --- | --- |
| `openai-completions` | ant-ling、Cerebras、Cloudflare Workers AI、DeepSeek、Groq、Hugging Face、Moonshot AI、NVIDIA、OpenRouter、Together、xAI、Xiaomi、z.ai，以及部分 Fireworks、GitHub Copilot、OpenCode、OpenCode Go、Cloudflare AI Gateway 模型 |
| `openai-responses` | OpenAI，以及部分 GitHub Copilot、OpenCode、Cloudflare AI Gateway 模型 |
| `anthropic-messages` | Anthropic、Kimi Coding、MiniMax、Vercel AI Gateway，以及部分 Fireworks、GitHub Copilot、OpenCode、OpenCode Go、Cloudflare AI Gateway 模型 |
| `google-generative-ai` | Google，以及部分 OpenCode 模型 |
| `google-vertex` | Google Vertex |
| `bedrock-converse-stream` | Amazon Bedrock |
| `mistral-conversations` | Mistral |
| `azure-openai-responses` | Azure OpenAI Responses |
| `openai-codex-responses` | OpenAI Codex |

混合协议 provider 必须根据具体 `Model.api` 分发，不能只根据 provider ID 选择 codec。

## pi 的归一化边界

### 可统一的部分

pi 可以稳定统一以下语义：

- 文本和图片输入
- 文本、推理和工具调用输出
- JSON Schema 工具声明
- 工具结果及错误标记
- 流式内容块生命周期
- 正常结束、长度限制、工具调用、错误和取消
- 输入、输出、缓存和推理 token
- 基于模型目录的成本计算

### 有损或带条件的部分

以下内容只能有损转换或依赖 provider metadata：

- provider-specific reasoning signature 和密文
- citation、grounding 和安全评分
- 音频、视频和通用文件对象
- provider-native server tools
- computer use 和 code interpreter 的专用事件
- logprobs 和部分生成诊断
- 上游原始错误结构

跨模型重放时，pi 优先保证目标 API 接受历史消息，而不是逐字段保持原始请求。这包括删除不可重放的密文、将推理转为文本、规范化工具 ID、跳过未完成的 assistant turn，以及为孤立工具调用补充错误结果。

## 主要兼容性风险

1. OpenAI-compatible 端点共享 URL 和基础字段，但 reasoning、tool use、cache 和 usage 仍可能不兼容。
2. provider-specific signature 通常绑定模型，跨模型原样重放可能被拒绝。
3. cache retention 是意图级抽象，无法保证各 provider 使用相同的保留时间和计费方式。
4. `onPayload` 可以替换完整请求体，调用方需要自行维持适配器依赖的不变量。
5. 厂商可能新增 stop reason 或流事件。适配器必须显式映射，不能默认视为正常结束。
6. 上游协议能力大于当前内部 AST。新增音频、引用或 server tool 时，需要扩展公共内容块或提供明确的 provider metadata。

## 官方参考资料

- [OpenAI Chat API reference](https://developers.openai.com/api/reference/resources/chat)
- [OpenAI Responses: Create a model response](https://developers.openai.com/api/reference/resources/responses/methods/create)
- [Azure OpenAI Responses API](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/responses)
- [Anthropic Messages API](https://platform.claude.com/docs/en/api/messages)
- [Gemini API: Generating content](https://ai.google.dev/api/generate-content)
- [Vertex AI generative inference](https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/inference)
- [Amazon Bedrock ConverseStream](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_ConverseStream.html)
- [Mistral Chat API](https://docs.mistral.ai/api/endpoint/chat)
- [OpenRouter image generation](https://openrouter.ai/docs/guides/overview/multimodal/image-generation)
- [OpenRouter provider routing](https://openrouter.ai/docs/guides/routing/provider-selection)

