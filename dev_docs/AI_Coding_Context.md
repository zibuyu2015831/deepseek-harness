---
title: DeepSeek Harness AI 编码上下文主文档
summary: dev_docs 文档体系的入口与导航层，把开发场景映射到仓库权威事实源，覆盖项目概览、目录速查、场景导航、文档索引、核心代码模式、开发流程、命名规范、模块映射、AI 编码禁忌与常见任务速查。
keywords: entry-point | navigation | deepseek-harness | dsh | cordis | ai-coding-context
scope: deepseek-harness 仓库 dev_docs 文档体系的主入口
related_files: AGENTS.md | docs/architecture.md | docs/AGENTS.md | packages/README.md | package.json | docs/testing.md
dependencies: docs/architecture.md | AGENTS.md | packages/README.md | dev_docs/_analysis/generation_plan.md
verified_at: 2026-09-06
---

# DeepSeek Harness — AI 编码上下文

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。
>
> 仓库自带一套成熟且由 CI 强制的文档体系，其第一原则是 [one home per fact](../docs/AGENTS.md)——每个事实只有一个归属地，其他地方链接过去。`docs/` 在双语配对门禁作用域内**已全量双语**（仅 [`scripts/translation-pairing.manifest.json`](../scripts/translation-pairing.manifest.json) 显式豁免的 5 篇文档治理/翻译工具页仅英文），所以"提供中文"不构成 `dev_docs` 的存在理由。
>
> `dev_docs` 只做两件上游按 one-home-per-fact 原则不会做的事：
>
> 1. **导航**：把"我要做什么"映射到对应机制与权威文档，做场景 → 机制 → 事实源的三级跳转
> 2. **串联**：把被拆散在 `docs/architecture.md`、`docs/subsystems/`、包 README 与 Agent Notes 中的线索，串成一条完整脉络
>
> **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即修正 `dev_docs`。

---

## 📊 项目概览

| 项 | 值 |
| --- | --- |
| 项目 | DeepSeek Harness（CLI 名 `dsh`，根包 `@deepseek-ai/dsh-root`） |
| 类型 | Agent Harness 运行时 × Monorepo；次级类型 CLI + 库/SDK + Web 全栈 |
| 核心框架 | [Cordis](../docs/cordis-primer.md)（`vendor/` 下的固定源码副本，非 npm 依赖） |
| 语言与运行时 | TypeScript 6（全仓 ESM）、Node `^22.19.0 \|\| >=24.0.0`、pnpm 11.7.0 |
| 构建 | `tsc -b` 出类型与 `lib/`，tsdown 打包运行时，Vite 构建 Web 前端 |
| 测试 | Vitest 4，四层主干（unit / coverage / e2e / snapshot）+ owner-local expected 与 Web 浏览器快照两组旁支 |
| 许可 | MIT；第三方依赖披露于 [`THIRD_PARTY_NOTICES.md`](../THIRD_PARTY_NOTICES.md) |
| 稳定性 | **开发者预览期**，公开 API 前稳定，明确会有破坏性变更 |

**规模**（`verified_at: 2026-09-06` 实测，口径见 [`_analysis/generation_plan.md`](_analysis/generation_plan.md)）：

```
workspace 包    255 个（packages/ 下 50 个能力组）+ vendor/ 9 个 + apps/ 2 个
TypeScript      684,902 行（含 vendor/ 与 scripts/）
TSX              82,677 行
Markdown        255,250 行
Python            8,718 行
测试             260 个测试目录；packages/ 下 854 个 *.spec.ts / *.test.ts
决策记录         .agents/notes/ 下 1,742 篇 Agent Notes
```

> ⚠️ 这些数字**会快速漂移**——2026-08-18 到 2026-09-06 的 19 天内，包数 +36、TypeScript +36.6%、Agent Notes +352 篇。引用规模数字前请重新实测，不要信任本表的绝对值。

---

## 📂 关键目录速查

> **上游事实源**：[`AGENTS.md`](../AGENTS.md) 的 Repository layout 章节是目录职责的归属地，本表只做中文索引。

