---
title: Token 成本与上下文管理
summary: 把 deepseek-harness 里控制上下文成本的四种手段——压缩、溢出落盘、工具结果裁剪、有界读取——并排比较，说明各自的触发条件、代价与 KV cache 影响，并指回 docs/ 与包 README 的权威归属地
keywords: token | compaction | spill | kv-cache | context-budget | token-meter
scope: deepseek-harness 上下文成本控制手段的横向对比
related_files: packages/compaction | packages/spill | packages/llm/token-meter | docs/subsystems/compaction.md
dependencies: docs/subsystems/compaction.md | docs/subsystems/spill.md | docs/subsystems/token-meter.md
verified_at: 2026-09-06
---

# Token 成本与上下文管理

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。压缩、溢出、计量各自的机制归 [`docs/subsystems/`](../docs/subsystems/README.md) 对应页与包 README，配置项归生成产物 [`docs/config-catalog.md`](../docs/config-catalog.md)。本文不复述它们，只做一件上游没做的事：**把控制上下文成本的几种手段并排比较，说明各自的代价。**
>
> **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即修正本文。

## 本文定位与上游事实源

> **上游事实源**：[`docs/subsystems/compaction.md`](../docs/subsystems/compaction.md)（中文版：[`compaction.zh.md`](../docs/subsystems/compaction.zh.md)）、[`docs/subsystems/spill.md`](../docs/subsystems/spill.md)（中文版：[`spill.zh.md`](../docs/subsystems/spill.zh.md)）、[`docs/subsystems/token-meter.md`](../docs/subsystems/token-meter.md)（中文版：[`token-meter.zh.md`](../docs/subsystems/token-meter.zh.md)）

四种手段分属**三个包组**、挂在四条不同的路径上：压缩在 [`packages/compaction/`](../packages/compaction/README.md)，溢出落盘在 [`packages/spill/`](../packages/spill/README.md)，工具结果裁剪与压缩同住 compaction 组、但走的是另一条路径（无模型调用），有界读取则是一个零依赖工具库 [`packages/util/output-retention/`](../packages/util/output-retention/README.md) 加上各个工具包自己的分页渲染。上游按 one-home-per-fact 原则把它们各自钉死在自己的页面上，因此**没有任何一页回答"我该用哪个"**——那正是本文存在的理由。

读法：先看第 2 节确认预算被谁吃掉，再用第 3 节的对比表选手段，然后顺着表里的"权威文档"列进上游看具体语义。第 5 节解释为什么每个包 README 都必须写一段 KV-cache 影响；第 6 节是提交前的自查清单。

**本文不复述的内容**：`CompactionResult` / `SpillRef` / `TokenMeasurement` 的字段定义（在对应 subsystem 页的类型块里）、每个插件的完整配置字段表（在生成产物 [`docs/config-catalog.md`](../docs/config-catalog.md) 里）、以及各包的实现内幕（在包 README 的 `Understand the implementation` 折叠区里）。

## 上下文预算从哪里被消耗

