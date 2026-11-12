# 插件配置：用 Schemastery 接收配置

> **实时文档（先读，可能与本文不同步）**
> - 插件配置（入门）：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/config>
> - Cordis tutorial 第 5 章 配置：<https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/05-config>
> - 插件配置目录（每个内置包接受的配置项）：<https://deepseek-harness.github.io/deepseek-harness/reference/config-catalog>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/config>

## 定义 `Config` 类型与同名 schema

导出一个 `Config` **类型**和同名的 Schemastery schema，默认值直接写在 schema 字段上：

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'my-plugin'

export interface Config {
  greeting: string
  maxRetries: number
  verbose?: boolean
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
  verbose: Schema.boolean().default(false),
})

export function apply(ctx: Context, config: Config) {
  console.log(config.greeting)   // 用户值或 schema 默认值
}
```

在 `cordis.yml` 的插件行里给出配置：

```yaml
- insert:
    - id: hello
      name: './src/my-plugin.ts'
      config:
        greeting: 'Hi there'
        maxRetries: 5
```

加载时 Cordis 用导出的 schema 校验配置并填充默认值。

> **不要导出普通对象当 `Config`**：它不实现 Cordis 要求的 Standard Schema 接口。

## 更严格的校验

```ts
export interface Config {
  apiKey: string
  timeout: number
  mode: 'fast' | 'accurate'
}

export const Config = Schema.object({
  apiKey: Schema.string().required(),
  timeout: Schema.number().default(30000),
  mode: Schema.union(['fast', 'accurate']).default('fast'),
})
```

schema 在插件加载时运行。配置非法会让加载**失败**，并给出可操作的错误信息。

## 设计原则

### 不要把可调值写死

DSH 要求：**任何两个部署可能想设成不同值的量，都必须是配置字段**。

```ts
// 错误：写死 timeout
const TIMEOUT = 30000

// 正确：可配置
export interface Config {
  timeoutMs: number   // 默认 30000
}
```

判断标准：能否在不改代码的情况下通过 `cordis.yml` 改掉它。

### 配置非法要大声失败

自包含的约束尽量写进 schema，让非法配置在插件加载时失败。需要引用服务或已注册资源的约束要靠依赖注入，见 [04-services-and-lifecycle.md](04-services-and-lifecycle.md)。

## 与 HMR 配合

改配置会**热替换**插件：框架卸载旧实例、加载新实例。因为注册都是 effect、会自我清理，替换不会残留旧实例的注册。

## 常见坑

- **配置没生效**：检查它写在哪一层。层顺序见 [07-packaging-and-publish.md](07-packaging-and-publish.md)；后应用的层按行覆盖，且 patch 会**整体替换**该行的 `config`，不是深合并。
- **`Config` 导出成普通对象**：类型上看似对，运行时会因缺少 Standard Schema 接口而失败。
- **只在 schema 里写默认值却在代码里再兜一次底**：默认值只放 schema 一处，代码直接信任 `config`。
- **`!!js` 表达式取不到值**：Loader 只对 entry 的 `config`（在声明的注入激活后、针对该插件上下文）和 `disabled`（每次 mount 决策、针对 loader 上下文）做插值，其他 entry 元数据保持字面量。
