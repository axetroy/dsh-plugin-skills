# LLM adapter：接入新的模型 provider

> **实时文档（先读，可能与本文不同步）**
> - LLM adapter：<https://deepseek-harness.github.io/deepseek-harness/develop/practice/llm-adapter>
> - 新增 LLM adapter cookbook：<https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/adding-an-llm-adapter>
> - LLM 流式子系统的服务与事件面：<https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/llm-streaming>
> - 子系统首页：<https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/practice/llm-adapter>

## 概览

LLM adapter 继承 `LlmAdapter` 并实现 `stream()`：把 harness 的 provider-neutral 请求翻译成 provider API 调用，再把响应翻译回 harness 的 chunk。

## 最小实现

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'
import { LlmAdapter, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

class MyAdapter extends LlmAdapter {
  private apiKey: string

  constructor(apiKey: string) {
    super()
    this.apiKey = apiKey
  }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // 1. 把 options.messages 转成 provider 格式
    // 2. 调用流式 API
    // 3. 把响应转成 StreamChunk
  }
}

export interface Config {
  apiKey: string
  providers: string[]
}

export const Config: Schema<Config> = Schema.object({
  apiKey: Schema.string().required(),
  providers: Schema.array(Schema.string()).required(),
})

export const name = 'my-llm-adapter'
export const inject = ['llm']

export function apply(ctx: Context, config: Config) {
  const adapter = new MyAdapter(config.apiKey)
  ctx.llm.registerAdapter(config.providers, adapter)
}
```

## StreamChunk 协议

`stream()` 按下面的顺序产出 chunk：

```ts
import { brandString } from '@deepseek-ai/dsh-brand'
import type { StreamChunk, ToolCallId } from '@deepseek-ai/dsh-llm'

async function* exampleChunks(): AsyncIterable<StreamChunk> {
  // 1. 每个 content block 以 block-start 开始
  yield { type: 'block-start', index: 0, blockType: 'text' }

  // 2. 文本通过 text-delta 流式输出
  yield { type: 'text-delta', index: 0, text: 'Hello' }
  yield { type: 'text-delta', index: 0, text: ' world' }

  // 3. 每个 block 以 block-end + 完整 block 结束
  yield { type: 'block-end', index: 0, block: { type: 'text', text: 'Hello world' } }

  // 4. tool-call block
  yield { type: 'block-start', index: 1, blockType: 'tool-call' }
  yield {
    type: 'tool-call-delta',
    index: 1,
    id: brandString<ToolCallId>('call-123'),
    name: 'bash',
    argumentsDelta: '{"command":"ls"}',
  }
  yield {
    type: 'block-end',
    index: 1,
    block: {
      type: 'tool-call',
      id: brandString<ToolCallId>('call-123'),
      name: 'bash',
      arguments: '{"command":"ls"}',
    },
  }

  // 5. token 用量
  yield { type: 'usage', usage: { inputTokens: 100, outputTokens: 50 } }

  // 6. 结束原因
  yield { type: 'finish', reason: { kind: 'stop' } }
  // 或者 { kind: 'tool-calls' } 请求执行工具
}
```

### 关键规则

- 每个 `block-start` 都要有配对的 `block-end`。
- `index` 从 0 递增，标识 content block 的顺序。
- `tool-call-delta` 在 `argumentsDelta` 里携带**原始 JSON 文本**，可以一次给全，也可以分多个 chunk。
- `finish` 是最后一个 chunk。
- `usage` 要在 `finish` 之前发出。

## GenerateOptions

`stream()` 收到导出的 `GenerateOptions` 类型，包含 model、adapter 自有的 reasoning-effort id、对话历史、system prompt、工具 schema、生成参数、stop 序列与 abort signal。**以 `@deepseek-ai/dsh-llm` 导出的 TypeScript 类型为准**。把支持的字段映射到 provider API；provider 无法满足某字段时，抛带稳定 code 的 `LlmError`，不要静默丢弃。

覆写 `resolveModel(provider, model, signal?)`，在一次查询里返回精确的 provider/model 身份，外加可选的 `context` 与 `reasoning` 元数据。reasoning 元数据包含有序的不透明 id 与展示名，以及可选的配置默认值；保留 adapter 权威的可选列表（包括上游能力 API 返回的 `off`），不要把这些值提升成核心枚举。要尊重可选的 signal，以便取消和 dispose 能到达静止状态。service 会在 `stream()` 之前校验聚合结果并拒绝不支持的显式 effort；省略 `reasoning` 表示该模型没有可选的 reasoning-effort 能力。

## 注册 adapter

```ts ignore-check
ctx.llm.registerAdapter(['my-provider'], adapter)
```

第一个参数列出该 adapter 处理的 provider 路由。`GenerateOptions.provider` 选择已注册的 adapter，`GenerateOptions.model` 传 adapter 自有的 model id、不参与生命周期注册。adapter 能向选择器公布模型时，覆写 `listModels()`。

## 在 cordis.yml 里使用

```yaml
- id: my-llm
  name: './src/my-llm-adapter.ts'
  config:
    apiKey: !!js process.env.MY_API_KEY
    providers:
      - my-provider

- id: agent-loop
  name: '@deepseek-ai/dsh-agent-loop'
  config:
    agents:
      - id: main
        provider: my-provider
        model: my-model-v1
```

## 错误处理

adapter 把传输与协议失败抛成带稳定 code 的 `LlmError`。`new LlmError` 只接受 `LLM_FAILURE_CODES` 的内置 code；adapter 自有的 code 走 `LlmError.fromFailure`。agent loop 会保留错误与 code 用于诊断和策略，**不会**自动把普通 `Error` 转换过来。每个 provider HTTP 请求还必须合并 `attributionHeaders()` 并转发 `options.signal`。

```ts
import {
  attributionHeaders,
  LlmAdapter,
  LlmError,
  type GenerateOptions,
  type StreamChunk,
} from '@deepseek-ai/dsh-llm'

class HttpAdapter extends LlmAdapter {
  constructor(private readonly endpoint: string) {
    super()
  }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    const response = await fetch(this.endpoint, {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        ...attributionHeaders(),
      },
      body: JSON.stringify({ model: options.model, messages: options.messages }),
      ...options.signal ? { signal: options.signal } : {},
    })
    if (!response.ok) {
      throw LlmError.fromFailure({
        message: `Provider API error: ${response.status}`,
        code: 'PROVIDER_HTTP_ERROR',
        status: response.status,
      })
    }
    yield { type: 'finish', reason: { kind: 'stop' } }
  }
}
```

## 参考实现

- `packages/llm/llm-deepseek/` — 用 Messages API 的 DeepSeek adapter。
- `packages/llm/llm-pi-ai/` — 用另一种 API 格式的 Pi AI adapter。

对照这两个已发布的 adapter，可以看到同一套 harness 契约如何落在不同的 provider SDK 上。

## 常见坑

- **漏发 `block-end`**：流式解析在 block 未闭合时行为异常。
- **`usage` 放在 `finish` 之后**：顺序错了，用量会丢。
- **把 tool 参数拆成解析好的对象**：`argumentsDelta` 必须是原始 JSON 文本，拼接后才是完整 JSON。
- **忘记转发 `signal`**：取消无效，dispose 到不了静止状态。
- **用普通 `Error` 抛 provider 失败**：诊断与策略拿不到 code，改用 `LlmError` / `LlmError.fromFailure`。