| 目录 | 职责 | 进一步阅读 |
| --- | --- | --- |
| `packages/` | 255 个 workspace 包，按 50 个能力组划分 | [`packages/README.md`](../packages/README.md) |
| `apps/` | 产品装配层：`cli`（`dsh` 命令）、`web` | [`docs/user/`](../docs/user/index.md) |
| `vendor/` | Cordis 生态固定源码副本（9 包），rescope 为 `@deepseek-ai/*` | [`vendor/README.md`](../vendor/README.md)、[`docs/rescope.md`](../docs/rescope.md) |
| `native/` | `landlock-run` 原生启动器（Linux 进程约束） | [`native/README.md`](../native/README.md) |
| `python/` | Python SDK 与打包运行时 | [`python/README.md`](../python/README.md) |
| `docs/` | **仓库自有权威文档**，分层双语，含生成产物 | [`docs/AGENTS.md`](../docs/AGENTS.md) |
| `scripts/` | 门禁与生成器（`verify-*` / `gen-*` / `run-gates`） | [`scripts/run-gates.ts`](../scripts/run-gates.ts) |
| `snapshots/` | 会话驱动的快照期望输出 | [`snapshots/AGENTS.md`](../snapshots/AGENTS.md) |
| `website/` | VitePress 文档站点 | — |
| `.agents/` | Agent Notes（1,742 篇决策记录）与 Skills | [`.agents/notes/README.md`](../.agents/notes/README.md) |
| `dev_docs/` | **本文档体系**（中文导航层） | 本文 |

---

## 🎯 场景快速导航

先找到你的场景，再顺着链接走。**`dev_docs` 列给中文脉络，`权威事实源` 列给必须遵守的契约**——两者冲突以后者为准。

| 我要做什么 | dev_docs | 权威事实源 |
| --- | --- | --- |
| 第一次接触这个仓库，想搞懂它怎么装起来的 | [架构总览](architecture_overview.md) | [`docs/architecture.md`](../docs/architecture.md) |
| 新增一个模型可见工具 | [插件开发指南](plugin_development_guide.md) | [`docs/cookbook/adding-a-tool.md`](../docs/cookbook/adding-a-tool.md) |
| 接入一个新的模型提供方 | [模型配置与 LLM 适配器](model_configuration.md) | [`docs/cookbook/adding-an-llm-adapter.md`](../docs/cookbook/adding-an-llm-adapter.md) |
| 新增一种能力（如新的执行后端） | [能力接缝设计](capability_seams.md) | [`docs/capability-seams.md`](../docs/capability-seams.md) |
| 让某个信息进入模型上下文 | [系统提示词与上下文组装](prompt_management.md) + [会话日志与事件系统](session_and_events.md) | [`docs/subsystems/system-prompt.md`](../docs/subsystems/system-prompt.md) |
| 拦截或修改某次模型请求 | [Agent 循环与工具执行管线](agent_loop_and_tools.md) | [`docs/agent-lifecycle.md`](../docs/agent-lifecycle.md) |
| 新增一个 npm 包 | [Monorepo 结构与构建体系](monorepo_and_build.md) | [`docs/cookbook/adding-a-package.md`](../docs/cookbook/adding-a-package.md) |
| 改 Web UI | [CLI 与 Web 应用装配](apps_cli_and_web.md) | [`docs/subsystems/client-modules.md`](../docs/subsystems/client-modules.md) |
| 提交前该跑哪些检查 | [质量门禁导航](quality_gates.md) | [`.agents/skills/dsh-pre-push-checks/SKILL.md`](../.agents/skills/dsh-pre-push-checks/SKILL.md) |
| 写测试 / 快照不通过 | [测试指南](testing_guide.md) | [`docs/testing.md`](../docs/testing.md) |
| 理解沙箱与审批为什么拦了我 | [安全、沙箱与数据边界](security_and_sandbox.md) | [`docs/subsystems/sandbox.md`](../docs/subsystems/sandbox.md)、[`docs/subsystems/approval.md`](../docs/subsystems/approval.md) |
| 发布一个版本 | [发布与分发](deployment_guide.md) | [`scripts/release/`](../scripts/release) |
| 上下文爆了 / token 成本过高 | [Token 成本与上下文管理](cost_optimization.md) | [`docs/subsystems/compaction.md`](../docs/subsystems/compaction.md) |
| 从外部程序驱动 dsh（SDK / ACP / hooks） | [SDK 与外部协议](sdk_and_protocols.md) | [`packages/sdk/README.md`](../packages/sdk/README.md)、[`python/README.md`](../python/README.md) |
| 判断一次改动是否让 agent 变差了 | [评测与基准](evaluation_metrics.md) | [`BENCHMARK.md`](../BENCHMARK.md)、[`snapshots/AGENTS.md`](../snapshots/AGENTS.md) |
| 排查一个运行期故障 | [故障排查](troubleshooting.md) | [`docs/postmortem/README.md`](../docs/postmortem/README.md)、[`docs/defensive-patterns.md`](../docs/defensive-patterns.md) |

