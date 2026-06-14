# SLO 定义 — Game Fleet Platform

> **版本:** v1.0
> **最后更新:** 2026-06-14
> **状态:** 生效中
> **方法论:** Google SRE Workbook, 第 2 章 (SLI/SLO/SLA) 及第 5 章 (基于 SLO 告警)

---

## 概述

本文档定义了 Game Fleet Platform 的服务等级目标（SLO）。这些 SLO 驱动以下内容：

1. **告警** — 多窗口、多燃烧速率告警（见 `prometheus/rules/alerts.yaml`）
2. **仪表盘** — Grafana 中的 SLO 燃烧速率仪表盘（见 `grafana/dashboards/slo-burn-rate.json`）
3. **混沌工程** — Phase C 韧性验证标准（见 `fleet-chaos` 仓库）
4. **错误预算策略** — 何时冻结部署 vs 何时投入可靠性建设

---

## 核心概念

### SLI → SLO → SLA

| 术语 | 定义 | 示例 |
|------|------|------|
| **SLI**（服务等级指标） | 服务某一方面的量化度量 | 成功分配请求 / 总分配请求 |
| **SLO**（服务等级目标） | SLI 在时间窗口内的目标值 | 30 天内成功率 99.5% |
| **SLA**（服务等级协议） | 与用户关于未达 SLO 后果的合约 | 不适用 — 这是内部平台，无付费客户 |

### 错误预算

**错误预算**是在给定周期内允许的不可靠量：

```
错误预算 = 1 - SLO = 1 - 0.995 = 0.005（0.5%）
```

以 30 天月度计算：
- 总分钟数: 30 × 24 × 60 = 43,200 分钟
- 错误预算: 43,200 × 0.005 = **216 分钟（3.6 小时）**/月

当错误预算耗尽时，服务未达到其 SLO。

### 燃烧速率

**燃烧速率**表示相对于理想速率，你消耗错误预算的快慢：

- **1x 燃烧速率** = 按"配额"速率消耗错误预算（0.5% 错误）
- **10x 燃烧速率** = 比配额快 10 倍消耗预算（5% 错误）
- **100x 燃烧速率** = 比配额快 100 倍消耗预算（50% 错误）

**为什么重要：** 1% 的错误率看起来不大，但以 2x 燃烧速率计算，你会在 15 天内耗尽月度预算。这能捕获阈值告警会漏掉的缓慢恶化。

---

## SLO #1: 游戏服务器分配可用性

最关键的面向用户指标。当玩家想加入游戏时，能成功吗？

| 字段 | 值 |
|------|-----|
| **SLI** | 分配成功率 = `gfd_allocation_total{phase="Allocated"} / gfd_allocation_total` |
| **SLO** | **99.5%** 成功率（月度，30 天滚动窗口） |
| **SLI 实现** | `sum(rate(gfd_allocation_total{phase="Allocated"}[30d])) / sum(rate(gfd_allocation_total[30d]))` |
| **错误预算** | 0.5% = 约 216 分钟/月允许失败 |
| **度量窗口** | 30 天滚动 |

### 多窗口燃烧速率告警

遵循 Google SRE Workbook 第 5 章 — 基于 SLO 告警的黄金标准：

| 窗口 | 时长 | 燃烧速率 | 错误率阈值 | 严重度 | 原理 |
|------|------|----------|-----------|--------|------|
| **短** | 5 分钟 | > 14.4x | > 7.2% | **CRITICAL** | 捕获突发严重恶化。14.4x 持续 5 分钟 = 消耗 2% 月度预算 |
| **中** | 1 小时 | > 14.4x | > 7.2% | **WARNING** | 捕获持续问题。14.4x 持续 1 小时 = 消耗 5% 月度预算 |
| **长** | 6 小时 | > 6x | > 3.0% | **CRITICAL** | 捕获缓慢的错误预算侵蚀。6x 持续 6 小时 = 消耗 30% 月度预算 |

