---
title: AI 编码硬规则（合并版）
summary: 在 deepseek-harness 改代码前必须知道的红线、架构不变量与提交义务，每条附上游出处与自查方式。
keywords: ai-rules | agents-md | invariant | agent-note | pre-stable | model-visible | capability-seam
scope: 面向在本仓库改动代码的 AI 与人类贡献者的强制约束合集
related_files: AGENTS.md | docs/architecture.md | docs/AGENTS.md | docs/testing.md | .agents/notes/README.md
dependencies: AGENTS.md | docs/architecture.md | docs/AGENTS.md
verified_at: 2026-09-06
---

# AI 编码硬规则（合并版）

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。本仓库规则的唯一权威是根目录 [`AGENTS.md`](../../../AGENTS.md)（`CLAUDE.md` 是它的符号链接，改要改真身）。本文把散落在 `AGENTS.md`、[`docs/AGENTS.md`](../../../docs/AGENTS.md)、[`docs/architecture.md`](../../../docs/architecture.md)、[`docs/testing.md`](../../../docs/testing.md)、[`.agents/notes/README.md`](../../../.agents/notes/README.md) 中的强制条款，按"你什么时候会撞上它"重新编排成一份中文速查。
>
> **与上游冲突时一律以上游为准**，并立即修正本文。本文**不新增任何规则**——每条都能追溯到上游出处。
>
> 用法：动手前扫一遍 §1 红线与 §7 速查表；改到具体子系统时再看对应条目。

## 1. 红线：违反即需回滚重做

