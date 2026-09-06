---
title: 性能与成本记录索引
summary: 实测的性能与 token 成本数据，附测量方法与环境；区别于 cost_optimization.md 的可调项导航。
keywords: performance | 性能 | token | 成本 | 基准 | 测量
scope: dev_docs/knowledge/performance 的索引
related_files: dev_docs/cost_optimization.md | dev_docs/evaluation_metrics.md | BENCHMARK.md
dependencies: dev_docs/knowledge/index.md
verified_at: 2026-09-06
---

# 性能与成本记录索引

> 目录规则见 [`knowledge/index.md`](../index.md)。

**先去 [`cost_optimization.md`](../../cost_optimization.md) 与 [`evaluation_metrics.md`](../../evaluation_metrics.md)**——那里是可调项与指标口径的导航。本目录只放**实测数字**。

## 一条记录必须带的东西

数字脱离条件就是噪音。以下五项缺一不可：

- **测量对象** — 具体到 profile/bundle 与模型配置。
- **测量方法** — 命令与参数，可复制执行。
- **环境** — 平台、Node 版本、提交 SHA。这个仓库跨三平台且处于开发者预览期，环境不同结论就不同。
- **数据** — 原始值，不是"大约快了一半"。
- **`verified_at`** — 性能数据的过期速度快于其他知识。

## 脱敏

不写真实 API key、真实 base URL、真实 OTLP 端点、本地绝对路径。用环境变量名、`sk-***`、`$DSH_HOME/...` 代替。见 [`AI_RULES.md`](../../rules/combined/AI_RULES.md) R1 与 §6。

## 关于仓库自带的基准

根目录 [`BENCHMARK.md`](../../../BENCHMARK.md) 是上游的基准入口，但它当前指向一个已迁移的示例路径（详见 [`UPSTREAM_DOC_ISSUES.md`](../../../UPSTREAM_DOC_ISSUES.md) U7）。按 [`docs/user/guide/python-sdk.md`](../../../docs/user/guide/python-sdk.md) 的现行说明执行即可——那里的 `sdk-minimal` profile 与 `python/sdk/examples/` 路径是正确的。

## 列表

_（暂无）_