**为什么需要多窗口？** 单窗口告警要么触发太频繁（短窗口 = 吵闹），要么触发太晚（长窗口 = 错过急性问题）。多窗口两者兼顾。

### 错误预算策略

| 剩余预算 | 措施 |
|---------|------|
| > 50% | 正常运维。部署自由进行 |
| 20% – 50% | ⚠️ 注意。检查近期变更。考虑仅金丝雀部署 |
| 5% – 20% | 🔴 告警。冻结所有非关键部署。聚焦可靠性 |
| < 5% | 🚨 紧急。所有部署冻结。启动战时机制 |

---

## SLO #2: 扩缩容决策延迟

系统对玩家数量变化的响应有多快？

| 字段 | 值 |
|------|-----|
| **SLI** | Reconcile 循环耗时 P95 = `histogram_quantile(0.95, rate(gfd_reconcile_duration_seconds_bucket[5m]))` |
| **SLO** | **P95 < 5 秒** |
| **错误预算** | 1% 的 reconcile 循环可超过 5 秒 |
| **度量窗口** | 7 天滚动 |

### 原理

如果 reconcile 超过 5 秒，扩容决策会滞后于玩家需求。最坏情况下，玩家等待超过 10 秒才等到游戏服务器 — 体验上就是"游戏坏了"。

### 告警

| 窗口 | 条件 | 严重度 |
|------|------|--------|
| 5 分钟 | P95 > 5s | WARNING |
| 15 分钟 | P95 > 10s | CRITICAL |

---

## SLO #3: 排水安全性（零玩家驱逐）

缩容时，我们是否曾踢出正在游戏的玩家？

| 字段 | 值 |
|------|-----|
| **SLI** | 驱逐了活跃玩家的排水事件 = `gfd_drain_duration_seconds` < 30s 的排水次数（太快 = 强制杀死） |
| **SLO** | **0 次玩家驱逐**（100% 优雅排水） |
| **错误预算** | 0 — 这是硬性要求，不是目标 |
| **度量窗口** | 30 天滚动 |

### 原理

踢出正在游戏中的玩家是排名第一的最差体验。三阶段排水（Cordon → Drain → Decommission）专门为防止此问题设计。如果这个 SLO 被违反，说明排水逻辑存在 bug。

---

## 在 Grafana 中可视化 SLO

SLO 燃烧速率仪表盘（`slo-burn-rate.json`）可视化以下内容：

1. **错误预算仪表** — 本月剩余多少预算
2. **多窗口燃烧速率图表** — 短/中/长燃烧速率叠加在同一张图上（"面试截图"）
3. **30 天成功率趋势** — 分配成功率 + SLO 阈值线
4. **燃烧速率对比柱状图** — 各窗口燃烧速率并排对比
5. **SLO 合规状态表** — 逐 SLO 合规状态

### 关键 PromQL Recording Rules

所有指标在 `prometheus/rules/recording.yaml` 中预计算，以提升仪表盘性能：

```promql
# 30 天成功率（SLO 合规）
slo:gfd_allocation_success_rate:rate30d

# 多窗口燃烧速率
slo:gfd_allocation_burn_rate:short_5m    # 5 分钟窗口燃烧速率
slo:gfd_allocation_burn_rate:medium_1h   # 1 小时窗口燃烧速率
slo:gfd_allocation_burn_rate:long_6h     # 6 小时窗口燃烧速率

# 错误预算剩余（%）
slo:gfd_allocation_error_budget_remaining_pct
```

---

## 参考文献

- [Google SRE Workbook — 第 2 章: 实施 SLO](https://sre.google/workbook/implementing-slos/)
- [Google SRE Workbook — 第 5 章: 基于 SLO 告警](https://sre.google/workbook/alerting-on-slos/)
- [Prometheus SLO 燃烧速率告警示例](https://prometheus.io/docs/practices/alerting/)
- [Grafana SLO 仪表盘指南](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)

---

## 变更记录

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-06-14 | v1.0 | 初始 SLO 定义: 分配可用性、扩缩容延迟、排水安全性 |
