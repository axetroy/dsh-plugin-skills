# dsh-plugin-skills

一个 [Agent Skill](https://agentskills.io/specification)，用于指导 **DeepSeek Harness (DSH)** 插件的开发。

DSH 的插件框架是 Cordis：插件是一个导出 `apply` 函数的 TypeScript 模块，通过 `ctx` 注册工具、service、事件与 LLM adapter，卸载时自动回收。

## 安装

推荐用 [`skills` CLI](https://github.com/vercel-labs/skills)（Vercel Labs 出品的 agent skills 生态 CLI）安装。它支持 Claude Code、Codex、Cursor、OpenCode 等 80 多个 agent，会自动把技能放进对应 agent 的 skills 目录。

要求 Node.js **22.20 或更高版本**。在你的项目目录下执行：

```sh
npx skills add axetroy/dsh-plugin-skills
```

交互式安装会让你选择要装的技能与目标 agent。常用变体：

```sh
# 只装到指定 agent
npx skills add axetroy/dsh-plugin-skills -a claude-code

# 装到全局（所有项目可用），非交互
npx skills add axetroy/dsh-plugin-skills -g -y

# 先看看仓库里有哪些技能，不安装
npx skills add axetroy/dsh-plugin-skills --list
```

默认装到**当前项目**的 `.agents/skills/`（可随项目提交、团队共享）；加 `-g` 装到用户目录（跨项目可用）。

安装后用 `npx skills list` 查看，用 `npx skills update dsh-plugin-skills` 更新，用 `npx skills remove dsh-plugin-skills` 卸载。

### 手动安装

不想用 CLI 时，直接把技能目录放进 agent 的 skills 目录即可：

```sh
git clone --depth 1 https://github.com/axetroy/dsh-plugin-skills.git \
  ~/.claude/skills/dsh-plugin-skills
```

把路径换成你所用 agent 的 skills 目录。技能名必须与目录名一致（`dsh-plugin-skills`）。

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
