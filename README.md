# dsh-plugin-skills

一个 [Agent Skill](https://agentskills.io/specification)，用于指导 **DeepSeek Harness (DSH)** 插件的开发。

DSH 的插件框架是 Cordis：插件是一个导出 `apply` 函数的 TypeScript 模块，通过 `ctx` 注册工具、service、事件与 LLM adapter，卸载时自动回收。

## 内容

```
SKILL.md                      # 入口：任务路由 + 实时文档入口 + 排查流程
references/
├── 01-plugin-basics.md       # 第一个插件、apply/inject、Fiber 状态机、ctx.effect
├── 02-tool-authoring.md      # defineTool、execute 契约、后台任务、UI 卡片、PTC
├── 03-plugin-config.md       # Config + Schemastery、校验、HMR
├── 04-services-and-lifecycle.md  # Service 子类、inject、声明合并、service 隔离
├── 05-events.md              # emit/bail/serial/waterfall、typed events
├── 06-llm-adapter.md         # LlmAdapter、StreamChunk 协议、LlmError
├── 07-packaging-and-publish.md   # bundle / profile、层顺序、GitHub 安装
└── 08-capability-layering.md # Definition / Provider / Consumer 三角色
```

## 设计

- 遵循 [agentskills.io 规范](https://agentskills.io/specification)：单层 `references/`，渐进披露，`SKILL.md` 保持精简并做路由。
- 每个 reference 顶部都列出**实时在线文档链接**。文档可能比技能新，**冲突时以在线文档为准**。
- 正文为中文，代码与标识符保持英文原文。

## 在线文档

- 中文：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/>
- 英文：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/>
- 参考手册：<https://deepseek-harness.github.io/deepseek-harness/reference/>
- 上游仓库：<https://github.com/deepseek-ai/deepseek-harness>

## License

MIT