---

## 🚀 文档索引

| 优先级 | 文档 | 覆盖内容 | 状态 |
| --- | --- | --- | --- |
| 🔴 P0 | [架构总览](architecture_overview.md) | Cordis 插件树、profile/bundle 分层、核心包与 ctx key、事件三域、双编译面、vendoring | ✅ |
| 🔴 P0 | [能力接缝设计](capability_seams.md) | 三角角色、如何设计新接缝、`resolve(request): Spec`、反模式 | ✅ |
| 🔴 P0 | [会话日志与事件系统](session_and_events.md) | 追加式日志、`deriveMessages()`、`SessionEventMap` 声明合并、版本与相邻迁移、投影接缝 | ✅ |
| 🔴 P0 | [Agent 循环与工具执行管线](agent_loop_and_tools.md) | turn/step 语义、瀑布与 `next()` 契约、inbox 与注入、工具三阶段、**拦截决策地图** | ✅ |
| 🔴 P0 | [系统提示词与上下文组装](prompt_management.md) | 分段注册、工具 schema 并入、**"让 X 进入模型上下文"的 7 条路径对比**、快照锁定 | ✅ |
| 🔴 P0 | [模型配置与 LLM 适配器](model_configuration.md) | `ctx.llm` 接缝、DeepSeek 直连与 pi-ai、**配置在哪一层生效**、token 计量 | ✅ |
| 🔴 P0 | [插件开发指南](plugin_development_guide.md) | **扩展点选择决策树**、注册即效果、包落位、subagent/workflow/skill 三条链路 | ✅ |
| 🟡 P1 | [Monorepo 结构与构建体系](monorepo_and_build.md) | workspace 布局、**vendoring 与 rescope**、**双编译面**、源码面与产物面 | ✅ |
| 🟡 P1 | [质量门禁导航](quality_gates.md) | **门禁报错反向索引**、检查选择策略、Agent Note 义务 | ✅ |
| 🟡 P1 | [测试指南](testing_guide.md) | 四层测试拓扑、覆盖率门禁语义、**"改了 X 该测什么"**、快照实操 | ✅ |
| 🟡 P1 | [CLI 与 Web 应用装配](apps_cli_and_web.md) | 应用启动硬规则、profile/bundle 层叠、host/client 双半边、**UI 改动定位表** | ✅ |
| 🟡 P1 | [SDK 与外部协议](sdk_and_protocols.md) | **四条自动化路径的选择表**、JSON-RPC SDK、Python SDK、ACP、hooks 桥 | ✅ |
| 🟡 P1 | [安全、沙箱与数据边界](security_and_sandbox.md) | **数据外流边界总表**、三道防线、遥测三模式与匿名 id | ✅ |
| 🟡 P1 | [发布与分发](deployment_guide.md) | npm 发布、单文件可执行、Python 运行时 wheel 三条链路 | ✅ |
| 🟢 P2 | [Token 成本与上下文管理](cost_optimization.md) | **压缩/溢出/裁剪/有界读取横向对比**、KV cache 友好性 | ✅ |
| 🟢 P2 | [评测与基准](evaluation_metrics.md) | keyless 快照回放为主的取舍、**"担心什么退化 → 看哪份证据"** | ✅ |
| 🟢 P2 | [故障排查](troubleshooting.md) | **运行期现象索引**、postmortem 中文索引、诊断工具 | ✅ |

**规则与流程目录**：

