---
title: Monorepo 结构与构建体系
summary: 把 deepseek-harness 的 workspace 分层、vendoring 决策、host/client 双编译面与源码面/产物面规则串成一条脉络，每个事实都指回 docs/ 与 vendor/ 的权威归属地
keywords: monorepo | pnpm-workspace | vendoring | rescope | compiler-face | tsdown | esm
scope: deepseek-harness 仓库结构与构建体系的中文导航层
related_files: pnpm-workspace.yaml | docs/development.md | vendor/README.md | docs/rescope.md | tsdown.config.ts
dependencies: docs/development.md | vendor/README.md | docs/rescope.md
verified_at: 2026-09-06
---

# Monorepo 结构与构建体系

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。构建体系与 TypeScript 布局的权威说明归 [`docs/development.md`](../docs/development.md)，vendoring 清单与同步流程归 [`vendor/README.md`](../vendor/README.md)，rescope 映射归 [`docs/rescope.md`](../docs/rescope.md)，依赖图归生成产物 [`docs/module-graph.md`](../docs/module-graph.md)。本文不复述它们，只把新贡献者最容易卡住的三件事串成一条脉络：**vendoring 为什么这么做、双编译面为什么必须拆、源码面与产物面为什么不能混。**
>
> **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即修正本文。

## 1. 本文定位与上游事实源