> **上游事实源**：[`docs/subsystems/system-prompt.md`](../docs/subsystems/system-prompt.md)、[`packages/core/system-prompt/README.md`](../packages/core/system-prompt/README.md)、[`docs/subsystems/token-meter.md#tokenmeasurement`](../docs/subsystems/token-meter.md#tokenmeasurement)

一次请求的 token 压力可以拆成两块：**信封**（envelope，即系统提示词加工具 schema）和**会话表面**（surface，即当前可见的消息序列）。`ctx.tokenMeter` 的 `TokenMeasurement` 就是按这个划分给的：`totalTokens` 是请求加响应的整体压力，`surfaceTokens` 只是表面部分，等于各 `nodes[].tokens` 之和。信封部分不出现在 `nodes` 里，所以**压缩永远动不了它**——这一点是选择手段时的第一个分岔。

| 消耗源 | 每轮是否重复计费 | 归属 | 可用手段 |
|---|---|---|---|
| 系统提示词（identity / persona / 各插件 section） | 是，每次请求全量重发 | [`packages/core/system-prompt/`](../packages/core/system-prompt/README.md) | 只能减少 section 本身；压缩/裁剪都无效 |
| 工具 schema | 是，每次请求按当前可见工具集全量重发 | [`docs/tool-catalog.md`](../docs/tool-catalog.md) | 按 agent 限制工具可见性；压缩无效 |
| 会话历史（消息、工具调用与结果） | 是，直到被压缩替换 | [`docs/subsystems/session.md`](../docs/subsystems/session.md) | 压缩、工具结果裁剪 |
| 工具结果（历史里最容易失控的一项） | 是，直到被裁剪或压缩替换 | 各工具包 | 有界读取（进入历史前）、溢出落盘（进入历史时）、裁剪（进入历史后） |

`packages/core/system-prompt/README.md` 对系统提示词与 schema 的表述是"每次请求重复计费"（`Identity is a fixed per-request cost when enabled`、`Schema tokens repeat on every request`），因此**一个多余的 prompt section 或一个没人用的工具，它的成本是按轮次线性累加的**，比一条长历史消息更值得砍。这也解释了为什么 `docs/cookbook/adding-a-package.md` 要求每个包 README 都申报自己的 token 影响：新增一个插件往往就是给每一轮请求加一笔固定开销。

**关键推论**：手段的选择点取决于内容处在生命周期的哪一段——还没产生（有界读取）、正要进历史（溢出落盘）、已经在历史里（裁剪）、还是整段历史都太长（压缩）。四种手段不是替代关系，是流水线上的四道闸。

## 四种手段横向对比

> **上游事实源**：[`docs/subsystems/compaction.md#the-service`](../docs/subsystems/compaction.md#the-service)、[`docs/subsystems/spill.md#the-service`](../docs/subsystems/spill.md#the-service)、[`packages/compaction/compaction-tool-result-pruner/README.md`](../packages/compaction/compaction-tool-result-pruner/README.md)、[`packages/util/output-retention/README.md`](../packages/util/output-retention/README.md)

### 总表

| | 压缩（compaction） | 溢出落盘（spill） | 工具结果裁剪（prune） | 有界读取（bounded read） |
|---|---|---|---|---|
| **何时触发** | `agent/pre-step` 上压力越过阈值，或 provider 确认 `CONTEXT_WINDOW_EXCEEDED`，或人工 `/compact` | 工具执行后 `tools/post-execute`，纯文本最终结果超过 `maxInlineBytes` | 压缩触发**已经成立之后**、选择压缩范围**之前** | 工具产出结果的当下，由工具自己施加 |
| **作用对象** | 一段平衡的旧历史区间 | 单次工具调用的最终结果 | 当前表面上所有超预算的工具结果节点 | 单次工具调用尚未成形的输出 |
| **丢什么** | 原始表述——旧区间被一条摘要消息**替换**，细节只在会话日志里 | 全文的正文——模型看到头尾预览加定位符，全文落盘可再读 | 结果的中段——保留头尾，中间换成裁剪标记 | 超出上限的条目/字节——附带精确的省略计数 |
| **可恢复性** | 不可（模型侧）；日志保留被遮蔽事件 | 可，模型用 `read`/`grep` 按 `retrievalHint` 取回 | 不可（模型侧）；原始事件仍在日志里 | 可，改分页参数或收窄条件重新调用 |
| **是否需要模型调用** | 需要（一次独立的摘要请求） | 不需要 | 不需要 | 不需要 |
| **KV cache** | **失效**：从第一个被替换的历史 token 起 | **不失效**：append-only | **失效**：从第一个被改动的 token 起 | **不失效**：append-only |
| **由什么配置控制** | `dsh-compaction-basic` 的 `thresholdRatio` / `retainRatio` / `retainTokens` / `maxTokens` / `compactionRetries` / `maxOverflowRetries` / `modelPolicies` / `auto` | `dsh-spill-policy` 的 `maxInlineBytes`（省略即整体停用）+ `dsh-spill-local` 的 `root` / `cleanupPeriodDays` | `dsh-compaction-tool-result-pruner` 的 `thresholdChars` / `headChars` / `tailChars` | 各工具自己的配置，如 `dsh-tool-fs` 的 `readLimit` |
| **权威文档** | [compaction subsystem](../docs/subsystems/compaction.md)、[compaction-basic README](../packages/compaction/compaction-basic/README.md) | [spill subsystem](../docs/subsystems/spill.md)、[spill-policy README](../packages/spill/spill-policy/README.md) | [pruner README](../packages/compaction/compaction-tool-result-pruner/README.md)、[compaction subsystem § Tool-result pruning outcomes](../docs/subsystems/compaction.md#tool-result-pruning-outcomes) | [output-retention README](../packages/util/output-retention/README.md)、[tool-fs README](../packages/fs/tool-fs/README.md) |

**表里最重要的一行是 KV cache 那一行**：四种手段里只有两种是免费的。溢出落盘和有界读取只改变"新写进历史的内容有多大"，不动已有前缀；压缩和裁剪都是**替换式**的，会让 provider 侧已缓存的前缀从改动点起全部作废。第 5 节展开这条代价。

### 压缩：唯一能整体缩小历史的手段，代价最贵

> **上游事实源**：[`docs/subsystems/compaction.md#ctxcompaction--compactionengine-abstract-seam`](../docs/subsystems/compaction.md#ctxcompaction--compactionengine-abstract-seam)、[`packages/compaction/compaction-basic/README.md`](../packages/compaction/compaction-basic/README.md)

接缝按 [capability seam](capability_seams.md) 三角拆分：Service Definition 是 [`dsh-compaction`](../packages/compaction/compaction/README.md)（`ctx.compaction`），Service Provider 是 [`dsh-compaction-basic`](../packages/compaction/compaction-basic/README.md)，人类 Consumer 是 [`dsh-command-compact`](../packages/compaction/command-compact/README.md)（`/compact`）。三个入口分别是 `compactIfNeeded()`（自动，带 `'pressure' | 'context-overflow'` 触发原因）、`compactNow()`（低于阈值也做一次有用的空闲缩减）、`compactRegion()`（显式区间）。

阈值和保留量都是按**路由模型的上下文容量**换算出来的绝对 token 数，而不是固定常量：

```ts
  const thresholdTokens = Math.floor(contextWindow * policy.thresholdRatio)
  const retainTokens = policy.retainTokens === undefined
    ? Math.floor(contextWindow * policy.retainRatio)
    : policy.retainTokens
```

`packages/compaction/compaction-basic/src/config.ts:144-147`（`resolveCompactSpec`）

默认 `thresholdRatio` 为 `0.8`、`retainRatio` 为 `0.16`；`modelPolicies` 可以按 `{ provider, model }` 精确覆盖，让一个 backend 同时服务不同上下文尺寸的路由。完整字段表见 [compaction-basic README 的 Tuning 小节](../packages/compaction/compaction-basic/README.md#use-this-package)，穷尽来源是 [`docs/config-catalog.md#deepseek-aidsh-compaction-basic`](../docs/config-catalog.md#deepseek-aidsh-compaction-basic)。

压缩的三笔代价要分开算。**第一笔是摘要请求本身**：它是一次独立的 `ctx.llm.stream()` 调用，输入是被遮蔽区间的完整重放，输出受 `maxTokens`（默认 `8192`）封顶，而 `compactionRetries` 允许在压力仍未回落时再来一次——也就是说这笔钱可能付不止一次。**第二笔是信息损失**：旧区间被一条 `<compacted-summary>` 用户消息替换，模型再也看不到原文。**第三笔是缓存失效**，见第 5 节。

区间选择保证工具调用与结果成对（Service Definition 导出 `toolPairingBalancedBefore()` / `toolPairingBalancedAfter()`，缓存行为见[包契约](../packages/compaction/compaction/README.md#tool-pairing-boundaries)），但**不保证按整轮对齐**——一个超大回合的早期已结束步骤可以被单独压掉。

失败时的行为值得记住：`ManualCompactionErrorCode` 里的 `changed` 和 `summary` 会保持会话表面不变，但仍然把这次失败的尝试落进日志；摘要失败时自动路径只记一条 warning 并**带着超预算的完整历史继续跑**，不会把请求打挂。

### 溢出落盘：把大结果换成一个可再取的定位符

> **上游事实源**：[`docs/subsystems/spill.md#the-save-request`](../docs/subsystems/spill.md#the-save-request)、[`packages/spill/spill-policy/README.md`](../packages/spill/spill-policy/README.md)

同样是三角接缝：Service Definition [`dsh-spill`](../packages/spill/spill/README.md)（`ctx.spillStore`，唯一操作 `saveText`）、Service Provider [`dsh-spill-local`](../packages/spill/spill-local/README.md)（会话作用域的私有文件）、Consumer [`dsh-spill-policy`](../packages/spill/spill-policy/README.md)（挂在 `tools/post-execute` 上的策略）。判定极其朴素——把内容拍平成纯文本，量它的 UTF-8 字节数：

```ts
    const content = decision.content ?? result.content
    const text = flattenPlainText(content)
    if (text === undefined) return decision
    const totalBytes = Buffer.byteLength(text, 'utf8')
    if (totalBytes <= maxInlineBytes) return decision
```

`packages/spill/spill-policy/src/index.ts:199-203`（`tools/post-execute` 监听器）

超限时模型看到的是**头尾预览 + 一条注记**，注记里带 `locator` 与 `retrievalHint`，整体不超过 `maxInlineBytes`（注记的字节开销先从预算里扣掉）。预览用的是 `dsh-output-retention` 的 `TextRetainer`——这就是"有界读取"和"溢出落盘"在实现上的接合点。

三个必须知道的边界：`maxInlineBytes` **默认省略，省略等于整个策略不生效**，所以它是个显式的部署决定，不是自动生效的保护；策略是 best-effort 的，落盘失败时保留原始内联结果，绝不把一次成功的调用变成 `isError`；`read` 被显式跳过，避免 read → spill → 再 read 的循环（源码注释：``Skip `read` to avoid a read → spill → read again loop.``，`packages/spill/spill-policy/src/index.ts:195`）。

代价一栏写"可恢复"是有条件的：模型必须真的去 `read`/`grep` 那个定位符，而那次取回本身又要花 token。溢出把**一次性的大成本**换成了**可选的、按需的小成本**——如果模型根本不需要正文，这笔钱就省下了；如果它每次都要全文，溢出反而多花了一趟往返。

### 工具结果裁剪：免模型调用的第一道闸

> **上游事实源**：[`docs/subsystems/compaction.md#ctxtoolresultpruner--toolresultpruner`](../docs/subsystems/compaction.md#ctxtoolresultpruner--toolresultpruner)、[`packages/compaction/compaction-tool-result-pruner/README.md`](../packages/compaction/compaction-tool-result-pruner/README.md)

`ctx.toolResultPruner` 对当前表面上的工具结果节点做确定性的头/中/尾裁剪：超过 `thresholdChars`（默认 `8192` 个 Unicode 码点）的文本被替换为前 `headChars`（默认 `4096`）+ 裁剪标记 + 后 `tailChars`（默认 `1024`）。计量单位是**码点而非 token**——pruner README 自己把这条列在了 Known Limitations 里（`Character budgets are not token budgets`），判断裁剪是否真的缓解了压力仍然要回到 `ctx.tokenMeter`。

它和压缩的关系是**顺序关系而非并列关系**：裁剪只在压缩触发已经成立之后才跑，跑完立刻重新计量，如果压力回落到阈值以下，摘要调用就被完全跳过。

```ts
    const spec = resolveCompactSpec(policy, context.contextWindow)
    if (measurement.totalTokens < spec.thresholdTokens) return null

    // Once pressure qualifies, land the model-free pass before choosing a
    // summary range, then remeasure through the singleton replay fold.
    if (prune !== undefined) {
      prune.pruneSession(agent.session)
      measurement = meter.measure(agent.session)
    }
    if (measurement.totalTokens < spec.thresholdTokens) return null
```

`packages/compaction/compaction-basic/src/index.ts:304-313`（自动压力路径）

**这是本节最有价值的一条**：挂上 pruner 的收益不是"少一点历史"，而是**有机会完全避免那次摘要模型调用**——省下的是一次 LLM 往返，不只是几千 token。代价是裁剪纯粹是语法性的（保头保尾，不理解中段哪几行重要），且每次替换都会打断 KV cache。

裁剪的替换事件带一个 `compaction/prune` 影子定价事件，用注入的 token meter 给被遮蔽节点定价，让纯消费者不必维护逐节点状态就能做减法——这是计量与裁剪之间的契约，细节在 [compaction subsystem 的 `ctx.toolResultPruner` 小节](../docs/subsystems/compaction.md#ctxtoolresultpruner--toolresultpruner)。

### 有界读取：在内容进入历史之前就封顶

> **上游事实源**：[`packages/util/output-retention/README.md#use-this-package`](../packages/util/output-retention/README.md#use-this-package)、[`docs/subsystems/session-query.md#bounded-event-reads`](../docs/subsystems/session-query.md#bounded-event-reads)

`dsh-output-retention` 提供两个 retainer：`ItemRetainer` 按条目数封顶有序逻辑单元（路径、匹配、来源），`TextRetainer` 按字节封顶文本流，支持 `head` / `tail` / `headTail` 三种窗口并在每个切点保持 UTF-8 边界有效。库只回答"留了什么、省了什么"这个机械问题；分组、行号、落盘文件、provider 错误状态都留在工具里。README 里那张表列了当前消费者：`glob` / `grep` / `web_search` 用 `ItemRetainer`，`bash` / `web_fetch` 用 `TextRetainer`。

`read` **不在这张表里**，因为它的 `offset` / `limit` 行窗分页是文件特有的渲染器，单个省略计数表达不了窗口的两侧——它自己封顶（`readLimit` 默认 `2000` 行，另有 `readMaxLineLength` 与 `readMaxBytes`）。同样属于"有界"家族的还有 `sessionQuery` 的[有界事件读取](../docs/subsystems/session-query.md#bounded-event-reads)：一个目标 seq 加可选的前后邻居数，返回一个有界的原始日志窗口，而不是整段日志。

一条容易搞错的语义：`truncated` 是**预算事实**，只表示"因为上限而省略了本来可得的内容"，它**从不**表示上游不完整——权限失败、跳过的二进制文件、provider 部分失败都留在工具自己的字段里。把这两件事混在一起，模型会误判它是否应该重试。

有界读取是四种手段里唯一**零代价**的：不失效缓存、不调模型、不丢已进入历史的信息，因为它根本没让那些内容进历史。代价被前移成了设计负担——工具作者必须在写工具的时候就决定上限和恢复指引。

## token 计量：`ctx.tokenMeter` 记什么

> **上游事实源**：[`docs/subsystems/token-meter.md`](../docs/subsystems/token-meter.md)、[`packages/llm/token-meter/README.md`](../packages/llm/token-meter/README.md)

上面四种手段里有三种要靠同一个服务来判断"现在到底多大"。`ctx.tokenMeter.measure(session, requestHeader?)` 返回一个**脱离引用的、深度不可变的快照**：`logRevision`（消费掉的持久事件数）、`baseline`（provider usage 锚点还是启发式估计）、`surfaceDeltaTokens`（相对锚点的带符号重定价）、`totalTokens`、`surfaceTokens`，以及按位置排序的 `nodes`。字段语义在 [subsystem 页的 `TokenMeasurement` 块](../docs/subsystems/token-meter.md#tokenmeasurement)，本文不复述。

**"回放感知"（replay-aware）的含义**：测量不是维护一个可变计数器，而是从持久会话日志推进一个每会话隔离的 fold。三个后果——测量是确定性的（同样的日志给同样的数）、不花模型调用、且**只反映日志里真实存在的东西**。这条直接锚在仓库的 `Model-visible ⟺ logged` 约定上：任何进入模型请求的东西都必须能从会话日志重建，所以计量只读日志就够了。反过来，一个绕过会话事件偷偷塞进请求的内容，计量看不见它，压力判断就会偏低。

`nodes[]` 上每个节点带两个价格，这个双价格设计是理解计量的关键。`tokens` 是**按路由定价**的请求压力：当路由适配器声明了图像定价时，图像按视觉 token 加模型可见文本计价；否则回落固定启发式。`heuristicTokens` 是**路由无关**的固定启发式价格，替换类操作的影子定价用它，好让 O(1) 的投影 fold 跟自己的追加保持一致。压力判断、保留尾巴、区间选择读的都是前者。

那个"固定启发式"就是字面意义上的固定：

```ts
/** Fixed text-density estimate used until exact tokenization is needed. */
const CHARS_PER_TOKEN = 4

/** Per-block structural overhead for JSON framing and type tags. */
const BLOCK_OVERHEAD = 4

/** Role-field framing overhead added to every priced message. */
export const ROLE_OVERHEAD = 4
```

`packages/llm/token-meter/src/estimate.ts:12-19`

token-meter README 对这条规则的误差有明确警告：四字符一 token 会**严重低估 CJK 文本和 JSON schema**。所以 UI 上的 `contextBreakdown`（`systemTokens` / `toolsTokens` / `messageTokens`）被 README 明确定性为"近似构成，永远不要当作总量呈现"，它们不会加总等于 `projectedTokens`。做成本判断时，把它当量级参考，不要当账单。

provider 上报的 usage 只在**规范请求信封完全匹配**且其总数不低于该次调用的完整路由定价锚点时才被复用；prompt、前缀、工具、provider、model 或调用配置一变，就故意回落到完整的启发式估计。这解释了一个常见困惑：切换模型之后压力数字会跳变——不是历史变了，是锚点没了。

## KV cache 友好性：为什么这是一条制度

> **上游事实源**：[`docs/cookbook/adding-a-package.md#4-write-the-package-readme`](../docs/cookbook/adding-a-package.md#4-write-the-package-readme)、[`docs/subsystems/system-prompt.md#dynamic-prompt-context`](../docs/subsystems/system-prompt.md#dynamic-prompt-context)

仓库要求**每个包 README 都以固定格式收尾**：`## Model Experience`，下面按 H3 分条，每条依次是 `What the model sees` / `Token effect` / `KV Cache effect` 三个 H4，然后是 `## Known Limitations and Deferred Work`。这不是文风建议——`scripts/verify-package-readme-model-experience.ts` 是一个被执行的门禁，缺章节或用了未审计的省略句式会直接失败；连"我没有模型影响"也必须写成审计过的 `None, as ` / `Indirectly, through ` 句式加一段非空的 `KV Cache effect`。

为什么单挑 KV cache 出来立规矩：**它是唯一一个"局部改动、全局代价"的属性**。一个包新增两行 prompt，token 成本是可加的、可预测的、局部的；但如果这两行落在请求前缀里且每轮都变，它作废的是**它后面的全部缓存**——代价随对话长度增长，而且账单落在别的包头上。cookbook 因此要求 `KV Cache effect` 一栏必须区分四种行为并**点名本包哪些改动会让复用失效**：append-only 增长、稳定的重复前缀、替换早先的请求 token、以及独立的模型请求。

顺着这条规则读仓库里已有的 README，会看到一套一致的词汇：

| 表述 | 含义 | 例子 |
|---|---|---|
| `Append-only; newly visible content follows the reusable request prefix` | 新内容追加在可复用前缀之后，不作废任何已有条目 | [`dsh-tool-fs`](../packages/fs/tool-fs/README.md#model-experience)、[`dsh-spill-policy`](../packages/spill/spill-policy/README.md#model-experience)、[`dsh-time-context`](../packages/context/time-context/README.md) |
| `Prefix-stable while …` | 只要列出的那些输入渲染结果不变就可复用，变了则从第一个变动的 token 起失效 | [`dsh-system-prompt`](../packages/core/system-prompt/README.md)（identity、persona、变量、section 文本与**顺序**） |
| `Each checkpoint invalidates reuse from the first replaced history token` | 替换式，从第一个**被替换**的历史 token 起失效 | [`dsh-compaction-basic`](../packages/compaction/compaction-basic/README.md#model-experience) |
| `Replacing an earlier result invalidates reuse from the first changed token` | 替换式，从第一个**被改动**的 token 起失效（措辞与上一行不同，别混用） | [`dsh-compaction-tool-result-pruner`](../packages/compaction/compaction-tool-result-pruner/README.md#model-experience) |
| `No direct invalidation; the named consumer owns any request-prefix changes` | 本包不碰请求前缀，影响由**具名消费者**负责 | [`dsh-token-meter`](../packages/llm/token-meter/README.md#model-experience) |
| `No direct invalidation; the retention consumers own any request-prefix changes` | 同上，但负责方是 **retention 消费者** | [`dsh-output-retention`](../packages/util/output-retention/README.md#model-experience) |

**什么样的改动会破坏缓存前缀**，按破坏力从大到小：

- **在系统提示词里放每轮都变的内容**。这是最贵的错误：前缀在最前面，一变整条历史的缓存全废。仓库为此专门分了两种贡献——`PromptSection` 是静态段落，`PromptContext` 是"`PromptSection` 的 cache-safe 对应物"（[subsystem 原话](../docs/subsystems/system-prompt.md#dynamic-prompt-context)），动态上下文被物化成排在保留历史**之后**的 user 角色快照，而且只在内容变化或被压缩移除时才重新落一份。时间戳这类天生每轮都变的东西走的正是这条路——`dsh-time-context` 的 README 因此可以写 `Append-only`。
- **改变工具 schema 的集合、渲染或顺序**。schema 在 prompt 之后、历史之前，改一处就废掉后面全部历史。`dsh-system-prompt` 的原话是"重排改变缓存形态但不改变语义内容"——也就是说，**为了好看而重排工具顺序是纯亏损**。注册/注销插件、按 agent 限制工具可见性同理。
- **切换路由（provider/model）**。[`dsh-ui-model-selection`](../packages/client/ui-model-selection/README.md) 说得很直白：切路由会削弱或作废 provider 侧的缓存复用，尽管 prompt 前缀本身没被碰过。
- **任何替换式的历史改写**。压缩和裁剪就是故意付这笔钱的两个手段，它们换回的是"对话还能继续"。

反过来，仓库里有一处**刻意为缓存做的设计**值得抄：compaction-basic 的摘要调用把上次路由请求的系统提示词、工具与被遮蔽区间的消息**逐字重放**，只在末尾追加一条压缩指令，让这次辅助调用成为主对话的一个真前缀，从而复用 provider 的暖前缀而不是把它打掉：

```ts
  const messages: Message[] = [
    ...input.messages,
    createUserMessage({
      content: [{ type: 'text', text: COMPACTION_INSTRUCTION }],
      source: { kind: 'plugin', plugin: 'dsh-compaction-basic' },
    }),
  ]
```

`packages/compaction/compaction-basic/src/summarizer.ts:146-152`（`summarize` 请求构造）

**模式**：需要一次辅助模型调用时，尽量把它构造成主请求的前缀加后缀，而不是另起一份提示词。

## 写插件时的成本自查清单

> **上游事实源**：[`docs/cookbook/adding-a-package.md#4-write-the-package-readme`](../docs/cookbook/adding-a-package.md#4-write-the-package-readme)、[`packages/AGENTS.md`](../packages/AGENTS.md)、[`AGENTS.md#conventions`](../AGENTS.md#conventions)

提交前逐条过一遍；具体怎么写在 [plugin_development_guide.md](plugin_development_guide.md)，门禁怎么跑在 [quality_gates.md](quality_gates.md)。

**贡献的内容**

- 我往系统提示词加了东西吗？它是每轮固定成本。能不能只在需要时出现、或者做成按 agent 作用域的 section？
- 我加的是 `PromptSection` 还是 `PromptContext`？**只要内容会随轮次变化，就必须是后者**，否则每轮都在废掉整条缓存。
- 我注册了新工具吗？schema 每轮全量重发。字段名和描述能不能更短而不损失语义？
- 我改动了工具的顺序或可见集合吗？如果只是为了美观，撤销它。

**产出的内容**

- 我的工具可能返回多大的输出？有没有封顶？封顶用 `dsh-output-retention` 的 retainer 还是自己写的渲染器（像 `read` 那样）？
- 超限时我给了模型可执行的恢复指引吗？（retention 库只负责省略子句，恢复动作的措辞归工具自己。）
- 我的 `truncated` / `omitted` 只表达预算事实吗？有没有把上游失败混进去？
- 大文本结果适合走 `dsh-spill-policy` 吗？注意它只处理**最终的纯文本结果**，嵌套结果、`read`、被拦截的决策、含非文本块的结果都会原样通过。

**计量与配置**

- 我引入的可部署调节量是不是都做成了校验过的 `Config` 字段？仓库规则是"插件里不许有硬编码可调项"，一个 `DEFAULT_*` 常量不算可配置。
- 我的字节/码点上限和 token 之间的换算，有没有在文档里说清楚是近似？（pruner 就是把这条写进 Known Limitations 的范例。）
- 需要判断压力时，我用的是 `ctx.tokenMeter` 还是自己数字符？应该是前者——单一计量服务是压缩、占用率显示与遥测共享的同一份账。

**文档**

- 包 README 的 `Model Experience` 三段（`What the model sees` / `Token effect` / `KV Cache effect`）都填了吗？`KV Cache effect` 有没有**点名**本包哪些改动会让复用失效？
- 稳定的模型可见文本有没有逐字粘贴进 `markdown` 围栏？数据相关的部分才允许概述。
- 新增/改动的配置字段跑过生成器了吗？[`docs/config-catalog.md`](../docs/config-catalog.md) 是生成产物，手改无效。

## 延伸阅读

**上游权威页**

- [`docs/subsystems/compaction.md`](../docs/subsystems/compaction.md) — 压缩事件、`CompactionResult`、服务语义、裁剪结果类型（中文：[`compaction.zh.md`](../docs/subsystems/compaction.zh.md)）
- [`docs/subsystems/spill.md`](../docs/subsystems/spill.md) — 保存请求、`SpillRef`、本地后端的落盘布局（中文：[`spill.zh.md`](../docs/subsystems/spill.zh.md)）
- [`docs/subsystems/token-meter.md`](../docs/subsystems/token-meter.md) — `TokenMeasurement` 与 `TokenSurfaceNode` 字段（中文：[`token-meter.zh.md`](../docs/subsystems/token-meter.zh.md)）
- [`docs/subsystems/session-query.md#bounded-event-reads`](../docs/subsystems/session-query.md#bounded-event-reads) — 有界事件读取
- [`docs/subsystems/system-prompt.md#dynamic-prompt-context`](../docs/subsystems/system-prompt.md#dynamic-prompt-context) — 动态上下文为什么是 cache-safe 的那一侧
- [`docs/cookbook/adding-a-package.md#4-write-the-package-readme`](../docs/cookbook/adding-a-package.md#4-write-the-package-readme) — Model Experience 的规范格式
- [`docs/config-catalog.md`](../docs/config-catalog.md) — 全部配置字段的穷尽来源（**生成产物**）

**包 README**

- [`packages/compaction/README.md`](../packages/compaction/README.md) — 压缩家族的包地图
- [`packages/compaction/compaction-basic/README.md`](../packages/compaction/compaction-basic/README.md) — 阈值、保留策略、溢出恢复
- [`packages/compaction/compaction-tool-result-pruner/README.md`](../packages/compaction/compaction-tool-result-pruner/README.md) — 头/中/尾裁剪
- [`packages/compaction/command-compact/README.md`](../packages/compaction/command-compact/README.md) — `/compact`
- [`packages/spill/README.md`](../packages/spill/README.md) — 溢出家族的包地图
- [`packages/spill/spill-local/README.md`](../packages/spill/spill-local/README.md) — 本地后端与启动清理
- [`packages/llm/token-meter/README.md`](../packages/llm/token-meter/README.md) — 计量服务与三个投影
- [`packages/util/output-retention/README.md`](../packages/util/output-retention/README.md) — 两个 retainer 与省略注记

**兄弟中文导航页**

- [architecture_overview.md](architecture_overview.md) — 插件树、profile/bundle、回合流程
- [capability_seams.md](capability_seams.md) — 三角接缝为什么这样拆
- [prompt_management.md](prompt_management.md) — 系统提示词的组装与作用域
- [agent_loop_and_tools.md](agent_loop_and_tools.md) — `agent/pre-step`、`tools/post-execute` 等扩展点
- [session_and_events.md](session_and_events.md) — 会话日志、表面与事件
- [model_configuration.md](model_configuration.md) — 路由、适配器与上下文容量
- [plugin_development_guide.md](plugin_development_guide.md) — 写插件的完整流程
- [quality_gates.md](quality_gates.md) — 文档与配置门禁怎么跑
