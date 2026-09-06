---
title: dev_docs 首版质量验收报告
summary: dev_docs 28 篇正式文档的首版验收结论；AICC 工具链不可用故采用降级验收，机器可验证项全部通过，语义层验收标注 NOT_RUN。
keywords: health-check | acceptance | machine-checks | not-run | accepted-issues | downgraded
scope: deepseek-harness dev_docs 首版质量验收记录
related_files: dev_docs/_analysis/generation_plan.md | dev_docs/_analysis/project_analysis_report.md | dev_docs/_analysis/generation_progress.md | UPSTREAM_DOC_ISSUES.md
dependencies: dev_docs/_analysis/generation_plan.md | dev_docs/_analysis/project_analysis_report.md
verified_at: 2026-09-06
---

# dev_docs 首版质量验收报告

> **验收方式已降级。** AICC 工具链在本机不可用（见 §2），原定的 `doc_health_checker` / `semantic_review_checker` 双实现交叉验证**无法执行**。本报告采用 `project_analysis_report.md` 建议 4 的替代方案：借用仓库自有门禁 + 人工执行方案质量检查清单 + 全部 AICC 检查如实标注 `NOT_RUN`。
>
> **本报告不包含任何未实际执行的检查结果。** 凡标注 `NOT_RUN` 的项，其状态为"未知"，不得解读为通过。

## 1. 元信息

| 项 | 值 |
| --- | --- |
| 验收日期 | 2026-09-06 |
| 验收对象 | `dev_docs/` 正式文档 28 篇，7,090 行 |
| 过程档案 | `dev_docs/_analysis/` 3 篇，2,344 行（不计入验收对象） |
| 基准提交 | `998014f`（`zibuyu` 分支） |
| 环境 | macOS (darwin 25.5.0) / Node v26.3.1 / Python 3.14.6 / pnpm workspace 273 成员 |
| 总体结论 | **有条件通过** — 机器可验证项全部通过；语义层验收因工具缺失未执行 |

## 2. AICC 检查：全部 `NOT_RUN`

```yaml
aicc_checks:
  summary_validator:        NOT_RUN
  doc_health_checker:       NOT_RUN
  semantic_review_checker:  NOT_RUN
  reason: >
    AI-Coding-Context 原为指向仓库外绝对路径的符号链接。按 project_analysis_report.md
    问题 2 的修复，该链接已加入 .gitignore，因此不随 Git 传播到本机 clone。
    `ls -ld AI-Coding-Context` 返回 No such file or directory，Python 与 JS 两套实现均不存在。
  evidence_level: E4  # 证伪
  forbidden: >
    不得复用 2026-08-18 的历史 PASS 结果——那是另一台机器、另一个仓库快照上的结论，
    与本次生成的 28 篇文档没有任何关系。
```

**这一节缺失的是什么**：语义层验收。`semantic_review_checker` 本应检查的"文档陈述是否与源码语义一致"，本次由**逐条源码核实 + 代码块逐字可追溯性验证**部分替代（见 §3.4），但两者覆盖面不同——前者是语义级，后者是字面级。**这是本次验收的已知缺口。**

## 3. 机器可验证检查（全部实跑）

```yaml
machine_checks:
  - id: repo_gates
    status: PASS
    detail: verify-translation-pairing 1135 pairs / verify-md-links 2273 files / verify-doc-budgets 8 docs
  - id: dev_docs_links
    status: PASS
    detail: 借用作用域后 2305 files，dev_docs 全部相对链接与 #fragment 锚点可解析
  - id: dev_docs_mermaid
    status: PASS
    detail: 合计 36 个 mermaid 块全部解析成功（门禁基线 20 + dev_docs 内 16）
  - id: code_block_authenticity
    status: PASS
    detail: 85 个源码块，83 个每行逐字可追溯，0 处不一致；2 个为显式标注的示例配置
  - id: hygiene
    status: PASS
    detail: 16 gates passed, 0 failed（构建后）
  - id: frontmatter_fields
    status: PASS
    detail: 28 篇全部具备 title/summary/keywords/scope/related_files/dependencies/verified_at
  - id: command_existence
    status: PASS
    detail: dev_docs 引用的全部 pnpm run 命令均存在于 package.json 的 142 个 script 中
  - id: redaction
    status: PASS
    detail: 无本地绝对路径、无密钥形态字符串、无 OTLP 端点、无模板残留
  - id: main_doc_coverage
    status: PASS
    detail: 主文档链接全部 17 篇子文档
```

### 3.1 仓库自有门禁

在含 `dev_docs/` 与 `UPSTREAM_DOC_ISSUES.md` 的工作树上运行，**未对仓库脚本做任何提交性改动**：

