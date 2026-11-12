---
name: dsh-plugin-skills
description: 指导 DeepSeek Harness (DSH) 插件开发：从零写第一个插件、定义模型可调用工具、用 Schemastery 接收配置、提供与消费 Cordis service、使用 typed events、实现 LLM adapter，以及打包成 bundle/profile 发布安装。当用户要开发、调试、打包或发布 DSH 插件，提到 cordis.yml、defineTool、ctx.tools、inject、apply、dsh plugin、bundle、profile、cordis.patch.yml、LlmAdapter、StreamChunk 或 DeepSeek Harness 插件体系时使用本技能。
license: MIT
compatibility: 需要能访问 https://deepseek-harness.github.io/deepseek-harness/ 以查阅实时文档；本地开发插件需要 Node.js、pnpm 以及 DeepSeek Harness 仓库 checkout。
metadata:
  author: dsh-plugin-skills
  version: "1.0"
  upstream: https://github.com/deepseek-ai/deepseek-harness
---

# DeepSeek Harness 插件开发

本技能指导为 **DeepSeek Harness (DSH)** 编写插件。DSH 的插件框架是 **Cordis**：一个插件就是一个 TypeScript 模块，导出 `apply` 函数（或一个 `Service` 子类），Cordis 在加载时调用它并传入 `ctx`；插件通过 `ctx` 注册一切能力（工具、服务、事件、适配器），卸载时这些注册会被自动回收。

## 使用方式

1. 先判断任务属于哪一类，打开下面**唯一对应**的 reference（每个文件独立成篇，不要一次全读）。
2. 每个 reference 顶部都有 **实时文档链接**。动手前先读链接里的在线文档，因为它可能已经比本技能更新。
3. 任何一步与在线文档冲突时，**以在线文档为准**，并回填修正本技能。
4. 在 DSH 仓库里开发时，先读仓库根的 `AGENTS.md`，它以仓库为准。

## 任务路由

| 任务 | 打开 |
| --- | --- |
| 第一个插件、`apply`/`inject`、加载与调试、`ctx.effect` 清理 | [references/01-plugin-basics.md](references/01-plugin-basics.md) |
| 给模型注册一个可调用工具、工具参数/输出 schema、后台任务、UI 卡片 | [references/02-tool-authoring.md](references/02-tool-authoring.md) |
| 插件读取用户配置、Schemastery 校验、HMR 热替换 | [references/03-plugin-config.md](references/03-plugin-config.md) |
| 提供/消费 service、`Service` 子类、声明合并、service 隔离 | [references/04-services-and-lifecycle.md](references/04-services-and-lifecycle.md) |
| 监听/派发事件、emit/bail/serial/waterfall、typed events | [references/05-events.md](references/05-events.md) |
| 接入新的 LLM provider、`LlmAdapter`、`StreamChunk` 协议、错误码 | [references/06-llm-adapter.md](references/06-llm-adapter.md) |
| 打包成 bundle、安装进 profile、层叠加顺序、从 GitHub 安装 | [references/07-packaging-and-publish.md](references/07-packaging-and-publish.md) |
| 可替换能力的三层拆分（Definition / Provider / Consumer） | [references/08-capability-layering.md](references/08-capability-layering.md) |

## 实时文档入口

- 中文开发文档首页：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/>
- 英文开发文档首页：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/>
- 参考手册（reference）：<https://deepseek-harness.github.io/deepseek-harness/reference/>
- 仓库源码：<https://github.com/deepseek-ai/deepseek-harness>

站点支持中英双语：中文页面路径为 `/develop/...`、`/reference/...`，英文页面在同样路径前加 `/en`，例如 `/en/develop/framework/service`。每个页面右上角都有语言切换与 `index.md` 原文入口。

## 排查流程

插件出问题时，按下面的顺序定位，不要凭记忆猜：

1. **插件没加载**：看它是否停在 `PENDING`（依赖未就绪）。对照 [04-services-and-lifecycle.md](references/04-services-and-lifecycle.md) 的 Fiber 状态机与 [Cordis tutorial 第 6 章](https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/06-composition-and-hmr) 的“插件永不加载”诊断章节。
2. **配置没生效**：确认配置写在了正确的层（bundle patch / profile patch / `--patch`），见 [07-packaging-and-publish.md](references/07-packaging-and-publish.md) 的层顺序。
3. **类型报错或 API 对不上**：在线查 <https://deepseek-harness.github.io/deepseek-harness/reference/> 下对应的 subsystem 页，那里由代码生成，是最新事实来源。
4. **仍不确定**：直接读仓库源码里同类插件的实现，例如 `packages/shell/tool-bash`（工具）、`packages/llm/llm-deepseek`（LLM adapter）。
