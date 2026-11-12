# 打包与发布：bundle、profile 与层顺序

> **实时文档（先读，可能与本文不同步）**
> - 打包并安装插件：<https://deepseek-harness.github.io/deepseek-harness/develop/basic/publish>
> - 新增 workspace package cookbook（含插件展示元数据）：<https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/adding-a-package>
> - 插件索引（插件市场）：<https://deepseek-harness.github.io/deepseek-harness/guide/plugin-market>
> - 插件配置目录：<https://deepseek-harness.github.io/deepseek-harness/reference/config-catalog>
> - 英文版：<https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/publish>

## 两个概念，两份 manifest

两者都由 `package.json` 描述，但 `dsh` key 下承载不同类型的 manifest，回答不同问题：

- **bundle** — 一个携带配置层的 npm 包。manifest 声明 `dsh.bundle`，回答“这个包贡献了什么”：一个插入或覆盖插件行的 patch 文件。
- **profile** — `$DSH_HOME/profiles/<name>` 下的目录，描述一次可运行的组合。manifest 声明 `dsh.profile`，回答“哪些 bundle 按什么顺序组成这套配置”。

bundle 是你编写和分发的；profile 是用户用 `dsh --profile <name>` 启动的。**没有东西同时是两者。**

## bundle manifest

```
hello-plugin/
├── package.json       # 声明 dsh.bundle
├── cordis.patch.yml   # profile 列出该 bundle 时应用的层
└── index.js           # patch 行引用的插件模块
```

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } },
  "engines": { "dsh": ">=0.2.0 <0.3.0" }
}
```

`cordis.patch.yml` 是一个 YAML 数组；与 `--patch` overlay 一样，区别在于插件行**按包名**引用，以便 Node 解析到已安装的代码：

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin
```

`patch` 也接受有序文件列表，例如 `["./base.patch.yml", "./web.patch.yml"]`：launcher 按该顺序把它们当作**一个层**应用，每个文件的相对插件路径相对该文件解析。

用 `engines.dsh` 的 SemVer range 声明 bundle 支持的 DSH 版本。安装与 profile 启动会拒绝运行中的 DSH 不满足该 range 的 bundle；没有 `engines.dsh` 时改查其 `@deepseek-ai/dsh*` peer 依赖 range。

没有 `dsh.bundle` 声明的包仍可安装，但只作为普通依赖：`dsh plugin` 打印警告且不激活任何层。库（被其他插件包 import）用这种格式。

## profile manifest

profile 目录放两个文件：

- `package.json` — profile 的 out-of-tree 插件依赖（pnpm 管理）+ `dsh.profile` manifest 及其有序 `bundles` 列表。
- `cordis.patch.yml` — 用户自己的 patch 层，在所有 bundle 层之后应用。

**不要手写 profile manifest**：`dsh --profile <name> --from-default-profile <template>` 可以从内置应用模板创建或重置；`dsh plugin` 会创建 base-backed profile 并维护其已安装 bundle 列表。

## 安装进 profile

`dsh plugin --profile <name> <args...>` 转发给 profile 目录下的 pnpm，因此所有 pnpm 动词都可用：

```sh
dsh plugin --profile demo add ./hello-plugin
```

首次使用会初始化 profile（以 `@deepseek-ai/dsh-base` 作为第一个 bundle），pnpm 链接该 checkout，`dsh` 因包声明了 `dsh.bundle` 而把该 bundle 追加到 `dsh.profile.bundles`：

```json
{
  "name": "dsh-profile-demo",
  "private": true,
  "dependencies": { "dsh-hello-plugin": "link:/path/to/hello-plugin" },
  "dsh": {
    "profile": {
      "bundles": ["@deepseek-ai/dsh-base", "dsh-hello-plugin"]
    }
  }
}
```

插件必须与宿主共享实例的 dsh 包，要**同时**声明在 `peerDependencies` 和 `devDependencies`。独立版本的第三方依赖与无状态 dsh 工具放 `dependencies`。

先不启动、只验证层，再启动：

```sh
dsh --profile demo --dump-config   # 应显示 "# == dsh-hello-plugin" 层
dsh --profile demo
```

移除：`dsh plugin --profile demo remove dsh-hello-plugin`（同时移除依赖与层）。

## 加载顺序（层叠加）

有效配置从空根开始，按顺序应用：

