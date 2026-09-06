---
title: 评测与基准
summary: 这个仓库不靠 benchmark 分数判断好坏，而靠无密钥的录制会话回放判断"输出是否变了"——本文解释这个取舍及其边界。
keywords: benchmark | snapshot | replay | evaluation | regression
scope: deepseek-harness 如何判断改动是否让 agent 变差
related_files: BENCHMARK.md | vitest.snapshot.config.ts | snapshots/AGENTS.md | packages/test-support/llm-replay
dependencies: BENCHMARK.md | docs/testing.md | snapshots/AGENTS.md
verified_at: 2026-09-06
---

# 评测与基准

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。基准的权威说明归 [`BENCHMARK.md`](../BENCHMARK.md)，快照归属规则归 [`snapshots/AGENTS.md`](../snapshots/AGENTS.md)，测试策略归 [`docs/testing.md`](../docs/testing.md)。本文不复述它们，只回答一个上游没有集中回答的问题：**这个仓库靠什么判断一次改动让 agent 变好还是变差。**
>
> **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即修正本文。

## 1. 本文定位与上游事实源

> **上游事实源**：[`BENCHMARK.md`](../BENCHMARK.md)、[`docs/testing.md`](../docs/testing.md)、[`snapshots/AGENTS.md`](../snapshots/AGENTS.md)

如果你带着"这个 agent 在 SWE-bench 上多少分"的问题打开本文，先接受一个结论：**这个仓库里没有这样的数字，也没有产生这类数字的流水线。** `docs/` 下没有 benchmark 或 evaluation 页面，CI 里没有打分作业，仓库根的 `BENCHMARK.md` 只有三行，讲的是"你自己怎么跑基准任务"而不是"我们跑出了什么分数"。

