# Service 与依赖

> **实时文档（先读，可能与本文不同步）**
> - Service 与依赖：<https://deepseek-harness.github.io/deepseek-harness/develop/framework/service>
> - 插件与生命周期：<https://deepseek-harness.github.io/deepseek-harness/develop/framework/>
> - Cordis tutorial 第 3 章 Services：<https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/03-services>
> - Cordis API — Service：<https://deepseek-harness.github.io/deepseek-harness/reference/cordis-api/service>
> - Cordis API — Context：<https://deepseek-harness.github.io/deepseek-harness/reference/cordis-api/context>
> - 子系统页（由代码生成的服务名、公开方法、源码位置）：<https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/core>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/framework/service>

## 什么是 service

service 是某个插件向其他插件暴露的能力，挂载在 `ctx` 上、有稳定的 key。DSH 内置的 `tools`、`llm`、`agents` 都是 service：

```ts ignore-check
ctx.tools    // ToolRuntime service
ctx.llm      // LLM service
ctx.agents   // Agent service
```

任何插件都可以提供 service 供别的插件消费。

## 消费 service

声明 `inject` 即可使用已有 service；`apply` 运行时，`inject` 里每个 service 都已就绪：

```ts ignore-check
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(/* ... */)
}
```

service 未就绪时，插件**等待**而不是运行（对应 Fiber 的 `PENDING`）。

## 提供 service

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

export default class MetricsService extends Service {
  static inject = ['llm']      // service 本身也可以依赖其他 service

  constructor(ctx: Context) {
    super(ctx, 'metrics')      // 'metrics' 是 service 名
  }

  record(event: string, value: number) {
    // 公开方法
  }
}
```

加载后，消费者通过 `ctx.metrics` 访问：

```ts ignore-check
export const inject = ['metrics']

export function apply(ctx: Context) {
  ctx.metrics.record('tool_call', 1)
}
```

### 声明类型（declaration merging）

用 TypeScript 声明合并给 `ctx.metrics` 加类型：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    metrics: MetricsService
  }
}

export default class MetricsService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'metrics')
  }

  record(event: string, value: number) { /* ... */ }
}
```

## 依赖行为

### 必需 vs 可选

```ts ignore-check
// 必需：service 缺失时插件不加载
export const inject = ['tools']

// 可选：不写 inject，在使用点用 ctx.get() 查询
export function apply(ctx: Context) {
  const metrics = ctx.get('metrics')
  metrics?.record('plugin_loaded', 1)
}
```

### service 消失时

若必需 service 在运行中消失（例如其 provider 卸载）：① 依赖它的插件自动 dispose；② service 回来时重新加载。这样插件不会调用已不存在的 service。

## service 隔离

`cordis.yml` 可以隔离 service，让不同插件组看到同一 service 的不同实例：

```yaml
- id: group-a
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 5000
    - name: './src/plugin-a.ts'

- id: group-b
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 60000
    - name: './src/plugin-b.ts'
```

`plugin-a` 与 `plugin-b` 各自看到本组的 Bash 实例，互不影响。

## 内置 service 去哪查

**不要维护第二份静态清单。** 服务名、公开方法、源码位置由仓库生成到各 service 的 subsystem 页：<https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/core>。开发时同时对照该页与 service 的 TypeScript 接口。

## 常见坑

- **`inject` 名字写错**：插件永远停在 PENDING，且没有明显报错。对照 subsystem 页核名。
- **把必需依赖写成可选**：provider 消失后插件继续运行并调用空 service。真需要就写进 `inject`。
- **忘记声明合并**：运行正常但 `ctx.metrics` 无类型，退化成 `any`。
- **在同一 group 内期望共享实例**：`isolate` 之后不同组拿到的是不同实例，跨组通信要走事件或上层 service。