| 门禁 | 结果 |
| --- | --- |
| `verify-translation-pairing` | PASS — 1135 组配对一致 |
| `verify-md-links` | PASS — 2273 文件 |
| `verify-doc-budgets` | PASS — 8 篇预算文档在上限内 |
| `pnpm run hygiene` | PASS — 16 gates passed, 0 failed |

`hygiene` 首轮曾报 3 项失败（`publint`、`verify-built-package-invariants`、`verify-node-next-types`），错误均为 `missing lib/types/index.d.ts`——本 clone 从未执行过 `pnpm run build`，属于产物面缺失。执行 `pnpm run build`（exit 0）后重跑，16 项全绿。**该失败与 `dev_docs` 无关**，此处记录以免被误读为回归。

### 3.2 `dev_docs` 链接与锚点

`dev_docs` 不在任何门禁的 `PATTERNS` 作用域内，故采用**借用—还原**流程验证：临时向 `scripts/verify-md-links.ts` 的 `PATTERNS` 追加 `'dev_docs/**/*.md'` 与 `'UPSTREAM_DOC_ISSUES.md'`，运行后 `git checkout` 还原。

结果：**2305 个文件，全部相对链接与 `#fragment` 锚点可解析**（基线 2273 + dev_docs 32 + 根文档 1 = 2306；初版记为 2305 系少算一篇 dev_docs，已订正）。这覆盖了 `AI_RULES.md` 对 `AGENTS.md` 各章节锚点的引用（`#pre-stable-apis-and-released-session-data`、`#secrets--env`、`#conventions` 等），以及对 `docs/` 的全部章节锚点引用。

同法验证 mermaid：**合计 36 个块全部解析成功**——其中门禁基线 20 个、`dev_docs` 内 16 个。（注意别把 36 读成 dev_docs 的数量。）

改动未提交，`git status` 确认工作树只含 `dev_docs/` 与 `UPSTREAM_DOC_ISSUES.md`。

### 3.3 命令真实性

对 28 篇正式文档提取全部 `pnpm run <x>` 形态，与 `package.json` 的 142 个 script 比对。

结果：**唯一未命中项是 `quality_gates.md:288` 的 `pnpm run gen-<x>`**——散文中的占位写法，非实际命令。判定 PASS。

### 3.4 代码块真实性

这是本次验收最重要的一项，方法本身经过三次修正，过程记录在 `generation_progress.md`：

| 迭代 | 判据 | 结果 | 问题 |
| --- | --- | --- | --- |
| v1 | 从路径标注向下找代码块 | 21 处"不一致" | 假阳性：多数文档把标注写在代码块**下方**作图注 |
| v2 | 双向查找标注 | 6 处"不一致" | 假阳性：相邻代码块共用标注时归属歧义 |
| **v3** | **块内每一行是否逐字存在于邻近标注的任一源文件中** | **83/85 通过，0 处不一致** | 判据不依赖配对，不受标注位置影响 |

剩余 2 个块无路径标注，均为文档中显式标注为"修复建议"的示例配置，非源码引用。

**这项检查是字面级的**，它证明代码块没有被编造或改写，但**不证明**上下文对该代码的解释正确——后者本应由 `semantic_review_checker` 覆盖，见 §2。

## 4. 方案质量检查清单（人工执行）

### 4.1 准确性验证

| 项 | 结果 | 依据 |
| --- | --- | --- |
| 代码示例路径可追溯 | ✅ | §3.4，83/85 逐字可追溯，其余 2 个为显式标注示例 |
| 架构描述与 `docs/architecture.md` 一致 | ✅ | 各篇开头声明上游事实源；冲突处理原则写入 `AI_RULES.md` §6 |
| 事件与工具表未手工重述生成产物 | ✅ | 5 个生成产物均以链接引用（`tool-catalog` 11 篇、`config-catalog` 13 篇、`module-graph` 8 篇、`event-producer-consumer` 7 篇、`persistence-catalog` 4 篇），无手写清单 |
| 配置说明与 `docs/config-catalog.md` 一致 | ✅ | 同上，只链接不复制 |
| 命令与 `package.json` scripts 一致 | ✅ | §3.3 |

### 4.2 完整性验证

