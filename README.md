# Fleet Observability

> Kubernetes observability configuration based on Prometheus, Loki, Grafana, and AlertManager.

[![Status](https://img.shields.io/badge/status-in%20development-yellow)](https://github.com/Shaohan-He/fleet-observability)
[![Stack](https://img.shields.io/badge/stack-Prometheus%20%2B%20Loki%20%2B%20Grafana-orange)](https://github.com/Shaohan-He/fleet-observability)

## 概述

Fleet Observability 用于集中管理 Kubernetes 集群的监控、日志、告警规则和 Grafana 仪表盘。仓库内容以配置为主，目标是让可观测性组件可以通过 Git 进行版本控制和重复部署。

当前覆盖范围：

- Prometheus 抓取配置、记录规则和告警规则。
- Loki 与 Promtail 日志采集配置。
- Grafana 数据源、仪表盘和自动装载配置。
- 基于 SLO 的告警规则示例。
- 与节点巡检、节点诊断工具的指标或日志接入说明。

## 架构

```text
Kubernetes cluster
    |
    +-- node-exporter / kube-state-metrics
    +-- application metrics
    +-- node-health-watcher metrics
    +-- node-guardian textfile output
    |
    v
Prometheus + Loki
    |
    +-- AlertManager
    +-- Grafana dashboards
```

## 目录结构

```text
fleet-observability/
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/
│   │   ├── alerts.yaml
│   │   └── recording.yaml
│   └── scrape_configs/
│       ├── nhw.yaml
│       └── node-guardian.yaml
├── loki/
│   ├── loki-config.yaml
│   └── promtail-config.yaml
├── grafana/
│   ├── grafana.ini
│   ├── dashboards/
│   └── provisioning/
└── docs/
    ├── slo-definitions.md
    └── adr/
```

## 快速开始

### 前提条件

- Kubernetes 集群
- `kubectl`
- Helm 3+
- 至少 4 GB 可用内存用于本地 `kind` 环境

### 部署 Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin
```

### 部署 Loki

```bash
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=5Gi \
  --set promtail.config.lokiAddress=http://loki.monitoring.svc:3100/loki/api/v1/push
```

### 应用规则和仪表盘

```bash
kubectl create configmap prometheus-rules \
  --from-file=prometheus/rules/ \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -

kubectl create configmap grafana-dashboards \
  --from-file=grafana/dashboards/ \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -

kubectl label configmap grafana-dashboards -n monitoring grafana_dashboard=1 --overwrite
```

### 访问 Grafana

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-grafana 3000:80
```

打开 `http://localhost:3000`，默认用户为 `admin`，密码为安装命令中设置的 `admin`。

## SLO 与告警

仓库包含基于 SLO 的告警示例。相比单纯的资源阈值，SLO 告警更适合描述用户可感知的可用性问题。

| 类型 | 示例 |
| --- | --- |
| SLI | 请求成功率、关键操作延迟、节点健康状态 |
| SLO | 月度成功率目标、P95/P99 延迟目标 |
| 告警 | 多窗口燃烧速率、错误预算耗尽、节点健康异常 |

完整定义见 [docs/slo-definitions.md](docs/slo-definitions.md)。

## 仪表盘

| 仪表盘 | 文件 |
| --- | --- |
| 节点健康概览 | [grafana/dashboards/node-health-overview.json](grafana/dashboards/node-health-overview.json) |
| 运行状态概览 | [grafana/dashboards/fleet-operations.json](grafana/dashboards/fleet-operations.json) |
| SLO 燃烧速率 | [grafana/dashboards/slo-burn-rate.json](grafana/dashboards/slo-burn-rate.json) |
| 日志浏览 | [grafana/dashboards/log-explorer.json](grafana/dashboards/log-explorer.json) |

## 项目状态

- [x] Prometheus 规则与抓取配置
- [x] Loki 与 Promtail 配置
- [x] Grafana provisioning 配置
- [x] Grafana 仪表盘 JSON
- [x] SLO 定义文档
- [ ] 在真实集群中完成端到端部署验证
- [ ] 补充仪表盘截图

## 设计取舍

| 主题 | 当前选择 | 说明 |
| --- | --- | --- |
| 指标系统 | Prometheus | Kubernetes 生态通用，规则和抓取配置易于版本控制 |
| 日志系统 | Loki | 资源占用较低，适合与 Grafana 集成 |
| 仪表盘管理 | JSON + provisioning | 避免手动 UI 配置丢失，便于 code review |
| 告警策略 | AlertManager + SLO 规则 | 兼顾资源异常和服务目标异常 |

## 相关项目

| 仓库 | 关系 |
| --- | --- |
| [fleet-gitops](https://github.com/Shaohan-He/fleet-gitops) | 可用于部署本仓库中的可观测性配置 |
| [node-health-watcher](https://github.com/Shaohan-He/node-health-watcher) | 节点巡检数据来源 |
| [node-guardian](https://github.com/Shaohan-He/node-guardian) | 节点诊断日志和 textfile 指标来源 |
| [k8s-healing-agent](https://github.com/Shaohan-He/k8s-healing-agent) | 可消费 AlertManager 告警的修复实验项目 |

## License

MIT
