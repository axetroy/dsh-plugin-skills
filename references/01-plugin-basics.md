# 插件基础：第一个插件

> **实时文档（先读，可能与本文不同步）**
> - 你的第一个插件：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/>
> - 插件与生命周期：<https://deepseek-harness.github.io/deepseek-harness/develop/framework/>
> - Cordis tutorial（可运行、无需 API key）：<https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/>
> - 第 1 章 第一个插件：<https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/01-first-plugin>
> - 第 2 章 生命周期与 effects：<https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/02-lifecycle-and-effects>
> - Cordis primer：<https://deepseek-harness.github.io/deepseek-harness/reference/cordis-primer>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/>

## 前置：跑起来

在 DSH 仓库 checkout 中（源码运行路径）：

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

`pnpm dsh web` 使用已构建产物，不重新构建。

## 最小插件

插件是一个 TypeScript 模块，导出 `apply` 函数；Cordis 加载时传入 `ctx`，插件通过 `ctx` 注册能力：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello-plugin'   // 可选，仅用于诊断显示

export function apply(ctx: Context) {
  console.log('[hello-plugin] plugin loaded!')
}
```

## 用 `--patch` overlay 加载本地插件

`scratch-plugin/cordis.yml`（Web overlay）：

```yaml
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

```sh
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

要点：**插件路径必须是绝对路径**。patch 文件只贡献配置，不会改变 loader 解析模块路径的目录。

## 声明依赖：`inject`

插件若使用 `tools`、`llm` 等服务，必须在 `inject` 中声明；框架会等到这些服务就绪后才加载插件：

```ts ignore-check
export const name = 'my-tool-plugin'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(/* ... */)   // 此处 ctx.tools 已就绪
}
```

## 自动清理与 `ctx.effect`

通过 `ctx` 完成的注册（事件监听、工具、定时器）在插件卸载时**自动撤销**，无需手写 `removeListener` / `clearInterval`。框架管理这些注册：

- `ctx.on(event, handler)`
- `ctx.tools.register(tool)`
- `ctx.llm.registerAdapter(names, adapter)`
- `ctx.effect(() => cleanup)`

框架管不到的资源（网络连接、watcher）用 `ctx.effect()` 返回 disposer：

```ts
export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => console.log('heartbeat'), 5000)
    return () => clearInterval(timer)   // 插件卸载时执行
  })
}
```

卸载时 disposer 按**注册的逆序**启动，但多个异步 disposer **并发**执行、无串行完成保证。有顺序依赖的清理要放在**同一个** `ctx.effect()` 返回的单个 disposer 里串行 await。

## Fiber 状态机

每个已加载插件实例拥有一个 **Fiber**：

```
PENDING → LOADING → ACTIVE
                 ↘ FAILED
ACTIVE → UNLOADING → DISPOSED
```

| 状态 | 含义 |
| --- | --- |
| PENDING | 已声明，但依赖服务未就绪 |
| LOADING | 依赖就绪，`apply` 正在执行 |
| ACTIVE | 插件运行中 |
| FAILED | `apply` 抛错 |
| UNLOADING | 正在卸载并释放资源 |
| DISPOSED | 已完全卸载 |

依赖驱动的加载：所需服务消失时（例如 provider 被替换），插件自动卸载（ACTIVE → DISPOSED），服务恢复后重新加载。

## 三种插件形态

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

// 1. 函数式（最常用）
export function apply(ctx: Context) {}

// 2. 对象式
export default {
  name: 'my-plugin',
  inject: ['tools'],
  apply(ctx: Context) {},
}

// 3. 类式：Service 子类，用于向其他插件提供服务
export default class MyService extends Service {
  static inject = ['tools']
  constructor(ctx: Context) {
    super(ctx, 'myService')   // 同步初始化放在构造函数
  }
}
```

大多数场景用函数式即可；需要对外提供服务时用类式，见 [04-services-and-lifecycle.md](04-services-and-lifecycle.md)。

## 嵌套上下文与手动 dispose

`ctx.plugin(child)` 创建子 Fiber：继承父上下文，生命周期独立，随父插件一起卸载。

```ts
import type { Context } from '@deepseek-ai/cordis'

declare const ctx: Context
declare function myPlugin(ctx: Context): void

const fiber = ctx.plugin(myPlugin)
await fiber.dispose()
```

`dispose()` 保证：① 移除该插件所有注册；② 递归卸载子插件；③ 返回的 promise 在所有异步清理完成后才 resolve。

## HMR

`cordis.yml` 中加载 `@deepseek-ai/dsh-hmr` 后，改动插件源文件会：卸载旧插件并清理注册 → 加载新代码 → 运行新 `apply`。因为注册会自我清理，热替换不会残留旧实例的注册。

## 常见坑

- **插件没加载**：`inject` 的服务不存在 → 停在 PENDING。先查服务名拼写，再查 provider 是否被加载。
- **用了相对路径**：`--patch` 里的本地插件路径必须是绝对路径。
- **手动清理由 `ctx` 管理的注册**：多余且会出错，交给框架。
- **在 `apply` 外注册**：模块顶层副作用不受 Fiber 管理，卸载不会回收。