| 目录 | 用途 |
| --- | --- |
| [规则合集 `AI_RULES.md`](rules/combined/AI_RULES.md) | 动手前必看：红线、架构不变量、提交义务，每条附上游出处与自查方式 |
| [`plans/`](plans/index.md) | 中文工作计划，[`active`](plans/active/index.md) → [`done`](plans/done/index.md) → [`archive`](plans/archive/index.md) 单向流转；不替代 `.agents/notes/` 的 Agent Note 义务 |
| [`memos/`](memos/index.md) | 待查问题、一次性发现、上下文交接；默认结局是被删除或沉淀 |
| [`knowledge/`](knowledge/index.md) | 经验证、可复用的结论，分 [排障案例](knowledge/troubleshooting/index.md)、[实现模式](knowledge/patterns/index.md)、[性能记录](knowledge/performance/index.md) |

`plans/`、`memos/`、`knowledge/` 三个目录（含各自子目录）共 9 篇索引，一律命名为 `index.md` 而非 `README.md`：仓库的双语配对门禁把任意路径下的 `README.md` 纳入语料并要求成对翻译，而 `dev_docs` 是刻意的纯中文层。理由见 [`AI_RULES.md`](rules/combined/AI_RULES.md) §6 第 8 条。

**方案与进度**：[生成方案](_analysis/generation_plan.md) · [问题报告](_analysis/project_analysis_report.md) · [进度记录](_analysis/generation_progress.md) · [健康检查](_analysis/health_check_report.md)

**上游缺陷台账**：[`UPSTREAM_DOC_ISSUES.md`](../UPSTREAM_DOC_ISSUES.md) — 13 条经复查确认的**上游文档与源码不符**之处。读上游文档踩到坑时先查它。上游不接受外部 PR 且 Issues 已关闭（见该文件"投递可行性"），所以它的定位是 **fork 内自用的已知陷阱清单**，不是待提交队列。

---

## 💻 核心代码模式

> **上游事实源**：模式的权威说明在 [`docs/cookbook/`](../docs/cookbook/adding-a-package.md) 与 [`packages/AGENTS.md`](../packages/AGENTS.md)。本节只给一个可直接照抄的最小真实样本，帮助快速建立形状认知。

### 函数插件 + 模型可见工具（最小完整样本）

`packages/todo/tool-todo` 是全仓最小的完整工具插件，一个文件里同时演示了插件形态、配置契约、工具注册、会话写入与 UI 呈现五件事。

**插件形态与注入声明** —— `packages/todo/tool-todo/src/index.ts:22-23`（symbol: `name` / `inject`）：

```typescript
export const name = 'tool-todo'
export const inject = ['tools', 'sessionProjections']
```

函数插件**具名导出** `name` / `inject` / `Config` / `apply` 且**没有 default export**；服务包才 default-export 服务类。两种形态混用会让 Loader 丢弃函数插件的命名空间（[postmortem](../docs/postmortem/0001-acp-default-export-drops-inject.md)）。

**配置即部署选择，不是常量** —— `packages/todo/tool-todo/src/index.ts:37-43`（symbol: `Config`）：

```typescript
  allowParallelInProgress: boolean
}

/** Schemastery configuration for the todo tool consumer. */
export const Config: z<Config> = z.object({
  allowParallelInProgress: z.boolean().required(),
})
```

注意它是 `required()` 而非带默认值——[`AGENTS.md`](../AGENTS.md) 的 no hardcoded tunables 规则要求：随部署变化的选择必须是可从 `cordis.yml` 改的 `Config` 字段，`DEFAULT_*` 常量或测试钩子**不算**可配置性。

**工具注册与"模型可见 ⟺ 已记录"** —— `packages/todo/tool-todo/src/index.ts:146-221`（symbol: `apply` 内的 `ctx.tools.register`）：

```typescript
  ctx.tools.register(defineTool({
    name: 'todo_write',
    description: describe(allowParallel),
    parameters: { /* JSON Schema，略 */ },
    output: { schema: { /* 略 */ }, render: (_args, value) => [...] },
    execute(args, exec) {
      const todos = toTodoList(args.todos, allowParallel)
      if (!exec.agent) {
        throw new Error('todo_write requires an owning agent session')
      }
      exec.agent.session.append('todo/write', { todos })
      /* 略 */
    },
    presentCall: args => ({ card: 'generic', title: 'Update todo list', kind: 'other', rawInput: args.todos }),
  }))
```

