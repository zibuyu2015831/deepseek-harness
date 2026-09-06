# 上游文档缺陷记录（fork 内自用，暂不对外）

> **本文件是 fork 内的工作稿，不是上游仓库内容。** 它记录在生成 `dev_docs/` 中文导航层过程中，附带发现的**上游文档与代码实际不符**之处。
>
> **定位（2026-09-07 更新）**：核实后确认**上游不接受任何形式的外部代码贡献**（详见下节"投递可行性"）。因此本文件的用途改为——
>
> - ✅ **fork 内自用**：这 13 条是读上游文档时的**已知陷阱清单**。踩到其中任何一条，先查这里，别浪费时间怀疑自己。
> - ✅ **可在本 fork 内直接修掉**这些文档缺陷，不需要任何人批准（但会增加与 upstream 的合并冲突面，见"在 fork 内修复"一节）。
> - ⏸️ **暂不对外投递**。用户已决定暂缓。下方"若将来决定对外"一节保留了完整的投递材料，随时可用。

## 元信息

| 项 | 值 |
| --- | --- |
| 记录日期 | 2026-09-06（同日经独立对抗性复核修订；同日第二轮逐条回源复查） |
| 复查状态 | ✅ 第二轮逐条复查完毕（2026-09-06）；投递可行性核实完毕（2026-09-07） |
| 对外状态 | ⏸️ **暂不对外**（用户决定，2026-09-07）——上游不收外部 PR，Issues 已关闭 |
| 基准提交 | `cfde65f`（`zibuyu` 分支，merge `master`） |
| 上游基准 | `d347e70` — `Merge pull request #3554 from deepseek-harness/release/dsh-0.1.3-alpha.1`（2026-09-04，复查时 `git ls-remote upstream master` 仍为此值） |
| 上游远端 | `git@github.com:deepseek-ai/deepseek-harness.git` |
| 条目数 | 7 条主条目（U1–U7）+ 6 条附带发现（S1–S6） |
| 复核结论 | 13 条**全部成立**；U5 需收窄；**A/B 分组曾经错误已重新归并**；第二轮另修正 3 处本文件自身的记录错误（见下） |

## 投递可行性（2026-09-07 核实）

**结论：上游明文拒绝外部 PR，且 Issues 已整体关闭。唯一对外渠道是 Discussions，而官方在其中基本不公开回复。**

上游 `CONTRIBUTING.md`（仓库自带，双语）：

> DeepSeek Harness is still at an early stage and under active development. **We are sorry that we cannot accept external pull requests at the moment.**

中文版 `CONTRIBUTING.zh.md` 同义：「很抱歉，我们目前无法接受外部 PR」。

三条互相独立的机制性佐证：

| 检查 | 结果 | 含义 |
| --- | --- | --- |
| `gh repo view --json hasIssuesEnabled` | **`false`** | Issues 标签页整个关闭；`.github/ISSUE_TEMPLATE/config.yml` 另有 `blank_issues_enabled: false` |
| `gh api repos/deepseek-ai/deepseek-harness/pulls` | **HTTP 404** | PR 入口在该仓库上不可达 |
| `gh pr list --state all --limit 300` | **0 条** | 不是"外部 PR 被拒"，而是该仓库从无 PR 记录 |

**该仓库是单向发布镜像。** `git log --merges` 显示全部 merge commit 来自 `deepseek-harness/…` 分支（另一个 GitHub 组织），而 `gh api orgs/deepseek-harness` 显示该组织 `public_repos = 0`——真实开发在私有仓库，代码单向推送到 `deepseek-ai/deepseek-harness`。仓库里那份中文 `pull_request_template.md` 与 `.github/ISSUE_TEMPLATE/` 是内部仓库的产物被一并镜像出来的，在公开侧点不开。

**Discussions 的真实生态**：`CONTRIBUTING.md` 指定用 GitHub Discussions 报告问题（不是 Issues）。已启用，6 个分类（Announcements / General / Ideas / Polls / Q&A / Show Your Plugins!），**无 Bug 专用分类**——社区实际做法是发在 General 并加 `[Bug]` 前缀，中英文均可。但截至 2026-09-07，共 5,707 个 discussion；抽查最近 100 帖的全部 160 条评论，`authorAssociation` **全部为 `NONE`**，即无一条官方成员的公开回复。这与官方自述一致：

> We are a very small team and may not be able to reply to every post, **but we monitor them and consider them when allocating resources.**

即：发帖会被看到、会影响排期，但不要期待回复，也不存在"被合入"这种确认。

**第二轮复查修订**（2026-09-06，逐条回源核对全部 `file:line` 与 `git blame`）：13 条结论全部维持，但本文件自身有三处记录错误已改正——

