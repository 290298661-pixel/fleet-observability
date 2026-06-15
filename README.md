# 📊 Fleet Observability

> **统一可观测性平台** — Prometheus + Loki + Grafana + SLO 燃烧速率仪表盘
>
> Phase B of [K8s 全栈运维平台](https://github.com/290298661-pixel)

[![Status](https://img.shields.io/badge/status-in%20development-yellow)](https://github.com/290298661-pixel/fleet-observability)
[![Phase](https://img.shields.io/badge/phase-B-blue)](https://github.com/290298661-pixel)
[![Stack](https://img.shields.io/badge/stack-Prometheus%20%2B%20Loki%20%2B%20Grafana-orange)](https://github.com/290298661-pixel/fleet-observability)

---

## 概述

Fleet Observability 是一个**统一可观测性平台**，将三个 Phase 0 服务的监控整合到一起：

| 服务 | 提供什么 | Phase B 集成方式 |
| ---- | ------- | ----------------- |
| **Game Fleet Director**（Go） | `:8080/metrics` 上的 11 个 Prometheus 指标 | Prometheus 直接抓取 — 无需改动 |
| **Node Health Watcher**（Python） | 节点健康检查、IM 告警 | 新增 `/metrics` 端点（prometheus_client） |
| **Node Guardian**（Bash） | 诊断工具集 | 通过 node_exporter 的 textfile collector |

**核心洞察：** 这三个服务当前的监控是割裂的 — NHW 发送一次性 IM 告警，GFD 暴露指标但无人采集，Node Guardian 写本地日志。Phase B 将它们统一为 **指标 + 日志 + SLO 仪表盘** 平台。

---

## 架构

```
┌──────────────────────────────────────────────────────────────────┐
│                     统一可观测性平台                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  数据来源                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │   GFD    │  │   NHW    │  │ Guardian │  │  K8s 集群      │  │
│  │ :8080    │  │ :9090    │  │ textfile │  │  (节点/Pod)    │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬────────┘  │
│       │             │             │                  │           │
│       ▼             ▼             ▼                  ▼           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │           Prometheus Stack（kube-prometheus-stack）           │ │
│  │  ┌──────────────┐  ┌──────────┐                            │ │
│  │  │  Prometheus   │  │   Loki   │                            │ │
│  │  │  • 抓取指标    │  │  • 日志  │                            │ │
│  │  │  • 告警规则    │  │  • 14天  │                            │ │
│  │  └──────┬───────┘  └────┬─────┘                            │ │
│  └─────────┼───────────────┼──────────────────────────────────┘ │
│            │               │                                     │
│            ▼               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                      Grafana                                  │ │
│  │                                                               │ │
│  │  🏥 节点健康    🎮 舰队运营    🔥 SLO 燃烧    📋 日志        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  告警                                                             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  AlertManager ──→ 飞书 / 钉钉                                │ │
│  │  多窗口燃烧速率告警 + 节点健康告警                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 快速开始

### 前提条件

- `kind` 集群运行中，至少 4 GB RAM
- `helm` v3+
- `kubectl` 已配置

### 1. 部署 Prometheus Stack

```bash
# 添加 Helm 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 部署 kube-prometheus-stack（Prometheus + Grafana + AlertManager + node_exporter）
helm upgrade --install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin
```

### 2. 部署 Loki + Promtail

```bash
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=5Gi \
  --set promtail.config.lokiAddress=http://loki.monitoring.svc:3100/loki/api/v1/push
```

### 3. 应用自定义配置

```bash
# 应用 Prometheus 规则
kubectl create configmap prometheus-rules \
  --from-file=prometheus/rules/ \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -

# 导入 Grafana 仪表盘
kubectl create configmap grafana-dashboards \
  --from-file=grafana/dashboards/ \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -

# 打标签以便 Grafana sidecar 发现
kubectl label configmap grafana-dashboards -n monitoring grafana_dashboard=1
```

### 4. 访问 Grafana

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-grafana 3000:80
# 打开 http://localhost:3000
# 用户名: admin / 密码: admin
```

### 5. 验证

检查 Prometheus 是否正在抓取目标：

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-prometheus 9090:9090
# 打开 http://localhost:9090/targets
```

预期目标：`gfd-controller`、`gfd-apiserver`、`nhw`、`node-exporter`、`kube-state-metrics`

---

## 目录结构

```
fleet-observability/
├── README.md                              ← 你在这里
├── .gitignore
│
├── prometheus/
│   ├── prometheus.yml                     # Prometheus 主配置
│   ├── rules/
│   │   ├── alerts.yaml                    # SLO 燃烧速率 + 节点健康告警
│   │   └── recording.yaml                 # 预计算 SLO 指标
│   └── scrape_configs/
│       ├── gfd.yaml                       # GFD 抓取配置文档
│       ├── nhw.yaml                       # NHW 抓取配置文档
│       └── node-guardian.yaml             # Textfile collector 文档
│
├── loki/
│   ├── loki-config.yaml                   # Loki 服务端配置
│   └── promtail-config.yaml               # Promtail DaemonSet 配置
│
├── grafana/
│   ├── grafana.ini                        # Grafana 服务端设置
│   ├── dashboards/
│   │   ├── node-health-overview.json      # 🏥 节点健康仪表盘
│   │   ├── fleet-operations.json          # 🎮 舰队运营仪表盘
│   │   ├── slo-burn-rate.json             # 🔥 SLO 燃烧速率仪表盘
│   │   └── log-explorer.json              # 📋 日志浏览器仪表盘
│   └── provisioning/
│       ├── datasources/
│       │   ├── prometheus.yaml            # 自动装载 Prometheus 数据源
│       │   └── loki.yaml                  # 自动装载 Loki 数据源
│       ├── dashboards/
│       │   └── all.yaml                   # 自动装载仪表盘
│       └── notifiers/
│           └── alertmanager.yaml           # AlertManager 通知通道
│
└── docs/
    ├── slo-definitions.md                 # ★ SLO 定义
    └── adr/
        ├── 001-choose-loki-over-elk.md    # 为什么选 Loki 而非 ELK
        └── 002-slo-burn-rate-alerts.md    # 为什么用多窗口燃烧速率告警
```

---

## 核心设计决策

### 1. 基于 SLO 的告警（而非阈值告警）

我们使用**多窗口、多燃烧速率 SLO 告警**，遵循 Google SRE Workbook 第 5 章的方案，而不是 "CPU > 90% → 告警" 的传统做法。

```
SLI:  allocation_success_rate = Allocated / (Allocated + Failed)
SLO:  99.5%（月度）
错误预算: 0.5%（约 216 分钟/月）

告警:
  短窗口 (5m):  燃烧速率 > 14.4 → CRITICAL
  中窗口 (1h):  燃烧速率 > 14.4 → WARNING
  长窗口 (6h):  燃烧速率 > 6    → CRITICAL
```

这意味着我们对**用户体验**告警，而非系统内部指标。

### 2. 选择 Loki，而非 ELK

Loki 作为单个 Go 二进制运行仅需约 200 MB RAM。Elasticsearch 需要 3+ 个 JVM 节点。在本地 `kind` 集群上，Loki 是唯一可行的选择。[阅读 ADR →](docs/adr/001-choose-loki-over-elk.md)

### 3. 版本控制的仪表盘

所有 Grafana 仪表盘以 JSON 格式存储在本仓库中，通过 provisioning 自动装载 — 无需在 UI 中手动配置。这意味着：

- 仪表盘变更需要 code review
- `git diff` 精确显示变更内容
- 集群销毁/重建后仪表盘不丢失

---

## 仪表盘

| 仪表盘 | UID | 描述 |
| ------ | --- | ---- |
| 🏥 [节点健康概览](grafana/dashboards/node-health-overview.json) | `node-health-overview` | 健康分、巡检历史、告警趋势、逐节点状态 |
| 🎮 [舰队运营](grafana/dashboards/fleet-operations.json) | `fleet-operations` | 副本/玩家/缓冲区、分配延迟热力图、扩缩容事件 |
| 🔥 [SLO 燃烧速率](grafana/dashboards/slo-burn-rate.json) | `slo-burn-rate` | 错误预算仪表、多窗口燃烧速率、SLO 合规表 |
| 📋 [日志浏览器](grafana/dashboards/log-explorer.json) | `log-explorer` | 全文日志搜索、按级别/应用统计日志量、指标到日志关联 |

---

## SLO 定义

完整的 SLO 定义见 [docs/slo-definitions.md](docs/slo-definitions.md)，包括 SLI 公式、燃烧速率告警逻辑、错误预算计算和运维策略。

| SLO | 目标 | 错误预算 |
| --- | ---- | ------- |
| 分配可用性 | 月度 99.5% | 216 分钟/月 |
| 扩缩容决策延迟 | P95 < 5s | 1% 的 reconcile 次数 |
| 排水安全性 | 0 次玩家驱逐 | 0（硬性要求） |

---

## 告警规则

| 告警 | 严重度 | 描述 |
| ---- | ------ | ---- |
| `SLOBurnRateCritical_Short` | CRITICAL | 5 分钟燃烧速率 > 14.4x（突发严重恶化） |
| `SLOBurnRateWarning_Medium` | WARNING | 1 小时燃烧速率 > 14.4x（持续问题） |
| `SLOBurnRateCritical_Long` | CRITICAL | 6 小时燃烧速率 > 6x（预算缓慢侵蚀） |
| `SLOErrorBudgetExhausted` | CRITICAL | 月度错误预算完全耗尽 |
| `FleetZeroReplicas` | CRITICAL | 某舰队运行中游戏服务器为 0 |
| `FleetAllocationFailureSpike` | WARNING | 分配失败率 > 10% |
| `FleetCircuitBreakerOpen` | CRITICAL | 扩缩容熔断器触发 |
| `FleetBufferDepleted` | WARNING | 缓冲池无预热服务器 |
| `NodeHealthCritical` | CRITICAL | 节点健康状态 = CRITICAL |
| `ConntrackPressure` | WARNING | Conntrack 表使用率 > 90% |
| `DiskSpaceLow` | WARNING | 磁盘使用率 > 85% |

---

## 项目状态

- [x] Prometheus 配置（抓取配置、规则、告警）
- [x] Loki + Promtail 配置
- [x] Grafana 仪表盘（4 个仪表盘，版本控制 JSON）
- [x] SLO 定义 + 燃烧速率告警规则
- [x] ADR: Loki vs ELK
- [x] ADR: 多窗口燃烧速率告警
- [ ] NHW metrics 端点接入（在 NHW 仓库中）
- [ ] Node Guardian `--prometheus` textfile 输出（在 node-guardian 仓库中）
- [ ] 部署到 kind 集群并端到端验证
- [ ] README 仪表盘截图

---

## 相关项目

| Phase | 仓库 | 描述 |
| ----- | ---- | ---- |
| Phase 0 | [node-guardian](https://github.com/290298661-pixel/node-guardian) | Bash 诊断工具集 |
| Phase 0 | [node-health-watcher](https://github.com/290298661-pixel/node-health-watcher) | Python 节点巡检 + IM 告警 |
| Phase 0 | [game-server-orchestrator](https://github.com/290298661-pixel/game-server-orchestrator) | Go K8s Operator 游戏服务器自动扩缩容 |
| Phase A | [fleet-gitops](https://github.com/290298661-pixel/fleet-gitops) | ArgoCD GitOps 交付流水线 |
| **Phase B** | **fleet-observability** ← 你在这里 | **Prometheus + Loki + Grafana + SLO** |
| Phase C | [fleet-chaos](https://github.com/290298661-pixel/fleet-chaos) | Chaos Mesh 韧性测试 |
| Phase D | [fleet-brain](https://github.com/290298661-pixel/fleet-brain) | LLM Agent 根因分析 |

---

## License

MIT