三个可迁移的要点：

- `exec.agent.session.append('todo/write', ...)` 是"模型可见即已记录"落地的样子——状态写进会话日志，UI 从会话事件渲染，而不是维护一份进程内状态
- `presentCall` 是**纯函数**，只依据 `args` 决定渲染意图（`card` / `title` / `kind`），不做副作用
- 拿不到 owning agent 时**抛错而不是静默 no-op**——对应 [`AGENTS.md`](../AGENTS.md) 的 misconfiguration fails loud

### 其他模式的落位

| 模式 | 去哪看 |
| --- | --- |
| 能力接缝三角 + `resolve(request): Spec` | [能力接缝设计](capability_seams.md)；模板是 `packages/shell/` |
| 会话事件扩展（`SessionEventMap` 声明合并） | [`docs/subsystems/session.md`](../docs/subsystems/session.md) |
| LLM 适配器 | [`docs/cookbook/adding-an-llm-adapter.md`](../docs/cookbook/adding-an-llm-adapter.md) |
| `cordis.yml` 行结构与 `!!js` 边界 | [`docs/cordis-primer.md`](../docs/cordis-primer.md) |
| 防御性模式（生命周期 / 并发 / 子进程 / teardown） | [`docs/defensive-patterns.md`](../docs/defensive-patterns.md) |

---

## 🛠️ 开发流程规范

> **上游事实源**：命令清单的归属地是 [`AGENTS.md`](../AGENTS.md) 的 Commands 章节与 [`package.json`](../package.json) 的 scripts；检查选择策略的归属地是 [`dsh-pre-push-checks`](../.agents/skills/dsh-pre-push-checks/SKILL.md)。本节**不复述命令清单**（它会漂移），只讲三条最容易踩错的原则。

**一、按 diff 选最小检查集，不要反射式跑全量套件。** 仓库明确要求把证据匹配到改动面：行为改动跑聚焦测试，模型/用户输出改动跑快照，文档改动跑 `doc-sync`，provider 改动跑真实 API e2e。穷尽覆盖与平台矩阵由 CI 负责。

**二、覆盖率门禁是 `test:coverage`，不是 `test`。** `test:coverage` 对 `packages/*/*/src` 要求**每文件 100%**。用 `pnpm run test` 通过来冒充覆盖率通过是明确的错误（[why](../docs/testing.md)）。