| 项 | 结果 | 依据 |
| --- | --- | --- |
| 主文档包含全部 17 篇子文档链接 | ✅ | 机器核对，无遗漏 |
| 回合流程有完整 mermaid 图 | ✅ | `architecture_overview.md` 与 `agent_loop_and_tools.md`；dev_docs 内 16 个块全部可解析（合计 36） |
| 至少 3 个真实扩展场景的 step-by-step | ✅ | `plugin_development_guide.md` 的扩展点决策树 + subagent/workflow/skill 三条链路 |
| 1.3B 全部不可破坏约束落入正式文档 | ✅ | 14 条全部落入 `rules/combined/AI_RULES.md`（R1–R24），并交叉分发到对应子文档 |
| 1.3C 全部外部服务边界落入 `security_and_sandbox.md` | ✅ | 8 条边界；`AI_RULES.md` §5 另附速查表 |

### 4.3 可用性验证

| 项 | 结果 | 依据 |
| --- | --- | --- |
| 全部 markdown 链接可跳转 | ✅ | §3.2，2305 文件 |
| mermaid 图表能正确渲染 | ✅ | §3.2，36 块 |
| frontmatter 通过 `summary_validator --strict` | ⚠️ **NOT_RUN** | 工具不可用。**替代**：机器核对 28 篇七字段齐全（`title`/`summary`/`keywords`/`scope`/`related_files`/`dependencies`/`verified_at`）——这只覆盖字段存在性，**不覆盖 strict 模式的内容规则** |
| AI 依据主文档 5 分钟内定位信息 | ⚠️ **未验证** | 需要实际使用观察，非本次可测。主文档提供 12 个场景导航与 P0/P1/P2 分级作为设计依据 |

### 4.4 一致性验证

| 项 | 结果 | 依据 |
| --- | --- | --- |
| 文档间交叉引用有效 | ✅ | §3.2 |
| 量化数值全仓一致 | ✅ | 见下表 |
| 与 `AGENTS.md` 治理约束无冲突 | ⚠️ **部分** | 见 §5 accepted_issues 第 1 条 |

量化数值复核（口径与实测值）：

| 数值 | 出现处 | 口径 | 实测 | 一致 |
| --- | --- | --- | --- | --- |
| 255 个包 | `monorepo_and_build.md` ×2、`plugin_development_guide.md` | `packages/*/*/` | 255 | ✅ |
| 50 个包组 | `AI_Coding_Context.md`、`architecture_overview.md` | `packages/*/` | 50 | ✅ |
| 1,742 篇 Agent Note | `AI_Coding_Context.md` ×2 | `.agents/notes/**/*.md` | 1742 | ✅ |
| 854 个 spec 源文件 | `AI_Coding_Context.md`、`testing_guide.md` | `packages/` 下 `*.spec.ts`/`*.test.ts` | 854 | ✅ |
| 684,902 行 TS / 255,250 行 MD / 260 个测试目录 | `AI_Coding_Context.md` | 2026-09-06 全量复算 | 同 | ✅ |

无相互冲突的变体（如同时出现 254/255/256）。

## 5. 已接受问题

```yaml
accepted_issues:
  - id: AI-1
    severity: high
    title: dev_docs 与 docs/ 存在事实归属地重复风险
    status: ACCEPTED_WITH_MITIGATION
  - id: AI-2
    severity: medium
    title: 语义层验收未执行
    status: ACCEPTED_KNOWN_GAP
  - id: AI-3
    severity: low
    title: 索引文件命名偏离 AICC 约定
    status: ACCEPTED_BY_DESIGN
  - id: AI-4
    severity: low
    title: 部分子文档的延伸阅读交叉链接不完整
    status: ACCEPTED_MINOR
```

**AI-1 — 事实归属地重复。** `docs/AGENTS.md` 的 one-home-per-fact 治理与"再加一层文档"天然张力。用户已接受该风险。强制缓解措施：每篇开头声明上游事实源；生成产物只链接不复制；冲突时以 `docs/` 为准并立即修正；代码逐字粘贴；`verified_at` 驱动复核。规则固化在 `AI_RULES.md` §6。**残余风险**：`dev_docs` 不入 CI 门禁，漂移不会被自动检出，只能靠 `verified_at` 触发人工复核。

**AI-2 — 语义层验收未执行。** 见 §2。字面级检查（代码逐字可追溯、链接可解析、命令存在）全部通过，语义级检查（陈述与源码含义是否一致）无工具可用。**部分补偿**：生成过程中对每条断言做了源码核实，副产品是查实了 7 处上游文档缺陷（记录在根目录 `UPSTREAM_DOC_ISSUES.md`）——这说明核实过程确实在起作用，但它是人工的，不构成可重复的验收证据。

