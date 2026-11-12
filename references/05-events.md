# 事件系统

> **实时文档（先读，可能与本文不同步）**
> - 事件系统：<https://deepseek-harness.github.io/deepseek-harness/develop/framework/events>
> - Cordis tutorial 第 4 章 Events：<https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/04-events>
> - Cordis API — Events：<https://deepseek-harness.github.io/deepseek-harness/reference/cordis-api/events>
> - Cordis primer（派发模式与 waterfall 语义）：<https://deepseek-harness.github.io/deepseek-harness/reference/cordis-primer>
> - 子系统页（`cordis-surface` 区域记录完整签名与模式）：<https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/core>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/events>

事件是 Cordis 插件之间最核心的通信机制，DSH 大量用它做松耦合扩展点。

## 基本用法

```ts ignore-check
ctx.on('event-name', (payload) => { /* 处理 */ })
ctx.emit('event-name', payload)
```

## 五种派发模式

事件只能用它声明的那个模式对应的方法来派发。

| 模式 | 是否 await | 派发顺序 | 有返回值 |
| --- | --- | --- | --- |
| `emit` | 否 | 按注册顺序，listener 只观测 | 无 |
| `waterfall` | 否 | 按注册顺序，listener 可包裹结果 | 有 |
| `parallel` | 是 | 所有 listener 并行观测 | 无 |
| `serial` | 是 | 按注册顺序 | 有 |
| `bail` | 否 | 按注册顺序，直到有人 bail | 有 |

### emit — 广播

```ts ignore-check
ctx.emit('my-plugin/ready', { id: 'worker-1' })
ctx.on('my-plugin/ready', ({ id }) => console.log(`${id} is ready`))
```

### bail — 短路

listener 按顺序运行，第一个返回值**不是** `null`/`false`/`undefined` 的结果即最终结果：

```ts ignore-check
const result = ctx.bail('some-check', input)

ctx.on('some-check', (input) => {
  if (shouldBlock(input)) return 'blocked'
  // 返回 null/false/undefined 继续给下一个 listener
})
```

### serial — 有序执行

按注册顺序运行，异步结果会被 await；第一个非空结果停止后续执行：

```ts ignore-check
await ctx.serial('setup-phase', context)
```

### waterfall — 管线（around-middleware）

每个 listener 可以包裹下游结果形成处理链。listener 收到 `(...args, next)`；**必须调用 `next()` 才能委托下游**，不调用即按设计短路：

```ts ignore-check
const output = await ctx.waterfall('my-plugin/transform', input, async () => input)

ctx.on('my-plugin/transform', async (_input, next) => {
  const downstream = await next()
  return downstream.trim()
})
```

::: warning
waterfall listener **必须调用 `next()`**。不调用会短路整条管线——这是刻意设计，用来支持拦截与网关行为。
:::

派发模式是事件的公开契约的一部分。新增 DSH 事件用 `@mode` 标注，生成的目录会核对声明与派发点是否一致。只有必须抢在普通注册之前运行时才用 `prepend: true`。

## 类型化事件

DSH 用 TypeScript 声明合并做类型安全的事件：

```ts
import '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-plugin/ready': (payload: { id: string }) => void
    'my-plugin/check': (input: string) => boolean | undefined
    'my-plugin/transform': (input: string, next: () => Promise<string>) => Promise<string>
  }
}
```

## Cordis 事件 ≠ 会话事件

DSH 的 Cordis 事件用 `namespace/action` 命名，例如 `agent/pre-step`、`agent/request-error`、`tools/result`、`session/event`。完整签名与模式记录在 subsystem 页的 `cordis-surface` 区域。

**注意区分**：`turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` 是**持久化会话事件类型**，不是同名的 Cordis 事件。要观测它们，监听 `session/event` 并检查 `event.type`。

## 监听器就是 effect

`ctx.on()` 注册的 listener 在插件卸载时自动移除：

```ts ignore-check
export function apply(ctx: Context) {
  ctx.on('tools/result', handler)   // 插件 dispose 时移除
}
```

## 示例：工具日志插件

```ts
import type { Context } from '@deepseek-ai/cordis'
import '@deepseek-ai/dsh-tools'

export const name = 'tool-logger'

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    console.log(`[tool] ${exec.name}(${JSON.stringify(exec.arguments)})`)
    const text = result.content
      .map(block => block.type === 'text' ? block.text : '')
      .join('')
    console.log(`[tool result] ${text.slice(0, 100)}`)
  })
}
```

## 常见坑

- **用 `emit` 想拿返回值**：`emit` 丢弃返回值，需要结果就用 `bail` / `serial` / `waterfall`。
- **waterfall 忘了 `next()`**：下游静默不执行，表现为“策略没生效”。
- **监听 `tool/call` 拿不到事件**：那是会话事件类型，要监听 `session/event`。
- **用 `ctx.on` 之外的方式注册**：例如直接挂 EventEmitter，卸载不会清理，热替换后重复触发。
