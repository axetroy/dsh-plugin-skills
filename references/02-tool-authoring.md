# 工具开发：给模型注册可调用工具

> **实时文档（先读，可能与本文不同步）**
> - 构建一个工具（入门）：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/tool>
> - 工具开发参考（完整契约）：<https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/adding-a-tool>
> - 工具 schema 目录（每个内置工具的入参/出参）：<https://deepseek-harness.github.io/deepseek-harness/reference/tool-catalog>
> - 扩展插件形态 cookbook（工具/钩子/UI）：<https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/extension-cookbook>
> - Cordis tutorial 第 7 章（在真实 harness 服务上注册工具）：<https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/07-into-the-harness>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/tool>

## 最小形态

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',        // 模型看到的就是这句
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                   // 默认可选
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args 由 schema 推导出类型：{ path: string; limit?: number }
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

`defineTool` 把 `parameters` 转成给模型的 JSON Schema，推导 `args` 类型，并在 `execute` 之前校验模型给的参数。注册是 effect：插件 fiber 释放即注销工具。schema 会自动进入 system prompt 组装。

## `execute()` 契约要点

- **参数已被校验**：`defineTool` 在 `execute` 前按统一的 `ParameterSchemaSpec` 校验（类型、必填、字面量、exact-one union、嵌套值）。DSL 表达不了的约束（非空字符串、正数、跨字段规则）仍需自己检查，并用 `ToolArgsError` 拒绝。
- **注册借用你的只读定义**：不要注册后再改 schema 或替换回调。要热替换工具，就释放持有它的 effect 再注册新的。
- **执行身份受保护**：`exec` 上 `callId`、`name`、`arguments`、`agent`、`token`、必需的 `signal`、可选的 `parent` 在派发期间不可变。把 `args` 当只读输入。
- **只声明并返回一个规范 JSON 值**：`output.schema` 用 `ValueSchemaSpec`，根可以是对象、数组、标量或 null。`execute` 只返回该值；不要返回 content blocks，也不要让调用方从 prose 里解析 id 和字段。
- **抛出或返回非法值即 `isError`**：每个预期失败都抛一个带稳定 `code` 的 `HarnessError` 子类；只有抛出的 `HarnessError` 会把 `error.name` 和 `error.code` 保留到结果上。
- **遵守 `exec.signal`**：signal 触发时取消在途工作，让 signal 的 reason 或名为 `AbortError` 的依赖错误向上传播（注册表记为 `AbortError`，`TOOL_ABORTED`）。不要替换成普通 `Error`。
- **用 `exec.agent` 做异步通知**：`agent.inject({ content, source: { kind: 'my-plugin' } })` 追加的是**下一次**模型请求才看到的持久上下文，不是唤醒信号（idle 的 agent 保持 idle）。要对该 `kind` 做声明合并；session format V4 拒绝已废弃的 `{ kind: 'plugin', plugin: '<name>' }`。对已 dispose 的 agent 要 try/catch。

## 长任务与后台执行

`run_in_background` 由 producer 配置开关，然后通过 `ctx.jobs.start({ kind, label, owner: exec.agent, run })` 注册。注册表会在 producer body 之前拒绝已 abort 的调用；运行时在 `run()` 开工前校验归属与 task-controller 可用性，并负责 id、session fence、通用控制工具、通知与 owner 清理。

- 后台分支成功时返回类型化的规范 handle，例如 `{ kind: 'background', jobId }`；其 renderer 可以保留人类 prose（如 `started background job bash-1`），但 **PTC 模式绝不能解析这句 prose 去取 id**。
- 一旦 `ctx.jobs.start()` 发布了 id，就用**任务自己的**取消 signal，而不是 `exec.signal`：外层调用取消只停止等待，不杀死已发布的工作；`job_kill`、owner dispose、服务拆除才拥有它的生命周期。前台工作仍与 `exec.signal` 绑定。

## 执行策略与观测（不要把手艺写进工具）

优先不要把部署策略做进工具里，改用这些扩展点：

| 事件 / API | 用途 |
| --- | --- |
| `tools/pre-execute` | 可扩展的 allow/deny/ask 策略 |
| `ctx.tools.guard()` | 最终单调 deny，后续 listener 无法撤销 |
| `tools/execute` | 包裹派发，加 deadline、重试、metrics |
| `tools/post-execute` | 替换展示内容或返回值、阻断结果、附加模型可见上下文 |
| `tools/result` | 观测不可变的规范化结果 |

内容替换不影响程序化访问 `value`；机密性策略会阻断或替换 `value`。

## PTC 模式

PTC 模式下每个可见的已注册工具都自动成为 `await tools.<name>(args)`，无需额外集成。`ToolArgsMap` / `ToolOutputMap` 由同一套 schema 推导精确类型，调用重新进入正常执行管线。成功调用 resolve 到策略之后的最终规范 JSON 值，而不是渲染后的内容；失败调用 reject 真正的 `ToolCallError`。

因此把 `output.schema` 设计成好用的编程接口：直接返回 handle 和字段；标量/数组/null 根是诚实值时就允许；人类解释留在 `output.render`。

## UI 卡片

`output.render` 产出**模型可见**内容；**UI 卡片**是另一件事，由纯展示投影和可选的 `presentCall` / `presentResult` 声明。两者都返回 `card`-tagged render intent：

- `presentCall(args)` → 待定卡片：`{ card: 'generic', title, kind?, rawInput?, content?, locations? }`、`{ card: 'terminal', title, description?, cwd? }`、`{ card: 'diff', title, diffs, locations? }`。
- `presentResult(args, { content, isError, meta? })` → 完成卡片：`generic` / `terminal` / `diff` / `read` / `search` / `web`。

硬性规则：

- **纯函数**：这些方法在实时流式**和**会话日志回放时都会跑，因此必须是 `args`（+ 结果）的纯函数——禁止 I/O、禁止读会话状态、禁止时钟/随机。
- **UI 专用格式化不要进模型结果**：`` ```console `` 代码块、diff、相对化路径都不该为了 UI 而进入规范值或模型内容。
- **`defineTool` 对展示路径做软校验**：格式不对或旧日志参数会让 wrapper 返回 `undefined`（走 generic 回退），而不是抛错——展示绝不能弄崩回放。

内置 Web Client 不消费 `presentCall` / `presentResult`；它通过 `tool.call.toolview` 的 keyed slot 由 Client 插件自行派生卡片。需要给已有 Web 卡片提供有界的结构化结果事实时，用 `output.presentationMeta(args, value)`。

## 参考实现

- `packages/shell/tool-bash` — 生产级三包示例（Service Definition / Provider / Consumer），terminal 卡片。
- `packages/fs/tool-fs`（`write`/`edit`/`read`）— diff 与 read 卡片。

## 验证

按仓库的测试策略与所属包的测试文档执行。任何模型可见或 UI 可见的改动都需要对应的组装级覆盖。