这不是缺失，是取舍。仓库把"回归检测"和"能力评估"拆成了两件事，只把前者做成硬门禁：**每次改动都必须证明 agent 的可观测输出没有意外变化**，而"变化是否是变好"交给人在 review diff 时判断。承载这个证明的机制是**无密钥的录制会话回放**（keyless recorded-session replay），它是 [`docs/testing.md` 的 `## Tiers`](../docs/testing.md#tiers) 里的 Snapshot 层。

本文回答三个上游分散着说、没人集中说的问题：**为什么是回放而不是打分**、**回放到底比对了什么**、**我这次改动要不要动快照**。具体规则一律以上游为准，本文只做导航。

## 2. 这个仓库怎么评测：回放而不是打分

> **上游事实源**：[`vitest.snapshot.config.ts`](../vitest.snapshot.config.ts)、[`docs/testing.md` 的 `## Tiers`](../docs/testing.md#tiers)、[`docs/testing.md` 的 `## The with-key policy: inference is cheap here`](../docs/testing.md#the-with-key-policy-inference-is-cheap-here)

核心机制写在快照配置的注释里，这是全仓库对该取舍最凝练的一句话：

```ts
// Replay is the keyless default: boot real subprocess paths from recorded model responses and diff
// assembled requests, normalized protocol or transcript output, and persisted-log expected outputs.
// `record` calls the real API and updates fixtures and expected outputs; `refresh` replays committed scripts
// and updates current expected outputs. Replay/refresh never load `.env`; only record reads a key from the
// environment or root `.env`.
```

（`vitest.snapshot.config.ts:24-28`）

拆开看，回放同时买到了三样东西：

- **无密钥**——回放不读 `.env`、不发任何 provider 请求，因此秘密缺失的 CI 和没有 key 的外部贡献者都能跑完整的快照层。只有录制需要真实 key。
- **确定性**——模型这个唯一的非确定性来源被换成了一份committed 的 JSONL，同一份输入永远产出同一份输出，失败就是真失败，不是模型今天心情不好。
- **可 diff**——比对的对象不是"分数"而是**字节**：组装出的请求、归一化后的协议/转录输出、持久化日志的期望输出，以及（对会改动工作区的场景）完整的工作区结果树。任何一处变了，PR 里就出现一段可以逐行读的 diff。

代价必须说清楚，否则这套机制会被误用：**快照衡量的是"输出是否变化"，不是"输出是否更好"。** 它能在你改坏 prompt 组装、改乱事件顺序、悄悄丢掉一个工具字段时立刻变红；它无法告诉你新 prompt 是不是让模型更聪明了。判断"更好"的责任被显式地交回给人——[`snapshots/AGENTS.md`](../snapshots/AGENTS.md) 要求录制与刷新产生的每一份 JSONL、prompt、schema、协议、UI 与工作区 diff 都在 commit 前被审阅。**红色快照不等于 bug，绿色快照也不等于没变差**，它只是把"变了什么"完整摊开在你面前。

补上"是否更好"这一侧的，是另外两层，都在 [`docs/testing.md` 的 `## Tiers`](../docs/testing.md#tiers) 里：带 key 的真实 API e2e（`pnpm run test:e2e`），以及它背后那条明确的政策——"We are DeepSeek — do not ration real-API tests"，无密钥测试只证明管道通了，只有真实模型跑通才证明 agent 能用。所以完整的判断链是：**回放守住"没有意外变化"，带 key 的 e2e 与人工 review 守住"确实变好了"。**

按"你担心什么退化"反查该看哪份证据，比按测试分层正着记要实用得多：

| 你担心的退化 | 会变红的证据 | 跑什么 |
|---|---|---|
| 发给模型的请求组装变了（prompt、工具 schema、历史） | 快照场景比对的 assembled request | `pnpm run test:snapshot` |
| 事件顺序、生命周期、持久化日志形状变了 | 快照场景的 expected 持久化输出 | `pnpm run test:snapshot` |
| 工具真的没干成事（只是模型说干成了） | 场景独立的 `workspace.expected/` 结果树 | `pnpm run test:snapshot` |
| 协议线缆（ACP / JSON-RPC）输出变了 | `snapshots/acp/`、`snapshots/sdk/` 的归一化协议输出 | `pnpm run test:snapshot` |
| 浏览器渲染 / ARIA 变了 | `snapshots/web/` 与 `apps/web/tests/expected/` | `pnpm run test:web` |
| adapter 在网络异常下的重试/退避坏了 | `dsh-llm-mock-server` 驱动的恢复策略测试 | `pnpm run test` |
| agent 面对真实模型是否还能完成任务 | 带 key 的真实 API e2e（无 key 自动跳过） | `pnpm run test:e2e` |

注意最后一行的性质差异：**前六行在无密钥 CI 上是必跑门禁，最后一行没有 key 就自跳过。** 这正是"回归检测硬、能力评估软"这个取舍的具体形状。

## 3. `BENCHMARK.md` 实际是什么

> **上游事实源**：[`BENCHMARK.md`](../BENCHMARK.md)、[`docs/user/guide/python-sdk.md`](../docs/user/guide/python-sdk.md)

`BENCHMARK.md` 全文三行，内容是一条操作指引：按 [Get started with the Python SDK](../docs/user/guide/python-sdk.md) 安装 SDK 并运行最小变体，**并且给互相独立的基准任务使用独立的 workspace 与 session id**。

所以它的定位是：**给"想拿这个 harness 去跑自己的基准"的人的入口**，不是仓库自己的成绩单。它不定义任务集、不定义指标、不产出数字、不在任何 CI 作业里被执行（全仓库唯一的其他引用在 fork 本地的 [`UPSTREAM_DOC_ISSUES.md`](../UPSTREAM_DOC_ISSUES.md) U7 条目里，上游代码与 CI 均不引用它）。

它唯一带信息量的约束值得单独强调，因为它是最容易踩的坑：**独立任务必须用独立的 workspace 和 session id**。原因在 [`docs/user/guide/python-sdk.md` 的 `## Understand the minimal profile`](../docs/user/guide/python-sdk.md#understand-the-minimal-profile) 里能读到——最小 profile 的会话以未压缩 JSONL 持久化在 `<dsh_home>/sessions` 下，复用 home 与 session id 意味着**继续同一段持久对话**，而不是开始一次干净的新任务。基准任务之间串了历史，测出来的东西就没有意义了。同一节还写明这个 profile 固定 `danger-full-access`，跑基准请用一次性 checkout 或容器。

顺带一提命名：`BENCHMARK.md` 里说的 "`jsonrpc-agent` minimal variant"，其可运行程序如今位于 [`python/sdk/examples/`](../python/sdk/examples/README.md)，profile 名为 `sdk-minimal`，完整组合树归 [`packages/bundle/sdk-minimal`](../packages/bundle/sdk-minimal/README.md)。以后者为准。

## 4. 回放的机制：三种模式与两个 mock 包

> **上游事实源**：[`packages/test-support/llm-replay/README.md`](../packages/test-support/llm-replay/README.md)、[`packages/test-support/llm-mock-server/README.md`](../packages/test-support/llm-mock-server/README.md)、[`packages/test-support/session-snapshot/README.md`](../packages/test-support/session-snapshot/README.md)

三个脚本对应三种模式，只有中间那个需要 key：

```json
    "test:snapshot": "vitest run --config vitest.snapshot.config.ts",
    "test:snapshot:record": "DSH_SNAPSHOT=record vitest run --config vitest.snapshot.config.ts --update",
    "test:snapshot:refresh": "DSH_SNAPSHOT=refresh vitest run --config vitest.snapshot.config.ts",
```

（`package.json:42-44`）

- **`replay`（默认）**——无密钥，只读，不写任何 committed 输出。CI 跑的是它。
- **`record`**——调真实 API，重新录制 fixture 并更新期望输出。**需要 key**，是唯一会加载 `.env` 的模式。当模型转录本身应当改变时用它。
- **`refresh`**——无密钥，回放 committed 脚本但把当前期望输出写回。当回放输入仍然有效、只是下游渲染/归一化结果变了时用它。

模式类型在 `packages/test-support/session-snapshot/src/suite.ts:267` 的 `SnapshotSuiteOptions.mode` 上声明为 `'replay' | 'record' | 'refresh'`；哪些场景在写模式下真的会被改写，由 `manifest.ts` 的 `writesCurrentSessionFixtures` 决定：

```ts
export function writesCurrentSessionFixtures(
  manifest: SnapshotManifest,
  mode: SnapshotSessionWriteMode,
): boolean {
  return mode !== 'replay' && manifest.session === undefined && manifest.sessionFormat === undefined
}
```

（`packages/test-support/session-snapshot/src/manifest.ts:136-141`）

读法：**回放永不写**；借用别人 session 的场景（`session` 字段）和刻意保留历史世代的场景（`sessionFormat` 字段）也永不被写模式改写。这是"独立 oracle"原则的代码化——用来验证的东西不能被被验证的东西重写。

两个 mock 包分工完全不同，别混：

| 包 | 替换的边界 | 用途 |
|---|---|---|
| [`dsh-llm-replay`](../packages/test-support/llm-replay/README.md) | LLM **适配器**（进程内） | 把一份录制的 session JSONL 投影成模型流，让真实 agent 在固定转录上跑完整场景 |
| [`dsh-llm-mock-server`](../packages/test-support/llm-mock-server/README.md) | **provider HTTP/SSE 线缆** | 脚本化 stream reset、stall、畸形 chunk、限流、5xx，让真实 DeepSeek adapter 在真实 wire 上被测恢复策略 |

即：**回放测"agent 在正常模型输出下的行为"，mock server 测"adapter 在异常线缆下的恢复"。** 前者是快照层的模型来源，后者服务重试/退避/超时这类恢复策略测试。

还有一个容易被忽略的第三层保护：**语料本身也被门禁烧机。** `test:snapshot` 的 include 列表里除了场景套件还有 `scripts/session-snapshot-corpus.corpus.ts`（见 `vitest.snapshot.config.ts:48`），它检查整个语料的归属与存储不变量；`packages/test-support/llm-replay/tests/session-format-corpus.spec.ts` 则发现 `snapshots/`、`packages/`、`scripts/snapshots/python-sdk-single-exe/` 下每一份带世代的 `session*.jsonl`，要求它们都能通过真实的 Session 格式目录还原到当前视图，任何一次拒绝都以 artifact 路径失败。**含义是：Session 格式演进不能悄悄让历史 fixture 变成不可读的死数据。** 这是把"评测基线本身的有效性"也纳入了门禁。

回放本身有两个必须知道的边界，来自 [`llm-replay` README 的 `## Use this package`](../packages/test-support/llm-replay/README.md#use-this-package)：**纯抛出（第一个 chunk 之前就失败）和 cancel/hang 无法从持久化的 Assistant 结算中重建**，需要 `replay.override.json` 旁车文件；**父子 agent 的脚本按首次调用顺序绑定**，所以并发同级 subagent 的场景绑定是不确定的。第三方封装 [`dsh-session-snapshot`](../packages/test-support/session-snapshot/README.md) 提供其余支撑：清单、身份脱敏、归一化、工作区比对、fixture 守卫，以及 headless / SDK / ACP / Web 四种协议适配器。

## 5. 四类作用域与归属规则

> **上游事实源**：[`snapshots/AGENTS.md`](../snapshots/AGENTS.md)、[`docs/testing.md` 的 `## When a snapshot test is required`](../docs/testing.md#when-a-snapshot-test-is-required)

`snapshots/` 下恰好四个作用域，由语料门禁的常量固定：

```ts
const profiles = ['acp', 'sdk', 'session', 'web'] as const
```

（`scripts/session-snapshot-corpus.corpus.ts:25`）

分工按**入口的所有权**切，不按功能切：`session/` 归 headless 一次性行为，`sdk/` 归持久控制，`acp/` 归自动化协议行为，`web/` 在同一份 Session 旁边保留浏览器/ARIA 证据。**四个作用域下的每个进程都从 `dsh` CLI 带着出厂 profile 启动**——[`snapshots/AGENTS.md`](../snapshots/AGENTS.md) 明确禁止为测试新增另一个应用入口、隐藏 CLI 模式或可执行场景驱动器。

三条最容易违反的归属规则，原文都在 [`snapshots/AGENTS.md`](../snapshots/AGENTS.md)，这里只做索引：

1. **只有 committed session JSONL 同时充当回放输入与期望持久化输出的测试才放这里。** 非会话驱动的 ARIA、几何、生成器、CLI、单元期望输出留在各自 app/script/package 的 `tests/expected/` 下，走 `test:expected` / `test:web` / `test` 层，并且**不用 `*.snapshot.ts` 后缀**。
2. **每个场景拥有一个主 Session 角色加连续的子角色**，文件名 `session[.vN].jsonl` / `session.<ordinal>[.vN].jsonl`，回放与录制一律选择数字最高的世代，文件名必须与 header 一致。共享引用只读、无环、指向拥有者选定的父世代。
3. **会改动工作区的场景要把完整结果树 commit 到 `workspace.expected/` 并设 `workspace.final: true`**，record 与 refresh 都不会重写它——模型的自述文字和工具结果文本不构成外部效果的证据。这是 [`docs/testing.md` 的 `## Verify the world, not the self-report`](../docs/testing.md#verify-the-world-not-the-self-report) 在快照层的落地。

每个场景目录的 `snapshot.yml` 声明 profile、组合、header 类与旁车归属、录制策略、例外的回放/输入元数据、平台与权限、工作区事实；可接受的字段以 `packages/test-support/session-snapshot/src/manifest.ts` 的 `SnapshotManifest` 接口（第 93 行起）为准。

## 6. 什么改动必须更新快照

> **上游事实源**：[`docs/testing.md` 的 `## When a snapshot test is required`](../docs/testing.md#when-a-snapshot-test-is-required)

判据只有一句，但覆盖面比大多数人预期的宽：**每一个非平凡的、模型可见 / 协议可见 / 人类可见的改动，都要在同一个 PR 里新增或更新一个无密钥录制会话场景。** 上游明确写了什么**不能**替代它：包级测试、e2e、纯 mock 测试、以及 rationale 说明，都不能代替那份组装出来的转录。

三个高频误判：

- **"我只改了 prompt 里一个词"**——那是模型可见改动，必须更新。快照层的 diff 正是给这种改动准备的。
- **"我只改了 Web 的一个渲染"**——那是人类可见改动。Web 场景在 `snapshots/web/` 下，可以显式借用另一个场景的 canonical session 来渲染同一段录制行为。
- **"我改的是 agent loop 内部"**——这是最容易漏的一类：**agent-loop、session 生命周期、`SessionEventMap` 的改动要同时更新两个 SDK 投影**。`snapshots/sdk/` 拥有 TypeScript 侧，而 Python 侧由必跑的 Python-runtime CI 拥有，期望输出在 [`scripts/snapshots/python-sdk-single-exe/`](../scripts/snapshots/python-sdk-single-exe) 下。只更新一侧会在 CI 才炸。

模式选择的判断很简单：**模型转录本身应当改变 → `test:snapshot:record`（需要 key）；回放输入仍然有效、只是下游输出变了 → `test:snapshot:refresh`（无密钥）。** 两种情况都必须逐条审阅产生的 diff 再 commit。新增能力接缝、生命周期或转录变体时，在**计划阶段**就要点名需要哪几层证据，而不是写完代码再补。

还有一类 diff 看着像回归、其实是 fixture 缺陷，值得单独识别：**committed session 是归一化的不动点。** [`snapshots/AGENTS.md`](../snapshots/AGENTS.md) 要求把易变身份替换成保持关系的类型化 token、把请求 system prompt 与工具 schema 替换成 token、每个 header 类恰好保留一个可读旁车拥有者。所以如果你看到的 diff 是一串随机 id、时间戳或绝对路径在抖，那不是 agent 变差了，而是某个易变值没被 token 化；**修 fixture 的归一化，不要用 refresh 把抖动 commit 进去。** 反过来同样重要：不要仅仅因为某段用户或工具文本"长得像标识符"就去脱敏它——那会掩盖真实的行为变化。

CI 侧的形态：快照与 Web 浏览器快照都是必跑门禁，且 CI 强制只读回放——`vitest.web.config.ts` 的注释写明 Linux PR CI 固定 `DSH_SNAPSHOT=replay` 并比对 committed golden，record/refresh 只是本地工作流。回放模式下文件级并行开启，并发度由 `DSH_SNAPSHOT_MAX_CONCURRENCY` 控制（默认取 `5` 与 `availableParallelism()` 的较小值，见 `vitest.snapshot.config.ts:6` 的 `DEFAULT_SNAPSHOT_MAX_CONCURRENCY`）；record 与 refresh 强制串行，因为并发写入会破坏期望输出。本地机器吃不消时把它设为 `1` 恢复完全串行。

## 7. 自查命令：不要相信任何手写的快照清单

场景集合每周都在变，任何文档里的清单在写下的那一刻就开始腐烂。下面的命令随时给你当前真相（在仓库根执行）：

```sh
# 四个作用域各有多少个场景
for p in acp sdk session web; do printf '%s\t%s\n' "$p" "$(find "snapshots/$p" -mindepth 1 -maxdepth 1 -type d | wc -l | tr -d ' ')"; done

# 全部场景清单（每个含 snapshot.yml 的目录就是一个场景）
find snapshots -name snapshot.yml | sort

# 哪些场景带独立的工作区 oracle
grep -rl 'final: true' snapshots --include=snapshot.yml | sort

# 哪些场景需要 replay.override.json 旁车（抛出 / cancel / 注入重试）
find snapshots -name 'replay.override.json' | sort

# 某个场景保留了哪些 Session 世代
find snapshots/session/compaction-recovery -name 'session*.jsonl' | sort

# 快照套件的驱动器（每个作用域一个入口，Web 的两个在 apps/web/tests 下）
find snapshots apps/web/tests -name '*.snapshot.ts' | sort
```

## 8. 其他信号：会话统计与人类反馈

> **上游事实源**：[`packages/session/session-stats/README.md`](../packages/session/session-stats/README.md)、[`packages/feedback/README.md`](../packages/feedback/README.md)、[`docs/subsystems/feedback.md`](../docs/subsystems/feedback.md)

快照之外，仓库里还有两类与"这次跑得好不好"相关的产物。**它们都不是门禁，也不参与 CI 判定**——把它们当成运行期可观测信号，不要当成评测指标。

**会话统计**由 [`dsh-session-stats`](../packages/session/session-stats/README.md) 以 `sessionStats` 投影单元的形式提供，字段语义见该 README 的 [`## Use this package`](../packages/session/session-stats/README.md#use-this-package) 表格：turn 与 step 计数，以及 LLM、工具、首 token、解码的墙钟时间。关键性质是**它从完整持久化日志折叠而来，因此分页与压缩都不会改变数值**；它只在挂载了投影注册表的组合里注册（Web 聊天 bundle 的统计条是参考消费者），没有注册表的装配退回窗口范围计数。想比较两次运行的耗时构成，这是仓库里现成的口径——但注意它是**单次会话的描述性数字，不是跨版本可比的基准指标**。

**人类反馈**由 [`feedback/` 包组](../packages/feedback/README.md)捕获，两个包彼此独立：`/feedback` 命令记录整段会话的自由文本评价，`messageFeedback` 服务提供逐条助手消息的好评/差评与备注。有两条性质必须分开记，混在一起会得出危险的结论。

**都不进入模型历史。** 两种反馈都是关于输出的信号，从不作为模型的输入（[`command-feedback/README.md`](../packages/feedback/command-feedback/README.md) 的记录事件"never surfaces to the model"）。所以反馈是给人看的、离线的质量信号，不构成任何自动化闭环。

**遥测侧两者相反。** `messageFeedback` 确实"never enter model history or telemetry"（[`message-feedback/README.md`](../packages/feedback/message-feedback/README.md)）；但会话级 `/feedback` 写的是一条普通会话日志事件，而出厂默认的遥测策略正是 `FEEDBACK_ONLY`——[`session-telemetry/README.md`](../packages/session/session-telemetry/README.md) 定义它为"nothing is handed over until a `feedback/record` event releases the unreleased prefix"。**记录一次 `/feedback`，就是把此前尚未共享的整段会话记录一并上传的动作**，而且 [`apps/cli/reference/README.md`](../apps/cli/reference/README.md#shared-deployment-behavior) 写明出厂 base **没有任何遥测脱敏规则**，导出内容可含消息正文、工具参数与结果、工作区路径。退出口径与关闭方式见 [`security_and_sandbox.md`](security_and_sandbox.md)。

命令侧只在 Web 客户端可用，headless / ACP / JSON-RPC 入口不提供 slash 命令。

## 9. 延伸阅读

- [`docs/testing.md`](../docs/testing.md) — 测试策略的权威归属地：分层、with-key 政策、真实入口路径、快照何时必需。
- [`snapshots/AGENTS.md`](../snapshots/AGENTS.md) — 快照语料的归属、命名、世代选择与工作区 oracle 规则。
- [`packages/test-support/README.md`](../packages/test-support/README.md) — 测试支撑包组的地图。
- [`.agents/skills/dsh-ci-test-reliability/SKILL.md`](../.agents/skills/dsh-ci-test-reliability/SKILL.md) — 并发执行下的资源、同步、超时与拆解规则；快照层的偶发失败先查这里。
- [`quality_gates.md`](./quality_gates.md) — 从门禁报错反查该改哪里，以及按改动面选最小检查集。
- [`testing_guide.md`](./testing_guide.md) — 测试分层与写法的中文导航。
- [`session_and_events.md`](./session_and_events.md) — Session 日志与事件模型；快照 fixture 就是它的投影。
- [`sdk_and_protocols.md`](./sdk_and_protocols.md) — 两个 SDK 投影与 ACP 协议面，对应快照的 `sdk/` 与 `acp/` 作用域。
