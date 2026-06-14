# ADR-001: 选择 Loki 而非 Elasticsearch 做日志聚合

| 字段 | 值 |
|------|-----|
| **状态** | 已采纳 |
| **日期** | 2026-06-14 |
| **决策者** | @290298661-pixel |
| **替代** | 无 |
| **被替代** | 无 |

---

## 背景

Phase B（Fleet Observability）需要一个日志聚合系统来采集、存储和查询来自三个 Phase 0 服务（NHW、GFD、Node Guardian）以及 Kubernetes 系统组件的日志。平台运行在本地 `kind` Kubernetes 集群上，资源有限（通常分配给 Docker 4-8 GB RAM）。

需要在以下方案中选择：

1. **ELK Stack**（Elasticsearch + Logstash + Kibana） — 行业标准，生态成熟
2. **Loki + Promtail** — Grafana 原生，轻量级，专为 Kubernetes 设计
3. **Graylog** — 中间选项

---

## 决策

**我们将使用 Loki + Promtail** 进行日志聚合。

---

## 评估标准

### 资源占用（关键 — 本地 kind 集群）

| | Elasticsearch | Loki |
|---|---|---|
| **最低 RAM** | 每节点 2-4 GB（3 节点 = 6-12 GB） | 单二进制 200 MB |
| **最低磁盘** | 推荐 20+ GB | 5 GB 足够 14 天留存 |
| **JVM 开销** | 有（需要堆调优） | 无（Go 二进制，无需 GC 调优） |
| **启动时间** | 30-60 秒 | < 5 秒 |

**结论：** Loki 是本地 kind 集群的唯一可行选择。运行哪怕单节点 Elasticsearch 也会消耗 50% 以上的可用 RAM。

### Kubernetes 集成

| | Elasticsearch | Loki |
|---|---|---|
| **K8s 服务发现** | 需要 Filebeat/Metricbeat 配置 | Promtail 原生 `kubernetes_sd_configs` |
| **K8s 元数据** | 通过 Filebeat processor 添加 | 自动注入 pod/namespace/container 标签 |
| **DaemonSet** | Filebeat 作为 DaemonSet | Promtail 作为 DaemonSet（轻量） |
| **Helm Chart** | ECK Operator（复杂） | `loki-stack` chart（简单） |

**结论：** Loki 的 Kubernetes 原生设计意味着更少的配置和更少的组件。Promtail 无需额外 processor 即可自动用 K8s 元数据丰富日志。

### Grafana 集成

| | Elasticsearch | Loki |
|---|---|---|
| **数据源** | 需要 Elasticsearch 插件 | 原生数据源（Grafana 内置） |
| **查询语言** | Lucene / Elasticsearch DSL | LogQL（与 PromQL 语法一致） |
| **指标 ↔ 日志关联** | 手动拼接 URL | 原生 `derivedFields` 链接 |
| **Dashboard 自动装载** | 标准 | 标准 |

**结论：** Loki 在开发体验上胜出。LogQL 与 PromQL 使用相同的思维模型，团队只需要学一种查询语言就能同时处理指标和日志。从 Prometheus 告警一键跳转到对应 Loki 日志的能力是显著的运维优势。

### 运维复杂度

| | Elasticsearch | Loki |
|---|---|---|
| **分片管理** | 手动或 ILM 策略 | 自动（tsdb/boltdb 索引分片） |
| **索引生命周期** | 需要 ILM 策略 | 配置文件中简单的留存时间 |
| **备份** | Snapshot API + 仓库配置 | 文件系统拷贝或对象存储 |
| **升级** | 滚动重启 + 分片再均衡 | 单二进制重启（< 5s 停机） |

**结论：** Loki 运维更简单。对于个人项目，运维负担的减少是显著的。

---

## 后果

### 正面

- **能在 kind 集群上运行**，不会挤占其他服务的资源
- **单一查询语言**（LogQL ≈ PromQL）处理指标和日志 — 降低认知负担
- **零依赖部署** — 一个二进制、一个配置文件
- **原生 Grafana 集成** — 无需额外插件，指标与日志自动关联
- **对项目规模足够好** — 不需要对日志正文做全文索引

### 负面

- **无全文索引** — 在大数据量下搜索日志正文比 Elasticsearch 慢。可接受，因为我们的日志量很小（< 100 MB/天）
- **无内置仪表盘** — Kibana 的日志探索比 Grafana Logs 面板更丰富。Grafana 10 改进的 Logs 面板可缓解
- **社区较小** — StackOverflow 答案比 ELK 少。活跃的 Grafana 社区和 Slack 可缓解

### 中性

- **未来迁移路径** — 如果项目超出 Loki 的能力范围（对展示项目来说不太可能），OpenTelemetry Collector 可以同时输出到 Loki 和 Elasticsearch，支持渐进式迁移

---

## 备选方案

### Elasticsearch + Filebeat + Kibana

**拒绝**，原因是资源需求。一个最小的生产级 Elasticsearch 集群需要 3 节点各 2 GB = 仅 Elasticsearch 就要 6 GB RAM。加上 Filebeat 和 Kibana 会超过 8 GB，kind 集群上没空间留给实际应用负载。

### Graylog

**拒绝**，因为它同样需要 Elasticsearch 做后端。同样的资源问题，还多一个 Java 服务（Graylog Server）要管理。

### Grafana Cloud Logs（SaaS）

**拒绝**，因为项目设计为完全本地运行（演示/面试时不依赖互联网）。SaaS 依赖会破坏"我现在就能演示给你看"的面试体验。

---

## 参考文献

- [Loki 文档](https://grafana.com/docs/loki/latest/)
- [LogQL: 日志查询语言](https://grafana.com/docs/loki/latest/logql/)
- [Elasticsearch 规模规划指南](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html)
- [Grafana Loki vs Elasticsearch 对比](https://grafana.com/blog/2022/03/21/how-to-choose-the-right-log-aggregation-tool/)