1. **簇① 的根因句原先写反了**：原文称 `a2d0f7f411` "没有触碰 `AGENTS.md` 的目录树"。实测该提交**恰恰改了这棵树**（同一 block 内 `bash/`→`shell/`、`pty/`→`terminal/`、`compact/`→`compaction/`，并在 `self-modification/` 下方三行插入 `identity/`）。改正后的事实**对 issue 更有利**，见簇①。
2. **U1 的 private 统计口径错误**：原文称"全仓 255 个 workspace 清单中带 `private: true` 的只有 9 个"。255 是 `packages/*/*` 的数量，不是全仓；全仓 273 份受管清单中有 **13** 份 `private: true`。已改为限定口径。
3. **U6 的源码行号偏一**：`baseURL` 三元回退在 `src/index.ts:378-380`，`:377` 是 `apiKeyEnv`。
4. 附带补全：总览表与提交策略表原先漏列正文已有的 S6；U4 补上与 U7 对称的"写的时候是对的"时间线注记；一键复核脚本补 `--exclude=UPSTREAM_DOC_ISSUES.md`（本文件自身会命中两条 grep）。

**发现方式**：`dev_docs` 遵循 one-home-per-fact 原则，每条引用都必须回源码核实。这些缺陷是那次交叉审计的副产品，不是专门排查出来的——意味着**同类问题可能还有，本清单不完备**。

**本轮修订说明**（重要，影响提交策略）：初版把 U2/U3/U7 归为"2026-08-11 重命名台账的同一根因"。**该归因经 `git blame` 核实为错误**：台账是一份**持续追加**的活台账，`:260`/`:261` 两行由 `869b5c7bf23` 于 2026-08-11 写入，而 `:279` 行由 `4125514a088` 于 **2026-08-24** 写入，相隔十三天，实施提交也完全不同。据此重新归并为两个真正同源的簇（见下）。初版建议的 issue 标题 `…still reference pre-2026-08-11 paths` 在两个半句上都不成立，**不要使用**。

**门禁说明**：本文件位于仓库根目录，但不在任何门禁作用域内，可安全存在。依据：

- `scripts/verify-md-links.ts:19`、`verify-mermaid.ts:18`、`verify-md-wrap.ts:18` 的 `PATTERNS` 均为显式白名单，根目录仅列 `README.md` / `README.zh.md` / `AGENTS.md`。
- 双语配对语料由 `scripts/translation-pairing.ts:185` 的 `isTranslationScopeFile` 判定，根目录白名单是 `ROOT_PAIRED_DOCUMENT_ARTIFACT = /^(?:brand_guidelines|contributing|safety)…/`（**第 133 行**；第 132 行是 `README_ARTIFACT`），不含本文件名。

---

## 总览

| # | 簇 | 位置 | 一句话 | 证据 | 影响 | 复查 |
| --- | --- | --- | --- | --- | --- | --- |
| U1 | 独立 | `AGENTS.md:103` + `vendor/README.md:34` | 称 vendored 包 `private: true`，实际全部公开发布；**上游自己的已实施 Note 记录了该约定失效，但从未执行** | E4 | 高 | ⬜ |
| U2 | ① 命名契约 | `AGENTS.md:38` | 目录树列出已改名的 `self-modification/` | E4 | 中 | ⬜ |
| U3 | ① 命名契约 | `AGENTS.md:49` | 目录树列出已改名的 `support/` | E4 | 中 | ⬜ |
| U4 | ② examples 退役 | `docs/testing.md:40` | 要求维护一个已不存在的测试路径 | E4 | 中 | ⬜ |
| U7 | ② examples 退役 | `BENCHMARK.md:3` | 指向已迁移的 `jsonrpc-agent` 示例 | E4 | 中 | ⬜ |
| U5 | 独立 | `packages/core/session/src/types.ts:325` | JSDoc 写错函数名（仅此，见收窄说明） | E4 | 中 | ⬜ |
| U6 | 独立 | `packages/llm/llm-deepseek/README.md:54` | 优先级措辞与同表 `:57` 的"win"用法不一致 | E4 | 低 | ⬜ |
| S1 | 附带 | `AGENTS.md` 目录树 | 树只列 35 组，实际 50 组，缺 17 个 | E4 | 中 | ⬜ |
| S2 | 附带 | `vendor/README.md:5` vs `:34` | 同一文件自相矛盾 | E4 | 中 | ⬜ |
| S3 | 附带 | `vendor/README.md:11` | 泄漏维护者本机路径 `~/repos/cordis-workspace` | E4 | 低 | ⬜ |
| S4 | 附带 | `packages/README.md` | 分组总表漏登 `packages/mcp/`（49 行 vs 50 组） | E4 | 低 | ⬜ |
| S5 | 附带 | `docs/development.md:62` | "Six packages split Host and Client tsconfigs"，实际 8 个 | E4 | 低 | ⬜ |
| S6 | 附带 | `AGENTS.md:127` vs `docs/testing.md:54` | 快照触发条件两处措辞不一致（少 protocol-visible） | E4 | 低 | ⬜ |

**分组**（已按修订后的根因重排）。⏸️ 下表原为"提 issue 的拆分方案"，因上游不收外部贡献而**暂不执行**；保留是因为这个分组同样是**理解这些缺陷成因的正确方式**，也是将来若改用 Discussions 投递时的现成拆分。