**AI-3 — 索引文件命名。** `dev_docs` 内索引文件使用 `index.md` 而非 AICC 约定的 `README.md`。理由：配对门禁的 `README_ARTIFACT` 正则（`scripts/translation-pairing.ts:132`）匹配任意路径下的 `readme`，而 `dev_docs` 是刻意的纯中文层。三个候选方案中，本方案是唯一不需要改动仓库门禁脚本或配置、因而不需要 Agent Note 的做法。代价是偏离 AICC 目录约定；由于 AICC 工具不可用，该偏离当前无法被检出。**若未来接入 AICC 工具链，此项需重新评估。**

**AI-4 — 交叉链接不完整。** 部分子文档在写作时其兄弟文档尚不存在，"延伸阅读"因此不完整。不影响正确性（所有已写出的链接都可解析），但导航密度低于设计目标。

## 6. 复现方式

```bash
# 仓库自有门禁（dev_docs 在其作用域外，此处验证不引入回归）
pnpm exec tsx scripts/verify-translation-pairing.ts
pnpm exec tsx scripts/verify-md-links.ts
pnpm exec tsx scripts/verify-doc-budgets.ts
pnpm run build && pnpm run hygiene        # hygiene 的三个产物面门禁需要先构建

# dev_docs 内部链接与 mermaid（借用—还原，改动不提交）
#   1. 向 scripts/verify-md-links.ts 与 scripts/verify-mermaid.ts 的 PATTERNS
#      追加 'dev_docs/**/*.md' 与 'UPSTREAM_DOC_ISSUES.md'
#   2. pnpm exec tsx scripts/verify-md-links.ts && pnpm exec tsx scripts/verify-mermaid.ts
#   3. git checkout scripts/verify-md-links.ts scripts/verify-mermaid.ts

# frontmatter 七字段
python3 - <<'PY'
import glob
req=['title','summary','keywords','scope','related_files','dependencies','verified_at']
for f in sorted(x for x in glob.glob('dev_docs/**/*.md',recursive=True) if '_analysis' not in x):
    lines=open(f).read().split('\n'); end=lines[1:].index('---')+1
    keys={l.split(':')[0].strip() for l in lines[1:end] if ':' in l}
    miss=[k for k in req if k not in keys]
    if miss: print(f, miss)
PY

# 脱敏与模板残留
find dev_docs -name '*.md' -not -path '*/_analysis/*' -print0 \
  | xargs -0 grep -nE '/Users/|/home/[a-z]|sk-[A-Za-z0-9]{8,}|待填写|4317|4318'
```

## 7. 后续复核触发条件

`dev_docs` 不入 CI 门禁（见 AI-1 残余风险），因此复核靠以下条件触发，而非自动化：

1. **`docs/` 对应章节改动** — 受影响篇目的 `verified_at` 失效，需重新核实。
2. **`AGENTS.md` 的 `## Conventions` 或 `## Pre-stable APIs` 改动** — `AI_RULES.md` 必须同步。
3. **仓库门禁脚本改动** — 特别是 `translation-pairing.ts` 的语料判定（会影响 AI-3 的成立前提）与 `verify-md-links.ts` 的 `PATTERNS`。
4. **距 `verified_at` 超过一个发布周期** — 本仓库处于开发者预览期、公开 API 为 pre-stable，陈述过期速度快。
5. **AICC 工具链变为可用** — 届时应补跑 `summary_validator --strict`、`doc_health_checker --full-check`、`semantic_review_checker --full-check`，用真实结果替换本报告 §2 与 §4.3 的 `NOT_RUN` 标注，并重新评估 AI-3。

## 7bis. 第二轮全面复查（2026-09-06）

首版验收（§1–§7）是**自查**。本节记录随后进行的一轮**独立复查**：9 个子代理分工审计全部 28 篇正式文档 + `UPSTREAM_DOC_ISSUES.md`，指令要求逐条回源码实证、默认不采信文档自述，其中一个专门以"推翻这些指控"为立场对抗性复核上游缺陷记录。所有结论在采纳前由主控二次实证。

```yaml
review_round_2:
  date: 2026-09-06
  method: 9 个独立审计代理 + 主控机器级复检
  scope: dev_docs 全部 28 篇正式文档、4 篇 _analysis、UPSTREAM_DOC_ISSUES.md
  findings_accepted: 62      # 已修正
  findings_rejected: 3       # 经主控实证为审计员误判，未采纳
  machine_checks_after_fix:
    frontmatter:      PASS   # 28/28 恰好七字段
    links_anchors:    PASS   # 2291 条链接 0 断链 0 死锚
    code_authenticity: PASS  # 75/75 探针逐字命中源码
    redaction:        PASS   # 无 key / 无端点 / 无本机绝对路径
    non_chinese_prose: PASS  # 命中项全为上游原文引用或英文锚点标题
    repo_gates:       PASS   # 1135 配对 / 2273 文件 / 8 预算 / 20 mermaid
```

