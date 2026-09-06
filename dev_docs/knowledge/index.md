---
title: 知识沉淀索引
summary: 经验证、可复用的结论按排障/模式/性能三类归档，以及它与 dev_docs 导航文档的分工。
keywords: knowledge | troubleshooting | patterns | performance | 沉淀 | 可复用
scope: dev_docs/knowledge 的组织规则与索引
related_files: dev_docs/troubleshooting.md | dev_docs/cost_optimization.md | dev_docs/plugin_development_guide.md
dependencies: dev_docs/memos/index.md
verified_at: 2026-09-06
---

# 知识沉淀索引

> **本文定位**：`dev_docs` 是 deepseek-harness 的**中文导航层**。本目录存放**经过实际验证、可复用**的结论——从 [`memos/`](../memos/index.md) 与 [`plans/done/`](../plans/done/index.md) 里提取出来的部分。
>
> **与 `docs/` 冲突时一律以 `docs/` 为准。**

## 与导航文档的分工

`dev_docs/` 根目录下的 18 篇是**导航文档**：回答"这件事的权威说明在哪"。本目录是**经验记录**：回答"我们实际撞上过什么，怎么解决的"。

| 主题 | 导航文档（指路） | 本目录（案例） |
| --- | --- | --- |
| 排障 | [`troubleshooting.md`](../troubleshooting.md) — 症状 → 对应文档/事实源 | [`troubleshooting/`](troubleshooting/index.md) — 具体案例的现场、根因、解法 |
| 实现模式 | [`plugin_development_guide.md`](../plugin_development_guide.md) — 官方扩展点与流程 | [`patterns/`](patterns/index.md) — 反复出现的具体写法与它的适用边界 |
| 性能与成本 | [`cost_optimization.md`](../cost_optimization.md) — 可调项与其上游说明 | [`performance/`](performance/index.md) — 实测数据与调优结论 |

判据：**如果一条内容在上游有归属地，它就属于导航文档，用链接指过去。** 只有上游没有、且经过本地验证的内容才进本目录。

## 三个子目录

- [`troubleshooting/`](troubleshooting/index.md) — 遇到过的具体故障：症状、复现条件、根因、解法、验证方式。
- [`patterns/`](patterns/index.md) — 在这个代码库里反复出现的实现模式，以及什么时候**不该**用它。
- [`performance/`](performance/index.md) — 实测的性能与成本数据，附测量方法与环境。

## 命名约定

```
YYYY-MM-DD-<kebab-case-主题>.md
```

## 一条知识的最低要求

不满足以下四条的不要收录——否则本目录会退化成第二个备忘目录：

1. **可复现** — 写清在什么条件下成立，而不只是"我遇到过"。
2. **已验证** — 结论经过实际执行确认，不是推测。推测留在 [`memos/`](../memos/index.md)。
3. **有边界** — 说明它在什么情况下**不**适用。没有边界的经验最容易被误用。
4. **不与上游重复** — 若上游 `docs/` 或源码 JSDoc 已经说了，写链接而不是重述（见 [`AI_RULES.md`](../rules/combined/AI_RULES.md) §6）。

## 保鲜

每条知识在 frontmatter 里带 `verified_at`。这个仓库处于开发者预览期、公开 API 是 pre-stable（见 [`AI_RULES.md`](../rules/combined/AI_RULES.md) R2），**知识会过期**。引用一条超过其 `verified_at` 较久的记录前，先按它写的验证方式复核一遍；确认失效就删除，不要留着加"可能已过时"的标注。

## 延伸阅读

- [`AI_Coding_Context.md`](../AI_Coding_Context.md) — dev_docs 总入口
- [`memos/index.md`](../memos/index.md) — 沉淀前的暂存地
- [`AI_RULES.md`](../rules/combined/AI_RULES.md) — 硬规则合集