1. `dsh.profile.bundles` 中列出的每个 bundle patch，按列表顺序（`@deepseek-ai/dsh-base` 最先，然后按安装顺序）。
2. profile 自己的 `cordis.patch.yml`。
3. home 级 `$DSH_HOME/cordis.patch.yml`（所有 profile 共享的机器本地偏好）。
4. 每个 `--patch <path>` overlay，按 argv 顺序。

**后面的层按行胜出**，且 patch 会**整体替换**该行的 `config`，而不是深合并 key。对 bundle 作者的两个后果：

- 你的 patch 可以按 `id` 覆盖前面层的行，但必须**重述该行需要的每个 key**，而不只是改动的那一个。
- 用户可以在自己 profile 的 `cordis.patch.yml` 里覆盖你的行而不碰你的包，所以配置默认值要选用户大概率会保留的，其余交给 schema。

in-box bundle 名总是从 dsh 安装本身解析；pnpm 只管 out-of-tree 包。

## 从 GitHub 安装的构建脚本陷阱

```sh
dsh plugin --profile demo add github:you/hello-plugin
```

git 安装拉的是**源码而不是构建产物**：不会跑你的 `build` 脚本，因此 TypeScript 包到手时没有 `lib/` 产物、加载失败。两边各要做一件事：

- **作者**：提供 `prepare` 脚本（pnpm 在 git 安装后执行），从源码构建发布入口点，且必须自包含——不能假设存在 sibling monorepo checkout。用专门的 tsdown 配置转译 `src/`，不用 project references、不做类型检查。
- **用户**：允许该构建。pnpm ≥10 在显式允许前拒绝执行 git 依赖的 `prepare`，因此第一次 `add` 会失败；`dsh` 会指出修法——把 pnpm 打印的包 key 原样复制进 profile 的 `pnpm-workspace.yaml`：

  ```yaml
  allowBuilds:
    dsh-hello-plugin: true
  ```

  然后重跑 `add`。

> **把该允许视为“安装时在本机执行该包代码”的许可**，它在 agent 沙箱之外。只允许你信任其源码的包，并钉住 commit（`github:you/hello-plugin#<sha>`），以免后续 push 悄悄改掉实际运行的代码。

不想要求用户授权，就分发构建产物（两种都不需要构建许可）：

- **发布到 npm**，在 `pnpm publish` 时构建好 `lib/`；`dsh plugin add your-package` 直接装预构建代码。
- **提供 tarball**（`pnpm pack`）；用户 `dsh plugin add ./hello-plugin-0.1.0.tgz`。

## 插件展示元数据

在包的 `locale/en.json` 里定义标题与描述，其他语言文件（如 `locale/zh.json`）用同样字段：

```json
{ "meta": { "title": "Workspace Tools", "description": "Tools for your workspace." } }
```

同时在 `package.json` 里导出这些资源：`"./locale/*.json": "./locale/*.json"`，并把 `locale/*.json` 列入 `files`。图标用顶层 `"icon": "./icon.svg"`（或导出 `./icon`）；支持 SVG/PNG/JPEG/WebP，上限 256 KiB。可用 `meta.detail` 指向包内 Markdown 详情文档。

仓库内开发时跑 `pnpm run verify-package-meta` 与 `pnpm run check:i18n` 校验字段、资源导出与发布覆盖。

## 插件索引（可选分发渠道）

插件索引是一个 JSON 文件，列出可安装的 bundle（标题、图标、作者、版本、来源）。打开实验性的 **Custom plugin indexes** bundle 后，Plugins 页出现 **Discover** 标签。把索引文件和图标目录放在任意静态 host 上即可发布。索引是第三方内容，DSH 只校验文件格式、不校验其中的声明，只添加你信任的来源。详见 <https://deepseek-harness.github.io/deepseek-harness/guide/plugin-market>。

## 常见坑

- **patch 行引用相对路径**：bundle 的 patch 行要用**包名**，让 Node 解析到安装位置。
- **覆盖了某行却漏写 key**：patch 整体替换 `config`，漏写的 key 会消失。
- **忘了 `engines.dsh`**：安装时不拦版本，运行期才炸。
- **git 安装后加载失败**：多半是没构建产物，检查 `prepare` 脚本与 `allowBuilds`。
- **以为 `--patch` 能改 profile 目录**：它只贡献配置，不改变模块解析目录。