> **上游事实源**：[`AGENTS.md` 的 `## Pre-stable APIs and released Session data`](../../../AGENTS.md#pre-stable-apis-and-released-session-data)、[`## Conventions`](../../../AGENTS.md#conventions)、[`## Secrets / .env`](../../../AGENTS.md#secrets--env)

### R1 — 绝不提交凭证

`DEEPSEEK_API_KEY`、`DEEPSEEK_BASE_URL` 及任何密钥只从环境变量或根 `.env` 读取。文档、测试夹具、示例配置、Agent Note 里都不得出现真实值。（具体到 `dev_docs` 用什么占位，是本地写作约定，见 §6 第 7 条。）

CI 的真实 API e2e 在无 key 时跳过，所以"本地能跑"不构成写死 key 的理由。

### R2 — 公开 API 是 pre-stable：改了就要改全部消费者

不加兼容垫片、不留过渡期。重命名或重构一个公开 API，必须在同一次改动中更新**每一个**调用点。

与之并列的是已发布的 Session JSONL——它走相邻迁移，见 R3。（上游把两者并列陈述，并未称后者是前者的"例外"。）

### R3 — 已发布的 Session 世代永不改写

已提交的 Session JSONL 世代**不得移动、覆盖或删除**。读取侧走[相邻迁移](../../../.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)：只能新增一个版本命名的后继物；前驱的存在既不意味着回退支持，也不意味着降级支持。SQLite 域使用单调递增的 `SCHEMA_VERSION`。

这条与 R2 方向相反，因为它保护的是**用户磁盘上已经存在的数据**，而不是代码接口。

### R4 — Model-visible ⟺ logged

任何进入模型请求的内容都必须能从会话日志重建。**新增一个模型可见的输入，就必须同时新增一个会话事件**——没有例外，没有"这个不重要所以不记"。

这是整个 harness 可复现性的地基：快照测试、会话回放、SDK 投影全都建立在它之上。

详见 [`session_and_events.md`](../../session_and_events.md) 与 [`agent_loop_and_tools.md`](../../agent_loop_and_tools.md)。

### R5 — 只有 `dsh` profile 能启动应用

包的 `bin`、demo 脚本、公开 SDK 的 argv 逃逸口都**禁止**用作应用启动路径。规则归 [`docs/architecture.md#application-launch`](../../../docs/architecture.md#application-launch)。

### R6 — 非平凡改动必须在同一 PR 内附 Agent Note

只有机械性、局部性的编辑豁免。判定范围见 [`.agents/notes/README.md#when-to-write-one`](../../../.agents/notes/README.md#when-to-write-one)。

**已归档的 Note 是冻结的**：永远不要编辑它们，也不要把它们当作当前权威（[归档策略](../../../.agents/notes/README.md#archiving-and-deletion)）。这一点容易踩——`.agents/notes/archived/` 里的内容读起来和 `implemented/` 一样自信。

## 2. 架构不变量：改 `packages/` 前必须成立的前提

> **上游事实源**：[`docs/architecture.md`](../../../docs/architecture.md)、[`AGENTS.md` 的 `## Conventions`](../../../AGENTS.md#conventions) 与 [`## Type safety and documentation`](../../../AGENTS.md#type-safety-and-documentation)、[`docs/glossary.md`](../../../docs/glossary.md)

### R7 — 一切皆插件，无特权内核

新行为挂在**已文档化的扩展点**上。想改 `agent-loop` 本身，必须同步更新 `docs/architecture.md`——这个要求本身就是一道刹车，用来阻止把特例塞进核心循环。

### R8 — 能力接缝是三角整体

一个能力接缝由 **Service Definition / Service Provider / Consumer** 三个角色构成，是完整的，从不只做一个角色。只有当三个角色确实开始独立演化时才拆分（[glossary](../../../docs/glossary.md#capability-seam)）。

新增能力时如果只写了 Provider 而没有 Definition，说明你在往别人的能力里塞私货。详见 [`capability_seams.md`](../../capability_seams.md)。

### R9 — 注册即效果

每一处贡献都通过 `ctx.effect()` / `ctx.on()` 完成；registry 的 `register()` **返回 disposer**。

禁止不可撤销的全局注册——插件必须能被卸载干净，否则 HMR 与按会话组合都会漏。

### R10 — Waterfall 监听器必须调用 `next()`

不调用就短路整条链。这是沉默失败的高发点：你的监听器看起来"生效了"，实际是把后面所有人挤掉了（[语义](../../../docs/cordis-primer.md#cordis-waterfall-semantics)）。

### R11 — 显式优于隐式：默认值是一个 `resolve(request): Spec` 步骤

在包边界上，defaulting 必须是拥有方实现里一个**显式的 `resolve` 步骤**，绝不是 `run()` 内部藏着的 `?? default`。`dsh-shell` 的 request/spec 拆分是模板。

### R12 — 插件里不得有硬编码可调参数

随部署变化的选择必须是**经过校验的 `Config` 字段**，可从 `cordis.yml` 改。一个 `DEFAULT_*` 常量或测试钩子**不算**可配置性。协议常量、外部规范、安全不变量保持固定。

### R13 — 配置错误要大声失败

自包含的错误在加载时失败，否则在最早可解析处失败。**永远不要静默跳过一个缺失的引用对象。**

### R14 — 类型约定

- 跨边界的不透明 id 一律用 `Branded<B>`（来自 `dsh-brand`），不用裸 `string`。
- 判别式联合上用 `switch`；封闭联合以 `assertNever` 收尾，可合并扩展的联合走一个有文档的 default 分支。
- 类型化事件用**声明合并**与可合并扩展的映射。事件 JSDoc 需要 `@mode` 与 payload 的 `@param`；payload 中不存在的 scoped key 需要 `@dshScopeScan unsupported`。
- **在同进程的类型化边界上信任 TypeScript**：不要仅仅因为静态接口已经要求了某个值，就给它加运行时校验、兜底行为或恶意输入测试。校验只发生在 parser/config、队列、模型/工具 JSON、持久化/文件、worker、进程、wire 这些真实边界上。
- 全仓 `strict: true` + `noImplicitAny`；每一个残留的 `any` 都要解释为什么无法收窄。

### R15 — 源码面与产物面不得混用

静态门禁与测试通过 tsconfig 的 `paths` 解析到 `src`，并在干净工作树上通过；消费构建产物 `lib/` 的门禁必须**声明**这个依赖（[布局](../../../docs/development.md#typescript-project-layout)）。

同时保持**编译面显式**：同时有 Host 与 Client 程序的包暴露面特定的 leaf config 与一个纯 solution 根；仓库级程序播种一个面 config，绝不用根 solution。

### R16 — 全仓 ESM

`"type": "module"`。跨包用包名导入，包内相对导入带 `.ts` 后缀。`dsh` CLI 的源码启动走 tsx 的 ESM-only 钩子（`node --import tsx/esm`），它能到达的模块**必须保持 ESM，不得引入 CJS-only 导出**。

### 其它带独立门禁的硬约束

以下条款同样出自 `AGENTS.md`，都各自有门禁在守，撞上就是红灯。它们没有单独编号，是为了不打乱 §7 速查表的引用；触到对应面时按表回上游读原文。

| 约束 | 守它的门禁 | 上游出处 |
|---|---|---|
| **Client UI 文案归 locale 所有**——产品文本走类型化字典与 `t` 或本地化 primitive props，禁止硬编码文案 | `verify-client-ui-i18n` | [`AGENTS.md#conventions`](../../../AGENTS.md#conventions) |
| **运行期不变量只断言自己拥有的关系**——仅当独立观察可能分歧时才发布 `./invariant`；空 installer、检查服务存在/插件元数据/effect/固定样例一律无效，不发布就要在 README 写明理由 | `verify-package-invariants` | [`packages/AGENTS.md`](../../../packages/AGENTS.md) |
| **`SessionEventMap` 成员默认 required-on-read**——不认识某类型的构建会拒绝整份日志，除非事件带 envelope 的 `ignorable: true`；**只有结构性格式变更**才 bump `SESSION_FORMAT_VERSION` | 读取期失败 + 格式目录生成器 | [`AGENTS.md#conventions`](../../../AGENTS.md#conventions)，与 R3/R4 同域 |
| **`cordis.yml` 只允许 `!!js`（绝不是 `!js`）**，且仅限插件 `config` 与条目 `disabled`；其余元数据保持字面量，条件组合走 overlay | 加载即失败 | [`AGENTS.md#secrets--env`](../../../AGENTS.md#secrets--env) |
| **Raw/Web `cordis.yml` 的裸插件名必须出现在其 resolver manifest 的 `dependencies` 里** | `verify-cordis-config` | [`AGENTS.md#conventions`](../../../AGENTS.md#conventions) |
| **每个工具的 UI 呈现要提前设计**——Host presenter 保持纯函数，Web card 由 raw event 与持久化 result metadata 派生 | 评审 + 快照 | [`docs/cookbook/adding-a-tool.md`](../../../docs/cookbook/adding-a-tool.md) |
| **文件以且仅以一个换行结尾** | pre-commit 的 `git diff --cached --check` | [`AGENTS.md#conventions`](../../../AGENTS.md#conventions) |
| **PR 标签**：一个 `kind/*`、全部相关 `area/*`、native Issue Type | 评审 | [`AGENTS.md#conventions`](../../../AGENTS.md#conventions) |
| **可机检的不变量要接进已执行的顶层门禁**，并证明每条改动的接受路径能拒绝非法用例；用**窄而有理由的例外**，不要全局禁用规则 | 评审 | [`AGENTS.md#type-safety-and-documentation`](../../../AGENTS.md#type-safety-and-documentation) |
| **改了已文档化的类型，其 `docs/subsystems/` 页必须同一次改动更新**；文档里的 ```ts 围栏必须能编译 | `verify-type-equiv` / `doc-typecheck` | [`docs/AGENTS.md`](../../../docs/AGENTS.md) |

## 3. 提交前的义务

> **上游事实源**：[`AGENTS.md` 的 `## Commands`](../../../AGENTS.md#commands)、[`docs/testing.md`](../../../docs/testing.md)、[`.agents/skills/dsh-pre-push-checks/SKILL.md`](../../../.agents/skills/dsh-pre-push-checks/SKILL.md)

### R17 — 按 diff 选最小检查集，禁止反射式跑全量

本地 Git 钩子是**故意做窄的**；穷尽覆盖与平台矩阵由 CI 拥有。"想保险所以跑全量套件"是被明确禁止的默认行为——上游只留了三个例外：**用户显式要求、诊断 CI 失败、改动确实是不可再拆的仓库级改动**。选择策略归 [`dsh-pre-push-checks`](../../../.agents/skills/dsh-pre-push-checks/SKILL.md)，反查表见 [`quality_gates.md`](../../quality_gates.md)。

### R18 — 覆盖率门禁是 `test:coverage`，不是 `test`

`test:coverage` 对 `packages/*/*/src` 要求**每文件 100%**。`pnpm run test` 通过**不等于**覆盖率通过——不得用前者冒充后者。（已知豁免：无 `pwsh` 的主机上 `pwsh-local` / `pwsh-sandbox` 由 `vitest.config.ts` 动态排除，CI runner 上仍强制满格。）

### R19 — 快照与双 SDK 投影

每一处非平凡的、模型可见、**协议可见**或人类可见的改动，都要更新一个 keyless 的录制会话快照。（注意上游两处措辞不一致：`AGENTS.md` 写 "model- or product-user-visible"，`docs/testing.md#when-a-snapshot-test-is-required` 写 "model-, protocol-, or human-visible"。按 §6 第 2 条，此处取范围更大的 `docs/` 版本。）[快照归属](../../../snapshots/AGENTS.md) 把顶层树保留给会话驱动的用例，其他期望输出归各自 owner。夹具在 macOS/Linux 上回放；**修夹具，不要修 normalizer**。

Agent-loop、会话生命周期、`SessionEventMap` 的改动必须在**同一个 PR 内**同时更新 TypeScript SDK 与 Python SDK 的期望输出——`pnpm run test` 两者都不覆盖（[覆盖面](../../../docs/testing.md#when-a-snapshot-test-is-required)）。

### R20 — 测试描述行为，不描述正确性

行为过时了就连同它的测试一起改，并在 PR 里说明为什么。不要为了让测试变绿而扭曲实现。

### R21 — PR 历史要刻意选择

拆分独立改动；在传播之前先修引入它的那个 PR。改写历史用 `--force-with-lease`，远端有移动就中止，**绝不用裸 `--force`**（[理由](../../../.agents/notes/implemented/process/2026-08-02-native-github-stacks-and-optional-rebases.md)）。

## 4. 文档义务

> **上游事实源**：[`docs/AGENTS.md`](../../../docs/AGENTS.md)、[`AGENTS.md` 的 `## Type safety and documentation`](../../../AGENTS.md#type-safety-and-documentation)

### R22 — 文档随代码同改

改代码就同步更新受影响的 README 与 JSDoc 契约，在同一个 PR 内。

每个模块与导出都要有简洁 JSDoc 说明其非显然契约；函数类导出要有 `@param`/`@returns`，由 `verify-export-jsdoc` 强制。Heritage 声明的成员、插件协议槽位、构造函数，其文档留在**声明处**（Service Definition、协议或类），不在实现处重复。

### R23 — 一个事实一个归属地

`docs/` 的分层治理规则全在 [`docs/AGENTS.md`](../../../docs/AGENTS.md)：当前态叙述、每段一个物理行、一个事实一个家、字数预算。注意**只有两条有机器门禁**——"每段一个物理行"由 `verify-md-wrap` 强制、字数预算由 `verify-doc-budgets` 强制；"当前态叙述"与"一个事实一个家"靠评审与 `dsh-doc` 审计。

**这条直接约束 `dev_docs` 自身**——见 §6。

### R24 — 双语配对

`docs/`、`python/`、`.agents/notes/` 下的文档，以及**仓库内任意位置的 `README.md`**，都必须成对翻译。语料判定见 `scripts/translation-pairing.ts` 的 `isTranslationScopeFile`。改动这些范围内的源文档，必须同步更新配对文件与一致性记录。

只有用户显式调用才能运行 `dsh-translate-docs`。

### R25 — 散文风格

陈述完整契约与上下文，不写推理过程。用直接、具体的词，**不用比喻**。写 `contract`、`boundary`、`shape` 之前先问有没有更精确的词——写 `response fields` 而不是 `response shape`。不要叙述控制流或测试、不要保留评审历史、不要复述代码。保留行为、失败、时序、归属、安全使用这些事实，理由用链接指过去。

注释保持局部：不复述代码、不解释远处的行为（除非局部确有必要）、不顺手扩写无关注释。

空 `catch` 必须命名它吞掉了什么、以及为什么不会有别的东西到达这里；`try` 保持只有一条语句。

## 5. 外部服务与数据边界

> **上游事实源**：[`AGENTS.md` 的 `## Secrets / .env`](../../../AGENTS.md#secrets--env)、各能力包 README；完整边界表见 [`security_and_sandbox.md`](../../security_and_sandbox.md)

| 边界 | 出站/存储内容 | 由什么控制 |
| --- | --- | --- |
| DeepSeek 官方 API | 会话消息、系统提示词、工具 schema 与工具结果 | `DEEPSEEK_API_KEY`；可选 `DEEPSEEK_BASE_URL` |
| pi-ai 多 provider 适配器 | 同上，目标端点取决于所选 provider | `cordis.yml` 配置行 |
| Web 能力（search / fetch） | 模型发起的检索关键词与抓取 URL | profile/bundle 是否挂载该 provider |
| OpenTelemetry 遥测 | 会话事件账本记录 + 运行记录；资源标识含 `service.name`/`service.version` 与匿名 `user.id` | `mode` 配置（实时上传 / 仅反馈时回放 / 仅本地）；匿名 id 存于 `$DSH_HOME/.anonymous-user-id`，删文件即重置 |
| E2B 远程沙箱（POC） | 文件系统与子进程操作转发至远程沙箱 | 仅在挂载 E2B provider 时生效 |
| 本地进程约束 | 无外传 | base bundle 的 policy 配置 |

**本仓库没有向量数据库、没有 RAG、没有 embedding 流水线**（对 `packages/**/package.json` 的多关键词检索命中为 0）。不要为不存在的东西写文档或加抽象。

新增依赖会触发 pre-commit **自动重生成** [`THIRD_PARTY_NOTICES.md`](../../../THIRD_PARTY_NOTICES.md)（暂存文件改到它的输入时就跑）；测试道的 `gen-third-party-notices.spec.ts` 断言已提交字节一致，`pnpm run verify-third-party-notices` 是独立入口。注意披露义务来自**各第三方自己的许可证**，不是本仓库的 MIT——该文件开头就写着"Each project remains under its own license; nothing in this file changes those terms."

## 6. `dev_docs` 自身的规则

这一节是 `dev_docs` 对自己的约束，不是上游规则，但它是从 R23 推导出来的必然结果。

1. **`dev_docs` 不是任何事实的归属地。** 它是导航层：把"场景"映射到"文档"再映射到"上游事实源"。任何一段陈述都必须能指向 `docs/`、源码或 `.agents/notes/`。
2. **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即更新 `dev_docs`。若判定是上游写错了，记录到根目录的 [`UPSTREAM_DOC_ISSUES.md`](../../../UPSTREAM_DOC_ISSUES.md) 并在正文显式标注该冲突——**不要沉默地跟随任何一方**。
3. **代码块只能逐字粘贴**，且必须带 `path:line（symbol: name）` 标注。禁止凭印象重写签名。
4. **引用 `docs/` 用章节锚点，不用行号**；引用源码用 `path:line`，因为源码没有稳定锚点。
5. **不复制生成产物。** `tool-catalog.md`、`config-catalog.md`、`module-graph.md`、`event-producer-consumer.md`、`persistence-catalog.md` 一律链接，绝不转述——它们由生成器维护，手抄必漂移。
6. **不写实现状态标注，不写变更历史。** 当前态叙述，与 `docs/AGENTS.md` 一致。
7. **脱敏**：不写真实 key、真实 base URL、真实 OTLP 端点、本地绝对路径。
8. **`dev_docs` 内部索引文件命名为 `index.md`，不用 `README.md`。** 理由：`README.md` 会被 R24 的双语配对语料捕获（`scripts/translation-pairing.ts` 的 `README_ARTIFACT` 正则匹配任意路径下的 `readme.md`），而 `dev_docs` 是刻意的纯中文层。改名是三个候选方案里唯一不需要改动仓库门禁或配置的做法。

## 7. 速查表

改动落在哪里 → 至少要想起哪几条：

| 你在改…… | 必看 |
| --- | --- |
| 任何公开 API 的签名/名字 | R2、R22 |
| `SessionEventMap` / 会话事件 | R3、R4、R14、R19 |
| `agent-loop` 或核心循环 | R7、R19 |
| 新增一个能力 | R8、R9、R11、R12 |
| Cordis 监听器 | R9、R10 |
| 插件配置项 | R11、R12、R13 |
| 工具（tool） | R4、R19，另见 [`docs/cookbook/adding-a-tool.md`](../../../docs/cookbook/adding-a-tool.md) |
| 任何 README 或 `docs/` | R23、R24 |
| `vendor/` 下的任何东西 | [`AGENTS.md` 的 `## Vendoring policy`](../../../AGENTS.md#vendoring-policy)——按 `vendor/README.md` 的同步流程走，重跑 `pnpm run test && pnpm run build` |
| 生命周期 / 并发 / 子进程 / 拆卸 | 先读 [`docs/defensive-patterns.md`](../../../docs/defensive-patterns.md) |
| 准备提交 | R6、R17、R18、R19、R21 |

## 延伸阅读

- [`AI_Coding_Context.md`](../../AI_Coding_Context.md) — dev_docs 总入口与场景导航
- [`quality_gates.md`](../../quality_gates.md) — 从门禁报错反查该改哪里
- [`architecture_overview.md`](../../architecture_overview.md) — 全插件架构与 Cordis 基础
- [`capability_seams.md`](../../capability_seams.md) — 三角接缝的完整说明
- [`session_and_events.md`](../../session_and_events.md) — 会话事件与 model-visible ⟺ logged
- [`plugin_development_guide.md`](../../plugin_development_guide.md) — 写一个插件的完整流程