| 组 | 内容 | 理由 |
| --- | --- | --- |
| 1 | **U1 + S2** | 同一主题（vendored 包的发布性质），S2 是 U1 的自证材料 |
| 2 | **U2 + U3 + S1** | 同一处 `AGENTS.md` 目录树；只修两行会留下一棵仍然误导的树 |
| 3 | **U4 + U7** | 同一次 examples 退役漏掉的两个文档调用点 |
| 4 | **U5** | 独立文件、独立子系统 |
| 5 | **U6** | 独立包、独立作者，**不要并进 U1** |
| 6 | S3 / S4 / S5 / S6 | 琐碎项，彼此无关联，合并成一组即可 |

---

## 簇 ①：命名契约提交漏改 `AGENTS.md` 目录树

**共同根因**：提交 `a2d0f7f411`（2026-08-13，*"refactor: apply repository naming contract"*）把 `packages/self-modification/` 改名为 `packages/extensions/`、`packages/support/` 改名为 `packages/test-support/`，**并且同一提交就在改这棵目录树**——它在同一个 block 内改了四项：

```diff
-  bash/        bash capability: Service Definition + local/pwsh providers + shell Consumers
+  shell/        bash capability: Service Definition + local/pwsh providers + shell Consumers
-  pty/         persistent PTY capability
+  terminal/         persistent sessions
-  compact/     compaction capability + basic provider
+  compaction/     compaction capability + basic provider
@@ -35,6 +35,7 @@
   self-modification/  the agent inspects/mounts its own plugins
   hooks/       Claude Code/Codex hook bridges + wire-protocol library
   session/     durable session data: persistence, projection, titles, telemetry
+  identity/    anonymous identity
```

最后那个 hunk 的 diff 上下文第一行就是 `self-modification/`——新增的 `identity/` 落在它下方三行处。**所以这不是"忘了这个文件"，而是在同一次树内改名里漏掉了两项。** 两行的 `git blame` 分别停在 `36cf4f32f48`（2026-07-30）与 `c9dc0977491`（2026-07-26），均早于该提交，证实它们确实一直没被跟着改。

> 这条对 issue 措辞很关键：不要写成"文档没跟上重命名"（听起来像作者不知道有这个文件），而应写成"同一次树内改名改了四项、漏了两项"——后者是可核对的事实，也解释了为什么这两行至今仍在。

顺带确认 **`AGENTS.md` 的树是路径断言而非概念标签**：`bash/`→`shell/`、`pty/`→`terminal/` 这些改动本身就是跟着磁盘路径走的。

改名台账 `.agents/notes/implemented/architecture/2026-08-11-repository-naming-contract-and-rename-ledger.md` 记录了这两次重命名（`:260`、`:261`），可作佐证——但注意它是**活台账**，不要在 issue 里把它当成"一次性决策"。

### U2 — `AGENTS.md:38` 列出不存在的 `self-modification/`

**上游原文**（位于 `packages/` 目录树内）：

```
  self-modification/  the agent inspects/mounts its own plugins
```

**实际情况**：`packages/self-modification/` 不存在。内容位于 `packages/extensions/`，含 `tool-cordis`、`ui-cordis`、`cordis-host-runner`、`cordis-client-runner` 四个包。

**佐证**：台账 `:260` 给出了改名理由——"`extensions` states the stable package role without asserting that the agent modifies itself"。

**这是路径断言而非概念标签**：树的表头（`AGENTS.md:15`）写的是 `packages/    @deepseek-ai/dsh-<pkg> workspaces at packages/<group>/<pkg>/`，所以每一行都是路径。

**建议改法**：

```
  extensions/  the agent inspects/mounts its own plugins
```

### U3 — `AGENTS.md:49` 列出不存在的 `support/`

**上游原文**：

```
  support/     dev/test infrastructure
```

**实际情况**：`packages/support/` 不存在。实际为 `packages/test-support/`，含 `agent-loop-testkit`、`client-runtime`、`llm-mock-server`、`llm-replay`、`loader-smoke`、`session-snapshot`。

**佐证**：台账 `:261` 的理由是 "The group is test-only infrastructure. Its path must say so."——说明 `support/` 这个名字是被刻意废弃的。另外 `packages/README.md:77` **已经**正确列出 `test-support/`，即 `AGENTS.md` 是唯一的滞后处。

**建议改法**：

```
  test-support/ dev/test infrastructure
```

### S1 — 同一棵树还缺 17 个能力组（建议一并修）

**实际情况**：树只列 35 项，磁盘实测 50 组。扣掉上述两个失效条目，有效条目 33，**缺 17 个**：

```
attachment, client, code-runtime, extensions, feedback, goal, host, jobs, mcp,
runtime-diagnostics, sandbox, schedule, session-query, spill, storage,
test-support, workspace
```

树上没有任何 "selected" / "partial" 之类的限定语，读者会当成完整清单。而 `packages/README.md` 的分组表规则写明"新增组要更新它自己的 README **和这张表**"——该规则显然没有覆盖 `AGENTS.md`。

**建议**：并入 U2/U3 的 issue。只改两行会留下一棵仍然缺 34% 的树；要么补全，要么给树加一句"选列，完整清单见 `packages/README.md`"。

---

## 簇 ②：examples 退役漏改两个文档调用点

**共同根因**：2026-08 下旬的 examples 退役提交簇。注意仓库当时有**两棵各自独立的示例树**，两条缺陷各挂一棵，写 issue 时不要混为一谈：

