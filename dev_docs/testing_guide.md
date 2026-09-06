---
title: 测试指南
summary: 按"我改了什么"反向索引该跑哪层测试、是否必须重录快照、是否必须同步两个 SDK 的期望输出。
keywords: testing | vitest | coverage | snapshot | e2e | real-composition
scope: deepseek-harness 测试策略的中文导航层
related_files: docs/testing.md | vitest.config.ts | snapshots/AGENTS.md | packages/AGENTS.md
dependencies: docs/testing.md | packages/AGENTS.md | .agents/skills/dsh-pre-push-checks/SKILL.md
verified_at: 2026-09-06
---

# 测试指南

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。测试策略的权威归属地是 [`docs/testing.md`](../docs/testing.md)，快照归属规则归 [`snapshots/AGENTS.md`](../snapshots/AGENTS.md)，包级测试约束归 [`packages/AGENTS.md`](../packages/AGENTS.md)。本文不重写它们，只把上游按测试类型组织的内容**按改动类型反向组织**——你改了什么，就该测什么。
>
> **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即修正本文。

## 1. 本文定位与上游事实源

> **上游事实源**：[`docs/testing.md`](../docs/testing.md)（[中文版](../docs/testing.zh.md)）、[`AGENTS.md` 的 `## Commands`](../AGENTS.md#commands)、[`.agents/skills/dsh-pre-push-checks/SKILL.md`](../.agents/skills/dsh-pre-push-checks/SKILL.md)

[`docs/testing.md`](../docs/testing.md) 已经把每一层测试的职责、以及"什么时候必须写快照""什么时候不能用 mock""什么是真实入口路径"讲完了，而且是双语的。你应该**通读它一遍**，本文不打算复述。

本文补的是上游结构上的一个空白：上游按**测试类型**组织（先讲 unit，再讲 coverage，再讲 snapshot……），而你在写代码时的提问方向是相反的——你手里有一个 diff，你想知道"这个 diff 该配什么证据"。第 4 节就是这张反向表。

另外两件事值得先建立心智模型。

第一，**本地不跑全量**。[`AGENTS.md` 的 `### Run relevant checks locally`](../AGENTS.md#run-relevant-checks-locally) 明确写着：CI 拥有穷尽覆盖与平台矩阵，本地只跑与改动面匹配的证据；"为了保险起见跑全量"是被点名禁止的行为，选择策略归 [`dsh-pre-push-checks`](../.agents/skills/dsh-pre-push-checks/SKILL.md)。

第二，**命令清单不在本文**。真实清单是 [`package.json`](../package.json) 的 `scripts`，本文引用的每个命令都能在那里找到；本文刻意不复制完整清单，因为手工重述必然漂移。

规模参考（`verified_at` 当天实测，会漂移，用前请自行 `find` 复核）：`packages/` 下 854 个 `*.spec.ts` / `*.test.ts`，`packages/` + `apps/` 下 168 个 `*.e2e.ts`，`packages/`、`apps/`、`scripts/` 下 258 个 `tests/` 目录，`snapshots/` 下 137 个带 `snapshot.yml` 的场景。

## 2. 测试拓扑：四层主干加两组旁支

> **上游事实源**：[`docs/testing.md` 的 `## Tiers`](../docs/testing.md#tiers)、各 `vitest.*.config.ts`、[`package.json`](../package.json)

上游的 `## Tiers` 定义了四层主干（unit / coverage / e2e / snapshot），另外还有 owner-local expected 与 Web 浏览器快照两组。它们不是"同一批测试的不同强度"，而是**不同的 include 集合 + 不同的执行前提**：每个配置文件各自决定跑哪些文件、要不要 key、要不要构建产物。

下表把这些前提集中到一处。**耗时列是量级判断，不是实测数字**——依据是各配置里的 `testTimeout` / `fileParallelism` / `maxWorkers` 设置，以及命令本身是否前置了一次 `build`。

| 层 | 配置文件 | 命令 | 需要 `DEEPSEEK_API_KEY` | 需要构建产物 | 耗时量级 | 覆盖什么 |
|---|---|---|---|---|---|---|
| unit | [`vitest.config.ts`](../vitest.config.ts) | `pnpm run test` | 否 | 否 | 全量分钟级；单文件秒级 | `packages/*/*/tests/**/*.spec.{ts,tsx}`、`apps/*/tests/**/*.spec.ts`、`scripts/**/*.spec.ts`（`testIncludes`，第 121 行） |
| coverage 门禁 | 同上（`coverage` 段，第 201 行） | `pnpm run test:coverage` | 否 | 否 | 明显长于 `test`（v8 插桩 + 全量） | 与 unit 同一批测试，外加对 `packages/*/*/src` 的**每文件 100%** 阈值 |
| real-API e2e | [`vitest.e2e.config.ts`](../vitest.e2e.config.ts) | `pnpm run test:e2e` | **是**（无 key 各 suite 自跳过） | 部分需要（见下） | 慢：`testTimeout: 120_000`、`retry: 2`、`maxWorkers` 默认 4 | `packages/*/*/tests/**/*.e2e.ts` 与 `apps/cli/tests/**/*.e2e.ts`，排除 `*.expected.e2e.ts` |
| 录制会话快照 | [`vitest.snapshot.config.ts`](../vitest.snapshot.config.ts) | `pnpm run test:snapshot` | 否（replay 是无 key 默认）；`:record` **是** | 否（`DSH_EXAMPLE_MODE=lib` 时是） | 中等：`testTimeout: 120_000`，replay 并发上限默认 5 | `snapshots/**/*.snapshot.ts` 与 `scripts/session-snapshot-corpus.corpus.ts` |
| owner-local expected | [`vitest.expected.config.ts`](../vitest.expected.config.ts) | `pnpm run test:expected` | 否 | 部分需要（同 e2e，见下） | 中等：`testTimeout: 120_000`，`maxWorkers` ≤ 5 | 仅 `apps/cli/tests/**/*.expected.e2e.ts` |
| Web 浏览器快照 | [`vitest.web.config.ts`](../vitest.web.config.ts) | `pnpm run test:web` | 否（真实模型用例无 key 自跳过） | **是**（脚本本身是 `build && test:web:built`） | 最慢：前置全量 build，`testTimeout: 180_000`，`fileParallelism: false` | `apps/web/tests/**/*.{e2e,snapshot}.ts` 与 inspector 的浏览器 e2e |
| Web 性能诊断 | [`vitest.web.perf.config.ts`](../vitest.web.perf.config.ts) | `pnpm run test:web:perf` | 否 | **是** | 手工级：`testTimeout: 600_000` | `apps/web/tests/**/*.perf.ts` 等；**不在任何 CI lane 里**（文件头注释） |
| Web 压力 | [`vitest.web-stress.config.ts`](../vitest.web-stress.config.ts) | `pnpm run test:web:stress` | 否 | **是** | 手工级：`testTimeout: 600_000` | `apps/web/stress-tests/**/*.stress.ts`；opt-in，无默认配置包含 `*.stress.ts` |

几个容易踩的细节：

**`test:e2e` 的"需要构建产物"是有条件的。** 大部分 e2e 走源码平面，但 built-artifact smoke（真实入口路径那一类）会检查 `lib/` 是否存在，不存在就整段 `describe.skipIf` 掉——例如 `packages/code-runtime/code-runtime-worker-thread/tests/built-lib.e2e.ts:15-18` 的 `built` 常量。**这意味着不先 `pnpm run build` 就跑 `test:e2e`，你会看到绿色，但发布路径根本没被测。** 这条规则的上游是 [`docs/testing.md` 的 `## Test the real entry path`](../docs/testing.md#test-the-real-entry-path)。

**所有 lane 都解析到源码平面。** 每个 `vitest.*.config.ts` 都把 `vite-tsconfig-paths` 指向 `tsconfig.base.json`，于是裸的 workspace 包名解析到 `src`，而不是通过 package `exports` 走到 `lib/`——否则陈旧产物会加载模块单例的第二份拷贝。只有 `lib` 模式子进程和上面那些 built smoke 才显式消费构建产物（[`## Test resolution: source plane only`](../docs/testing.md#test-resolution-source-plane-only)）。

**unit lane 被拆成两个 project。** `vitest.config.ts` 的 `projects`（第 167 行）分出 `thread-safe` 与 `process-bound` 两组，后者是一份显式白名单 `processBoundTests`（第 147 行）——那些依赖进程全局状态、进程 API 或时序敏感 I/O 的 spec。两组都用 `pool: 'forks'`：Node 24 在 worker thread 里的 CJS lexer 会 abort（第 173-175 行注释）。

**平台不支持的 spec 是被 exclude 掉的，不是运行时跳过的。** `windowsUnsupportedTests`（第 39 行）与 `nonLinuxWebWorkerTests`（第 63 行）按 `process.platform` 直接从 include 里剔除。所以"我在 macOS 上跑绿了"对这些文件不构成任何证据。

**`pnpm run test:gui` 不是第八条 lane。** 它就是 `vitest run packages/client packages/host`——unit lane 加一个路径过滤，配置、阈值、执行模型完全一样。别把它当成"GUI 有自己的测试体系"，Web 的真正证据在 `test:web`。

**Python SDK 有独立的 pytest 入口，不在任何 vitest lane 里。** [`pytest.ini`](../pytest.ini) 把 `testpaths` 限死在 `python/sdk/tests`；跑法在 [`python/development.md`](../python/development.md) 的 `## Validate the SDK`。第 7 节会讲它为什么和 TypeScript 那边是**同一次 PR 的义务**。

### 2.1 环境变量旋钮

这些旋钮分散在若干 vitest 配置与三个 `scripts/` 模块里，没有任何一处上游文档把它们并排列出来——所以这张表是本文的原创汇总，每一项都注明了归属地，改动请以归属文件为准。

| 变量 | 归属 | 作用 |
|---|---|---|
| `DEEPSEEK_API_KEY` | [`vitest.e2e.config.ts:9-14`](../vitest.e2e.config.ts)、[`vitest.snapshot.config.ts:29-37`](../vitest.snapshot.config.ts)、[`vitest.web.config.ts:9-14`](../vitest.web.config.ts) | 真实 API 凭据，来自环境或根 `.env`。e2e 与 web 无条件尝试加载 `.env`；快照 lane **只有 `record` 模式**才加载 |
| `DEEPSEEK_BASE_URL` | [`AGENTS.md` 的 `## Secrets / .env`](../AGENTS.md#secrets--env) | 可选的端点覆盖 |
| `EXA_API_KEY` / `PERPLEXITY_API_KEY` | [`docs/testing.md` 的 `## Tiers`](../docs/testing.md#tiers) | provider 专属 smoke 的独立凭据；各自缺失时该 suite 自跳过 |
| `DSH_SNAPSHOT` | [`vitest.snapshot.config.ts:29`](../vitest.snapshot.config.ts)、[`vitest.web.config.ts`](../vitest.web.config.ts) 文件头注释 | `replay`（默认，只读）/ `record`（调真实 API 重录）/ `refresh`（重放已提交脚本、更新期望输出）。CI 的 Web lane 强制 `replay`，从不写期望输出 |
| `DSH_SNAPSHOT_MAX_CONCURRENCY` | [`vitest.snapshot.config.ts:19-22`](../vitest.snapshot.config.ts) | replay 的文件内并发上限，默认 `min(5, availableParallelism())`；设为 `1` 恢复完全串行 |
| `DSH_EXAMPLE_MODE` | [`vitest.snapshot.config.ts:51`](../vitest.snapshot.config.ts) | 设为 `lib` 时把 `apps/web/tests/**/*.snapshot.ts` 纳入快照 lane——**这条路径需要先 build**，源码模式才是零构建路径 |
| `DSH_E2E_MAX_WORKERS` | [`vitest.e2e.config.ts:29`](../vitest.e2e.config.ts) | e2e 文件级并发，默认 4；共享内部 key 撞并发配额时设为 `1` |
| `DSH_WEB_SNAPSHOT_WORKERS` | [`scripts/run-web-snapshots.ts`](../scripts/run-web-snapshots.ts) | CI Web 快照池的并发度，必须是大于 1 的整数，否则脚本直接抛错 |
| `DSH_COVERAGE_EXEMPT_HEAVY` | [`scripts/coverage-exempt.ts:26`](../scripts/coverage-exempt.ts) | 值只能是 `'1'` 或不设；设为其它值 [`vitest.config.ts:130-133`](../vitest.config.ts) 会直接抛错而不是静默忽略 |
| `DSH_COVERAGE_PARTITION_MODE` | [`scripts/coverage-partitions.ts:13`](../scripts/coverage-partitions.ts) | 同样只接受 `'1'`；开启后 `thresholds` 为 `undefined`、`reporter` 为空 |

最后两个旋钮值得多解释一句，因为它们是 CI 覆盖率 lane 能跑完的原因。`coverageExemptHeavySuites`（[`scripts/coverage-exempt.ts:29`](../scripts/coverage-exempt.ts)）是一份**成对**的清单：`filter` 选出这些重型套件单独跑（不插桩），`exclude` 把同一批文件从插桩那一轮里排掉。清单里是 Typert 生成器（每个用例做全工作区编译分析，是这条 lane 最长的尾巴）、webworker-runtime（整语料导入门禁，单个用例 900 秒预算并 spawn 子进程扫每一个构建产物）、几个用真实子进程跑 `scripts/` 源的 spec，以及 webworker-packer 的构建产物证明。**它们的 src 本来就在 threshold `exclude` 里**，所以不插桩不损失任何门禁强度——这是纯粹的时间优化，不是覆盖率豁免。

## 3. 覆盖率门禁的准确语义

> **上游事实源**：[`docs/testing.md` 的 `## Tiers`](../docs/testing.md#tiers)、[`vitest.config.ts`](../vitest.config.ts) 的 `coverage` 段、[`AGENTS.md` 的 `### Run relevant checks locally`](../AGENTS.md#run-relevant-checks-locally)

这是本仓库最高频的误解，值得单独一节。

**门禁是 `pnpm run test:coverage`，不是 `pnpm run test`。** 根 [`AGENTS.md`](../AGENTS.md#run-relevant-checks-locally) 里有一整行专门讲这件事："`test:coverage`, not `test`, is the CI coverage gate"。两个命令跑的是**同一批测试文件**（`test:coverage` 就是 `vitest run --coverage`），区别只在于加不加 v8 插桩与阈值判定。因此：

**用 `pnpm run test` 通过来声称覆盖率通过，是错的。** 它不是"更快的近似"，它是**完全不检查阈值**。`test` 绿而 `test:coverage` 红是常态——你新增了一个分支没人走，`test` 不会有任何意见。

阈值本身在 [`vitest.config.ts:357-365`](../vitest.config.ts)：

```ts
      thresholds: coveragePartitionMode
        ? undefined
        : {
            perFile: true,
            statements: 100,
            branches: 100,
            functions: 100,
            lines: 100,
          },
```

`perFile: true` 是关键：**每个文件各自 100%**，一个覆盖良好的大文件不能替一个裸文件兜底。测量范围是 [`vitest.config.ts:206`](../vitest.config.ts) 的 `include`：

```ts
      include: ['packages/*/*/src/**/*.{ts,tsx}'],
```

注意这个 glob 的形状：`packages/<组>/<包>/src/**`。`apps/`、`scripts/`、`vendor/`、`native/` **不在覆盖率测量范围内**——它们仍有测试，但不受这个阈值约束。

`exclude` 列表（第 209-352 行）很长，但它不是"可以随便加"的口袋。里面每一条都带理由注释，大致分四类：纯类型文件与自执行入口（`types.ts` / `bin.ts` / `worker.ts`，导入它们会在单测进程里把它们启动起来）；证据在别处的（Typert 生成器由 fixture 套件与目录字节级复现证明、webworker-runtime 由未插桩套件与打包端到端证明）；平台条件性排除（`windowsOnlyCoverageExclusions`、`pwshCoverageExclusions` 等，按宿主是否有 `pwsh`、是否 win32 动态计算）；以及带 `TODO(gui)` / `TODO(inspector)` 标记的、显式登记的 client 侧欠债。**往里加一条来绕开你自己的红线，是在跟这份注释的意图对着干。**

还有一条不在 `exclude` 列表里、但同样是逃生口的东西：源码里的 `v8 ignore` 注释。[`vitest.config.ts:355-356`](../vitest.config.ts) 的注释规定**每一条 `v8 ignore` 都必须带理由**。它和 `exclude` 的区别是粒度——`exclude` 摘掉整个文件，`v8 ignore` 只放过一行或一段，因此它更容易被顺手滥用。写之前先确认这段代码是真的不可达（那通常意味着该删），而不是你没想好怎么测。

上游还有一条判断准则值得复述：**未覆盖的行往往是门禁在提示你删掉的死代码，而不是让你补测试的缺口**。反过来，行覆盖是必要条件而非充分条件——它只证明代码跑过了，不证明功能按发布形态工作。

本地想在改动范围内验证覆盖率，用 [`dsh-pre-push-checks`](../.agents/skills/dsh-pre-push-checks/SKILL.md) 的 `### Focus unit coverage on the affected source` 一节给出的写法（该文件第 48-51 行）：

```sh
pnpm exec vitest run packages/<group>/<package>/tests/<behavior>.spec.ts \
  --coverage \
  --coverage.include='packages/<group>/<package>/src/**/*.ts'
```

**测试选择和覆盖率选择是两件事**：文件过滤决定跑哪些测试，`--coverage.include` 决定测量哪些源文件。同一份技能文档明确禁止用 `--passWithNoTests`、调低阈值或缩窄 `--coverage.include` 来掩盖一个受影响却没被覆盖的文件。

`pnpm run test:coverage:partitioned` 是 CI 用的分区变体（见 `vitest.config.ts` 第 138-142 行的 `COVERAGE_PARTITION_MODE_ENV`）：分区模式下 `thresholds` 为 `undefined`，阈值由分区合并后统一判定，所以**单跑一个分区不构成门禁证据**。

## 4. 我改了 X，该测什么

> **上游事实源**：[`docs/testing.md`](../docs/testing.md) 全篇、[`.agents/skills/dsh-pre-push-checks/SKILL.md` 的 `## Select relevant evidence`](../.agents/skills/dsh-pre-push-checks/SKILL.md)、[`AGENTS.md` 的 `## Conventions`](../AGENTS.md#conventions)

这是本文的核心。左列是你的 diff 触到的东西，右边三列是必须交出的证据。**"必须更新快照"一列写"是"的行，缺快照就是缺证据，PR 不完整**——上游的措辞是"in the same PR"。

用表之前先做三步，顺序不能颠倒。**第一步，看清改动范围**：`pnpm --silent run change-scope --base <已核实的 base ref>` 输出结构化的改动路径清单（该命令**从不猜测也不 fetch** base，你必须自己提供核实过的 ref）。**第二步，为每一条改动路径找到归属层**——用下表，落到具体的测试文件或场景名，而不是落到某个聚合命令。**第三步，只跑那些**；已经通过的检查不要因为"要提交了"再跑一遍。

| 改动类型 | 该跑哪层 | 必须更新快照？ | 必须同步 TS + Python SDK 期望输出？ |
|---|---|---|---|
| 纯函数 / 工具函数（`dsh-util` 一类零依赖模块） | 归属包的单个 `*.spec.ts` + 该文件的 `--coverage.include` | 否 | 否 |
| 服务方法（Service Definition 的公开方法语义） | 归属包 spec；若有其它 Consumer，补相邻包 spec | 仅当行为对模型/用户可见 | 否 |
| **模型可见的工具 schema 或描述** | 归属包 spec + `test:snapshot` + `verify-tool-catalog` | **是**——`tool-schemas.expected.json` 属 header 固定场景 | **是**（Python 侧 `minimal/model-visible.json` 钉住 advertised tool schemas） |
| **系统提示词**（section 内容、顺序、条件装配） | `test:snapshot` + 归属包 spec | **是**——`system-prompt.expected.md` | **是**（同上，Python 侧钉 assembled system prompts） |
| **agent-loop / 会话事件**（`SessionEventMap`、生命周期） | 单测 + `test:snapshot` + `test:e2e`；改 `agent-loop` 还要更新 `docs/architecture.md` | **是** | **是**——见第 7 节，这是两个 SDK 的硬性同 PR 义务 |
| LLM provider / adapter | 归属包单测（mock adapter）+ **`test:e2e`**（真实 API 才证明它工作） | 若模型可见输出变了则是 | 否（除非同时改了 loop 或事件） |
| Web UI（`apps/web`、`packages/client/*`） | `pnpm run test:web`（前置 build）；纯逻辑部分另有 jsdom spec | **是**——`snapshots/web/` 或 `apps/web/tests/expected/` 二选一，见第 5 节归属规则 | 否 |
| CLI 输出（headless 一次性行为、进程级装配） | `test:snapshot`（走 `dsh` 的场景）或 `test:expected`（无录制会话往返的装配期望） | 取决于归属：有 Session 往返 → `snapshots/`；没有 → `tests/expected/` | 否 |
| 文档 / Agent Note / 目录 | `pnpm run doc-sync`（全量）或 `pnpm run test:docs`（快速子集） | 否 | 否 |
| **发布路径**（包 `bin`、`exports`、worker/bin 入口、构建配置、包清单） | `pnpm run build` + `pnpm run hygiene` + 归属的 built-artifact smoke | 否 | 否 |
| **SDK 协议**（JSON-RPC 方法、参数、事件投影） | `packages/sdk/*` 单测 + `snapshots/sdk/` + Python `pytest` | **是** | **是** |
| 新增 product-visible 插件 | 非单元的 REAL-composition 测试（见第 6 节）+ 该行为对应的快照层 | 若有模型/用户可见输出则是 | 否 |
| registry 贡献（新增可注册项的注册表） | 单测中的 HMR 安全测试（见第 6 节） | 否 | 否 |
| 工具执行管线 / 权限与审批策略 | **通过 executor 测试拒绝路径**，不是通过 facade 或监听器顺序；加 `test:snapshot` 证明模型看到的结果 | 若拒绝对模型可见则是 | 否 |
| 会话持久化格式 / `SESSION_FORMAT_VERSION` | 归属包单测 + `test:snapshot`；保留历史代的场景要在 `snapshot.yml` 声明 `sessionFormat.version` 与迁移 `coverage` | **是** | **是**（`restart/` 场景钉住持久日志） |
| 出厂 profile 的 `cordis.yml` 组成 | [`apps/cli/tests/profiles/`](../apps/cli/tests/profiles) 下的 profile 级集成测试 + `test:snapshot` | **是**（组成变了，装配后的 header 就变了） | **是** |
| Client UI 文案 | `pnpm run verify-client-ui-i18n` + `test:web` | 是（UI 可见输出） | 否 |
| `vendor/` 下的 Cordis 源 | 按 [`vendor/README.md`](../vendor/README.md) 的 sync 流程，然后 `pnpm run test && pnpm run build` | 否 | 否 |
| 门禁脚本 / `vitest.*.config.ts` 本身 | 对应的 `scripts/**/*.spec.ts`；改 include/exclude 时另外确认受影响的 lane 仍然真的在跑那些文件 | 否 | 否 |
| 资源持有型 / 异步型的测试、fixture、helper 本身 | 先读 [`dsh-ci-test-reliability`](../.agents/skills/dsh-ci-test-reliability/SKILL.md) 决定需要哪类证据，再选命令 | 否 | 否 |

三条贯穿全表的判据：

**"模型可见 ⟺ 有日志"。** 根 [`AGENTS.md` 的 `## Conventions`](../AGENTS.md#conventions) 规定：任何进入模型请求的东西都必须能从会话日志重建；新增一个模型可见的输入就要新增一个 session event。所以"我只是改了个提示词字符串"从来不是小改动——它同时是快照改动。

**mock 只能架在昂贵或不确定的边界上。** [`## Prefer the real implementation over a mock`](../docs/testing.md#prefer-the-real-implementation-over-a-mock)：只 mock LLM adapter、网络、时钟，下游全部保持真实。手搓的替身只能证明桥接搬运了字节，不能证明发布的工具按断言行事——参考 [`packages/acp/acp/tests/harness.ts`](../packages/acp/acp/tests/harness.ts) 的 `makeBridgeHarness()`，它挂载真实的 loop、session store、tool registry 与 JSONL 持久化，唯一的 mock 是 `MockAdapter`。

**e2e 断言必须外部复核世界。** [`## Verify the world, not the self-report`](../docs/testing.md#verify-the-world-not-the-self-report)：重跑命令或从外部重读文件，而不是在 agent 自己的输出里搜关键词——后者让一个作弊的 agent 也能通过。顺带一条易踩的：共享 fixture 放 `tests/harness.ts` 这类普通文件，**永远不要放在另一个 `*.e2e.ts` 里**，因为 import 一个 spec 会重新注册它的 `describe`，把真实 API 调用翻倍。

### 4.1 一条串起来的证据链：新增一个工具

上表是按行查的，但真实改动往往同时命中好几行。用"给出厂 profile 加一个工具"举例——完整的开发流程归 [`docs/cookbook/adding-a-tool.md`](../docs/cookbook/adding-a-tool.md)，这里只把**证据**那一段串起来，让你看清一次改动是怎么同时触发五层的。

新增一个工具意味着：注册表多了一个贡献（→ HMR 安全测试）；advertised tool schema 变了（→ header 固定场景的 `tool-schemas.expected.json`，以及 Python 侧的 `minimal/model-visible.json`）；模型可见的请求变了（→ `test:snapshot` 的会话往返）；如果它是 product-visible 插件（→ REAL-composition 测试）；工具目录是生成的（→ `verify-tool-catalog`）；它的 UI 呈现要前置设计（→ Web 卡片从原始事件与持久化结果元数据派生，`test:web`）。

于是命令序列大致是：先写归属包 `tests/` 下的单测（含 HMR 安全用例），用聚焦的 `--coverage.include` 确认新源文件达到每文件 100%；再 `pnpm run gen-tool-catalog` 让生成目录跟上；再 `pnpm run test:snapshot:record`（transcript 变了）或 `:refresh`（输入仍有效），逐条 review diff；如果动到了 Web 呈现，`pnpm run test:web`；最后按第 7 节重跑 Python 侧的归属场景。

**没有任何一条命令能替代其中另一条。** 这正是上游那句"包级测试、e2e、纯 mock 测试和理由说明都不能替代装配后的 transcript"的实际含义——它们证明的是不同的东西：单测证明函数正确，REAL-composition 证明它在真实装配里被挂上了，快照证明模型真的看到了你以为它会看到的东西，Python 场景证明另一个 SDK 的投影没掉队。

## 5. 快照测试实操

> **上游事实源**：[`snapshots/AGENTS.md`](../snapshots/AGENTS.md)、[`docs/testing.md` 的 `## When a snapshot test is required`](../docs/testing.md#when-a-snapshot-test-is-required)、[`packages/test-support/session-snapshot/README.md`](../packages/test-support/session-snapshot/README.md)

### 5.1 顶层 `snapshots/` 的归属规则

`snapshots/` 下有四个目录，对应四种驱动接口：`session/`（headless 一次性行为）、`sdk/`（SDK 的持久化控制）、`acp/`（自动化协议行为）、`web/`（浏览器与 ARIA 证据）。三个 `*.snapshot.ts` 驱动文件分别在 `snapshots/session/`、`snapshots/sdk/`、`snapshots/acp/`；Web 的驱动在 `apps/web/tests/` 下，由 `test:web` lane 跑。

[`snapshots/AGENTS.md`](../snapshots/AGENTS.md) 开宗明义地定义了准入条件：**这棵树只放"提交的 session JSONL 既是 replay 输入、又是期望的持久化输出"的测试。** 非会话驱动的 ARIA、几何、生成器、CLI、单元期望输出**不属于这里**，它们跟随各自的 app / script / package 放在 `tests/expected/` 下，并且不使用 `*.snapshot.ts` 后缀，由 `test:expected`、`test:web` 或 `test` 承载。

这条边界是最常被搞错的地方。判据只有一个问题：**这份期望输出是不是一次录制会话往返产生的？** 是 → `snapshots/`；否 → 归属方的 `tests/expected/`。

每个场景是一个目录。以 [`snapshots/session/agent-instructions/`](../snapshots/session/agent-instructions) 为例，里面有 `snapshot.yml`（声明 profile、composition/header class、recording policy、workspace facts）、`session.v2.jsonl`（选中的父代 Session）、`system-prompt.expected.md` 与 `tool-schemas.expected.json`（只有钉住 header 的场景才拥有这两个 sidecar）、`replay.override.json`、`cordis.yml` 与 `workspace/`。

命名规则是硬约束：父角色是 `session[.vN].jsonl`，子角色是 `session.<ordinal>[.vN].jsonl`；v0 省略 `.v0`，正数版本必须小写 `.vN`，**且文件名必须与文件头一致**。同一角色可以保留多代，replay / record / refresh **一律选数值最高的那一代**。

还有一条独立的 oracle：会修改工作区的场景要在 `snapshot.yml` 里设 `workspace.final: true`，并把完整结果提交到 `workspace.expected/`。**record 与 refresh 都不会重写这棵树**——模型的自述和工具结果文本不构成外部效果的证明，这棵目录树才是。

这些命名与归属规则不是靠 review 人肉守的：`vitest.snapshot.config.ts` 的 include 里除了 `snapshots/**/*.snapshot.ts` 还有一个 [`scripts/session-snapshot-corpus.corpus.ts`](../scripts/session-snapshot-corpus.corpus.ts)，它的文件头一句话说明了自己是什么——`Repository-wide ownership and storage invariants for the recorded-session corpus`。它跨整棵树检查文件名与 header 是否一致、版本代次是否合法、workspace 期望树、以及 `snapshot.yml` 里声明的 sidecar 符号链接是否解析到声明的目标。所以**你在 `snapshots/` 下随手放一个不合规的文件，会在快照 lane 里红，而不是在某次 review 里被口头指出。**

### 5.2 什么时候必须重录，用哪个命令

上游的规则是：**每一个非平凡的、模型可见 / 协议可见 / 人可见的改动，都要在同一个 PR 里新增或更新一个无 key 的录制会话场景。** 包级测试、e2e、纯 mock 测试和"理由说明"都**不能**替代这份装配后的 transcript。

两个命令的分工不能混：

- `pnpm run test:snapshot:record`（`DSH_SNAPSHOT=record`，**需要 key**）：**模型 transcript 变了**才用。它调真实 API，重写 fixture 与期望输出。`vitest.snapshot.config.ts:29-37` 显示只有 `record` 模式会去加载根 `.env`——replay 和 refresh 都不读。
- `pnpm run test:snapshot:refresh`（`DSH_SNAPSHOT=refresh`，无 key）：**replay 输入仍然有效、只是期望输出变了**才用。它重放已提交的脚本并更新当前期望输出。

record 与 refresh 都是**串行**的（`vitest.snapshot.config.ts:64` 的 `fileParallelism` 只在 replay 模式下为真）：record 每个场景都在花真实 API 配额，refresh 的写回要从磁盘上的 fixture 收割易变值，并发写会损坏期望输出。

**两者产生的每一处 JSONL、prompt、schema、协议、UI、workspace diff 都必须逐条 review 后再提交。** 这不是客套话——`snapshots/AGENTS.md` 把它列为规则，而 record/refresh 唯一的作用就是"把当前行为写下来"，它无法判断当前行为是否正确。

只跑一个场景用 `-t <name>` 过滤（[`AGENTS.md` 的 `## Commands`](../AGENTS.md#commands) 里就是这么写的），例如 `pnpm run test:snapshot -- -t fs-read`。场景名就是 `snapshot.yml` 里的 `scenario` 字段，也就是目录名。

历史代的处理是特例：需要保留某一代历史 fixture 的 owner，要在 `snapshot.yml` 里声明确切的 `sessionFormat.version` 与封闭的迁移 `coverage` 名字；record 和 refresh 就再也不会重写那份 Session fixture。没有这个声明的 owner 跟随当前 writer 的输出。

### 5.3 "改 fixture，不要改 normalizer"

根 [`AGENTS.md` 的 `## Conventions`](../AGENTS.md#conventions) 里的原话是：`Fixtures replay on macOS/Linux; fix fixtures, not normalizers.`

前半句是事实陈述：必需的快照 lane 在 macOS 和 Linux 上运行（`snapshots/AGENTS.md` 对 sidecar 符号链接别名也是这么规定的）。所以你在这两个平台上看到的 replay 差异**是真实差异**，不是"平台噪声"，不能靠"这台机器不一样"糊弄过去。

后半句是本节最重要的规则，理解它需要先理解 normalizer 是什么。快照对比不是裸字节对比：`packages/test-support/session-snapshot/src/normalize.ts` 里的一组**纯函数** normalizer 会把易变值替换成稳定 token——session id 换成首次出现序号、生成的 cwd 换成 `{{cwd}}`、请求头的 system prompt 与 tool schema 换成 `{{system}}` / `{{tools}}`、时间归零。这样"录制时的那一次运行"和"现在这一次运行"才能结构性地对比。

关键在于：**已提交的 session 本身是归一化的不动点**。`packages/test-support/session-snapshot/src/suite.ts:1648` 把这条写成了一个断言：

```ts
        expect(redactSessionSnapshotIds(fixtures), `${scenario.name}: identity redaction fixed point`).toEqual(fixtures)
```

也就是说，对已提交的 fixture 再跑一遍归一化，必须原样返回。

于是当快照红了，你面前有两条路，其中一条是错的：

- **正确**：这次运行产生了与 fixture 不同的内容。要么这个差异是你有意引入的行为改动——那就 `refresh`（或 transcript 变了就 `record`），review diff，提交；要么这个差异是 bug——那就修代码。
- **错误**：去 `normalize.ts` 里加一条规则，把这个差异也归一化掉，让它不再出现在 diff 里。

第二条为什么错：normalizer 的每一条规则都是在**永久放弃**对某一类内容的观测能力。放宽一次，这个场景以后就再也发现不了这一类变化了——包括真正的回归。而且 normalizer 是所有场景共享的纯函数，你为一个场景放宽的规则，会同时削弱另外一百多个场景。fixture 是**这一个**场景的局部资产，改它的代价是局部的、可 review 的、有 diff 可看的；改 normalizer 的代价是全局的、无声的。

同样的道理适用于 `suite.ts:1640-1645` 那几条守卫（未清洗的 tool schema / header 内容会被直接拒绝）：它们是在 fixture 进入对比之前就否决畸形或漂移的输入。绕过它们等于让后面的比较结果失去意义。

## 6. 包级测试约束

> **上游事实源**：[`packages/AGENTS.md`](../packages/AGENTS.md)、[`docs/testing.md` 的 `## How specs execute`](../docs/testing.md#how-specs-execute)

**测试放包级 `tests/`，不放 `src/__tests__/`。** 这是 [`packages/AGENTS.md`](../packages/AGENTS.md) 的命名规则之一，也和覆盖率 `include` 的形状对应：`packages/*/*/src/**` 被测量，`tests/` 不被测量——把测试塞进 `src/` 会让它自己成为覆盖率对象。

**spec 在 forked worker 里并发执行，而且是和其它门禁进程并排跑的。** 上游 [`## How specs execute`](../docs/testing.md#how-specs-execute) 讲得很直接：被隔离的只有**进程**——端口、可预测的路径、外部命名空间、继承来的子进程**都没有被隔离**；覆盖率门禁还会拆成并发分区，自托管 runner 共享一台宿主和一个卷。

由此得出一条判据，值得贴在心里：**一个只有单独跑才过的 spec，是这个 spec 的缺陷，不是 runner 不稳定。** 每一个获取到的端口、路径、子进程都必须由它自己的 teardown 拥有（哪怕失败、重试、超时也要释放）。分配、恢复、同步、超时预算、平台与 teardown 的完整规则归 [`dsh-ci-test-reliability`](../.agents/skills/dsh-ci-test-reliability/SKILL.md)；已经在概率性失败的用例，用它的 [flake 诊断流程](../.agents/skills/dsh-ci-test-reliability/references/ci-flake-diagnosis.md) 分类。

**registry 贡献必须有 HMR 安全测试**：dispose 掉贡献者 fiber，然后观察贡献被移除。形状可以照抄 [`packages/core/tools/tests/tools.spec.ts:1984-1996`](../packages/core/tools/tests/tools.spec.ts)：

```ts
  it('rejects duplicate names and unregisters on fiber dispose (HMR safety)', async () => {
    const ctx = await setup()
    ctx.tools.register(echoTool)
    expect(() => ctx.tools.register(echoTool)).toThrow('already registered')

    const fiber = await ctx.plugin(Object.assign((inner: Context) => {
      inner.tools.register({ ...echoTool, name: 'scoped' })
    }, { inject: ['tools'] }))
    expect(ctx.tools.schemas().map(t => t.name)).toEqual(['echo', 'scoped'])

    await fiber.dispose()
    expect(ctx.tools.schemas().map(t => t.name)).toEqual(['echo'])
  })
```

注意它证明的是两件事而不是一件：注册后**确实出现**，dispose 后**确实消失**。只断言其中一半的测试通不过这条要求——这和"注册即 effect、`register()` 返回 disposer"这条约定是同一件事的两面。

**product-visible 插件需要非单元的 REAL-composition 测试。** 手搓 `ctx.plugin(...)` 的套件**不够**：要把一份 test-only 的 `cordis.yml` 通过 Loader 与 app/process 真正启动起来，只 mock 外部服务或不确定输入，然后断言模型可见的请求/日志、持久化状态，或用户可见输出。opt-in 的东西不要混进发布默认值。

与之配套的一条反直觉规则来自 [`## Test the real entry path`](../docs/testing.md#test-the-real-entry-path)：**守卫只有在回归确实能让它红的时候才算守卫。** 对于没有 `inject` 的 bundle/composition 插件，一个 Loader smoke 在 default export 顶掉必需的具名导出时**仍然是绿的**——所以要显式加 `expect('default' in mod).toBe(false)` 加上 `unwrapExports` 往返断言，并且**实际证明它**：先引入回归、看它红、再改回来。这条规则的来历是 [postmortem 0001](../docs/postmortem/0001-acp-default-export-drops-inject.md)。

profile 级的跨包集成测试放 [`apps/cli/tests/profiles/`](../apps/cli/tests/profiles)（该目录有自己的 `AGENTS.md`）；包特定的 Loader composition 放该包的 `tests/fixtures/`。

## 7. 两个 SDK 都投影这个循环

> **上游事实源**：[`AGENTS.md` 的 `## Conventions`](../AGENTS.md#conventions)、[`docs/testing.md` 的 `## When a snapshot test is required`](../docs/testing.md#when-a-snapshot-test-is-required)、[`python/development.md`](../python/development.md)

这一节单独列出来，是因为它是最容易在本地"全绿"却在 CI 挂掉的一类改动。

规则本身很短：**agent-loop、session 生命周期、`SessionEventMap` 的改动，必须在同一个 PR 里更新 TypeScript 与 Python 两个 SDK 的期望输出。** 而根 `AGENTS.md` 在同一行补了一句关键限定——**`pnpm run test` 两个都不覆盖**。

两个 SDK 的期望输出各自归属不同的树：

- **TypeScript** 归 `snapshots/sdk/`，由 `pnpm run test:snapshot` 承载（record/refresh 见第 5 节）。
- **Python** 归 [`scripts/snapshots/python-sdk-single-exe/`](../scripts/snapshots/python-sdk-single-exe)，由必需的 `python-runtime` CI 作业拥有，驱动脚本是 [`scripts/smoke-python-runtime.py`](../scripts/smoke-python-runtime.py)（不是 pytest）。

Python 侧的三个场景值得知道它们各自钉住什么，因为这决定了你的改动会不会撞上它们（细节归属地是 [`python/development.md`](../python/development.md)）：`minimal/model-visible.json` 钉住 `sdk-minimal` profile 装配后的**系统提示词、对外暴露的工具 schema、以及模型可见的消息**（`minimal/win-x64/` 是 PowerShell 对应版本）；`advanced/` 钉住一次复杂进程的 SDK 结果与父/子会话日志；`restart/` 启动两个完整的 SDK runtime 进程打到同一个持久化根，快照它们各自隔离的模型历史与独立的持久日志。

**推论：一个"只多贡献了一个系统提示词 section"或"多发了一条用户消息"的插件，会让 Python 作业变红**——因为该 profile 发出的每一条消息都被逐条比较。更新方式是用 `--update-snapshots` 重跑归属场景，然后 review 那份 diff 再提交。

另外还有一条与 Python 相关但不同的 lane：`python/sdk/tests/` 下的 pytest 套件（由 [`pytest.ini`](../pytest.ini) 限定 `testpaths`）驱动的是**伪 runtime peer**，不是打包后的 runtime。它由 CI 的 `python 3.10 / keyless SDK` 作业跑，本地跑法见 [`python/development.md`](../python/development.md) 的 `## Validate the SDK`。两者不能互相替代：pytest 证明客户端协议处理，`smoke-python-runtime.py` 证明发布形态的端到端投影。

写规划时的做法：[`AGENTS.md`](../AGENTS.md#conventions) 要求为能力接缝、生命周期路径和 transcript 输出**在计划阶段就点名每一个必需的层**，并把缺失的快照 harness 支持放进同一个改动。事后补比事前列贵得多。

## 8. 平台矩阵与 Windows

> **上游事实源**：[`docs/development.md` 的 `### CI gates`](../docs/development.md#ci-gates)、[`.github/workflows/ci.yml`](../.github/workflows/ci.yml)、[`AGENTS.md` 的 `## Commands`](../AGENTS.md#commands)

**平台矩阵是 CI 的资产，不是你的。** [`AGENTS.md`](../AGENTS.md#run-relevant-checks-locally) 写得很清楚：CI 拥有穷尽覆盖与平台矩阵；本地只在明确要求、诊断 CI 失败、或改动确实是仓库级不可分割时才做全量排练。

Windows 在 CI 里被拆成多个作业（见 [`ci.yml`](../.github/workflows/ci.yml)）：一个在标准 Linux runner 上通过 Wine 跑真实 Windows Node 的**阻塞**作业，加上真实 Windows 上的 build、coverage、native tests 与 observational 作业。Wine 那条 lane 的门禁逻辑由 [`scripts/wine-windows-gates.sh`](../scripts/wine-windows-gates.sh) 拥有。

本地对应的命令是 `pnpm run check:windows-wine`，[`AGENTS.md` 的 `## Commands`](../AGENTS.md#commands) 给它加了一条罕见的全大写限定：`ONLY when diagnosing a known Windows failure`（需要 wine；CI 拥有这个信号）。换句话说，**它不是"推送前保险起见跑一下"的检查**，是诊断工具。

理解 Windows 差异的入口在 [`vitest.config.ts`](../vitest.config.ts)：`windowsUnsupportedPackages`（第 22 行）列出在 win32 上被整包排除的测试（缺真正的 POSIX shell），注释里特别说明 pwsh 相关套件**故意保留**——PowerShell 随 Windows 发行，它们在那里原生运行；而 `windowsOnlyCoverageExclusions`（第 80 行）反过来把只在 win32 执行的源文件从非 Windows 的覆盖率里摘掉。`pwshCoverageExclusions`（第 114 行）则是运行时探测：真的 spawn 一次 `pwsh` 看它是否可用，可用就照常要求每文件 100%，不可用就豁免——所以**同一份配置在有无 pwsh 的机器上要求不同**，这解释了"我这里绿的覆盖率在 CI 红了"。

跨平台不确定性的处理原则来自 [`dsh-pre-push-checks` 的 `## Handle failures`](../.agents/skills/dsh-pre-push-checks/SKILL.md)：如果一个失败看起来是环境特有的，**要证明它**——记录确切命令、失败用例、平台差异点，确认非平台证据仍然通过；能修跨平台不确定性就优先修，而不是绕过。

### 8.1 "本地绿、CI 红"的已知原因

本文各节反复出现同一类现象，这里集中成一张排查表。**看到 CI 红而本地绿时先过一遍这张表，再去怀疑 runner。**

| 现象 | 原因 | 在哪一节 |
|---|---|---|
| `test` 绿、覆盖率作业红 | `test` 根本不判定阈值，两者只差 `--coverage` | 第 3 节 |
| 覆盖率本地绿、CI 红，涉及 `pwsh-local` / `pwsh-sandbox` | 豁免是**运行时探测**出来的：你的机器没有 `pwsh` 所以被豁免，CI runner 有 | 第 8 节 |
| 覆盖率本地绿、CI 红，涉及 `sandbox-windows-acl` 等 | 这些源只在 win32 执行，非 Windows lane 把它们排除，Windows lane 不排除 | 第 8 节 |
| 只跑了一个覆盖率分区就当作证据 | 分区模式下 `thresholds` 是 `undefined`，阈值在合并后才判定 | 第 3 节 |
| built-artifact smoke "通过"了但其实没跑 | `lib/` 不存在时整段 `describe.skipIf` 掉；没先 `pnpm run build` 就跑 `test:e2e` 会看到假绿 | 第 2 节 |
| 本地 macOS 全绿，Windows 作业红 | 平台不支持的 spec 是从 include 里 **exclude** 掉的，不是运行时跳过——本地那份绿对它们零证据 | 第 2 节 |
| Python 作业红，但你没碰 Python | 插件多贡献了一个 system section 或一条用户消息就会撞上 `minimal/model-visible.json`；`pnpm run test` 不覆盖它 | 第 7 节 |
| 快照本地重跑就绿、CI 红 | 期望输出被本地某次 record/refresh 改写但没 review、没提交，或选中的是不同代的 fixture（一律取最高代） | 第 5 节 |
| spec 单跑绿、并发跑红 | 端口/路径/子进程没有被 teardown 拥有；按上游规定这是 spec 的缺陷 | 第 6 节 |

## 9. 延伸阅读

- [`docs/testing.md`](../docs/testing.md) / [中文版](../docs/testing.zh.md) — 测试策略的权威归属地，本文的全部上游。
- [`snapshots/AGENTS.md`](../snapshots/AGENTS.md) — 快照树的准入与归属规则。
- [`packages/AGENTS.md`](../packages/AGENTS.md) — 包级测试约束、REAL-composition 与 HMR 安全要求的规则原文。
- [`packages/test-support/session-snapshot/README.md`](../packages/test-support/session-snapshot/README.md) — 快照 harness 本身的实现契约：manifest、身份脱敏、归一化、workspace 比较、协议适配器。
- [`.agents/skills/dsh-pre-push-checks/SKILL.md`](../.agents/skills/dsh-pre-push-checks/SKILL.md) — 推送前的检查选择流程。
- [`.agents/skills/dsh-ci-test-reliability/SKILL.md`](../.agents/skills/dsh-ci-test-reliability/SKILL.md) — 资源分配、恢复、同步、超时预算与 teardown 规则。
- [`docs/development.md`](../docs/development.md) — 贡献者环境、日常命令与 CI 概览。
- [`python/development.md`](../python/development.md) — Python SDK 的验证与打包 runtime 冒烟流程。
- [`quality_gates.md`](quality_gates.md) — 从门禁报错反查该改哪里；本文的测试列在那里有对应的门禁视角。
- [`monorepo_and_build.md`](monorepo_and_build.md) — 源码平面与产物平面的分界，解释为什么 built smoke 需要先 build。
- [`session_and_events.md`](session_and_events.md) — `SessionEventMap` 与会话日志格式，第 7 节改动类型的背景。
- [`agent_loop_and_tools.md`](agent_loop_and_tools.md) — agent-loop 与工具执行管线，第 4 节多行的改动对象。
- [`prompt_management.md`](prompt_management.md) — 系统提示词的装配路径，理解为什么它是快照改动。
- [`plugin_development_guide.md`](plugin_development_guide.md) — 插件开发流程，REAL-composition 测试在那里有完整上下文。
- [`capability_seams.md`](capability_seams.md) — 能力接缝的三角色结构，决定"改了服务方法"该测哪些 Consumer。
- [`architecture_overview.md`](architecture_overview.md) — 整体架构地图。
- [`AI_Coding_Context.md`](AI_Coding_Context.md) — 在本仓库写代码的总体上下文入口。