> **上游事实源**：[`docs/development.md`](../docs/development.md)（中文版：[`development.zh.md`](../docs/development.zh.md)）的 [TypeScript project layout](../docs/development.md#typescript-project-layout) 一节。

这个仓库的文档治理遵循 one home per fact：每个事实只有一个归属地，别处只链接过去。`docs/` 在配对门禁作用域内已全量双语（仅 manifest 显式豁免的 5 篇除外），所以本文**刻意不解释**下面这些内容，只把它们指出来：

| 你想知道什么 | 去哪里读 |
|---|---|
| 五个 tsconfig 各自的角色、是否形成 program | [`docs/development.md#typescript-project-layout`](../docs/development.md#typescript-project-layout) 的表格 |
| 哪些包拆了 host/client leaf config | 同上一节的正文 |
| 每个 vendored 包的上游仓库、版本、commit | [`vendor/README.md#manifest`](../vendor/README.md#manifest) |
| 相对上游做过哪些本地改动（逐条） | [`vendor/README.md#local-modifications`](../vendor/README.md#local-modifications) |
| 怎么从上游同步一个 vendored 包 | [`vendor/README.md#sync-procedure`](../vendor/README.md#sync-procedure) |
| 新增一个 vendored 包的完整步骤 | [`docs/cookbook/adding-a-vendored-package.md`](../docs/cookbook/adding-a-vendored-package.md) |
| 改名前后的完整映射表 | [`docs/rescope.md#name-mapping`](../docs/rescope.md#name-mapping) |
| 包与包之间的共享实例依赖图 | 生成产物 [`docs/module-graph.md`](../docs/module-graph.md) |
| 当前完整脚本与 gate 清单 | [`package.json`](../package.json) 与 [`scripts/run-gates.ts`](../scripts/run-gates.ts) |

本文的增量只有一件事：把上面这些**分散的片段串成因果链**。第 5、6、7 节是核心，其余章节是把它们放进上下文所需的最小骨架。

本文所有主要小节都以 `> **上游事实源**` 开头。**看到那一行就意味着：这一节写的是索引和串联，权威定义在链接指向的地方。**

## 2. 仓库分层：workspace 成员与它们的角色

> **上游事实源**：[`pnpm-workspace.yaml`](../pnpm-workspace.yaml) 的 `packages` 键；仓库目录职责见 [`AGENTS.md#repository-layout`](../AGENTS.md#repository-layout)。

workspace 成员由 `pnpm-workspace.yaml` 的 `packages` 键定义，一共七条 glob，注释本身就说明了每条的理由：

```yaml
packages:
  - vendor/*
  - packages/*/*
  # The Landlock launcher is developed with its harness consumers but keeps
  # its native build and publication scripts under native/landlock-run.
  - native/landlock-run
  - native/landlock-run/packages/*
  # Product assemblies over the package tier; apps/cli owns the `dsh` bin.
  - apps/*
  - website
  # Deploy root of the single-exe build: a pure dependency manifest whose
  # closure is what the exe bundles and what the Python runtime distributes.
  - python/sdk-runtime
```

（`pnpm-workspace.yaml:1-13`）

按角色理解这七条：

| glob | 实测规模 | 角色 |
|---|---|---|
| `vendor/*` | 9 个包 | Cordis 框架及其基础库的**固定源码副本**，见[第 5 节](#5-vendoring为什么不是-npm-依赖) |
| `packages/*/*` | 50 组 / 255 个包 | 能力层：所有 `@deepseek-ai/dsh-*` 包，见[第 4 节](#4-包的组织能力组与命名) |
| `native/landlock-run` + `native/landlock-run/packages/*` | 1 + 3 | Linux Landlock 启动器的原生构建与发布，源头见 [`native/README.md`](../native/README.md) |
| `apps/*` | 2 个（[`apps/cli`](../apps/cli/package.json) = `@deepseek-ai/dsh`，[`apps/web`](../apps/web/package.json) = `@deepseek-ai/dsh-web-frontend`） | 在包层之上的产品装配，`apps/cli` 拥有 `dsh` 这个 bin |
| `website` | 1 个（`@deepseek-ai/website`） | 选定双语 `docs/` 源的 VitePress 投影 |
| `python/sdk-runtime` | 1 个（`dsh-python-runtime-closure`） | 单文件可执行构建的部署根：一份纯依赖清单，它的闭包就是 exe 打进去、Python runtime 分发出去的东西 |

**注意两个已经不成立的旧认知**：根目录 `examples/` 与 `packages/examples/` 都**已不存在**；`examples` 也**不再是** workspace 成员。如果你在旧笔记或旧 PR 描述里看到它们，那是历史残留。

`apps/cli` 与 `apps/web` 的职责分工不在本文，见 [`apps_cli_and_web.md`](apps_cli_and_web.md)。

## 3. 三条不变量，先记住

> **上游事实源**：[`AGENTS.md#conventions`](../AGENTS.md#conventions) 的 ESM / source plane / compiler faces 三条。

后面三节各展开一条，这里先给出压缩版，方便你带着问题往下读：

1. **vendored 包是源码副本，不是 npm 依赖**——所以你可以直接改 `vendor/*/src`，但必须同步更新 [`vendor/README.md#local-modifications`](../vendor/README.md#local-modifications)（pre-commit 有守卫）。
2. **host 与 client 是两个独立的 TypeScript program**——不是为了并行编译，而是因为两侧在**相同的 cordis `Context` key** 上合并不同服务，一个 program 同时看到两边会直接报冲突。
3. **源码面与产物面绝不混用**——静态门禁与测试一律经 tsconfig `paths` 解析到 `src`；任何消费 `lib/` 的门禁必须显式声明这个依赖。

## 4. 包的组织：能力组与命名

> **上游事实源**：[`packages/README.md#package-groups`](../packages/README.md#package-groups)（组清单与各组职责）、[`packages/AGENTS.md`](../packages/AGENTS.md)（包内约定）。

实测规模（会漂移，用前请自己数一遍：`ls -d packages/*/*/package.json | wc -l`）：

- **50 个能力组**（`packages/<group>/`）
- **255 个包**（`packages/<group>/<pkg>/`）
- 每个 npm 包命名为 `@deepseek-ai/dsh-<name>`
- **每个包只属于一个组**；新包加入已有组，新建组要同时更新组自己的 README 和 `packages/README.md` 的总表

组不是目录整理，是**能力家族**的边界：一个能力接缝（Service Definition / Service Provider / Consumer 三个角色）通常落在同一个组内。为什么这样切分、接缝三角怎么读，见 [`capability_seams.md`](capability_seams.md) 与 [`docs/capability-seams.md`](../docs/capability-seams.md)。

包目录名与包名**不一定一致**。举例：`packages/util/time` 的包名是 `@deepseek-ai/dsh-util-time`，`packages/bundle/acp-app` 的包名是 `@deepseek-ai/dsh-acp-app`。真正的映射表在 [`tsconfig.base.json`](../tsconfig.base.json) 的 `paths` 里，其中大部分由 [`scripts/gen-tsconfig-paths.ts`](../scripts/gen-tsconfig-paths.ts) 生成：

```jsonc
      // BEGIN generated package aliases — pnpm run gen-tsconfig-paths
      "@deepseek-ai/dsh-acp": ["./packages/acp/acp/src"],
      "@deepseek-ai/dsh-acp-app": ["./packages/bundle/acp-app/src"],
      "@deepseek-ai/dsh-agent": ["./packages/core/agent/src"],
```

（`tsconfig.base.json:233-236`）

**要找一个包在哪**：不要靠猜目录，直接在 `tsconfig.base.json` 里搜包名，`paths` 的值就是它的源码路径。

## 5. vendoring：为什么不是 npm 依赖

> **上游事实源**：[`vendor/README.md`](../vendor/README.md)（清单、本地改动日志、同步流程）、[`docs/rescope.md`](../docs/rescope.md)（改名映射）、[`AGENTS.md#vendoring-policy`](../AGENTS.md#vendoring-policy)（策略陈述）。

这是新贡献者最高频的困惑之一：`vendor/cordis/` 里躺着一整份 Cordis 源码，为什么不写成 `"cordis": "^4.0.0-rc.7"`？

### 5.1 三个理由，缺一不可

`vendor/README.md` 开头给出的理由是 harness 要**完整拥有它的框架层**：可审计、可打补丁、可固定。把这三个词拆开看，就能理解为什么 npm 依赖办不到：

- **可审计**：框架是安全边界的一部分。一个 agent harness 的插件加载、生命周期、配置解析全在 Cordis 里；这些代码必须逐行可读、可 diff，而不是 `node_modules` 里一个随时会被 lockfile 换掉的黑盒。
- **可打补丁**：[`vendor/README.md#local-modifications`](../vendor/README.md#local-modifications) 里现在有 19 条本地改动，其中相当一部分是**上游没有、也不一定会接受**的加固——比如 cordis `fiber.ts` 的可重入 dispose 缺口修复、Loader/Include 的事务化配置协调、Include 的持久化写入重试。这些不是"顺手改改"，是 harness 依赖的行为契约。用 npm 依赖 + `patches/` 也能打补丁，但十九条跨文件的语义改动用 patch 文件维护是不可行的。
- **可固定**：清单里记录的是上游 **commit SHA**，不是 semver 范围。见 [`vendor/README.md#manifest`](../vendor/README.md#manifest)。

**对比参照**：仓库里确实存在真正用 patch 文件管理的依赖，就两个，都在 [`patches/`](../patches)：

```yaml
patchedDependencies:
  '@yao-pkg/pkg@6.21.0': patches/@yao-pkg__pkg@6.21.0.patch
  node-pty@1.2.0-beta.15: patches/node-pty@1.2.0-beta.15.patch
```

（`pnpm-workspace.yaml:80-82`）

这个对比就是判断标准：**改动小、局部、面向单个 bug 的，走 `patches/`；需要长期共同演进的框架层，走 `vendor/`。**

### 5.2 rescope 到底改了什么

改名的动机不是审美，是**发布语义**。[`docs/rescope.md`](../docs/rescope.md) 说得很直接：每个 harness 包都把框架声明为 peer dependency，所以发布 harness 就等于把这一层框架一起发布出去；如果沿用上游的包名发布，就会在 registry 上**抢占（squat）**别人的名字。

所以做了机械改名：`cordis` → `@deepseek-ai/cordis`，`@cordisjs/plugin-<x>` → `@deepseek-ai/cordis-plugin-<x>`。完整映射表在 [`docs/rescope.md#name-mapping`](../docs/rescope.md#name-mapping)，本文不重复。

真正值得记住的是**改名不碰什么**——这一节 [`docs/rescope.md#what-the-rename-does-not-touch`](../docs/rescope.md#what-the-rename-does-not-touch) 是最容易踩坑的地方，摘出四个高频误判：

- **目录名不变**：`vendor/hmr/` 还叫 `vendor/hmr/`，这样清单读起来仍然是一份上游快照。
- **依赖 range 不变**，只换 key：`"cordis": "^4.0.0-rc.7"` 变成 `"@deepseek-ai/cordis": "^4.0.0-rc.7"`。
- **Loader 的 `cordis:` 前缀不是包名**：`cordis:include`、`cordis:group` 是协议前缀，不改。`cordis.yml` 配置族（含 `*.cordis.yml`、`cordis.patch.yml`）也不改。
- **上游运行时标识不改**：Schemastery 的 `Symbol.for('schemastery')` 和它的 `vendor:` 元数据字段保持上游取值。

改名**不由人手工做**。[`scripts/rescope-vendor.ts`](../scripts/rescope-vendor.ts) 拥有这份映射，`pnpm run rescope-vendor:check` 在 hygiene 门禁里断言改名后的状态。命令四件套（report / apply / check / reverse）见 [`docs/rescope.md#applying-verifying-and-reverting`](../docs/rescope.md#applying-verifying-and-reverting)。

### 5.3 改了名之后，pnpm 怎么还能解析

这是链条上最后一环，也是最容易被略过的一环。vendored 包**保留了上游的 semver range**（上一节说过 range 不变），但本地构建必须让这些 range 解析到 workspace 里的固定源码，而不是去 registry 拉一份。两个键合力做到这点：

```yaml
# Vendored framework packages keep their upstream semver ranges, while local
# builds must resolve those matching names to this workspace's pinned sources.
linkWorkspacePackages: true

overrides:
  '@deepseek-ai/cosmokit': 'link:vendor/cosmokit'
  '@deepseek-ai/schemastery': 'link:vendor/schemastery'
```

（`pnpm-workspace.yaml:15-21`）

`linkWorkspacePackages: true` 让匹配名字的 semver range 落到 workspace 包上（**包括从构建产物 `lib/` 发出的 import**）；`overrides` 里的两条 `link:` 是针对 cosmokit 和 schemastery 的额外钉死。

这条链有门禁保护：hygiene 里的 `verify-vendored-links`（[`scripts/verify-vendored-links.ts`](../scripts/verify-vendored-links.ts)）断言每个 vendored 名字在 `pnpm-lock.yaml` 里都解析成 workspace `link:`，且旁边**没有** registry 副本。这条断言存在的原因很实际：一旦某个 vendored 包同时有 link 副本和 registry 副本，模块单例（比如 cordis 的 `Context`）就会被加载两份，症状会以极难定位的形式出现在运行时。

Schemastery 还有一个额外细节：它的 manifest 声明了条件 `exports`（import → `.mjs`，require → `.cjs`）。原因写在 [`vendor/README.md`](../vendor/README.md) 里——pnpm link 的是目录本身，没有 `exports` 时 Node 的 ESM 解析器会回退到 `main` 加载 CJS 入口，而那个入口里对 cosmokit 的惰性 `require` 会和同一个 linked module 的 ESM 加载在 module-hook 宿主（vitest）下产生竞态。

### 5.4 你要动 `vendor/` 时的规矩

- **改 `vendor/*/src` 必须同时更新 [`vendor/README.md#local-modifications`](../vendor/README.md#local-modifications) 的日志**。lefthook 的 pre-commit 有 vendor manifest 守卫（`scripts/check-vendor-manifest.sh`），漏了会被挡下。日志要求"穷尽"：每一处相对上游的偏离都必须列出来。
- **同步上游**走 [`vendor/README.md#sync-procedure`](../vendor/README.md#sync-procedure) 的五步，本文不重述。同步完要 `pnpm run rescope-vendor --apply` 重新改名，再按它打印的提示做 `pnpm install`、`pnpm run gen-third-party-notices`、`pnpm run verify-translation-pairing --write`。
- **新增一个 vendored 包**不要照抄已有包，走 [`docs/cookbook/adding-a-vendored-package.md`](../docs/cookbook/adding-a-vendored-package.md)。
- 有些包是**刻意不 vendor** 的（`reggol`、`@cordisjs/utils`、`@cordisjs/element`、`@cordisjs/unyaml`），清单里注明了"已验证此集合未使用"。想引入之前先确认这个判断是否还成立。

## 6. 双编译面：host / client

> **上游事实源**：[`docs/development.md#typescript-project-layout`](../docs/development.md#typescript-project-layout)（五个 tsconfig 的角色表与三条纪律）、[`AGENTS.md#conventions`](../AGENTS.md#conventions) 的 "Keep compiler faces explicit"、[`packages/AGENTS.md`](../packages/AGENTS.md) 的 package tsconfig 一条。

### 6.1 为什么必须拆

拆分的理由**不是**编译性能，也**不是** DOM 类型与 Node 类型不兼容（那个用 `lib` 就能解决）。真正的理由写在配置文件自己的注释里：

```jsonc
{
  // Host aggregate (one of the two check units; see tsconfig.json). packages/client
  // type-checks in tsconfig.client.json: the two sides merge cordis Context under
  // the same keys, one program cannot see both.
```

（`tsconfig.host.json:1-4`）

client 侧说得更具体，点名了冲突的 key：

```jsonc
{
  // Client-side typecheck aggregate: packages/client tests (.ts and .tsx).
  // Split from the host aggregate because both sides merge cordis Context
  // under the same keys (sessions, loader) with different services; shared
  // leaves (session/llm/tools/...) build once and are referenced by
  // both programs through each client package's own references.
```

（`tsconfig.client.json:1-6`）

翻译成一句话：**host 和 client 都通过 declaration merging 往 cordis `Context` 上挂服务，而且用的是同一批 key（`sessions`、`loader` 等），挂的却是不同的类型。**一个 `ts.Program` 同时看到两套合并，就会报冲突。

关键在于 `docs/development.md` 补充的那句：**这个冲突只存在于 `ts.Program` 内部，模块解析永远不会触发它。**这解释了为什么 solution root 可以同时引用两个 aggregate、为什么一份 `paths` facade 可以横跨两侧——只要它们不被压进同一个 program。

所以 solution root 是**空 program**：

```jsonc
{
  // Solution file: the whole-repo graph for `tsc -b tsconfig.json` and the
  // tsserver entry. `extends` carries the base paths for get-tsconfig
  // consumers — tsx running scripts/ (no nearer tsconfig)
  // resolves workspace imports through this file. `files: []` keeps it
  // program-less, so the host/client cordis Context merges never meet.
  // NEVER add include/files entries, and NEVER flatten this solution into a
  // single ts.Program (scripts seed tsconfig.host.json or tsconfig.client.json).
```

（`tsconfig.json:1-8`）

而 base 也**不许有 `include`**，因为它同时充当 vite-tsconfig-paths 的解析门面：

```jsonc
{
  // Doubles as the resolution facade for vite-tsconfig-paths (vitest configs
  // point here). NEVER add include/files to this file: it would leak into
  // every extending package project and narrow the facade's match-all scope.
```

（`tsconfig.base.json:1-4`）

### 6.2 什么时候我需要关心它

绝大多数时候不需要。判断顺序如下：

**情形一：普通包（占绝大多数）。** 一个 `tsconfig.json`，`extends` base（client 包 extends `tsconfig.base.client.json`），`rootDir: src`、`outDir: lib/types`，然后列 references：

```jsonc
{
  "extends": "../../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "lib/types"
  },
```

（`packages/core/session/tsconfig.json:1-6`）

你要做的只有一件事：**把这个包注册进恰好一个 aggregate**——host 包写进 `tsconfig.host.json` 的 `references`，client 包写进 `tsconfig.client.json` 的 `references`。

**情形二：既有 Node loader 入口又有浏览器入口的 client 插件。** 这**不是**拆的理由。`docs/development.md` 明确写了：普通 client 插件在 Client 构建阶段一次产出两种运行时产物，tsconfig 不拆。

**情形三：真的有两个编译面。** 判据是这个包的 Host 侧和 Client 侧**作为程序**需要分别类型检查——比如 `api/remotes` 的 Host 入口要参与 Host 的 Typert 图，而它的 Client 入口要 import 生成出来的 `/remote` 声明；又比如 `session-log-export` 要把 Node 的归档生产挡在浏览器控制器之外。清单以 [`docs/development.md#typescript-project-layout`](../docs/development.md#typescript-project-layout) 为准，但**注意该处的数字已过期**：上游写"Six packages"并列了 6 个，磁盘实测是 **8 个**（另有 `client/file-upload` 与 `experimental/inspector`）。不要抄数字，实测一行就够：`for f in packages/*/*/tsconfig.host.json; do d=$(dirname "$f"); [ -f "$d/tsconfig.client.json" ] && echo "$d"; done`。会漂移的原因也在同一段里——门禁靠两个 leaf config 的存在性自动发现拆分包，**但文档里的数字没有门禁**。

这类包的包根 `tsconfig.json` 退化成**纯 solution**：

```jsonc
{
  "files": [],
  "references": [
    {
      "path": "./tsconfig.host.json"
    },
    {
      "path": "./tsconfig.client.json"
    }
  ]
}
```

（`packages/api/gateway/tsconfig.json`，全文）

引用它的一方**必须指名对应的 leaf**（`./tsconfig.host.json` 或 `./tsconfig.client.json`），不能指 solution root，也不能指反了。workspace `constraints` 门禁会走一遍可达的 project reference 图，按引用方自己的编译面检查这一点；它靠"是否同时存在两个 leaf config"来发现拆分包，所以**新拆的包会自动进入门禁**，不需要手动登记。

**情形四：你在写会构建全仓 `ts.Program` 的脚本。** 必须显式 seed `tsconfig.host.json` 或 `tsconfig.client.json`，**绝不能 seed 根 solution**——那会把两个 aggregate 压成一个 program，`Context` 合并当场冲突。

**例外：shared leaf。** 三个包（`host/webserver`、`compaction/compaction`、`typert/registry`）被两个 aggregate 同时引用，因为两侧都要对同一份源码做类型检查。它们能这样是因为自身不做 cordis `Context` 合并（`tsconfig.client.json:42` 的注释说 `webserver` "零 workspace 依赖"，准确说是除 vendored cordis / schemastery 外没有 `dsh-*` workspace 依赖），不会把 host 侧的 `Context` 增强拖进 client program。`tsconfig.client.json` 的 references 里给每一条都写了理由，值得一读。

## 7. 源码面与产物面

> **上游事实源**：[`AGENTS.md#conventions`](../AGENTS.md#conventions) 的 "Source plane vs artifact plane, never mixed"、[`docs/development.md#typescript-project-layout`](../docs/development.md#typescript-project-layout) 的末段、[`docs/testing.md#test-resolution-source-plane-only`](../docs/testing.md#test-resolution-source-plane-only)。

### 7.1 两个面分别是什么

**源码面**：workspace import 通过 `tsconfig.base.json` 的 `paths` 直接解析到各包的 `src`。这张表的注释说明了它为什么长这样：

```jsonc
    // Source-level resolution for every repo-local graph. Project references,
    // not declaration path aliases, keep each package/vendor source compiled
    // under its own tsconfig boundary.
    "paths": {
      "@deepseek-ai/cordis": ["./vendor/cordis/src"],
```

（`tsconfig.base.json:27-31`）

注意这句区分：`paths` 负责**解析**，project references 负责**编译边界**。`paths` 指向 `src` 而不是 `.d.ts`，所以每个包的源码仍然在自己的 tsconfig 边界下被编译。

**产物面**：分两层，产物路径不同、消费者也不同。

| 产物 | 由谁产出 | 给谁用 |
|---|---|---|
| `lib/types/**`（JS + `.d.ts` + `.d.ts.map`） | `tsc -b` | 下游 TypeScript 的类型解析（`package.json` 的 `types` 字段指向它）；tsdown 的输入 |
| `lib/*.js`（打包后的运行时入口） | tsdown | 真正被 Node 加载执行的东西（`main` / `exports` 的 `default` 指向它） |

以 `packages/core/session/package.json:16-24` 为例，`exports` 的两条腿正好落在这两层上：`"types": "./lib/types/index.d.ts"` 与 `"default": "./lib/index.js"`。

### 7.2 tsc 与 tsdown 的分工

构建是四段流水线，源自 `package.json` 的脚本：

```json
    "build:lib": "npm run build:lib:host && npm run build:lib:client",
    "build:lib:host": "node --max-old-space-size=4096 ./node_modules/typescript/bin/tsc -b tsconfig.host.json && tsdown --env.DSH_BUILD_FACE host",
    "build:lib:client": "tsc -b tsconfig.client.json && tsdown --env.DSH_BUILD_FACE client",
```

（`package.json:22-24`）

tsdown 两遍用的是**同一份完整 workspace 匹配**，不做产物扫描、也不维护 host/client 包名单：

```ts
    workspace: ['vendor/*', 'packages/*/*', 'apps/cli'],
    entry: client ? '' : ['lib/types/{index,invariant,startup}.js'],
    outDir: 'lib',
    format: ['esm'],
    platform: 'node',
    target: 'es2024',
    fixedExtension: false,
    dts: false,
    clean: false,
    plugins: client ? [] : [typertPlugin({ mode: 'workspace', faces: ['host'] })],
```

（`tsdown.config.ts:19-28`）

从这段配置能直接读出三件事：

1. **`entry` 指向 `lib/types/*.js`**——tsdown **只消费上一段 tsc 已经 emit 出来的 JavaScript**，它不自己编译 TypeScript。所以 tsc 必须先跑完，`dts: false` 也是因为声明文件归 tsc 所有。
2. **面的选择靠 `DSH_BUILD_FACE` 环境变量**下发到各包的 package-local tsdown config，而不是靠一份中央名单。
3. **Typert 只在 Host 阶段跑**（`plugins: client ? [] : [...]`，`faces: ['host']`）。它分析 Host 类型，生成 Host 反射产物和 Host-for-Client 的 Remote 投影；Client 阶段完全不启动 Typert。生成物在两侧怎么装配见 [`docs/api-gateway.md`](../docs/api-gateway.md)。

这个顺序解释了一个乍看奇怪的脚本定义：

```json
    "typecheck": "npm run build:lib:host && npm run typecheck:contracts-ready",
    "typecheck:contracts-ready": "tsc -b tsconfig.client.json",
```

（`package.json:28-29`）

**`typecheck` 会真的构建**。因为 Client 侧要 import Typert 生成的 Remote 声明，而那些声明只在完整的 Host lib 阶段（tsc + tsdown + Typert）之后才存在。`lint` 和 `doc-typecheck` 同理。名字带 `:contracts-ready` 的内部脚本则假定调用方（公开命令或调度门禁）已经保证了这个前置。

### 7.3 "绝不混用"在实践中怎么落地

规则一句话：**静态门禁与测试走源码面，且必须在干净树上通过；消费 `lib/` 的门禁必须显式声明这个依赖。**

落地成三条可操作的判断：

- **写测试时**：vitest 的所有配置都把 vite-tsconfig-paths 指向 `tsconfig.base.json`，bare workspace import 解析到 `src`，**永远不经过 package `exports` 走到 `lib/`**。原因见 [`docs/testing.md#test-resolution-source-plane-only`](../docs/testing.md#test-resolution-source-plane-only)：那里的陈旧产物会让模块单例被加载第二份。想消费产物必须显式：`lib` 模式子进程，或 built smoke 测试。
- **写门禁脚本时**：如果它读 `lib/`，就要在脚本依赖链上把 `build` 声明出来。`pnpm run hygiene` 就是这类——它含 `publint`（拿 `package.json` 入口对着构建出的 `lib/*.js` 校验）和 `verify-node-next-types`（拿构建出的声明对着一个临时 NodeNext 消费者校验）。**新 clone 的 worktree 在跑 `pnpm run build` 之前没有任何 bundled JS 和声明**，所以这类检查在干净树上必然失败——这不是 bug，是它按定义就依赖产物面。
- **排查诡异行为时**：如果你看到"同一个 service 出现两个实例"或"改了源码但行为没变"，第一反应应该是**两个面串了**——要么测试意外走到了陈旧的 `lib/`，要么某个 vendored 包同时存在 link 副本和 registry 副本（见 [5.3](#53-改了名之后pnpm-怎么还能解析)）。更多症状与处理见 [`troubleshooting.md`](troubleshooting.md)。

唯一的正式例外是生成的 Host-for-Client Remote 声明，处理方式见上一节的 `:contracts-ready` 约定。

## 8. ESM 全仓约束

> **上游事实源**：[`AGENTS.md#conventions`](../AGENTS.md#conventions) 的 ESM 一条、[`docs/testing.md#test-subprocess-launch-modes`](../docs/testing.md#test-subprocess-launch-modes)。

四条约束，都是硬的：

1. **每个包 `"type": "module"`**（例：`packages/core/session/package.json:13`、`apps/cli/package.json:13`、`vendor/cordis/package.json:14`）。
2. **跨包用包名，包内相对导入带 `.ts` 后缀**。这是 `tsconfig.base.json` 里 `allowImportingTsExtensions` 与 `rewriteRelativeImportExtensions` 两个开关配合的结果：源码写 `.ts`，TypeScript 把 emit 出的 JS 重写成 `.js`，而声明文件保留显式的、NodeNext 安全的 `.ts` 说明符。vendored 包也做了同样的改造，见 [`vendor/README.md#local-modifications`](../vendor/README.md#local-modifications) 第 4 条。
3. **`dsh` CLI 的源码启动走 tsx 的 ESM-only 钩子**（`node --import tsx/esm`）。推论很重要：**这条路径能到达的模块必须保持 ESM，不能有 CJS-only 导出**。Node 的原生 TypeScript 模式在本仓支持的引擎区间内不可用，所以没有别的选项。
4. **配置类子进程跑构建后的 `lib/`，在纯 Node 下**；源码回归测试用它们声明的 launcher。不要为这些子进程手写 `--import tsx`。

两个 aggregate 都把 `rewriteRelativeImportExtensions` 关掉（`tsconfig.host.json:8`、`tsconfig.client.json:10`），因为它们是 `noEmit` 的纯检查单元，不产出需要重写的 JS。

## 9. 常用构建命令与它们的产物

> **上游事实源**：[`AGENTS.md#commands`](../AGENTS.md#commands)（命令速查）、[`package.json`](../package.json) 与 [`scripts/run-gates.ts`](../scripts/run-gates.ts)（当前脚本与 gate 清单）、[`docs/development.md#daily-commands`](../docs/development.md#daily-commands)。

命令清单本身归 `AGENTS.md#commands`，本文不复制。这里只补一张**"这条命令会留下什么"**的对照表，因为这是新贡献者最常问、而命令清单本身不回答的问题：

| 命令 | 落地产物 | 要点 |
|---|---|---|
| `pnpm install` | `node_modules` + workspace link；顺带装 worktree 本地的 lefthook 钩子和翻译配对 merge driver | 钩子安装由 `scripts/install-lefthook.mjs` 负责，缓存恢复或跳过 `postinstall` 会导致缺失，手动补跑即可 |
| `pnpm run typecheck` | **有产物**：完整 Host lib 阶段（`lib/types/**` + `lib/*.js` + Typert 生成物），然后跑 Client tsc | 见 [7.2](#72-tsc-与-tsdown-的分工)；它不只是检查 |
| `pnpm run build` | 全部四段 + Web 构建；写一条 gitignore 的完整构建记录，把版本/commit/dirty 标记绑定到产物 | 发布打包和 built-Web 测试会拒绝缺失记录或被后续部分构建改动过的产物 |
| `pnpm run build:official` | 与 CI/发布产物构建等价的跨平台本地版本 | 不带本地 dirty 标记 |
| `pnpm run hygiene` | 无产物，但**消费产物** | 含 `publint` 与 `verify-node-next-types`；干净树上必须先 `build` |
| `pnpm run clean` | 删除构建输出，以及被删包留下的安全残留 | 由 [`scripts/clean.ts`](../scripts/clean.ts) 实现 |
| `pnpm run gen-module-graph` | 重新生成 [`docs/module-graph.md`](../docs/module-graph.md) | 见[第 10 节](#10-依赖图怎么看) |
| `pnpm run rescope-vendor:check` | 无产物；断言 rescope 后状态 | 在 hygiene 门禁内 |

`pnpm run dev:web` 需要一次完整构建留下的产物树，但它在启动时只采样一次当前版本和 Git 状态，然后在整个会话里共享这个环境；它**不**校验完整构建记录，因为 watcher 阶段会重写记录里的产物。

要跑哪些检查、跑到什么程度，归 [`AGENTS.md#run-relevant-checks-locally`](../AGENTS.md#run-relevant-checks-locally) 和 [`quality_gates.md`](quality_gates.md)。CI 的 gate 编排见 [`docs/development.md#ci-gates`](../docs/development.md#ci-gates)。

## 10. 依赖图怎么看

> **上游事实源**：生成产物 [`docs/module-graph.md`](../docs/module-graph.md)（由 [`scripts/gen-module-graph.ts`](../scripts/gen-module-graph.ts) 生成）；更宏观的图集见 [`docs/graph-atlas.md`](../docs/graph-atlas.md)。

[`docs/module-graph.md`](../docs/module-graph.md) 是**生成文件，不要手改**，本文也不复制它的任何内容。读它之前需要知道两件事：

1. 它画的是 **peer dependency**，不是普通运行时依赖。一条 `a --> b` 的边意思是"`a` 需要与 `b` **共享同一个实例**"。普通运行时依赖和纯开发期关系都不在图里。
2. 图按 `packages/<group>/<pkg>` 分组，节点名省略了 `@deepseek-ai/dsh-` 前缀。

所以它回答的问题是"**改这个包会影响哪些共享实例**"，而不是"谁 import 了谁"。要回答后者，用 `tsconfig.base.json` 的 `paths` 加上各包 `tsconfig.json` 的 `references`。

修改包依赖后，用 `pnpm run gen-module-graph` 重新生成，不要手工编辑。

## 11. 延伸阅读

**上游权威源（英文 + `.zh.md` 中文版都有）**

- [`docs/development.md`](../docs/development.md) — 构建体系与 TypeScript 布局的**主事实源**，本文所有构建相关陈述的出处
- [`vendor/README.md`](../vendor/README.md) — vendoring 清单、19 条本地改动日志、同步流程
- [`docs/rescope.md`](../docs/rescope.md) — rescope 完整映射与应用/校验/回退命令
- [`docs/cookbook/adding-a-vendored-package.md`](../docs/cookbook/adding-a-vendored-package.md) — 新增 vendored 包的步骤指南
- [`docs/testing.md`](../docs/testing.md) — 测试解析面与子进程启动模式
- [`docs/api-gateway.md`](../docs/api-gateway.md) — Typert 生成物在 Host/Client 两侧的装配与 Web 构建顺序
- [`docs/module-graph.md`](../docs/module-graph.md) — 共享实例依赖图（生成产物）
- [`packages/README.md`](../packages/README.md) — 能力组总表与各组职责（注意该表当前列 49 组，漏了 `packages/mcp/`；磁盘实测 50 组）
- [`packages/AGENTS.md`](../packages/AGENTS.md) — 包内约定，含 package tsconfig 一条
- [`AGENTS.md`](../AGENTS.md) — 仓库布局、命令速查、Conventions、Vendoring policy

**dev_docs 兄弟文档（中文导航层）**

- [`AI_Coding_Context.md`](AI_Coding_Context.md) — 上手这个仓库的总入口
- [`architecture_overview.md`](architecture_overview.md) — 插件树、profile/bundle、回合流程；本文的结构视角在那里有运行时对应
- [`capability_seams.md`](capability_seams.md) — 能力接缝三角，解释[第 4 节](#4-包的组织能力组与命名)的分组逻辑
- [`plugin_development_guide.md`](plugin_development_guide.md) — 写一个新包时的完整流程，含 tsconfig 注册
- [`apps_cli_and_web.md`](apps_cli_and_web.md) — `apps/cli` 与 `apps/web` 的职责与启动路径
- [`quality_gates.md`](quality_gates.md) — 门禁全景，含哪些门禁消费产物面
- [`testing_guide.md`](testing_guide.md) — 测试分层与解析面
- [`sdk_and_protocols.md`](sdk_and_protocols.md) — SDK 与协议层，Typert/Remote 的消费侧
- [`troubleshooting.md`](troubleshooting.md) — 构建、解析、单例重复等症状的排查路径