| 树 | 相关条目 | 退役过程 |
| --- | --- | --- |
| 根目录 `examples/` | U7（`jsonrpc-agent`） | `f3402eff58`（08-23，*"relocate JSON-RPC example and runtime without edits"*，`examples/jsonrpc-agent/` → `examples/python-sdk-agent/`）→ `4125514a08`（08-24，*"refactor(repo): retire top-level examples"*，整棵移除） |
| `packages/examples/` | U4（`built-bin.e2e.ts`） | `d8dbb8235c`（08-23）删掉最后一个 `packages/examples/*/tests/built-bin.e2e.ts` → `244de7c18a`（08-26，*"refactor(bundle): remove the agent spine demo"*）清空整个目录 |

两棵树在同一周内退役，所以合并成一个 issue 是合理的；但**"top-level examples" 这个提交标题只覆盖 U7 那棵**，不要拿它去解释 U4。

### U4 — `docs/testing.md:40` 要求维护一个已不存在的测试路径

**上游原文**：

> Keep the built-artifact smokes green (`packages/examples/*/tests/built-bin.e2e.ts`, `packages/code-runtime/code-runtime-worker-thread/tests/built-lib.e2e.ts`), and assert a genuinely-missing config exits non-zero.

**实际情况**：`packages/examples/` 不存在。全仓唯一的 `built-bin.e2e.ts` 位于 **`apps/cli/tests/built-bin.e2e.ts`**。

**范围收敛**：同一句中的另外两个路径**仍然有效**，无需改动：

- `packages/code-runtime/code-runtime-worker-thread/tests/built-lib.e2e.ts` ✅ 存在
- `packages/sdk/server/tests/built-scope-carrier.e2e.ts`（同段前文）✅ 存在

所以这是一个精确的单路径替换。

**时间线注记（与 U7 同理，措辞上要给作者留余地）**：`docs/testing.md:40` 由 `a2d0f7f4112` 于 **2026-08-13** 写入，而那时 `packages/examples/acp-demo/tests/built-bin.e2e.ts` **还存在**——它是十天后 `d8dbb8235c`（08-23）删掉的最后一个同名文件，整个 `packages/examples/` 目录再由 `244de7c18a`（08-26）清空。所以这一条同样是"写的时候是对的，被后续迁移带失效的"。

`git log --diff-filter=D --name-only -- '*built-bin.e2e.ts'` 可完整重现这个通配符的消亡过程：`stdio-demo`（`4cadf096ce`，07-20）、`tui-demo`（`870fb1cafa`，07-25）、`cli-demo`（`dc57f7d854`，08-08）、`acp-demo`（`d8dbb8235c`，08-23）。

**建议改法**：`packages/examples/*/tests/built-bin.e2e.ts` → `apps/cli/tests/built-bin.e2e.ts`。

### U7 — `BENCHMARK.md:3` 指向已迁移的示例

**上游原文**（全文 3 行）：

```markdown
# Running benchmarks

Follow [Get started with the Python SDK](docs/user/guide/python-sdk.md) to install the SDK and run the `jsonrpc-agent` minimal variant. Use separate workspaces and session IDs for independent benchmark tasks.
```

**实际情况**：`jsonrpc-agent` 在仓库内**已无任何路径或包**（`find -name "*jsonrpc-agent*"` 零命中）。除本文件外，文本引用只出现在 Agent Note 里（这些是冻结的历史记录，按归档策略不应修改），因此 `BENCHMARK.md` 是唯一漏网的活引用。

**时间线注记**：`BENCHMARK.md:3` 由 `04fe477e7e6` 于 **2026-08-12** 写入——它**写的时候是对的**，是十天后的迁移让它失效的。所以不要用"引用了过时路径"这类暗示作者疏忽的措辞。

**关键点**：`BENCHMARK.md` 自己链接的 `docs/user/guide/python-sdk.md` **已经是更新后的正确内容**——该文档用的是 `python/sdk/examples/minimal.py`（`:62`、`:72`）和 profile 名 `sdk-minimal`（`:96`、`:106`、`:116`）。修复文本现成可用。

**建议改法**：将 "run the `jsonrpc-agent` minimal variant" 改为 "run the shipped `sdk-minimal` profile (`python/sdk/examples/minimal.py`)"。

---

## U1 — vendored 包被描述为 `private: true`，实际全部公开发布 ⚠️ 最高优先级

**上游原文（两处）**：

`AGENTS.md:103`：

> Every npm package is `@deepseek-ai/dsh-<name>`; vendored packages are rescoped ([mapping](docs/rescope.md)) and `private: true`.

`vendor/README.md:34`：

> **All `package.json` files**: regenerated — added `private: true`, added precise `files` entries …

**决定性证据 —— 上游自己已经记录了这条约定失效，但没执行。**
`.agents/notes/implemented/process/2026-08-10-npm-release-sequences.md:133`，位于一张标题为 "Repository changes this carried" 的表中：

> \| root `AGENTS.md` \| the convention that vendored packages are `private: true` no longer holds \|