**三、非平凡改动必须在同一 PR 内附 Agent Note。** 只有机械性/局部编辑豁免。范围定义见 [`.agents/notes/README.md`](../.agents/notes/README.md#when-to-write-one)。归档笔记是冻结历史，**不得编辑，也不得当作当前权威**。

---

## 📋 命名规范

> **上游事实源**：[`AGENTS.md`](../AGENTS.md) Conventions 与 [`packages/AGENTS.md`](../packages/AGENTS.md)；角色命名规则见 [`docs/cookbook/adding-a-package.md`](../docs/cookbook/adding-a-package.md#name-the-role-that-exists)。

| 对象 | 规则 |
| --- | --- |
| npm 包名 | `@deepseek-ai/dsh-<name>` |
| 包路径 | `packages/<group>/<pkg>/`，每个包**只属于一个组** |
| vendored 包 | rescope 为 `@deepseek-ai/*`（映射见 [`docs/rescope.md`](../docs/rescope.md)）；9 个 vendored 包**均无 `private` 字段**，且都带 `publishConfig.access: public`，即全部公开发布<br>ℹ️ 上游原文曾称其为 `private: true`，本 fork 已于 2026-09-07 修正（见 [`UPSTREAM_DOC_ISSUES.md`](../UPSTREAM_DOC_ISSUES.md) U1/S7）。合并上游时该处可能回退，以 `vendor/*/package.json` 为准 |
| 类型文件 | `src/types.ts` **只放类型**，不含运行时代码 |
| 测试位置 | 包级 `tests/`，**不是** `src/__tests__/` |
| 跨边界 id | 用 `Branded<B>`（`dsh-brand`），不用裸 `string` |
| 本地相对导入 | 带 `.ts` 后缀；跨包用包名 |
| 文件结尾 | 恰好一个换行符，由 pre-commit 的 `git diff --cached --check` 把关 |

---

## 🏢 业务模块映射

> **上游事实源**：[`packages/README.md`](../packages/README.md) 的分组表是 50 个包组的归属地。本表只把它们按**认知顺序**重排为 8 条主线，方便定位；每组的权威描述与包清单请点进各组 README。

| 主线 | 包组 | 关注点 |
| --- | --- | --- |
| 产品 API 主干 | [`core/`](../packages/core/README.md) | 会话、系统提示词、工具注册与执行、Agent 接口与默认循环、per-agent 作用域 |
| 模型接入 | [`llm/`](../packages/llm/README.md) | 消息与流协议、适配器接缝、DeepSeek/pi-ai 适配器、重试、token 计量 |
| 会话数据面 | [`session/`](../packages/session/README.md)、[`session-query/`](../packages/session-query/README.md)、[`storage/`](../packages/storage/README.md) | 持久化与投影接缝、日志派生标题、遥测、检索与全文搜索 |
| 执行能力族 | [`shell/`](../packages/shell/README.md)、[`subprocess/`](../packages/subprocess/README.md)、[`terminal/`](../packages/terminal/README.md)、[`code-runtime/`](../packages/code-runtime/README.md)、[`fs/`](../packages/fs/README.md)、[`lsp/`](../packages/lsp/README.md) | bash、进程树、持久 PTY、worker 代码执行、文件系统、语言服务 |
| 约束与人机协作 | [`sandbox/`](../packages/sandbox/README.md)、[`guard/`](../packages/guard/README.md)、[`interaction/`](../packages/interaction/README.md) | 进程约束后端、循环卫生与工具超时、审批与权限预设、命令 |
| 组合与分发 | [`bundle/`](../packages/bundle/README.md)、[`preset/`](../packages/preset/README.md)、[`boot/`](../packages/boot/README.md) | profile/bundle 补丁层、按会话 Agent 组合、启动胶水 |
| 协议与外部集成 | [`sdk/`](../packages/sdk/README.md)、[`acp/`](../packages/acp/README.md)、[`hooks/`](../packages/hooks/README.md)、[`api/`](../packages/api/README.md)、[`webhook/`](../packages/webhook/README.md) | JSON-RPC 协议与客户端/服务端、ACP、Claude Code/Codex 钩子桥、Typert RPC 网关、外部事件入站 |
| Web GUI | [`host/`](../packages/host/README.md)、[`client/`](../packages/client/README.md) | API 网关与 HTTP 路由；浏览器侧 shell、wire、slots 与 `ui-*` 插件 |

其余包组（`skill/`、`subagent/`、`workflow/`、`jobs/`、`goal/`、`plan/`、`compaction/`、`spill/`、`context/`、`extensions/` 等）按能力族组织，一律以 [`packages/README.md`](../packages/README.md) 为准。

---

## ⚠️ AI 编码禁忌

> **上游事实源**：[`AGENTS.md`](../AGENTS.md) Conventions 是这些规则的归属地，本节是中文速查，**不改写规则语义**。逐条详解与理由链接见该文件。

**架构层**

- ❌ 给 `agent-loop` 打补丁来加新行为 → ✅ 挂到文档化的扩展点；确需改循环则必须同步更新 [`docs/architecture.md`](../docs/architecture.md)
- ❌ 只实现能力接缝三角中的一个角色就宣称加了能力 → ✅ Definition / Provider / Consumer 三角是整体
- ❌ 让内容进入模型请求却不写会话事件 → ✅ **模型可见 ⟺ 已记录**，新增模型可见输入必须新增会话事件
- ❌ 不可撤销的全局注册 → ✅ 一切贡献走 `ctx.effect()` / `ctx.on()`，registry 的 `register()` 返回 disposer
- ❌ 瀑布监听器直接 return → ✅ **必须调用 `next()`**，否则短路整条链

**工程层**

- ❌ 引入 CJS-only 导出 → ✅ 全仓 ESM
- ❌ 在 `run()` 里用 `?? default` 做隐式默认 → ✅ 默认值是显式的 `resolve(request): Spec` 步骤
- ❌ 用 `DEFAULT_*` 常量代替可配置性 → ✅ 随部署变化的选择必须是 `Config` 字段
- ❌ 对同进程强类型边界加运行时校验和敌意输入测试 → ✅ 只在 parser/config、队列、模型/工具 JSON、持久化、worker、进程、wire 边界校验
- ❌ 静默跳过缺失的引用目标 → ✅ misconfiguration fails loud
- ❌ 空 `catch` 不说明吞了什么 → ✅ 注明吞掉的是什么、为什么不会有别的错到这里，且 `try` 只包一条语句

**兼容性层**（预发布阶段特有）

- ❌ 为旧 API 加兼容垫片 → ✅ 无外部消费者，重命名/重构时同步更新全部引用
- ❌ 承诺 API 向后兼容 → ✅ 开发者预览期，明确会有破坏性变更
- ⚠️ **例外**：已发布的 Session JSONL 走[相邻迁移](../.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)——已提交的世代路径**永不重命名、覆盖或删除**

**文档层**

- ❌ 把同一个事实写在两个地方 → ✅ one home per fact，其他地方链接过去
- ❌ 手工重述生成产物（工具目录、配置目录、模块图、事件表）→ ✅ 一律链接
- ❌ 在文档里写变更史（"以前"/"现在"/"不再"）→ ✅ 只描述当前状态
- ❌ 非平凡改动不写 Agent Note → ✅ 同 PR 必须附

---

## 🔧 常见任务速查

> **上游事实源**：[`docs/architecture.md`](../docs/architecture.md#where-new-behavior-goes) 的 "Where new behavior goes" 表是"新行为该挂在哪"的归属地，[`docs/cookbook/extension-cookbook.md`](../docs/cookbook/extension-cookbook.md) 是分步指南的索引。本表**只补充一列它们没有的东西**——中文文档入口。

| 任务 | 权威步骤 | dev_docs 脉络 |
| --- | --- | --- |
| 加一个模型可见工具 | [`adding-a-tool.md`](../docs/cookbook/adding-a-tool.md) | 上文"核心代码模式" |
| 加一个 npm 包 | [`adding-a-package.md`](../docs/cookbook/adding-a-package.md) | [Monorepo 与构建](monorepo_and_build.md) |
| 加一个 LLM 适配器 | [`adding-an-llm-adapter.md`](../docs/cookbook/adding-an-llm-adapter.md) | [模型配置](model_configuration.md) |
| 加一个设置卡片 | [`adding-a-settings-card.md`](../docs/cookbook/adding-a-settings-card.md) | [CLI 与 Web 应用](apps_cli_and_web.md) |
| 加一个 vendored 包 | [`adding-a-vendored-package.md`](../docs/cookbook/adding-a-vendored-package.md) | [Monorepo 与构建](monorepo_and_build.md) |
| 加一个远程 API | [`adding-a-remote-api.md`](../docs/cookbook/adding-a-remote-api.md) | [SDK 与外部协议](sdk_and_protocols.md) |
| 看当前机器启动的插件树 | `dsh --profile web --dump-config` | [架构总览](architecture_overview.md) |
| 查某个配置项 | [`docs/config-catalog.md`](../docs/config-catalog.md)（生成产物） | — |
| 查某个工具的 schema | [`docs/tool-catalog.md`](../docs/tool-catalog.md)（生成产物） | — |
| 查某个事件谁产谁消 | [`docs/event-producer-consumer.md`](../docs/event-producer-consumer.md)（生成产物） | — |
| 查包依赖关系 | [`docs/module-graph.md`](../docs/module-graph.md)（生成产物） | — |
| 查术语定义 | [`docs/glossary.md`](../docs/glossary.md) | — |

---

## 📌 使用本文档体系的规则

1. **`docs/` 优先**。任何冲突以 `docs/` 为准，并立即修正 `dev_docs`。
2. **不信任 `dev_docs` 里的绝对数字**。规模统计、包数、行数都会快速漂移，用前重测。
3. **`verified_at` 超过 90 天视为过期**，需重新核验后再依赖。
4. **`dev_docs` 不在仓库 CI 门禁作用域内**（已实测确认），所以它的正确性没有机器保障——这既是它可以自由组织的原因，也是它必须保守引用的原因。
