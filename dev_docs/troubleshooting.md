---
title: 故障排查
summary: 从运行期观察到的现象反查权威文档、事故记录与诊断工具的中文索引。
keywords: troubleshooting | postmortem | defensive-patterns | diagnostics | invariants
scope: deepseek-harness 运行期故障的现象索引与诊断入口
related_files: docs/postmortem | docs/defensive-patterns.md | packages/runtime-diagnostics | packages/session-query
dependencies: docs/postmortem/README.md | docs/defensive-patterns.md | docs/subsystems/invariants.md
verified_at: 2026-09-06
---

# 故障排查

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**，不是事实的归属地。事故经过归 [`docs/postmortem/`](../docs/postmortem/README.md)——那是仓库唯一允许写事故叙事的层；防御性模式归 [`docs/defensive-patterns.md`](../docs/defensive-patterns.md)。本文不复述它们，只做一件上游没做的事：**从你观察到的现象反查该读哪一篇。**
>
> **门禁报错**（覆盖率、快照、双语配对、链接失效等）不在本文范围，见 [质量门禁导航](quality_gates.md)。本文只管运行期故障。
>
> **与 `docs/` 冲突时一律以 `docs/` 为准**，并立即修正本文。

## 1. 本文定位与上游事实源

> **上游事实源**：[`docs/postmortem/README.md`](../docs/postmortem/README.md)、[`docs/defensive-patterns.md`](../docs/defensive-patterns.md)、[`docs/subsystems/`](../docs/subsystems/README.md) 各子系统页

排查一个运行期故障，通常卡在同一个地方：**你看到的是症状，而仓库的文档是按子系统组织的**。你看到"工具没出现在模型可见列表"，但没有一篇文档叫这个名字——真正的答案分散在事故记录 0002、`docs/subsystems/tools.md` 的作用域过滤、以及 preset 的组合规则里。本文就是那张缺失的映射表。

因此本文**刻意不解释任何机制**。每一行只回答"去哪读"，机制留在归属地。这条自律有个实际后果：如果你发现本文某一行讲了上游已经讲过的东西，那是缺陷，应当删掉换成链接。

三条边界值得先说清楚：

- **门禁 / 静态检查的报错不在这里。** `pnpm run doc-sync`、`test:coverage`、`verify-*` 这类报错的反查由 [质量门禁导航](quality_gates.md) 拥有。本文管的是**跑起来之后**出问题。
- **事故经过不在这里。** `docs/postmortem/` 是仓库唯一允许写 war story 的层（[README](../docs/postmortem/README.md) 定义了什么样的 bug 才配写一篇）。本文第 3 节只做索引，不复述任何时间线。
- **本文不是完整的错误码表。** 错误码的权威定义在各子系统页的 `Errors` 小节与包 README，本文只在需要"认出它"的时候引用码名。

## 2. 现象索引

> **上游事实源**：[`docs/postmortem/`](../docs/postmortem/README.md)、[`docs/subsystems/`](../docs/subsystems/README.md)、各包 README

这是本文的核心。**用法**：在"现象"列找到最接近你观察到的那一行，先做"先查什么"列的动作，再去"权威文档"列读机制。**不要跳过第三列直接读第四列**——大多数情况下第三列的一条命令就能把可能原因砍掉一半。

在翻表之前，有一个几乎总是值得先走一遍的顺序，因为它能最快地把"配置问题"和"代码问题"分开：

