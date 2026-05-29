# 📊 Fleet Observability

> **统一可观测性平台** — Prometheus + Loki + Grafana + SLO 燃烧率 Dashboard

## 状态

🔲 开发中 — Phase B of [K8s 全栈运维平台](https://github.com/290298661-pixel)

## 架构

```
NHW metrics ──┐
GFD metrics ──┼── Prometheus ── Grafana Dashboards
K8s metrics ──┘                    ├─ 节点健康总览
                                   ├─ 舰队运营面板
Loki ───────── Grafana Logs ───────├─ SLO 燃烧率
                                   └─ 日志浏览器
AlertManager ──→ 飞书/钉钉告警
```

## 目录

```
fleet-observability/
├── grafana/
│   ├── dashboards/      # Dashboard JSON (版本可控)
│   └── provisioning/    # 自动加载配置
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/           # 告警 + Recording 规则
│   └── scrape_configs/
├── loki/
│   ├── loki-config.yaml
│   └── promtail-config.yaml
└── docs/
    └── slo-definitions.md
```

## 快速开始

```bash
# 部署 Prometheus Stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack

# 导入 Grafana Dashboards
kubectl create configmap grafana-dashboards --from-file=grafana/dashboards/
```