**修正的高危问题**（会直接误导读者的）：

| 篇目 | 问题 | 性质 |
| --- | --- | --- |
| `evaluation_metrics.md` | 称"两种反馈都不进遥测"——实际会话级 `/feedback` **正是触发上传的闸门**，且出厂 base 无脱敏规则 | 隐私表述错误 |
| `security_and_sandbox.md` | 未提 `sdk-minimal` profile **硬编码 `danger-full-access` 且不挂审批**，`DSH_PERMISSION_MODE` 对其无效 | 把安全边界说得比实际强 |
| `security_and_sandbox.md` | 称"配置文件里也只写引用名，不写值"——多个随包 `Config` 结构性接受字面密钥，其中两处未标 `role('secret')` | 给虚假安全感 |
| `security_and_sandbox.md` | 遗漏权威硬关闭开关 `DSH_TELEMETRY_DISABLED` | 完整性 |
| `agent_loop_and_tools.md` | 称 `agent/request-error` 的 retry "开一个全新重试 turn"——实际是同 step 内 `continue` | 机制讲反 |
| `agent_loop_and_tools.md` | 称 `agent/pre-step` 是"请求推导前唯一的瀑布"——`system-prompt/assemble` 先于它派发 | 与本文自己的图矛盾 |
| `AI_RULES.md` | 编造"MIT 许可要求披露第三方依赖"的因果 | 事实错误 |
| `AI_RULES.md` | 把带三个例外的规定写成无条件禁令（全量套件） | 条件性→无条件 |
| `AI_RULES.md` | §1 混入无上游出处的本地写作约定，与本文"不新增规则"的声明冲突 | 自相矛盾 |
| 5 篇 | "`docs/` 已全量双语（每篇都有 `.zh.md`）"——实有 5 篇被 manifest 显式豁免 | 事实错误，扩散 5 篇 |
| `AI_Coding_Context.md` | 测试分层写成"四套配置：单元/快照/e2e/Web"，漏掉 coverage 这一最强门禁 | 事实错误 |
| `monorepo_and_build.md` | "六个包拆了 host/client"——实测 8 个（上游 `docs/development.md:62` 亦已过期） | 数字过期 |
| `quality_gates.md` | "三十余个 verify/gen 门禁"——实测 62 个 | 数字低估一半 |
| `quality_gates.md` | 把 `lint` / `typecheck` 列为"不需要构建"——两者都会跑完整 Host lib 阶段 | 跨文档矛盾 |
| `capability_seams.md` | 称 `dsh-user-approval` 是"定义与唯一实现同包"——该包无实现，answerer 在别的包 | 事实错误 |
| `UPSTREAM_DOC_ISSUES.md` | U2/U3/U7 "同一根因"的归因错误（台账 `:279` 行实为 2026-08-24 追加） | 归因错误，已重新归并 |

**未采纳的审计结论**（主控实证后判定为审计员误判）：

1. `docs/glossary.md#turn` 等锚点"失效"——实为 `<a id>` 显式锚点，检测器未解析。
2. `packages/acp/acp/README.md#standard-acp-v1-surface` "失效"——同上，该锚点在 `:57` 显式存在。
3. vendor 清单版本号与 `package.json` 不一致"说明清单陈旧"——按 `scripts/release/bump.ts` 的设计二者本就不等，已作为**反例**写入 `deployment_guide.md` 与 `UPSTREAM_DOC_ISSUES.md`。

**对已接受问题的影响**：AI-4（交叉链接不完整）已实质收敛——`architecture_overview.md` 与 `capability_seams.md` 这两个入度最高却零出边的"网络死端"已补上 dev_docs 内相关篇目区块，`troubleshooting.md` 补齐 6 条兄弟链接。AI-1（事实归属地重复）、AI-2（语义层无工具可验）、AI-3（`index.md` 命名偏离）维持原状。

**本轮未改变的结论**：AICC 三套 checker 仍为 `NOT_RUN`，语义层验收仍是真实缺口（AI-2）。本轮的独立审计**部分**替代了语义层检查——9 个代理确实逐条回源核实了论断，但那是人工/代理审计，不是工具化的可重复门禁。

## 8. 相关文档

- [生成方案](generation_plan.md) — 方案与质量检查清单的定义处
- [问题报告](project_analysis_report.md) — AI-1/AI-2/AI-3 的完整分析与降级验收方案的出处
- [进度记录](generation_progress.md) — 逐批次产物与验证过程
- [上游文档缺陷记录](../../UPSTREAM_DOC_ISSUES.md) — 核实过程的副产品，7 条待复查
