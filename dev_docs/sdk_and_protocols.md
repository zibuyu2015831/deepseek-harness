---
title: SDK 与外部协议
summary: 外部程序驱动 dsh 的四条集成路径（TypeScript JSON-RPC SDK、Python SDK、ACP 服务器、hook 桥）的选择依据、进程模型与代价，以及改动 SDK 时必须同步的两套投影。
keywords: sdk | json-rpc | acp | hooks | python-sdk | typert | api-gateway
scope: deepseek-harness 面向自动化的四条外部集成路径
related_files: packages/sdk | packages/acp | packages/hooks | python/README.md | docs/api-gateway.md
dependencies: docs/architecture.md | python/README.md | docs/api-gateway.md
verified_at: 2026-09-06
---

# SDK 与外部协议

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。各协议的契约归其包 README 与 [`python/README.md`](../python/README.md)，应用启动规则归 [`docs/architecture.md`](../docs/architecture.md)，网关归 [`docs/api-gateway.md`](../docs/api-gateway.md)。本文不复述它们，只回答一个上游没有集中回答的问题：**外部程序想驱动 dsh，四条路该选哪条。**
>
> 面向人类用户的入口（`dsh` CLI 与 Web GUI）见 [CLI 与 Web 应用装配](apps_cli_and_web.md)。
>
> **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即修正本文。

## 1. 本文定位与上游事实源

