# 能力分层：Definition / Provider / Consumer 三角色

> **实时文档（先读，可能与本文不同步）**
> - 三角色能力设计：<https://deepseek-harness.github.io/deepseek-harness/develop/practice/>
> - 内置能力家族与包链接（capability seams）：<https://deepseek-harness.github.io/deepseek-harness/reference/capability-seams>
> - 插件与生命周期：<https://deepseek-harness.github.io/deepseek-harness/develop/framework/>
> - 扩展插件形态 cookbook：<https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/extension-cookbook>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/practice/>

## 什么时候拆三个角色

当一项能力足够通用、需要**可替换的 provider**（例如 Bash 执行）时，DSH 把它拆成三个角色：

- **Service Definition** — 定义 Cordis service 以及请求/结果类型
- **Service Provider** — 真正实现该能力
- **Consumer** — 把该能力暴露成模型可调用的工具，或其他消费方

需要独立演进或被替换时，把三个角色放进**不同包**；否则一个包也可以承担多个角色。**完整能力才是 seam，单个角色不是。**

## Bash 示例

- Service Definition（`dsh-shell`）— 定义 Cordis service 与 Bash 请求/结果类型
- Service Provider（`dsh-bash-local`）— 在本机执行命令
- Consumer（`dsh-tool-bash`）— 把能力暴露成模型可调用的工具

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│  dsh-shell   │────▶│  dsh-bash-local  │     │ dsh-tool-bash│
│(definition) │     │    (provider)     │     │(consumer/tool)│
└─────────────┘     └──────────────────┘     └──────────────┘
       ▲                                            │
       └────────────────────────────────────────────┘
                    inject: ['shell']
```

## 拆分的收益

- **替换 provider**：一个 Service Definition 可以有多个 provider，由 `cordis.yml` 选择。Definition 与工具不变，只换 provider 行。
- **独立演进**：Definition 在调用方依赖其契约后很少变动；Provider 可独立优化性能与安全；Consumer 可独立改变对模型的呈现方式。
- **解耦依赖**：Provider 与 Consumer 都只依赖 Definition，**彼此不依赖**。

## 动手：开发一个三角色能力

### 第 1 步：写 Service Definition

```ts ignore-check
// packages/my-cap/my-cap/src/index.ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    myCap: MyCapService
  }
}

export abstract class MyCapService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myCap')
  }

  /** 执行该能力。 */
  abstract execute(request: MyCapRequest): Promise<MyCapResult>
}

export interface MyCapRequest { input: string }
export interface MyCapResult { output: string }
```

### 第 2 步：写 Service Provider

```ts ignore-check
// packages/my-cap/my-cap-local/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { MyCapService, type MyCapRequest, type MyCapResult } from '@deepseek-ai/dsh-my-cap'

class MyCapLocal extends MyCapService {
  async execute(request: MyCapRequest): Promise<MyCapResult> {
    return { output: request.input.toUpperCase() }
  }
}

export const name = 'my-cap-local'

export function apply(ctx: Context) {
  ctx.plugin(MyCapLocal)
}
```

### 第 3 步：写 Consumer

```ts ignore-check
// packages/my-cap/tool-my-cap/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'tool-my-cap'
export const inject = ['tools', 'myCap']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'my_cap',
    description: 'Execute my capability.',
    parameters: { input: { type: 'string', required: true } },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      const result = await ctx.myCap.execute({ input: args.input })
      return result.output
    },
  }))
}
```

### 在 cordis.yml 里组合

```yaml
- name: '@deepseek-ai/dsh-my-cap-local'
- name: '@deepseek-ai/dsh-tool-my-cap'
```

## 设计要点

- **不要预防性地拆分**：只有当各角色需要独立演进时才分独立包。简单工具插件不需要。
- **Request/Result 类型归 Service Definition 所有**：Provider 与 Consumer 只依赖 Definition 包。
- **显式优于隐式**：把默认值解析放到显式的 `resolve(request): Spec` 步骤，而不是把 `?? default` 藏在 `run()` 里。

## 常见坑

- **Provider 和 Consumer 互相 import**：破坏了 seam，替换任一方都要改另一方。
- **类型定义散落在 Provider 里**：Consumer 被迫依赖具体实现包，无法换 provider。
- **为一个只会有单一实现的简单工具拆三包**：徒增复杂度。先按简单工具插件写，出现第二个 provider 需求时再拆。
- **在 Consumer 里塞业务逻辑**：Consumer 只负责协议转换（工具 ↔ service），实现放 Provider。