这是一份 **implemented**（非 proposed、非 archived）的决策记录。`git blame` 显示 `AGENTS.md:103` 最后一次改动正是 2026-08-10、同一作者——该次改动编辑了这一行，却把 `private: true` 留在了句尾。**这不是可争辩的解读分歧，而是一项有案可查、但未执行完的后续项。** issue 里应当以此开篇。

**实际情况**：9 个 `vendor/*/package.json` **全部没有 `private` 字段**，且全部带 `publishConfig: { access: "public" }`：

| 包 | name | private | publishConfig |
| --- | --- | --- | --- |
| `vendor/cordis` | `@deepseek-ai/cordis` | 无 | `access: public` |
| `vendor/cosmokit` | `@deepseek-ai/cosmokit` | 无 | `access: public` |
| `vendor/group` | `@deepseek-ai/cordis-plugin-group` | 无 | `access: public` |
| `vendor/hmr` | `@deepseek-ai/cordis-plugin-hmr` | 无 | `access: public` |
| `vendor/include` | `@deepseek-ai/cordis-plugin-include` | 无 | `access: public` |
| `vendor/loader` | `@deepseek-ai/cordis-plugin-loader` | 无 | `access: public` |
| `vendor/logger-console` | `@deepseek-ai/cordis-plugin-logger-console` | 无 | `access: public` |
| `vendor/schemastery` | `@deepseek-ai/schemastery` | 无 | `access: public` |
| `vendor/timer` | `@deepseek-ai/cordis-plugin-timer` | 无 | `access: public` |

**对照口径（引用这个数字时按此写，不要笼统说"全仓"）**：`packages/*/*` 下 255 个包里，带 `private: true` 的只有 `packages/experimental/*` 那 9 个；`vendor/*` 九个一个也没有。全仓 273 份受管 `package.json` 里共 13 份 `private: true` = 上述 9 个 + `native/landlock-run`（原生构建工作区根）+ `python/sdk-runtime`（单文件可执行的部署根）+ `website` + 仓库根 `package.json`——**都不是 vendored 包**。

**旁证一：vendor 是一等发布家族。** `scripts/release/families.ts:371-375` 定义了 `class VendorFamily`（`patterns = ['vendor/*/package.json']`、`tagPrefix = 'vendor-'`），`.github/workflows/release-vendor-publish.yml:114` 实际执行 `pnpm run release:publish --family vendor --from dist/npm-vendor`。

**旁证二：若文档成立则发布跑不起来。** `scripts/release/verify.ts:44-49` 的 `verifyPublishable` 会拒绝任何 `private: true` 的发布成员：

```typescript
function verifyPublishable(members: readonly ReleaseMember[]): void {
  const priv = members.filter(member => member.manifest.private === true)
  if (priv.length > 0) {
    throw new Error(`publishing requires removing "private": true from:\n${priv.map(member => member.directory).join('\n')}`)
  }
}
```

**影响**：高。它会让贡献者误以为 vendored 包不进入发布流程，从而对这些包的版本、`files` 字段、破坏性改动采取错误的谨慎级别。

**建议改法**：`AGENTS.md:103` 删除 "and `private: true`"，可改为 "and published with `publishConfig.access: public`"；`vendor/README.md:34` 将 "added `private: true`" 改为 "added `publishConfig.access: public`"。

> **注意：初版的"附带发现"已删除且不要提交。** 初版据 `vendor/cordis/package.json` 是 `4.0.2`、而 manifest 记 `4.0.0-rc.7`，推断该清单"整体已陈旧"。**该推断错误**：这两个数字按设计就不相等。`vendor/README.md:5` 说明 manifest 记的是**上游快照**；`scripts/release/bump.ts:153-161` 的 JSDoc 明写 "a vendor re-sync restores upstream's version, which is lower than the release version this repository already reserved"，即 `package.json` 记的是本仓库自己的发布线。九个包全都因此不等。把这条错误主张放进一个正确的 issue 里，会连累整条的可信度。

### S2 — `vendor/README.md` 同一文件内自相矛盾（建议并入 U1）

`vendor/README.md:5` 已经写了 vendored 层是要发布的：

> …every harness package declares `cordis` as a peer dependency, so **publishing the harness publishes this framework layer too**, and a publication under the upstream names would squat them on the registry.

同一文件 `:34` 却说这些清单被加上了 `private: true`。把这两行并列贴进 issue，维护者不需要跑任何命令就能确认。

---

## U5 — `tool/result` 的 JSDoc 写错了函数名（已收窄）

**根因位置**：`packages/core/session/src/types.ts:325`（`SessionEventMap` 中 `tool/result` 事件的 JSDoc）。

**上游原文**：

> `meta` … MUST be JSON-serializable: `Session.append` runtime-validates all event data with `isJsonValue`, so a non-serializable `meta` is rejected at the source …

**实际情况**：`Session.append` 调用的是 **`snapshotJsonValue`**，不是 `isJsonValue`：

```typescript
    const dataSnapshot = snapshotJsonValue(data)
```

—— `packages/core/session/src/index.ts:709`（另见 `:713`；`:12` 的 import 只有 `deepEqualJson, deepFreeze, snapshotJsonValue`，**不含 `isJsonValue`**）。

`isJsonValue` 是真实存在的兄弟函数（`packages/util/values/src/index.ts:181`），README 明确区分二者：

