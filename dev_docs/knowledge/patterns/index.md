---
title: 实现模式索引
summary: 在本代码库中反复出现的实现写法及其适用边界，区别于官方扩展点文档。
keywords: patterns | 模式 | capability-seam | 写法 | 边界
scope: dev_docs/knowledge/patterns 的索引
related_files: dev_docs/plugin_development_guide.md | dev_docs/capability_seams.md | docs/defensive-patterns.md
dependencies: dev_docs/knowledge/index.md
verified_at: 2026-09-06
---

# 实现模式索引

> 目录规则见 [`knowledge/index.md`](../index.md)。

**先去这两处**——本目录只收它们没覆盖的：

- [`plugin_development_guide.md`](../../plugin_development_guide.md) — 官方扩展点与写插件的完整流程
- [`docs/defensive-patterns.md`](../../../docs/defensive-patterns.md) — 生命周期、并发、子进程、拆卸的既有防御模式（**上游权威**，改这些方面的代码前必读）

## 一条模式记录什么

- **模式** — 具体写法，附仓库中的真实出处 `path:line（symbol: name）`。代码逐字粘贴，不要凭印象重写。
- **它解决什么** — 不写这条就会出什么问题。
- **不适用的情况** — 必填。没有边界的模式会被无差别套用，那比没有模式更糟。
- **与硬规则的关系** — 它是否是某条 [`AI_RULES.md`](../../rules/combined/AI_RULES.md) 的具体实现（例如 R11 的 `resolve(request): Spec` 拆分、R9 的 disposer 返回）。

**注意**：如果一个"模式"其实是 `AGENTS.md` 已经规定的约定，那它属于 [`AI_RULES.md`](../../rules/combined/AI_RULES.md)，不属于这里。本目录收的是**上游没有明说、但代码里事实上一致**的写法。

## 列表

_（暂无）_