> **上游事实源**：[`docs/AGENTS.md` — The tier taxonomy: one home per fact](../docs/AGENTS.md#the-tier-taxonomy-one-home-per-fact)

仓库的文档治理规则是"一个事实一个归属地"：任何一条事实只在一处成文，其他地方链接过去。`docs/` 在配对门禁作用域内已全量双语（仅 manifest 显式豁免的 5 篇除外），因此本文**刻意不翻译**上游内容。下表是"你想找什么 → 去哪读"的路由表，本文只负责路由本身和四条路径之间的横向比较。

| 你想找的东西 | 归属地 | 本文的处理 |
| --- | --- | --- |
| JSON-RPC 线协议的帧格式、方法名、payload 语义、错误码 | [`packages/sdk/protocol/README.md`](../packages/sdk/protocol/README.md#use-this-package) | 只在第 4 节指出三件套分工，不复述字段 |
| TypeScript 客户端的 API（`DeepSeekHarness` / `HarnessClient`）、超时、错误类、拆卸阶梯 | [`packages/sdk/client/README.md`](../packages/sdk/client/README.md#use-this-package) | 第 4 节链接 |
| 服务端插件的挂载要求、stdout 纯净性、shutdown 语义 | [`packages/sdk/server/README.md`](../packages/sdk/server/README.md#use-this-package) | 第 4 节链接 |
| Python SDK 的完整用法、参数、结果对象 | [`python/sdk/README.md`](../python/sdk/README.md) | 第 5 节只讲运行时打包与 profile 模型 |
| 运行时 wheel 打包了什么、平台矩阵、虚拟文件系统代理包 | [`python/sdk-runtime/README.md`](../python/sdk-runtime/README.md) | 第 5 节链接 |
| ACP v1 支持的方法矩阵、MCP 信任模型、stopReason 映射 | [`packages/acp/acp/README.md`](../packages/acp/acp/README.md#standard-acp-v1-surface) | 第 6 节只讲"automation-only"的判据 |
| 两个 hook 桥各自支持的事件、配置字段、失败语义 | [`hooks-claude-code`](../packages/hooks/hooks-claude-code/README.md#use-this-package)、[`hooks-codex`](../packages/hooks/hooks-codex/README.md#use-this-package) | 第 7 节只讲"何时用桥、何时写原生插件" |
| profile / bundle 的分层与合并顺序 | [`docs/architecture.md` — Profiles and bundles](../docs/architecture.md#profiles-and-bundles) | 第 3 节引其结论 |
| 应用启动的硬规则与守卫脚本 | [`docs/architecture.md` — Application launch](../docs/architecture.md#application-launch) | 第 3 节引其结论 |
| Typert Remote 的编程模型与生成管线 | [`docs/api-gateway.md`](../docs/api-gateway.md#programming-model)、[`docs/subsystems/typert.md`](../docs/subsystems/typert.md) | 第 8 节只做定位，不复述 |
| 每个插件的完整配置字段 | [`docs/config-catalog.md`](../docs/config-catalog.md)（生成产物） | 一律链接，不复制 |

本文的读者是"要写一个程序去驱动 dsh"的开发者与 AI。它回答的是**选型动作**问题：四条路各自把什么交给你、把什么留给自己、代价在哪。

---

## 2. 四条路径速览与选择表

> **上游事实源**：[`packages/sdk/README.md`](../packages/sdk/README.md#packages)、[`packages/acp/README.md`](../packages/acp/README.md)、[`packages/hooks/README.md`](../packages/hooks/README.md#packages)、[`python/README.md`](../python/README.md)

这是本文存在的理由。上游的四份文档各自完整，但没有一处把它们放在同一张表里比较。先看一句话定性：

- **TypeScript JSON-RPC SDK** —— 仓库自有的 SDK 线协议，`packages/sdk/` 三件套（protocol / client / server）。客户端在自己的进程里 spawn 一个完整的 dsh 运行时，通过 stdio 上的换行分隔 JSON-RPC 驱动它。
- **Python SDK** —— 同一条线协议的 Python 实现（**镜像**而非导入类型），额外解决了"对端运行时怎么装到用户机器上"这个分发问题。
- **ACP 服务器** —— `packages/acp/acp/`，标准 [Agent Client Protocol](https://agentclientprotocol.com) 的服务端。你是客户端，dsh 是被驱动的 agent；协议不是仓库自有的。
- **hook 桥** —— `packages/hooks/`，方向相反：不是外部程序驱动 dsh，而是 dsh 在运行途中回调你已经写好的 Claude Code / Codex shell 钩子。

### 2.1 选择表

| 维度 | TypeScript JSON-RPC SDK | Python SDK | ACP 服务器 | hook 桥 |
| --- | --- | --- | --- | --- |
| **包/入口** | [`packages/sdk/`](../packages/sdk/README.md#packages) | [`python/sdk/`](../python/sdk/README.md) | [`packages/acp/acp/`](../packages/acp/acp/README.md) | [`packages/hooks/`](../packages/hooks/README.md#packages) |
| **进程模型** | 你的进程 + 一个 dsh 子进程 | 你的 Python 进程 + 一个 dsh 子进程 | 你的进程 + 一个 dsh 子进程（或你连到已启动的服务端） | 无新进程模型：dsh 在自己的运行途中 fork 你的 hook 命令 |
| **传输** | stdio，换行分隔 JSON-RPC 2.0 | 同左（同一线协议） | stdio，ACP JSON-RPC | 子进程 argv + stdin/stdout + 退出码 |
| **谁发起** | 客户端 spawn 运行时 | 客户端 spawn 运行时 | 客户端 spawn 或连接服务端 | dsh 发起，你被回调 |
| **协议归属** | 仓库自有（无版本协商，见[协议限制](../packages/sdk/protocol/README.md#known-limitations-and-deferred-work)） | 仓库自有，Python 侧镜像类型 | 外部标准，v1 表面 | 外部工具的 hook 方言（Claude Code / Codex） |
| **会话生命周期** | 一个 `sessionId` 一个 agent，进程存活期内一直在；线上**无**关闭单个会话的方法 | 同左 | `session/new` / `list` / `resume` / `close`，且跨进程重启可 resume | 不拥有会话；随 agent run 触发 |
| **能不能取消 / 打断** | 不能。放弃一个 turn 只能关掉整个运行时进程 | 同左 | 能。`session/cancel` 与 `$/cancel_request` 是协议一等公民 | hook 可以**阻断**一次 prompt/工具调用（退出码 2），但不能取消进行中的 turn |
| **模型路由何时定** | `initialize` 握手时定；服务端校验精确路由后才接受 prompt | 同左 | `session/set_config_option` 可换 `model` / `reasoning_effort`，下一轮生效 | 不管路由 |
| **MCP 服务器挂载** | 由 profile 组合决定，客户端不能按会话挂载 | 同左 | 客户端可在 `session/new` 时声明 stdio / Streamable HTTP MCP | 不适用 |
| **审批 / 权限交互** | 无。线上有 server→client 请求的传输能力，但服务端从不发（"死能力"） | 同左 | 有。`session/request_permission`，客户端可自动应答 | hook 自身就是审批点（Claude Code 侧还能 `ask`） |
| **可观测到什么** | 该运行时**全部**会话的 `session.event` + 整体 `session.status` + subagent 起止；会话树过滤在客户端做 | 同左 | 标准语义更新：已提交的 assistant 消息与 thought、通用工具生命周期、配置变更、上下文用量 | 只看到自己那一个 hook 事件的 payload |
| **典型适用场景** | 仓库邻近的 TypeScript 自动化；知道自己在启哪个运行时；[SDK subagent 后端](../packages/subagent/subagent-dsh-sdk/README.md)本身就是它的内部消费者 | Python 侧自动化、评测脚本、CI；**目标机器没有 Node.js** | 需要持久会话管理、按会话挂 MCP、需要取消与审批；或客户端不是 dsh（任何 ACP 兼容 agent 都行） | 你已经有一套 Claude Code / Codex 钩子，希望它们在 dsh 里原样生效 |
| **不适用场景** | 需要 per-prompt 结果、需要中途取消、需要按会话挂 MCP | 同左；另外 Python 侧不暴露完整 Cordis 树 | 需要 DSH 私有的展示数据（plan、todo、title、terminal、elicitation）——它**故意**不提供 | 需要参考工具里不存在的行为——那应该写原生 Cordis 插件 |
| **主要代价** | 无取消、无 per-prompt 结果，`run()` 的活动区间语义要自己理解 | 同上，加上 wheel 的平台矩阵约束 | 只有 ACP v1 标准表面；`session/load`、删除、fork、附加目录、SSE/ACP-transport MCP 都不支持 | 只跑 command 类 hook；`http` / `mcp_tool` / `prompt` / `agent` 处理器被跳过并告警 |

### 2.2 三条判据，按顺序问

上表信息量大，实际决策通常只需要按顺序回答三个问题：

1. **你写的是不是"驱动方"？** 如果你的程序是被 dsh 在 agent run 中途回调的，那不是 SDK 问题，是 hook 桥问题，直接跳到第 7 节。四条路里只有 hook 桥的箭头方向是反的。
2. **你的语言与部署环境是什么？** TypeScript 且与仓库同版本共存 → TS 客户端（它按 `@deepseek-ai/dsh` 的**同版本**解析可执行文件）。Python，尤其是目标机器没有 Node.js → Python SDK，因为只有它带了打包运行时。其他语言 → 只剩 ACP，因为那是唯一有第三方 SDK 生态的标准协议。
3. **你需要取消、按会话挂 MCP、或跨进程 resume 吗？** 需要任意一条 → ACP。这三件事恰好是自有 SDK 线协议**明确不做**的，而且都写在它的 Known Limitations 里，不是遗漏。

判据的来源全部是上游文档里的显式陈述：取消能力的缺失来自 [SDK 协议限制](../packages/sdk/protocol/README.md#known-limitations-and-deferred-work)，ACP 的方法矩阵与不支持面来自 [`dsh-acp` 协议契约](../packages/acp/acp/README.md#standard-acp-v1-surface)，"无 Node.js 依赖"来自 [运行时 wheel 参考](../python/sdk-runtime/README.md)，hook 桥的 command-only 约束来自 [`dsh-hook-protocol`](../packages/hooks/hook-protocol/README.md#use-this-package)。

### 2.3 三个横切事实，选哪条路都要面对

这三条散落在四份上游文档里，但对任何一条路径都成立，所以放在这里一次说清：

- **凭据策略由调用方拥有，不由协议拥有**。TypeScript 客户端的 `HarnessClientOptions.env` 一旦给出就**整体替换**子进程环境（不给则继承父进程），`dsh-subprocess` 的 `scrubbedParentEnv` 是共享的清洗基线；Python 侧的 `base_url` 与 `api_key` 是对子环境里 `DEEPSEEK_BASE_URL` / `DEEPSEEK_API_KEY` 的显式覆盖。没有哪条路径会替你决定"哪些环境变量该进子进程"。
- **没有传输层认证，信任来自你启动了它**。ACP 的 `authenticate` 立即成功，服务端不要求认证，客户端被明确当作受信任的控制器；自有 SDK 线协议同样没有认证概念。四条路径的安全边界都在"谁能启动这个进程、这个进程能访问什么"这一层，不在协议里——详见 [安全与沙箱](security_and_sandbox.md)。
- **隔离粒度是 Harness home + session id 这两个旋钮**。选定的 home 存放 profile、插件和一切 profile 拥有的持久资源；同时复用同一个 harness 与同一个 session id 会**继续**那段持久对话与会话级资源。需要资源隔离就换一个 home，需要独立工作就换一个 session id。跑批与并发自动化最常见的翻车点就在这里。

---

## 3. 共同的地基：所有自动化入口都落在 `dsh --profile <name>`

> **上游事实源**：[`docs/architecture.md` — Application launch](../docs/architecture.md#application-launch)、[`AGENTS.md`](../AGENTS.md)

这是理解全部四条路径的前提，也是仓库的一条硬规则：**每个受支持的 Node 应用都从 `dsh` CLI 加一个具名 profile 启动**。SDK 客户端不是"另一个应用"，它只是一个会 spawn `dsh` 的库；自定义组合的方式是"profile + 有序 patch 文件"，而**不是**再造一个可执行文件或在代码里内联一棵 Cordis 树。守卫脚本 [`scripts/verify-application-entrypoints.ts`](../scripts/verify-application-entrypoints.ts) 会把每个包的 bin、可执行源文件和根 demo 归入显式类别，并拒绝绕过 `dsh` 的 Node 应用路径。

各路径落在哪个 profile：

| 路径 | profile | 承载的 bundle |
| --- | --- | --- |
| TypeScript SDK（默认） | `sdk` | [`dsh-sdk-app`](../packages/bundle/sdk-app/README.md#use-this-package)（叠加在 `dsh-base` 之上） |
| TypeScript / Python SDK（最小树） | `sdk-minimal` | [`dsh-sdk-minimal`](../packages/bundle/sdk-minimal/README.md#use-this-package)（**不**叠加 `dsh-base`） |
| Python SDK（默认） | `sdk` | 同 TypeScript |
| ACP 服务器 | `acp` | [`dsh-acp-app`](../packages/bundle/acp-app/README.md#standard-automation-workflow) |
| hook 桥 | 任意 | 桥是插件行，挂在任何组合里都行 |

已发布的 profile 模板是 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`；分层合并顺序（bundle 顺序 → profile 的 `cordis.patch.yml` → home 级 → `--patch` 覆盖层）见 [Profiles and bundles](../docs/architecture.md#profiles-and-bundles)，组装机制归 [app-boot](../packages/boot/app-boot/README.md#profiles)。这里只强调两条对自动化最要命的推论：

- **`sdk` / `sdk-minimal` / `acp` 都是 `patchReload: startup`**，即启动时一次性套用全部层。原因写在架构文档里：在一个 stdio 应用已经持有工作之后替换它的依赖，会让那个生命周期失效。所以"改配置要重启进程"不是 bug。
- **stdout 就是协议**。三个 stdio 应用的 stdout 只能出现协议帧，诊断必须走 stderr。profile 与 per-launch patch 属于**受信任的应用组合**，一个被你插进去的 stdout logger 足以破坏 JSON-RPC 分帧，而服务端插件不会去检查或否决同级 logger。

---

## 4. TypeScript JSON-RPC SDK

> **上游事实源**：[`packages/sdk/README.md`](../packages/sdk/README.md#packages)、[protocol](../packages/sdk/protocol/README.md#use-this-package)、[client](../packages/sdk/client/README.md#use-this-package)、[server](../packages/sdk/server/README.md#use-this-package)

### 4.1 三件套的分工

三个包对应线协议的三个位置，边界很干净，记住这条就不会找错文档：

| 包 | 在哪一侧 | 拥有什么 |
| --- | --- | --- |
| [`dsh-sdk-protocol`](../packages/sdk/protocol/README.md) | 两侧共享 | 传输类 + 具名的请求/结果/通知类型。纯库：无插件、无配置、无注册 |
| [`dsh-sdk-client`](../packages/sdk/client/README.md) | 客户端进程 | spawn 子进程、握手、订阅扇出、拆卸阶梯、类型化错误。纯库：不在 Cordis 上注册任何东西 |
| [`dsh-sdk-jsonrpc-server`](../packages/sdk/server/README.md) | 运行时进程 | `jsonrpc` 插件，按 `sessionId` 建 agent，把会话/状态/subagent 事件转成通知 |

方法集本身很小，两张 map 就是全部入口——这也解释了为什么"取消"必须靠关进程：

```ts
/** Server-to-client notifications by JSON-RPC method name. */
export interface HarnessSdkNotificationMap {
  'session.event': SessionEventNotification
  'session.status': SessionStatusNotification
  'subagent.started': SubagentStartedNotification
  'subagent.finished': SubagentFinishedNotification
}

/** Client-to-server request methods with their param and result shapes. */
export interface HarnessSdkRequestMap {
  'initialize': { params: InitializeParams; result: InitializeResult }
  'session/prompt': { params: SessionPromptParams; result: SessionPromptResult }
  'shutdown': { params: undefined; result: Record<string, never> }
}
```

`HarnessSdkNotificationMap` / `HarnessSdkRequestMap`，`packages/sdk/protocol/src/types.ts:106-119`。每个字段的语义（尤其是 `SessionPromptResult.messageId` **不**标识后续的 assistant 消息或 turn 结束）在 [协议 README 的 Payload semantics](../packages/sdk/protocol/README.md#payload-semantics) 里，不在这里复述。

### 4.2 客户端只是个 launcher

第 3 节那条硬规则在客户端代码里是字面成立的——它拼的就是 `dsh --profile X --patch A --patch B`：

```ts
export function resolveDshLaunch(
  options: HarnessClientOptions = {},
  callerCwd: string = process.cwd(),
): RuntimeProcessOptions {
  const profile = options.profile ?? 'sdk'
  const dshLaunch = options.dshBin === undefined
    ? installedDshNodeLaunch()
    : { nodeArgs: [resolve(callerCwd, options.dshBin)], patches: [], environment: {} }
  const patches = [
    ...dshLaunch.patches,
    ...(options.patches ?? []).map(path => resolve(callerCwd, path)),
  ]
  const dshHome = options.dshHome === undefined ? undefined : resolve(callerCwd, options.dshHome)
  return {
    command: process.execPath,
    args: [...dshLaunch.nodeArgs, '--profile', profile, ...patches.flatMap(path => ['--patch', path])],
```

`resolveDshLaunch`，`packages/sdk/client/src/launch.ts:128-143`。三点值得记：默认 profile 是 `sdk`；`dshBin` 省略时解析**同版本**的 `@deepseek-ai/dsh` 包 bin（版本不符直接抛错）；caller 给的 patch 路径按调用方 cwd 解析成绝对路径，并排在内部 patch 之后。

TypeScript 侧**没有**打包运行时的解析逻辑——这是与 Python 的关键分歧，原因写在 [客户端的 Known Limitations](../packages/sdk/client/README.md#known-limitations-and-deferred-work)：在出现 TypeScript 分发消费者之前，打包可执行文件的发现留在 Python 侧。

### 4.3 `sdk` 与 `sdk-minimal`：不是"大小档位"，是两种组合方式

这两个 profile 常被误当成同一棵树的两个规格。它们的差别是结构性的：

| | `sdk` | `sdk-minimal` |
| --- | --- | --- |
| 是否叠加 `dsh-base` | 是 | **否**，[架构文档明说它是"deliberate exception"](../docs/architecture.md#profiles-and-bundles) |
| 树的来源 | base 提供适配器/工具/持久化/沙箱与审批策略/设置/凭据/遥测，`dsh-sdk-app` 只加 SDK 应用层 | 一个 bundle 的单次 insert 就是完整应用树 |
| 工具 | base 的完整默认工具册 | 只有平台选定的持久 shell（Linux/macOS 为 Bash，Windows 为 PowerShell）与 `str_replace_editor` |
| 持久化 | base 的持久化栈 | `$DSH_HOME/sessions` 下的未压缩 JSONL |
| 显式排除 | —— | Web、settings、托管凭据、遥测、compaction、workspace instructions、skills、jobs、subagents |
| 安全姿态 | base 的沙箱与审批策略 | danger-full-access：shell 与 editor 能改进程可达的任意路径，**只能配隔离工作区使用** |

关键是最后两行：`sdk-minimal` 少的不只是工具，还包括 compaction 与审批策略。选它是在换取"树完全可枚举、每加一行都是显式 profile 变更"，代价是安全边界要由你自己在外部提供（见 [安全与沙箱](security_and_sandbox.md)）。两个 profile 共用同一个启动 provider，所以 `--help` 行为一致：写完帮助就退出，不占用 stdin/stdout。

### 4.4 仓库内部也在消费它

[`dsh-subagent-dsh-sdk`](../packages/subagent/subagent-dsh-sdk/README.md) 用这个客户端把每个委派的子任务跑成一个完整的对等 harness 子进程。读它的源码是理解客户端契约的最好办法，因为它是唯一一个必须处理全部失败模式的仓库内消费者。子代理后端的整体图景见 [Agent 循环与工具](agent_loop_and_tools.md)。

---

## 5. Python SDK

> **上游事实源**：[`python/README.md`](../python/README.md)、[`python/sdk/README.md`](../python/sdk/README.md)、[`python/sdk-runtime/README.md`](../python/sdk-runtime/README.md)、[`docs/architecture.md` — Application launch](../docs/architecture.md#application-launch)

### 5.1 它不是"另一个应用"

这是最容易搞反的一点，也是架构文档专门用一整段澄清的：**Python SDK 遵循同一套应用架构**。它的运行时 wheel 打包的是**普通的 `dsh` CLI**（打成名为 `deepseek-harness-sdk-runtime-<platform>-<arch>` 的原生可执行文件），客户端默认启动的是 `dsh --profile sdk`。Python 暴露给你的是 **profile 选择 + 有序 patch 文件**，**不是**一棵完整的 Cordis 树；此前那个私有的 direct-config 载体已被移除，且没有兼容 bin 或回退解析器。

argv 的拼装与 TypeScript 侧同构：

```py
        patches = tuple(
            argument
            for patch in self.config.patches
            for argument in ("--patch", str(Path(patch).expanduser().resolve()))
        )
        return (*base, "--profile", self.config.profile, *patches)
```

`_default_launch_args`，`python/sdk/src/deepseek_harness/client.py:481-486`。

### 5.2 wheel 解决的是分发问题

Python 路径的独特价值不在 API，而在"对端怎么装上去"。三件事上游写得很细，这里只给结论与链接：

- **无需系统 Node.js**。wheel 把 `dsh` 及其闭合的 Node 依赖树打成原生可执行文件，同时也打包了 `dsh web` 与前端资源以供单独 CLI 使用。
- **Harness home 必须显式给**。`dsh_home` 参数或子环境里非空的 `DSH_HOME`，二选一。Python 侧**故意从不**去发现 `~/.dsh`——这条在 [`python/sdk/README.md`](../python/sdk/README.md) 与 [运行时 wheel 参考](../python/sdk-runtime/README.md) 里都写死了，是有意的隔离设计而非疏漏。
- **打包运行时里的模块解析需要真实代理包**。因为操作系统符号链接进不了打包可执行文件的虚拟文件系统，打包启动会在 `$DSH_HOME/profiles/node_modules` 下维护小的真实 ESM 代理包，让内建行与外部插件 peer 共享同一个 Cordis/模块实例。这是排查"外部插件在源码模式能跑、在 wheel 里报重复实例"类问题的关键背景。

平台矩阵、sidecar（ripgrep、macOS 的 `node-pty` spawn helper、Windows ConPTY）与构建脚本都在 [运行时 wheel 参考](../python/sdk-runtime/README.md)；构建与校验流程在 [Python contributor workflows](../python/development.md)。

### 5.3 三种定制层级，按持久性排

| 你想做的事 | 手段 | 持久性 |
| --- | --- | --- |
| 换一整套组合 | `profile="sdk-minimal"` 或另一个已存在的 profile | 由 profile 决定 |
| 单次调用改几行配置 | `patches=(...)`，绝对化后按序追加在 profile 与 home patch 层之后 | 仅本次启动 |
| 长期装一个外部插件 | `dsh plugin --profile sdk add file:/abs/path/to/bundle` | 写进 profile 包树与清单，跨进程持久 |

`dsh plugin` 仅在管理外部包时需要 `PATH` 上有 `pnpm`；**运行** SDK 本身不需要。另外，自选的 profile 必须保留 `@deepseek-ai/dsh-sdk-app` 或另一行 `@deepseek-ai/dsh-sdk-jsonrpc-server`，否则没有对端应答 —— 配置错误会在 CLI 启动或 SDK 初始化时失败，**没有**完整配置回退。插件开发本身见 [插件开发指南](plugin_development_guide.md)。

面向使用者的中文教程在 [`docs/user/guide/python-sdk.md`](../docs/user/guide/python-sdk.md)，可运行样例在 [`python/sdk/examples/`](../python/sdk/examples/README.md)。

---

## 6. ACP 服务器

> **上游事实源**：[`packages/acp/README.md`](../packages/acp/README.md)、[`packages/acp/acp/README.md`](../packages/acp/acp/README.md#standard-acp-v1-surface)、[`packages/bundle/acp-app/README.md`](../packages/bundle/acp-app/README.md#standard-automation-workflow)

### 6.1 "automation-only" 到底约束了什么

这个词不是"没做 UI"的委婉说法，而是一条**设计承诺**，并且是双向的：

- **线上只走标准语义**：已提交的消息与 thought、通用工具生命周期、配置变更、上下文用量。原始的 provider delta、重试尝试、DSH 展示数据、不支持的内容一律**不上线**。
- **不发明私有面**：ACP 应用 bundle 明说它不添加任何私有方法、能力、`_meta`、环境变量或传输字段，并且有一个 keyless 的控制面一致性测试用公开 ACP SDK 驱动真实 profile。
- **因此，人类界面所需的东西它故意没有**：DSH 特有的展示卡片、plan、title、todo、terminal 视图、elicitation 都不在表面上。`session/load`、删除、fork、附加目录、SSE 或 ACP-transport MCP、modes、commands、客户端文件系统操作同样被省略或拒绝。

换句话说：**如果你在犹豫"这条能不能给人看"，那就是选错了路径**——那属于 [CLI 与 Web 应用装配](apps_cli_and_web.md)。

### 6.2 什么时候它是最优解

三种情形只有 ACP 能满足，第 2.2 节的第三条判据就是从这里来的：

1. **需要持久会话管理**。`session/list` 给出确定性的最新优先分页，`session/resume` 能在进程重启之后接回持久化的非活跃会话（校验其规范工作区，且**不**重放历史更新），`session/close` 只处置被寻址的那个 Agent scope，同连接上的其他会话不受影响。
2. **需要按会话挂 MCP**。`session/new` 时声明标准 stdio 或 Streamable HTTP MCP 服务器，在发布 Agent 前完成校验；任何初始连接或发现失败都会回滚这个尚未发布的 Agent。ACP 客户端被当作**受信任的控制器**——stdio 条目授权其绝对命令与环境，HTTP 条目授权其绝对 HTTP(S) URL 与 header。
3. **需要取消与审批**。`session/cancel` / `$/cancel_request` 是协议一等公民；`session/request_permission` 给出一次性的允许/拒绝选项，你的客户端可以自动应答，因此全流程无需人在环。

`pnpm dsh --profile acp` 即可启动已配好的 stdio 服务端；`acp` profile 挂载了会话持久化，list/resume/close 才成立。仓库自己的 ACP 客户端是 [`dsh-subagent-acp`](../packages/subagent/subagent-acp/README.md)，它启动的正是同一个 profile——这也是它与 SDK 子代理后端的分工：ACP 后端的子进程**可以不是 dsh**，任何 ACP 兼容 agent 都行。

`dsh-acp` 同时是 [扩展 cookbook](../docs/cookbook/extension-cookbook.md) 里 automation-only 的样板案例，写新扩展时值得对照。

---

## 7. hook 桥

> **上游事实源**：[`packages/hooks/README.md`](../packages/hooks/README.md#packages)、[`dsh-hook-protocol`](../packages/hooks/hook-protocol/README.md#use-this-package)

### 7.1 它解决的问题：不重写

前三条路径回答"外部程序怎么驱动 dsh"，这一条回答的是相反方向的问题：**你已经为 Claude Code 或 Codex 写好了一堆 shell 钩子，怎么让它们在 dsh 的 agent run 里原样生效。** 挂上对应的桥，把 `configPath` 指向你现有的 `hooks.json`，钩子就在对应时机触发——不需要改写成原生插件。

两个桥覆盖各自参考工具所记录的 **command hook 子集**。共同的引擎是 [`dsh-hook-protocol`](../packages/hooks/hook-protocol/README.md)，你从不直接安装或配置它；它的存在保证了两种方言在协议一致的地方行为一致。一个 hook 能做的事就那么几件：以退出码 2 阻断并把 stderr 作为原因给模型看、附加上下文进入下一次请求、在选定事件上触发、以及非 2 的退出码作为非阻断失败被记录。

### 7.2 两种方言的差别

| | `dsh-hooks-claude-code` | `dsh-hooks-codex` |
| --- | --- | --- |
| 事件 | `SessionStart` / `UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `Stop` / `SubagentStart` / `SubagentStop` | 前五个 |
| `PreToolUse` 能力 | 可阻断，**也可请求确认**（映射到 `ask`） | 只能阻断 |
| 变量替换 | `${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_PROJECT_DIR}`，并设置 `CLAUDE_PROJECT_DIR` 环境变量 | 无；有 `model` 字段盖在每个 payload 上 |
| 额外跳过 | —— | `async: true` 的 hook 被跳过并告警 |

两边共同的失败语义：配置读不出或解析失败 → 记一条 warning、不跑任何 hook、agent 照常启动；hook 起不来或崩溃 → 记录并继续。完整字段表见各自 README 与生成的配置目录（[claude-code](../docs/config-catalog.md#deepseek-aidsh-hooks-claude-code) / [codex](../docs/config-catalog.md#deepseek-aidsh-hooks-codex)）。

### 7.3 什么时候不该用桥

上游给的判据很直接：**参考工具里没有对应物的行为，就不该用桥**。桥只跑参考工具的 command-hook 子集，`http`、`mcp_tool`、`prompt`、`agent` 类处理器会被跳过并告警；而原生 Cordis 插件拿到的是完整的 harness API，中间没有 hook 协议这一层。桥的每个事件都程序化地对应一个 harness 扩展点（waterfall 或 serial listener），所以"桥能做什么"永远是"扩展点能做什么"的子集。要走原生路线见 [插件开发指南](plugin_development_guide.md) 与 [能力接缝设计](capability_seams.md)。

---

## 8. Typert RPC 网关与 Remote BFF

> **上游事实源**：[`docs/api-gateway.md`](../docs/api-gateway.md#programming-model)、[`docs/api-gateway.md` — Boundaries](../docs/api-gateway.md#boundaries)、[`docs/subsystems/typert.md`](../docs/subsystems/typert.md)、[`packages/api/README.md`](../packages/api/README.md#packages)

把它放进这张图里，是为了**划清它不是第五条路径**。`packages/api/` 与 [`packages/typert/`](../packages/typert/README.md) 服务的是**应用内部**的 Client↔Host 边界（也就是 Web GUI 的浏览器侧调用宿主侧能力），不是给外部自动化程序用的公共协议。

| | 四条集成路径 | Typert Remote / API BFF |
| --- | --- | --- |
| 边界两端 | 你的进程 ↔ dsh 运行时进程 | 应用自己的 Client 环境 ↔ Host 环境 |
| 传输 | stdio（JSON-RPC / ACP）或子进程回调 | 应用共享的 Connection 与 `/api` 路由 |
| 契约来源 | 手写线协议 / 外部标准 | 从源码类型声明**生成**的 Host 与 Client 契约 |
| 消费者 | 外部程序 | Web 前端等应用内 Client |

分层是 `remotes → gateway → connection → webserver`：[`remotes/`](../packages/api/remotes/README.md) 决定哪些 Host 能力对 Client 可见、每个调用怎么落到正确会话的 agent 上；[`gateway/`](../packages/api/gateway/README.md) 承载类型化的一元调用、复用流与转发的 Host 事件（Host 侧 `ctx.typertGateway`，Client 侧 `ctx.remote`）。

有一条边界值得自动化作者知道：**Remote 只处理一元调用**（一请求一结果）。会话事件流、分页、增量 reduce、投影、实体子流需要独立的数据协议与注册模型，即使复用同一条 Connection 也不得伪装成 Remote 方法或进入调用描述符。这解释了为什么外部自动化拿事件走的是 SDK/ACP 的 stdio 通知，而不是 Remote。生成管线与运行时调用细节见 [`docs/api-gateway.md`](../docs/api-gateway.md#component-responsibilities)，公开类型契约见 [`docs/subsystems/typert.md`](../docs/subsystems/typert.md#host-gateway)。

---

## 9. 改动 SDK 时的义务

> **上游事实源**：[`docs/testing.md` — When a snapshot test is required](../docs/testing.md#when-a-snapshot-test-is-required)、[`docs/testing.md` — Tiers](../docs/testing.md#tiers)

这一节是本文对开发者最有约束力的部分，也是最容易漏做的一件事。测试政策的原文要求是：**agent-loop、session lifecycle、`SessionEventMap` 的改动，必须同时更新两套 SDK 投影**。

| 投影 | 归属 | 由谁执行 |
| --- | --- | --- |
| TypeScript | [`snapshots/sdk/`](../snapshots/sdk) | `pnpm run test:snapshot` |
| Python | `scripts/snapshots/python-sdk-single-exe/` | 必需的 `python-runtime` CI job |

**为什么必须两套都改**：Python SDK 是**镜像**协议形状而非导入它们（见 [协议 README 的 Dev Note](../packages/sdk/protocol/README.md)）。类型系统在这里帮不上忙——改了 TypeScript 侧的方法、payload 或线稳定的 `serverInfo.name`，Python 侧不会有任何编译错误，只有快照会失败。同一个 PR 里必须一起改 TypeScript 客户端与 Python 对应实现。

**为什么 `pnpm run test` 不够**：按 [Tiers](../docs/testing.md#tiers)，`pnpm run test` 是 vitest 单元层，覆盖各包 `tests/**` 下的 spec 与仓库脚本 spec，它**不**跑进程级的录制会话场景。SDK 与 ACP 的行为证据在 `test:snapshot` 层：场景通过 `dsh` 启动真实进程，SDK 场景负责"持久控制"语义，ACP 场景负责"自动化协议"行为。所以只跑 `pnpm run test` 绿了，不构成 SDK 改动的证据。

**实操顺序**：模型转录变了用 `test:snapshot:record`，回放输入仍有效用 `test:snapshot:refresh`，两者产生的每一处 diff 都要人工审阅后再提交。Python 侧的三个场景（`minimal/`、`advanced/`、`restart/`）用 `scripts/smoke-python-runtime.py` 驱动打包运行时，重跑对应场景时加 `--update-snapshots` 并审阅 diff——流程见 [Python contributor workflows](../python/development.md)。快照仓库自身的存放与命名规则见 [`snapshots/AGENTS.md`](../snapshots/AGENTS.md)，整体测试策略见 [测试指南](testing_guide.md) 与 [质量门禁](quality_gates.md)。

还有两条容易被忽略的连带义务：

- **改 stdout 行为等于改协议**。任何在 SDK/ACP 组合里新增日志输出的插件，都可能破坏分帧；服务端不会替你拦。
- **改 profile 组合要同时想清楚 `sdk-minimal`**。它是独立完整树，不会因为你改了 `dsh-base` 而自动跟进——反过来，你在 base 里修的 bug 它也不会自动得到。

---

## 10. 延伸阅读

**上游事实源（优先读这些）**

- [`docs/architecture.md`](../docs/architecture.md) — profile/bundle 分层与应用启动硬规则的归属地。
- [`packages/sdk/README.md`](../packages/sdk/README.md#packages) — SDK 三件套的包地图。
- [`packages/acp/acp/README.md`](../packages/acp/acp/README.md#standard-acp-v1-surface) — ACP v1 方法矩阵、MCP 信任模型、stopReason 映射。
- [`python/README.md`](../python/README.md) — Python 两个 dist 的分工入口。
- [`docs/api-gateway.md`](../docs/api-gateway.md) — Typert Remote 的编程模型与边界。
- [`docs/testing.md`](../docs/testing.md#when-a-snapshot-test-is-required) — 快照义务的原文。
- [`docs/config-catalog.md`](../docs/config-catalog.md) — 每个插件的完整配置字段（生成产物）。

**本仓中文导航层的兄弟文档**

- [AI 编码上下文](AI_Coding_Context.md) — 全局入口与阅读顺序。
- [架构总览](architecture_overview.md) — 本文第 3 节地基的上下文。
- [CLI 与 Web 应用装配](apps_cli_and_web.md) — 面向人类的两个入口，与本文互补。
- [会话与事件](session_and_events.md) — `SessionEvent` 词表，是 SDK 通知 payload 的一部分。
- [Agent 循环与工具](agent_loop_and_tools.md) — 子代理后端与工具执行管线。
- [能力接缝设计](capability_seams.md)、[插件开发指南](plugin_development_guide.md) — 桥不够用时的原生路线。
- [模型配置](model_configuration.md) — `provider` / `model` / `reasoningEffort` / `maxTokens` 的来龙去脉。
- [安全与沙箱](security_and_sandbox.md) — `sdk-minimal` 的 danger-full-access 姿态需要的外部边界。
- [测试指南](testing_guide.md)、[质量门禁](quality_gates.md) — 第 9 节义务的完整语境。
- [Monorepo 与构建](monorepo_and_build.md) — 运行时 wheel 构建脚本所处的构建体系。
- [发布与分发](deployment_guide.md)、[故障排查](troubleshooting.md) — 发布与排障。
- [成本优化](cost_optimization.md)、[评估指标](evaluation_metrics.md) — 自动化跑批时的两个配套话题。