> Use `isJsonValue()` for a predicate and `snapshotJsonValue()` when the caller also needs a detached copy.

—— `packages/util/values/README.md:29`

**传播范围**：该 JSDoc 被四处转载，均为衍生内容——修 JSDoc 后需重新生成：

- `docs/subsystems/session.md:92`
- `docs/subsystems/session.zh.md:92`
- `docs/persistence-catalog.md:932`
- `docs/persistence-catalog.zh.md:934`

（后两者带 `Generated by scripts/gen-persistence-catalog.ts` 头，必须重跑生成器。）

> **收窄说明：初版的第二项指控已删除，不要提交。** 初版称该 JSDoc "隐藏了 append 会复制数据这一契约"，调用方可能误以为改原对象会反映到日志。**这一点不成立**：该契约在它自己的归属地有完整文档——`packages/core/session/src/index.ts:691-693`（`append` 方法自身的 JSDoc）写着 "One iterative pass reads, validates, and copies each nested value once, so a stateful getter cannot supply one value to validation and another to storage."。按仓库自己的 one-home-per-fact 规则，`tool/result` 的事件 JSDoc **本就不该**复述它。维护者一查就会发现这半句是错的，从而连带削弱正确的那半句。
>
> 另外，`isJsonValue` 与 `snapshotJsonValue` 是同一个 `walkJsonValue` 的两层薄封装（`packages/util/values/src/index.ts:172-183`），所以 JSDoc 描述的**校验规则**是准确的——顺着错名字找过去的读者仍会落到正确语义上。这进一步说明它是纯粹的名称笔误，按 typo 提即可。

**建议改法**：改 `packages/core/session/src/types.ts:325` 一处，`isJsonValue` → `snapshotJsonValue`，然后重跑生成器同步四处衍生文档。**标题前缀用 `docs(session):` 而非 `fix(session):`**——这是纯注释改动，无行为影响。

---

## U6 — `llm-deepseek` README 的 `baseURL` 优先级说明与同表用法冲突

**上游原文**：

`packages/llm/llm-deepseek/README.md:54`（配置表）：

> \| `baseURL` \| `https://api.deepseek.com` \| Endpoint base; `$DEEPSEEK_BASE_URL` wins when set \|

**实际情况**：源码为**显式配置 > 环境变量 > 默认值**：

```typescript
    baseURL: config.baseURL
      ?? environment?.get(BASE_URL_ENV)?.value
      ?? PUBLIC_BASE_URL,
```

—— `packages/llm/llm-deepseek/src/index.ts:378-380`（`:377` 是同一 `return` 对象里的 `apiKeyEnv`）

**为什么不能辩解为"胜过本行展示的默认值"**：这是最强的辩护路径（该表第二列表头正是 Default），但**同一张表两行之后就否掉了它**。`README.md:57`：

> \| `maxTokens` \| `256,000` \| Per-request output cap; a model's own cap and **explicit request values win** \|

这里的 "win" 明确指**压过已配置的字段值**（由 `Config.maxTokens` 的 JSDoc `src/index.ts:134` 逐字印证）。也就是说在这张表自己的用语里，"win" = 覆盖配置值。按该用语读，`:54` 断言的是环境变量覆盖显式配置——与源码相反。

**其余两处都是对的**，可作为修复参照：`README.md:40` 的示例注释（`# optional; $DEEPSEEK_BASE_URL then this default`）与 `src/index.ts:128` 的 `Config.baseURL` JSDoc。

**建议改法**：`:54` 改为 `Endpoint base; used when set, else $DEEPSEEK_BASE_URL, else this default`。（初版建议的 "explicit config wins, then …" 在一个**本身就是该配置字段**的表行里读起来别扭。）

**不要并入 U1 的 PR**：不同包、不同子系统、不同作者（`git blame`：`Magolor` 2026-08-25，U1 是另一位）。

---

## 其余附带发现

### S3 — `vendor/README.md:11` 泄漏维护者本机路径

> Upstream workspace: `cordis-workspace` (local checkout: `~/repos/cordis-workspace`).

对外文档里的维护者本机目录布局，对读者无用。建议删括号或改为通用表述。

### S4 — `packages/README.md` 的分组总表漏登 `packages/mcp/`

`## Package groups` 表只有 49 行，磁盘 50 组，缺的是 `mcp/`（`packages/mcp/README.md` frontmatter 为 `kind: "package-group"`，下含 `mcp-client`）。该表自己的规则写着"新增组要更新它自己的 README 和这张表"。

### S5 — `docs/development.md:62` 的拆分包数字已过期

> Six packages split Host and Client tsconfigs: `api/remotes`, `api/gateway`, `api/session-controller`, `api/workspace-controller`, `client/connection`, and `session-query/session-log-export`.

实测 **8 个**，另有 `packages/client/file-upload` 与 `packages/experimental/inspector`（两者的包根 `tsconfig.json` 均为 `{"files": [], "references": [host, client]}`）。

有意思的是同一段接着写"门禁靠两个 leaf config 的存在性发现拆分包，所以新拆的包自动进入门禁"——**机制是自动的，散文里的数字不是**。建议直接删掉数字与枚举，只留机制描述。