1. **确认树。** `dsh --profile <name> --dump-config`——你以为挂上的插件是否真的在树里、配置值是否真的是你写的那个（见 [5.1](#51---dump-config看实际的插件树)）。
2. **确认身份。** 你正在观察的进程/端口/会话，是否就是出问题的那一个（事故 0003 的全部教训）。
3. **确认证据来源。** 会话日志是运行期最完整的证据源；从事件序列读事实，而不是从事后叙述里推测（见 [5.3](#53-会话日志导出与查询)）。
4. **再看代码。** 前三步都排除之后，才值得去读实现。

### 2.1 组合与加载

| 现象 | 可能原因 | 先查什么 | 权威文档 |
|---|---|---|---|
| 插件加载即崩，报 `cannot get property "<服务名>" without inject` | 插件模块里多了一行 `export default apply`，Loader 的 `unwrapExports` 优先取 `.default`，把带 `name`/`inject`/`Config` 的命名空间整个丢掉 | 打开该插件的 `src/index.ts`，确认它是纯命名空间导出、**没有** `export default` | [事故 0001 根因一](../docs/postmortem/0001-acp-default-export-drops-inject.md) |
| 插件里读一个**没写进 `static inject`** 的可选服务时抛同样的 without inject | 属性代理的 fiber 走查是**只走祖先**的；经由 traceable shadow 从别的 fiber 调进来时，兄弟分支上的服务走不到 | 检查那处读法是 `ctx.<name>` 还是 `ctx.get('<name>')`；可选服务必须用后者 | [事故 0001 根因二](../docs/postmortem/0001-acp-default-export-drops-inject.md) |
| `cordis.yml` 里 `disabled: !!js ...` 写法没报错，但那条插件**永远**处于禁用（或永远启用） | `!!js` 只在插件 `config` 与条目 `disabled` 内被求值，其他条目元数据保持字面量；条件化组合应当改用 overlay 文件 | `dsh --profile <name> --dump-config` 看该行实际是什么；`!!js` 在 dump 里原样打印、不求值 | [事故 0002](../docs/postmortem/0002-js-expression-disabled-filesystem-tools.md)、[Cordis 入门 Loader Configuration](../docs/cordis-primer.md#loader-configuration) |
| 组合树和你以为的不一样：某行被别的层覆盖了 | 层序是"每个 bundle → profile 的 `cordis.patch.yml` → home 级 → `--patch`"，后者按 id 整体替换前者的 config | `--dump-config` 里每一段**同源的行**前都带一条 `# ==` 注释，指出是哪个文件供的、被哪些 overlay 改过 | [`docs/architecture.md` 的 Profiles and bundles](../docs/architecture.md#profiles-and-bundles)、[CLI 行为参考](../apps/cli/reference/README.md) |
| 装了 bundle 但重启前不生效 | Bundle 成员关系是**启动边界**；`cordis.patch.yml` 的普通编辑才走热重载 | 确认改的是 bundle 依赖（需重启）还是 patch 文件（热重载） | [CLI 行为参考](../apps/cli/reference/README.md) |
| 热重载/HMR 后同一个包名再次注册 invariant 抛错 | 注册名是独占预留；重复、空白或含空格的包名会抛 | 确认前一次注册的 disposer 是否随 fiber 卸载执行——注册是 effect，卸载任一侧都应释放预留 | [`docs/subsystems/invariants.md` 的 The service](../docs/subsystems/invariants.md#the-service) |

### 2.2 工具与执行

| 现象 | 可能原因 | 先查什么 | 权威文档 |
|---|---|---|---|
| 工具没出现在模型可见列表；调用它得到 `UNKNOWN_TOOL` | 提供该工具的插件根本没挂上（见 2.1 各行），或被作用域的 `ToolRestriction` 过滤掉了 | 先 `--dump-config` 确认插件在树里；再看该 scope 的 allow/deny——allow-list 会排除后来新增的继承工具 | [事故 0002](../docs/postmortem/0002-js-expression-disabled-filesystem-tools.md)、[`docs/subsystems/tools.md`](../docs/subsystems/tools.md) |
| 工具调用挂住，最后返回 `tool call timed out after <ms>ms` | 协作式截止时间到期：插件通过 `exec.signal` 请求停止，等它 settle 后映射成这条错误 | 看该工具自身配置里的限时字段（例如 `dsh-tool-web` 的 `fetchTimeoutMs`/`searchTimeoutMs`） | [`dsh-tool-call-timeout-policy` README](../packages/guard/timeout-policy/README.md) |
| 工具调用永远挂住，**没有**超时错误 | 该工具没有配置限时（`bash`、`read`、`write`、`edit` 出厂即不被切断），或它忽略了取消信号——该插件**只能请求**停止，不会硬杀 | 确认工具是否 honor `exec.signal`；不 honor 的工具无法被这条策略保护 | [`dsh-tool-call-timeout-policy` README](../packages/guard/timeout-policy/README.md) |
| `web_search` / `web_fetch` 报 `WEB_PROVIDER_AMBIGUOUS` | 挂了多个可用 provider 又没指定 id；选择**不会**退化成先注册者优先 | 配置 `searchProvider`/`fetchProvider`，或只挂一个 provider | [`docs/subsystems/web.md` 的 Provider availability](../docs/subsystems/web.md#provider-availability) |
| 会话查询工具报 `SESSION_QUERY_TOOL_UNAUTHORIZED` | 工作区授权是保守的：跨会话访问要求目标与调用方会话的 `cwd` **完全相等**；没有 `cwd` 的调用方只能查自己 | 比对两个会话头里的 `cwd` | [`dsh-tool-session-query` README](../packages/session-query/tool-session-query/README.md) |
| 会话搜索返回 `SESSION_QUERY_SEARCH_DISABLED` 或 `SESSION_QUERY_INDEX_FAILED` | 全文检索需要 SQLite 后端注册在 `ctx.sessionQuery` 上；部署可以关掉它 | `--dump-config` 确认 `dsh-session-query-sqlite` 在树里 | [`docs/subsystems/session-query.md` 的 Errors](../docs/subsystems/session-query.md#errors) |
| 分页查询报 `SESSION_QUERY_INVALID_CURSOR` / `SESSION_QUERY_STALE_CURSOR` | 游标是**绑定**到归一化后的查询、元数据过滤与 limit 的；换了任意一项就不能复用旧游标 | 确认翻页时这三项完全没变 | [`docs/subsystems/session-query.md` 的全文检索分页](../docs/subsystems/session-query.md#full-text-search-pages) |
| 模型看到的工具结果被换成了预览 + 一个 locator | 挂了 spill 策略消费者：超过 `maxInlineBytes` 的纯文本终结果会被替换成头尾预览加 spill 引用，**完整内容仍被保存** | 用结果里的 `retrievalHint` 取回全文——`locator` 是不透明句柄，不要去解析它 | [`docs/subsystems/spill.md` 的 The service](../docs/subsystems/spill.md#the-service) |
| 结果非常大却**没有**被 spill，直接灌进上下文 | spill 是**可选**能力，不在 agent-loop 主干上；没挂 `dsh-spill-policy` 就不会有替换 | `--dump-config` 确认策略消费者在树里 | [`docs/subsystems/spill.md`](../docs/subsystems/spill.md) |

### 2.3 子进程、沙箱与审批

| 现象 | 可能原因 | 先查什么 | 权威文档 |
|---|---|---|---|
| 命令结束了，但 helper 进程残留 | teardown 只发了 kill 就返回，没等到静止 | 确认清理路径是 `terminate()` 之后 `await waitForExit()`（观察的是**整棵进程树**），且监听器/通知注册在 kill **之前**已关闭 | [`docs/defensive-patterns.md` 的 Dispose must reach quiescence](../docs/defensive-patterns.md#dispose-must-reach-quiescence-not-just-request-it)、[`docs/subsystems/subprocess.md`](../docs/subsystems/subprocess.md#handles-streams-readers-and-tree-scoped-termination) |
| 子进程"超时了"却被当成正常退出 | 超时与退出码是**正交事实**，把一个塞进另一个的分支里，调用方就会把被腰斩的运行读成干净成功 | 检查结果类型是否独立暴露 `timedOut` / `signal` / `exitCode` | [`docs/defensive-patterns.md` 的 Report orthogonal outcomes independently](../docs/defensive-patterns.md#report-orthogonal-outcomes-independently) |
| 子进程输出缺了中间一段 | 读取偏移滑出了内存尾窗，该次读被标记 `lossy`，完整内容只在 spill 文件里 | 看这次读返回的 `lossy` 与 `spillPath` | [`docs/subsystems/subprocess.md`](../docs/subsystems/subprocess.md#handles-streams-readers-and-tree-scoped-termination) |
| 命令失败并报 `SANDBOX_UNAVAILABLE` | 本机没有可用后端，`confine()` 快速失败——**对受限策略静默放行是永远非法的**，所以它宁可报错 | 先确认这是宿主真的没有后端，还是下一行那种误判 | [`docs/subsystems/sandbox.md` 的 Provider and fail-closed errors](../docs/subsystems/sandbox.md#provider-and-fail-closed-errors) |
| 在旧 Landlock ABI 的机器上，**正常**的非零退出（例如 ripgrep 无匹配的 exit 1）被报成沙箱不可用 | 把启动器的良性提示行与子进程自己的退出码错误地拼成了一个结论 | 该类误判的判定规则现在要求"状态码 + 每行致命签名 + 精确的信息行排除"三者合取 | [事故 0004](../docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md)、[`docs/subsystems/sandbox.md` 的分类方言](../docs/subsystems/sandbox.md#wrapped-argv-and-classification-dialects) |
| 沙箱模式看起来对，写操作却被拒 | `workspace-write` 的可写根来自**调用会话不可变的 cwd**（先按文件系统语义规范化再做词法规范化），不是你当前 shell 的目录 | 核对会话头里的 `cwd` 与 `workspaceRoot` | [`docs/subsystems/sandbox.md` 的 Per-call policy](../docs/subsystems/sandbox.md#per-call-policy) |
| 明明该有围栏，报告却说只是 `partial` | 旧内核 ABI 与 Windows ACL runner 的 Everyone/硬链接边界是当前已知的部分执行情形 | 读该后端报告的 enforcement 值——需要绝对边界的调用方**必须**拒绝 `partial` | [`docs/subsystems/sandbox.md` 的 Modes and enforcement](../docs/subsystems/sandbox.md#modes-and-enforcement) |
| 审批弹窗一直出现 | 会话策略是 `ask`，每次都下发到 answerer 链 | 会话内生效值是日志里最后一条 `approval/policy` 事件，回退到服务配置 | [`docs/subsystems/approval.md` 的 Per-session policy](../docs/subsystems/approval.md#per-session-policy) |
| 从不弹窗，所有需审批的动作直接失败 | 要么策略是 `never`（确定性返回 `rejected`，在服务内部就短路，后 prepend 的 answerer 也绕不过），要么没有 answerer——链落到 fail-closed 的 `unavailable` | 确认该部署是否挂了 UI answerer；`unavailable` 时调用方一律拒绝 | [`docs/subsystems/approval.md` 的 Identity and outcome](../docs/subsystems/approval.md#identity-and-outcome) |

### 2.4 会话、上下文与模型

| 现象 | 可能原因 | 先查什么 | 权威文档 |
|---|---|---|---|
| 打不开某个历史会话，报格式拒绝 | 后端拒绝一份它无法忠实解释的日志（`SessionFormatUnsupportedError`）——它与"损坏"（`SessionPersistenceCorruptionError`）是两回事：格式拒绝表示"读不懂"，不表示"数据坏了"，一个更高的未来代次即使旁边还留着可读的旧代次也照样拒绝；v0/v1 的历史迁移即使事件标了 `ignorable` 也会拒绝未知类型 | 错误消息会附上被拒的原始日志路径（当后端为每个会话保留单一产物时） | [`docs/subsystems/persistence.md` 的 Format refusal](../docs/subsystems/persistence.md#format-refusal--logs-a-build-cannot-faithfully-read) |
| 会话恢复（resume/load）时抛 without inject | 恢复路径读的是一个可选服务，经 shadow 走查失败——见 2.1 第二行 | 同 2.1 第二行 | [事故 0001 根因二](../docs/postmortem/0001-acp-default-export-drops-inject.md) |
| 崩溃重启后，被打断的那个 turn 状态可疑 | 崩溃恢复对"被打断的 turn"有明确约定，不是简单丢弃 | 读该小节确认预期形态，再判断是否真的异常 | [`docs/subsystems/persistence.md` 的 Crash recovery](../docs/subsystems/persistence.md#crash-recovery-preserves-an-interrupted-turn) |
| 上下文莫名变短，早先的内容不见了 | 压缩在 `agent/pre-step` 瀑布上按 `pressure` 或 `context-overflow` 触发；工具结果裁剪可能在选段前就先跑了 | 看会话日志里的 `compaction/*` 事件与替换用的 `user/message` checkpoint 来源 | [`docs/subsystems/compaction.md` 的 The service](../docs/subsystems/compaction.md#the-service) |
| 手动压缩失败但会话表面没变 | `changed` / `summary` 两类失败保持会话表面不变，但仍会关闭并持久化这次失败尝试 | 按 `ManualCompactionErrorCode` 的取值区分是哪一类 | [`docs/subsystems/compaction.md` 的 The service](../docs/subsystems/compaction.md#the-service) |
| 流式输出中断，且失败有时抛异常、有时是终止 chunk | 模型请求失败在公共 API 上**只**以终止 finish chunk 暴露；抛出的异常属于中间件或消费方缺陷 | 先按这条区分故障归属，再读归一化后的失败载荷字段（`code`、`status`、`providerRetryAfterMs`、`requestId`） | [`docs/defensive-patterns.md` 的 Honor public contracts on BOTH sides](../docs/defensive-patterns.md#honor-public-contracts-on-both-sides)、[`docs/subsystems/llm-streaming.md` 的 `LlmFailure`](../docs/subsystems/llm-streaming.md#llmfailure) |
| 等一次 follow-up 的"完成"永远等不到 | `agent.followup()` 没有单条消息的完成或结果；多条排队的 follow-up、steering 与注入的工作可能共享同一个 `running` 区间 | 明确定义你自己的区间（例如从消息的持久化收件回执到下一次整体 `idle`），并显式处理"根本没有可等的事"这一分支 | [`docs/defensive-patterns.md` 的 Async state is not synchronous state](../docs/defensive-patterns.md#async-state-is-not-synchronous-state) |

### 2.5 Web 前后端

| 现象 | 可能原因 | 先查什么 | 权威文档 |
|---|---|---|---|
| 页面白屏，控制台报 `window.__DSH_BOOT__ is missing or not an object` | 起的是裸 Vite，而不是完整的 `dsh web` 宿主——boot manifest 只由完整宿主注入；裸 Vite 的 HTTP 200 **不代表**应用可用 | 确认启动方式；`apps/web` 的 standalone Vite serve 模式现在会在配置阶段直接拒绝 | [事故 0003](../docs/postmortem/0003-web-agent-gui-feedback-loop.md)、[`docs/subsystems/web-client.md`](../docs/subsystems/web-client.md) |
| 改了源码，页面没变；或"验证通过"的是另一个端口的服务 | 验收对象没有对齐：一个替代服务证明不了**用户正在用的那个页面**变了 | 读 prompt 里 `app:web-surface` 段与 `$DSH_WEB_URL` 拿到当前 GUI 的规范 URL，在**那个 origin** 上外部观察 | [事故 0003](../docs/postmortem/0003-web-agent-gui-feedback-loop.md) |
| 生产模式下刷新后仍是旧界面 | 生产路径要求先重建产物，再在既有 URL 上刷新验证 | 先 `pnpm run build`，再刷新那个 URL | [事故 0003](../docs/postmortem/0003-web-agent-gui-feedback-loop.md)、[CLI 行为参考的 Web alias](../apps/cli/reference/README.md#web-alias) |
| 开发模式下客户端插件改动不热更 | HMR 接收端**始终**挂载但保持空闲，直到同一 checkout 里另起 `pnpm run dev:web` 重建客户端 bundle；shell 与普通包的改动仍需刷新 | 确认 watcher 是否在跑 | [CLI 行为参考的 Web alias](../apps/cli/reference/README.md#web-alias)、[`docs/subsystems/client-modules.md`](../docs/subsystems/client-modules.md) |
| 插件 bundle 请求返回 404 | 未知或被改动的资源列表、缺失的 revision、过期的 revision 一律回 404，而不是供给不同的字节或让 SPA fallback 把 HTML 当 JS 返回 | 对比页面里 `__DSH_BOOT__` 的 rev 与宿主当前图的 rev | [`docs/subsystems/client-modules.md` 的 bundle 路由](../docs/subsystems/client-modules.md#the-bundle-route-and-index-injection) |

### 2.6 启动、关停与部署默认值

这一组现象的共同点是：**它们全都是被设计成这样的**，只是文档在别处。误以为是 bug 而去改代码，是这一组最常见的浪费。

| 现象 | 可能原因 | 先查什么 | 权威文档 |
|---|---|---|---|
| `dsh web --host 0.0.0.0` 直接报用法错误 | CLI **有意**不支持该绑定 | 用 `--host` 指定具体地址，或用 `--trusted-host` 追加被 `/api` 浏览器信任栅栏接受的授权名 | [CLI 行为参考的 Web alias](../apps/cli/reference/README.md#web-alias) |
| 启动了 `dsh web` 但浏览器没自动打开 | 继承环境里 `SSH_CONNECTION` 或 `SSH_TTY` 非空时**故意**抑制浏览器交接（转发地址归 SSH 客户端或编辑器管），URL 仍会打印；或本机交接失败，stderr 会说明原因并留着服务器 | 读 stderr；`--no-open` 也会关掉这一步 | [CLI 行为参考的 Web alias](../apps/cli/reference/README.md#web-alias) |
| `Ctrl+C` 之后进程没有立刻退出 | 第一次 `SIGINT`/`SIGTERM` 启动优雅排空，插件树最多有五秒 dispose；第二次信号强制立即退出 | 退出码可区分来源：`SIGTERM` 各平台一律 0，`SIGINT` 报 130 | [CLI 行为参考](../apps/cli/reference/README.md) |
| headless 一次性任务退出码是 1，但看起来"跑完了" | 退出码取自持久区间里最后的 `turn/end` 原因：`completed` 才是 0，其余为 1 | 读会话日志里最后一个 `turn/end` 的原因 | [CLI 行为参考](../apps/cli/reference/README.md) |
| `DSH_TOOLS_MODE` 设了个值，进程启动就失败 | 该变量只接受 `native`、`ptc`、`both`，其他值在 boot 阶段失败 | 检查环境变量拼写 | [CLI 行为参考](../apps/cli/reference/README.md) |
| 在设置里改了权限，当前这个 Web 会话不受影响 | 存储的 General 权限设置作用于**之后**的 Web 会话，不改已打开的会话 | 新建会话验证；`DSH_PERMISSION_MODE` 改的是进程级回退 | [CLI 行为参考](../apps/cli/reference/README.md) |
| `sdk-minimal` 下没有任何审批弹窗，且沙箱形同虚设 | 这是**有意**的：该独立组合固定 `danger-full-access`，不挂审批与权限设置服务，也不做 instruction 发现与 SQLite | 需要围栏就别用 `sdk-minimal` | [CLI 行为参考](../apps/cli/reference/README.md) |

### 2.7 现象不在表里时

这张表覆盖不了所有情况，也不打算覆盖。现象不在表里时，用下面这条路径把它归位，而不是从代码入口开始读：

1. **先判断它属于哪个能力接缝。** 一个故障几乎总能落到某个 `ctx.<service>` 上；[`docs/subsystems/README.md`](../docs/subsystems/README.md) 是子系统页的总入口，[能力接缝](capability_seams.md) 是它的中文地图。
2. **该子系统页的 `Errors` / 失败小节通常直接给答案。** 这个仓库的惯例是把失败分类写成**封闭的码集合**并写进子系统页，所以拿到一个 `XXX_YYY` 形态的码，先在对应子系统页里搜它。
3. **码不在子系统页里，就去包 README 的"Failures"小节。** 工具层与消费者层自己拥有的失败（例如 `SESSION_QUERY_TOOL_UNAUTHORIZED`）归包，不归接缝。
4. **仍然找不到，且这个 bug 是"隐蔽 + 系统性 + 重发现代价高"三条齐备的**——那么修完之后，它可能就该成为第 5 篇 postmortem（收录门槛见 [README](../docs/postmortem/README.md)）。

## 3. `docs/postmortem/` 中文索引

> **上游事实源**：[`docs/postmortem/README.md`](../docs/postmortem/README.md)

仓库目前有 4 篇事故记录。它们是**唯一**允许写事故叙事的地方，收录门槛是三条同时成立：机制不显然、逃逸原因是流程/工具/约定的系统性缺口而非一次笔误、且重新发现的代价高。**下表只做索引**——事故经过、时间线、根因推导全部留在原文，本文只回答"它对应今天的哪条规则"。

每篇都以 **Executive summary** 开头，三十秒可读完；不确定要不要读全文时，先读那一段。

| # | 中文标题概括 | 它教会了什么现行规则 | 原文 |
|---|---|---|---|
| 0001 | ACP 服务器一连接就崩：`export default` 把插件的 `inject` 丢了 | ① 命名空间插件与 `export default` **互斥**，Loader 的 `unwrapExports` 会丢弃命名空间；② 不写进 `static inject` 的可选服务必须用 `ctx.get(name)`，属性代理的祖先走查会在 shadow 下失败；③ 至少要有一个测试走**真实 Loader / 真实导出路径**，且当那条操作不调模型时它就该进 CI 而不是躲在 key gate 后面 | [0001](../docs/postmortem/0001-acp-default-export-drops-inject.md) |
| 0002 | 文件系统快照工具被一个字面量 `!!js` 对象永久禁用 | ① `!!js` 只在插件 `config` 与条目 `disabled` 内求值，其余条目元数据保持字面量，条件化组合改用 overlay；② 快照 refresh 是**产出 fixture**，不是正确性评审，"工具根本没注册"这类语义不可能必须有独立断言；③ 权限控制只能描述它真正治理的能力——组合期的文件系统访问不能跟随运行期的 bash-only preset | [0002](../docs/postmortem/0002-js-expression-disabled-filesystem-tools.md) |
| 0003 | Web agent 验证了一个替身服务器，而不是承载它自己会话的那个 GUI | ① 当前 GUI 的规范 URL 与运行模式必须对模型可见（`app:web-surface` 段与 `$DSH_WEB_URL`）；② HTTP 就绪、构建成功、boot manifest 存在是**三件不同的事**，验收要点名确切 origin 并在那里外部观察；③ 替身服务证明不了既有页面变了，长驻进程要走受管任务生命周期；④ 回归测试必须能**因所报机制而失败**，进程超时不等于 fail-fast | [0003](../docs/postmortem/0003-web-agent-gui-feedback-loop.md) |
| 0004 | Landlock 部分执行提示把子进程失败错误分类了 | ① 进程归因需要**多项独立证据的合取**，共享前缀不是协议；② 信息性与致命性诊断可以共用命名空间，排除必须精确且窄，未知的致命行保持 fail-closed；③ 适配器必须保留下层 seam 拥有的结构化失败，不能替换成自己最近的泛化类别；④ 平台相关行为需要"原生边界上的确定性 fake + 一条组装后的产品路径"，会自跳过的真内核测试扛不起回归 | [0004](../docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md) |

一个使用建议：这 4 篇的根因**全都**落在同一个家族里："**测试走的路径不是产品走的路径**"。当你在排查一个"单测全绿但真跑就崩"的问题时，先假设自己也踩了同一类坑，比逐行读代码更快。

## 4. 防御性模式速查

> **上游事实源**：[`docs/defensive-patterns.md`](../docs/defensive-patterns.md)

[`docs/defensive-patterns.md`](../docs/defensive-patterns.md) 全文只有三十余行，每条都是**真的在这个仓库里出过或差点出的一类缺陷**，写成防止复发的规则。[`AGENTS.md`](../AGENTS.md) 把它列为写生命周期、并发、子进程、teardown 代码前的**必读**。

本节不复述任何一条，只给触发条件——**命中任意一条就去读全文**：

- 你要返回一个可能同时为真的多个结果事实（超时且退出 0、被信号杀死且有退出码）→ [Report orthogonal outcomes independently](../docs/defensive-patterns.md#report-orthogonal-outcomes-independently)
- 你在实现一个 provider，而它的失败可以既抛异常又发终止事件 → [Honor public contracts on BOTH sides](../docs/defensive-patterns.md#honor-public-contracts-on-both-sides)
- 你打算 `await` 某个状态转换来代表"我这条消息处理完了" → [Async state is not synchronous state](../docs/defensive-patterns.md#async-state-is-not-synchronous-state)
- 你在写 `dispose` / `close` / teardown，里面有 kill 或 abort → [Dispose must reach quiescence, not just request it](../docs/defensive-patterns.md#dispose-must-reach-quiescence-not-just-request-it)
- 你在写一个会调用外部注册监听器的分发循环 → [Contain callback exceptions in the dispatcher](../docs/defensive-patterns.md#contain-callback-exceptions-in-the-dispatcher)
- 你要 spawn 一个命令，或要落一个临时/spill 文件 → [Never hand untrusted output the ambient environment or predictable paths](../docs/defensive-patterns.md#never-hand-untrusted-output-the-ambient-environment-or-predictable-paths)
- 你要删除一个可能是符号链接或 Windows junction 的路径 → [Unlink link-shaped paths](../docs/defensive-patterns.md#unlink-link-shaped-paths)

测试层面的对应规则（真实入口路径、世界验证、资源归属）在 [`docs/testing.md`](../docs/testing.md)，中文导航见 [测试指南](testing_guide.md)。

## 5. 诊断工具

> **上游事实源**：[`docs/architecture.md`](../docs/architecture.md#profiles-and-bundles)、[CLI 行为参考](../apps/cli/reference/README.md)、[`docs/subsystems/invariants.md`](../docs/subsystems/invariants.md)、[`docs/subsystems/session-query.md`](../docs/subsystems/session-query.md)

### 5.1 `--dump-config`：看实际的插件树

排查"插件/工具没生效"类问题的**第一步永远是这个**，因为它把"我以为组合出了什么"换成"实际组合出了什么"：

```sh
dsh --profile web --dump-default-config     # 只有 bundle 层
dsh --profile web --dump-config             # 再叠 profile / home 的 cordis.patch.yml 与 --patch
dsh --profile web --patch ./extra.yml --dump-config
dsh web --dump-config                       # web 别名同样支持
```

需要知道的输出性质，全部来自 [CLI 行为参考](../apps/cli/reference/README.md)：每段同源的行前带注释指出**哪个文件供了这些行、哪些 overlay 改过它们**；`!!js` 表达式**原样打印、不求值**（这正是事故 0002 里能一眼看出问题的原因）；插入行里的相对插件名按 patch 文件所在位置解析；**未命中的 patch 目标会在 stderr 上报告**——这条常被忽略，patch 写错 id 时它是唯一线索。

两个边界：dump **不启动应用**，因此不会运行任何应用命令行 provider，展示的是解析 app 参数之前的组合树，并且会拒绝带了 app 参数的调用；它会初始化缺失的 profile 文件，但不准备 `$DSH_HOME/profiles/node_modules` 下的运行时模块回退。

入口实现是 [`apps/cli/src/dump-config.ts`](../apps/cli/src/dump-config.ts) 的 `runDumpConfig`（第 30 行），渲染在 [`packages/boot/app-boot/src/index.ts`](../packages/boot/app-boot/src/index.ts) 的 `renderConfigDump`（第 412 行）。另外提醒一句（这是本文的操作建议，不是上游契约）：**别拿该输出当序列化契约做程序化消费**，它没有承诺跨版本字节稳定。

### 5.2 运行时不变量：让违约当场大声失败

[`dsh-invariants`](../packages/runtime-diagnostics/invariants/README.md) 是包自有运行期不变量检查的注册表服务（`ctx.invariants`）。每个工作区包通过 `./invariant` 配套插件用**自己的 npm 包名**注册检查，违约时抛 `InvariantError`——稳定 `code: 'INVARIANT'`、带 `packageName`、消息前缀 `invariant violated by "<package>": …`，所以你**一眼就知道是哪个包的契约被破坏了**。

排查时它的用法是**选择性开关**：服务配置支持 `enabled` 全局开关，以及 `package_allowlist` / `package_blocklist` 两组正则（blocklist 命中优先于 allowlist）。正则用 `new RegExp(source)` 编译，**不加锚点就是非锚定匹配**，`/pattern/flags` 语法不被解析。配置校验在服务启动时**大声失败**：空白、含首尾空格、重复或非法的条目会抛，而不是被跳过。

有一条边界要记住：不变量断言的是**权威事件流或可变数据**，从不断言服务或方法是否存在。所以"某个服务没挂上"这类问题不会被不变量抓到——那是 `--dump-config` 的活。

注册表实现见 [`packages/runtime-diagnostics/invariants/src/index.ts`](../packages/runtime-diagnostics/invariants/src/index.ts) 的 `register`（第 136 行）与 `InvariantError`（第 50 行）。

### 5.3 会话日志：导出与查询

会话日志是运行期故障最完整的证据源——事故 0003 的整条时间线就是从持久化事件日志按 sequence 号重建的，而不是从事后报告里推测意图。

**程序化查询**走 `ctx.sessionQuery`（[子系统页](../docs/subsystems/session-query.md)）：精确读取、过滤列表、关系追踪，以及由 SQLite FTS5 后端支撑的全文检索；[包组 README](../packages/session-query/README.md) 是这一族的入口地图。检索结果与模型实际看到的会话历史一致，且**独立于压缩**——这一点在排查"模型说它做过某事但当前上下文里没有"时很关键。失败码是封闭集合，见 [Errors](../docs/subsystems/session-query.md#errors)。

排查时最常用的两种读法：

- **有界事件读**（[Bounded event reads](../docs/subsystems/session-query.md#bounded-event-reads)）——定位到一个 seq，然后取它前后各若干条原始事件。这是"我知道大概出问题的位置，想看上下文"的标准动作；返回的窗口带 `startSeq`/`endSeq` 与完整目标事件，不做省略。
- **会话谱系追踪**（[Session lineage](../docs/subsystems/session-query.md#session-lineage)）——拿到祖先链与递归的后代森林。它带一个完整性判别式：`complete: false` 时给出 `unresolvedParentId`，也就是**父链走出了可见语料**。排查子会话/subagent 相关问题时，先看这个标志位再下结论。

**让模型自己查**要显式挂 [`dsh-tool-session-query`](../packages/session-query/tool-session-query/README.md)——它**不在**出厂宿主组合里，挂上会给每次请求增加一段指引和五个 schema。五个工具是 `session_search`、`session_event_search`、`session_trace`、`session_event_trace`、`session_event_read`，结果无游标、跨会话访问要求 `cwd` 完全相等。

**整包导出**用 Web 的 [`dsh-session-log-export`](../packages/session-query/session-log-export/README.md)：Session Header 的 `Session log` 按钮或 `/export` 命令，把会话、子会话与附件打成 ZIP 由浏览器下载。要点：宿主在读之前会 flush 活跃根会话，所以斜杠命令触发的 ZIP **包含**触发它自己的 `command/run` / `command/done` 这一对；`/export <path>` 是错误——下载目标由浏览器决定，这不是宿主侧写文件的接口。

### 5.4 会话统计：判断"慢在哪里"

"这个会话很慢"是最难归因的一类现象，因为候选太多：模型、工具、压缩、持久化。[`dsh-session-stats`](../packages/session/session-stats/README.md) 的 `sessionStats` 投影单元把这个问题变成读几个数字。

它服务的是**整段会话**的口径：`turns` / `steps` 计数，以及 `llmMs`（组装出消息的那些 step 的模型墙钟时间）、`toolMs`（配对的 `tool/call` → `tool/result` 墙钟时间）、`ttftMs` / `ttftSteps`（首 token 延迟及贡献它的 step 数）、`decodeMs` / `decodeTokens`（解码墙钟与 provider 输出 token）。拿 `llmMs` 与 `toolMs` 一比，"模型慢还是工具慢"当场就分开了；`ttftMs / ttftSteps` 与 `decodeTokens / decodeMs` 则进一步把模型侧拆成"等第一个 token"和"吐字速度"两件事。

两个使用前提：它需要组合里已经挂了投影注册表（该单元只在注册表存在时注册），而且它是从**完整持久日志**折叠出来的——**分页与压缩都改不了这些数字**，这正是它比任何窗口内计数更可靠的原因。没有该单元的组合不受影响，消费方回退到窗口口径的计数。

上下文体量本身的估算与回放归 `ctx.tokenMeter`，压缩接缝自己不拥有计价 API（见 [`docs/subsystems/compaction.md` 的 The service](../docs/subsystems/compaction.md#the-service)、[`docs/subsystems/token-meter.md`](../docs/subsystems/token-meter.md)）；成本口径的中文导航另见 [成本优化](cost_optimization.md)。

## 6. 沙箱挡住开发命令时怎么办

> **上游事实源**：[`AGENTS.md` 的 `### Host sandbox failures`](../AGENTS.md#host-sandbox-failures)

这一节说的是**你（或 agent）在开发主机上跑命令被宿主沙箱挡住**，与第 2.3 节里"产品沙箱拒绝了模型发起的命令"是两回事，不要混。

规则原文很短，三条：

1. **触发条件要具体。** 只有当 `gh`、`pnpm`、构建、测试或生成器命令失败的原因是**沙箱阻断了凭据、网络、IPC、文件监视或嵌套 `sandbox-exec`** 时，这条路径才适用。
2. **原样重试 + 最窄提权。** 用**最窄**的宿主提权原样重试同一条命令，不要顺手改命令、改参数、跳过步骤。
3. **必须有沙箱证据。** 要求沙箱证据——也就是你得能指出失败**确实**来自沙箱阻断，而不是命令本身坏了。

以及那条不容协商的红线：**绝不以此绕过测试失败或产品沙箱**。测试红了就是红了，提权不会让它变绿，也不该被用来让它看起来变绿；产品沙箱（第 2.3 节那套 `ctx.sandbox` 的围栏）是安全边界，对受限策略静默放行在任何情况下都非法（见 [`docs/subsystems/sandbox.md`](../docs/subsystems/sandbox.md#provider-and-fail-closed-errors)）。

一个实践上的判据：如果你发现自己在想"提权之后这个测试大概就能过了"，那就**不是**这条规则覆盖的情形。

## 7. 延伸阅读

- [`docs/postmortem/README.md`](../docs/postmortem/README.md) — 什么样的缺陷才值得写一篇事故记录，以及 Executive summary 的写法要求
- [`docs/defensive-patterns.md`](../docs/defensive-patterns.md) — 生命周期 / 并发 / 子进程 / teardown 的防御性模式全文
- [`docs/architecture.md`](../docs/architecture.md) — profile / bundle / patch 的组合模型，`--dump-config` 的出处
- [`docs/subsystems/README.md`](../docs/subsystems/README.md) — 各子系统页的总入口
- [`docs/testing.md`](../docs/testing.md) — 测试层的对应规则（真实入口路径、覆盖率不等于行为覆盖）
- [`apps/cli/reference/README.md`](../apps/cli/reference/README.md) — CLI 精确行为：层序、flag、关停、部署默认值、源码执行
- [`docs/subsystems/sandbox.md`](../docs/subsystems/sandbox.md)、[`docs/subsystems/approval.md`](../docs/subsystems/approval.md) — 沙箱围栏与审批闸门的权威定义
- [质量门禁导航](quality_gates.md) — **门禁报错**的反查（与本文互补，不重叠）
- [测试指南](testing_guide.md) — 测试分层与本地选择策略
- [架构总览](architecture_overview.md)、[能力接缝](capability_seams.md) — 定位"这个故障属于哪个子系统"时的上层地图
- [会话与事件](session_and_events.md) — 读会话日志前的事件词汇入门
- [插件开发指南](plugin_development_guide.md) — 事故 0001 / 0002 涉及的插件导出与组合规则的中文导航
- [CLI 与 Web 应用](apps_cli_and_web.md) — 事故 0003 涉及的 Web 启动路径导航
- [安全与沙箱](security_and_sandbox.md) — 第 2.3 节与第 6 节涉及的沙箱不可用、Landlock ABI、审批策略的完整说明
- [模型配置与 LLM 适配器](model_configuration.md) — 第 2.4 节的流式失败、`LlmFailure` 字段与路由切换
- [SDK 与外部协议](sdk_and_protocols.md) — 第 2.6 / 5.3 节涉及的 ACP 与 JSON-RPC 入口
- [发布与分发](deployment_guide.md) — 第 2.6 节的部署默认值
- [Token 成本与上下文管理](cost_optimization.md) — 第 5.4 节压缩与溢出相关故障的可调项
- [`AI_Coding_Context.md`](AI_Coding_Context.md) — dev_docs 总入口