### S6 — 快照触发条件在两处措辞不一致

- `AGENTS.md:127`：`Every non-trivial model- or product-user-visible change …`
- `docs/testing.md:54`：`Every non-trivial model-, protocol-, or human-visible change …`

`docs/testing.md` 多了 **protocol-visible**。按 one-home-per-fact，`docs/testing.md` 是测试策略的归属地，`AGENTS.md` 那句应当对齐（或改为纯链接）。

---

## 一键复核

在仓库根目录执行，用于人工复查时快速确认每条仍然成立。**注意脚本必须排除本文件**——它自己引用了这些字符串，否则两条 grep 会自命中。

```bash
# lib/ 与 dist/ 必须排除：在已构建的工作区里，.js.map 会把这些字符串复制进来（实测输出会涨到 1.3 MB）。
X='--exclude=UPSTREAM_DOC_ISSUES.md --exclude-dir=node_modules --exclude-dir=.git
   --exclude-dir=dev_docs --exclude-dir=lib --exclude-dir=dist'

echo '--- U1: vendored 包的 private / publishConfig ---'
python3 -c "import json,glob;[print(f, json.load(open(f)).get('private'), json.load(open(f)).get('publishConfig')) for f in sorted(glob.glob('vendor/*/package.json'))]"
grep -n 'private: true' AGENTS.md vendor/README.md
grep -n 'no longer holds' .agents/notes/implemented/process/2026-08-10-npm-release-sequences.md
sed -n '44,49p' scripts/release/verify.ts
echo "vendor 里 private:true 的数量（期望 0）："
python3 -c "import json,glob;print(sum(json.load(open(f)).get('private') is True for f in glob.glob('vendor/*/package.json')))"

echo '--- S2: vendor/README.md 自相矛盾 ---'
sed -n '5p;34p' vendor/README.md

echo '--- U2 / U3 / S1: AGENTS.md 目录树 vs 实际 ---'
grep -n 'self-modification/\|^  support/' AGENTS.md
ls -d packages/self-modification packages/support 2>&1   # 期望：均不存在
ls -d packages/extensions packages/test-support           # 期望：均存在
echo "树列出组数（期望 35，含两个失效项）vs 实际组数（期望 50）："
awk '/^packages\/ /,/^python\//' AGENTS.md | grep -cE '^  [a-z]'
ls -d packages/*/ | wc -l
echo "缺失的组（期望 17 个；注意字符类要含数字，否则会误报 e2b）："
awk '/^packages\/ /,/^python\//' AGENTS.md | grep -oE '^  [a-z0-9-]+/' | tr -d ' /' | sort > /tmp/listed
ls -d packages/*/ | sed 's|packages/||;s|/||' | sort > /tmp/actual
comm -13 /tmp/listed /tmp/actual | tr '\n' ' '; echo
echo "同一提交是否改过这棵树（期望：能看到 bash→shell / pty→terminal / +identity）："
git show a2d0f7f411 -- AGENTS.md | grep -E '^[-+]  (bash|shell|pty|terminal|compact|identity)'

echo '--- U4 / U7: examples 退役的两个漏网引用 ---'
grep -n 'packages/examples' docs/testing.md
find . -path ./node_modules -prune -o -name 'built-bin.e2e.ts' -print  # 期望：仅 apps/cli
find . -path ./node_modules -prune -o -name '*jsonrpc-agent*' -print   # 期望：零命中
grep -rl 'jsonrpc-agent' $X --exclude-dir=.agents .                    # 期望：仅 BENCHMARK.md
git log --diff-filter=D --name-only --format='%h %ad %s' --date=short -- '*built-bin.e2e.ts'

echo '--- U5: JSDoc 函数名 vs 源码调用 ---'
grep -rn 'runtime-validates all event data' $X .   # 期望：源 types.ts:325 + 衍生 4 篇，共 5 命中
grep -n 'snapshotJsonValue\|isJsonValue' packages/core/session/src/index.ts | head

echo '--- U6: baseURL 优先级与同表 win 用法 ---'
sed -n '40p;54p;57p' packages/llm/llm-deepseek/README.md
sed -n '377,380p' packages/llm/llm-deepseek/src/index.ts   # :377 是 apiKeyEnv，:378-380 才是 baseURL
sed -n '128p;134p' packages/llm/llm-deepseek/src/index.ts  # 两处正确的 JSDoc 参照

echo '--- S4 / S5 / S6: 附带项 ---'
grep -c 'mcp' packages/README.md                                       # 期望：0
grep -n 'Six packages split' docs/development.md
for f in packages/*/*/tsconfig.host.json; do d=$(dirname "$f"); \
  [ -f "$d/tsconfig.client.json" ] && echo "$d"; done | wc -l          # 期望：8
grep -n 'product-user-visible' AGENTS.md                               # 期望：:127
grep -n 'protocol-, or human-visible' docs/testing.md                  # 期望：:54
```

---

## 在 fork 内修复（当前可做的事）

上游 MIT 许可，fork 内改这些文档不需要任何人批准。但**不建议无差别地全改**，理由是合并成本：

| 条目 | fork 内是否值得改 | 说明 |
| --- | --- | --- |
| U1 / S2 | ⭕ **值得** | `AGENTS.md:103` 与 `vendor/README.md:34` 会直接误导你判断 vendored 包的发布性质。改动量各一句 |
| U2 / U3 / S1 | ⭕ **值得** | `AGENTS.md` 目录树是最常被翻的导航面，两处死路径 + 缺 17 组，实际影响最大 |
| U4 / U7 / U5 / U6 / S3–S6 | ❌ **不必** | 影响面小，且 `docs/` 与 `packages/**/README.md` 属于双语配对语料——改一处就要同步改 `.zh.md` 与 `.i18n.yaml`，否则 `pnpm run doc-sync` 会红。收益不抵成本 |

**如果要改，注意两件事**：

1. **配对门禁**。`docs/**` 与 `packages/*/*/README.md` 受 `scripts/translation-pairing.ts` 管辖，`.md` / `.zh.md` / `.i18n.yaml` 必须同步。根 `AGENTS.md` 不在配对语料内（它只有单语版），所以 U1/U2/U3/S1 那几处改起来最干净。
2. **合并冲突**。每改一行上游文件，就在将来 `git merge upstream/master` 时多一个潜在冲突点。当前 `dev_docs/` 之所以零冲突，正是因为它只新增文件、不碰上游文件。**更稳的做法是完全不改上游文件，只依赖本清单**——踩到坑时来查，而不是预先改掉。

## 若将来决定对外

⏸️ 用户已决定暂不对外（2026-09-07）。以下材料保留完整，改主意时直接可用。

**渠道**：只能是 **GitHub Discussions → General 分类**，标题加 `[Docs]` 前缀（社区惯例，见"投递可行性"）。不能提 PR，也没有 Issues 可提。

**建议拆成两帖**，而不是一帖或六帖：

- **U1 单独一帖**——它有决定性证据（上游自己的 implemented note 写着这条约定 `no longer holds`）、影响最高、维护者不用跑任何命令就能确认。混在长帖里会被淹没，也拿不到独立 upvote（`CONTRIBUTING.md` 明说 upvote 影响资源分配）。
- **其余 12 条合一帖**，一键复核脚本原样附上——那是加分项，维护者可以直接跑。

**发帖前必须做的**：

- [x] **逐条回源复查全部 13 条**（2026-09-06 第二轮）：每个 `file:line`、每条 `git blame`、每个数字都在本机重新执行核对，13 条结论全部维持，本文件自身的 3 处记录错误已改正。
- [ ] 用上面的一键复核脚本，在**同步到最新 upstream/master 之后**重跑一遍——部分条目可能已被上游修复。（截至 2026-09-07 核对，`git ls-remote upstream master` 仍是 `d347e70`，与基准一致，全部条目仍然成立。）
- [ ] 确认每条的证据引用（`file:line`）在最新上游代码上仍然对得上，行号会漂移。**尤其是 U5 的 4 处衍生文档行号**（`persistence-catalog*.md` 由生成器产出，任何上游改动都会整体位移）。
- [ ] 检索现有 Discussions，避免重复——5,707 帖里可能已有人报过。用 `gh api graphql` 搜 `AGENTS.md`、`private: true`、`jsonrpc-agent` 等关键词。
- [ ] **删除本文件的元信息表**（含基准提交 `cfde65f`、`zibuyu` 分支名、fork 远端）、"第二轮复查修订"一节、"在 fork 内修复"一节与所有 `dev_docs/` 内部引用——这些都不能出现在对外内容里。
- [ ] 确认不含任何本机绝对路径或本地环境信息。（本文件唯一出现的 `~/repos/cordis-workspace` 是**上游文档里的原文**，即 S3 本身，不是本机路径。）

**建议的英文标题**（原为 issue 标题，用作 Discussion 标题同样合适）：

| 组 | 标题 |
| --- | --- |
| U1 + S2 | `[Docs] vendored packages are documented as private: true but all nine publish publicly` |
| U2 + U3 + S1 | `[Docs] AGENTS.md repository-layout tree lists two renamed groups and omits 17 others` |
| U4 + U7 | `[Docs] testing.md and BENCHMARK.md still reference paths retired in the August examples cleanup` |
| U5 | `[Docs] tool/result JSDoc names isJsonValue but Session.append calls snapshotJsonValue` |
| U6 | `[Docs] llm-deepseek baseURL row says the env var "wins", but explicit config takes precedence` |

**不要使用**初版的标题 `docs: AGENTS.md and BENCHMARK.md still reference pre-2026-08-11 paths`——两个半句都不成立（见开头的修订说明）。

## 相关文档

- 完整分析上下文：[`dev_docs/_analysis/project_analysis_report.md`](dev_docs/_analysis/project_analysis_report.md)（"生成过程中发现的上游文档缺陷" 一节）
- `dev_docs` 对这些冲突的处理原则：一律**以源码为准**，并在正文显式标注冲突，而不是沉默地跟随任一方。已按此处理 U1、U2/U3、U6。
- 上游贡献政策原文：[`CONTRIBUTING.md`](CONTRIBUTING.md) / [`CONTRIBUTING.zh.md`](CONTRIBUTING.zh.md)。
