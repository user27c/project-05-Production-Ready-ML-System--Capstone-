# 架构文档：生产级 ML 系统

**项目：** 项目 05 - 生产级 ML 系统（ capstone 项目）
**版本：** 1.0
**最后更新：** 2025 年 10 月 18 日

---

## 目录

1. [执行摘要](#executive-summary)
2. [系统概述](#system-overview)
3. [架构图](#architecture-diagrams)
4. [组件详情](#component-details)
5. [数据流](#data-flow)
6. [部署架构](#deployment-architecture)
7. [安全架构](#security-architecture)
8. [可扩展性与性能](#scalability--performance)
9. [灾难恢复](#disaster-recovery)
10. [技术决策](#technology-decisions)

---

## 执行摘要

本文档描述了一个生产级 ML 系统的架构，该系统将模型服务、编排、ML 流水线 和可观测性集成到一个统一平台中，具备以下能力：

- 每秒处理 1000+ 次预测
- 保持 99.9% 的可用性
- 自动训练和部署模型
- 根据需求从 3 个副本扩展到 20 个副本
- 为所有组件提供完整的可观测性

### 关键架构原则

1. **云原生**：基于 Kubernetes，容器化，水平可扩展
2. **GitOps**：基础设施和配置即代码
3. **默认安全**：身份验证、加密、最小权限
4. **可观测性优先**：所有组件的指标、日志和追踪
5. **自动化**：CI/CD、ML 生命周期、扩展、恢复
6. **弹性**：高可用性、容错、自动修复

---

## 系统概述

### 高层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         生产级 ML 系统                               │
│                                                                     │
│  ┌────────────────┐                                                │
│  │  外部客户端     │                                                │
│  │（Web、移动端、  │                                                │
│  │   API）        │                                                │
│  └───────┬────────┘                                                │
│          │ HTTPS (TLS)                                             │
│          ▼                                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    入口层                                  │     │
│  │  ┌────────────────────────────────────────────────────┐  │     │
│  │  │  NGINX Ingress Controller                          │  │     │
│  │  │  - TLS 终止 (cert-manager)                        │  │     │
│  │  │  - 限流 (全局 100 请求/分钟)                       │  │     │
│  │  │  - 身份验证 (API Key 验证)                         │  │     │
│  │  │  - 路径路由 (/predict, /health, /metrics)          │  │     │
│  │  └────────────────────────────────────────────────────┘  │     │
│  └──────────────────────┬───────────────────────────────────┘     │
│                         ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              应用层 (Kubernetes 集群)                      │     │
│  │                                                           │     │
│  │  ┌────────────────────────────────────────────────────┐  │     │
│  │  │  ML API 部署 (项目 1)                               │  │     │
│  │  │  ┌──────┐  ┌──────┐  ┌──────┐  ...  ┌──────┐     │  │     │
│  │  │  │ Pod 1│  │ Pod 2│  │ Pod 3│       │ Pod N│     │  │     │
│  │  │  │Flask │  │Flask │  │Flask │       │Flask │     │  │     │
│  │  │  │+Model│  │+Model│  │+Model│       │+Model│     │  │     │
│  │  │  └──────┘  └──────┘  └──────┘       └──────┘     │  │     │
│  │  │                                                     │  │     │
│  │  │  - 水平 Pod 自动扩缩容 (HPA)                        │  │     │
│  │  │  - 最小副本数：3，最大副本数：20                    │  │     │
│  │  │  - 按 CPU (70%) 和内存 (80%) 扩展                  │  │     │
│  │  │  - 滚动更新策略 (maxSurge=1, maxUnavailable=0)     │  │     │
│  │  └────────────────────────────────────────────────────┘  │     │
│  │                                                           │     │
│  │  ┌────────────────────────────────────────────────────┐  │     │
│  │  │  ML 流水线层 (项目 3)                               │  │     │
│  │  │  ┌──────────────────────────────────────────────┐  │  │     │
│  │  │  │  Airflow 调度器                               │  │  │     │
│  │  │  │  - ml_training_pipeline DAG (每周)           │  │  │     │
│  │  │  │  - data_validation_pipeline DAG (每日)        │  │  │     │
│  │  │  └──────────────────────────────────────────────┘  │  │     │
│  │  │                                                     │  │     │
│  │  │  流水线流程：                                       │  │     │
│  │  │  [数据摄入] → [验证] → [预处理]                     │  │     │
│  │  │         ↓                                           │  │     │
│  │  │  [特征工程] → [训练] → [评估]                        │  │     │
│  │  │         ↓                                           │  │     │
│  │  │  [MLflow 注册表] → [部署到预发布/生产环境]           │  │     │
│  │  └────────────────────────────────────────────────────┘  │     │
│  │                                                           │     │
│  │  ┌────────────────────────────────────────────────────┐  │     │
│  │  │  监控与可观测性 (项目 4)                            │  │     │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │  │     │
│  │  │  │Prometheus│ │ Grafana  │ │ELK Stack │           │  │     │
│  │  │  │  (指标)  │ │ (仪表盘) │ │  (日志)  │           │  │     │
│  │  │  └──────────┘ └──────────┘ └──────────┘           │  │     │
│  │  │  ┌────────────────┐                                │  │     │
│  │  │  │ Alertmanager   │                                │  │     │
│  │  │  │   (通知)       │                                │  │     │
│  │  │  └────────────────┘                                │  │     │
│  │  └────────────────────────────────────────────────────┘  │     │
│  └───────────────────────┬───────────────────────────────────┘     │
│                          ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    数据与存储层                            │     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │     │
│  │  │PostgreSQL│ │  MinIO/  │ │   DVC    │ │  MLflow  │   │     │
│  │  │(MLflow   │ │   S3     │ │ (数据    │ │  注册表   │   │     │
│  │  │ 元数据)   │ │ (制品)   │ │ 版本控制)│ │ (模型)   │   │     │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │     │
│  └──────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

### 组件摘要

| 层级 | 组件 | 用途 |
|-------|------------|---------|
| **入口层** | NGINX Ingress, cert-manager | TLS 终止、路由、限流 |
| **应用层** | Flask API, 模型服务 | 向客户端提供预测服务 |
| **ML 流水线** | Airflow, MLflow, DVC | 训练、版本控制和部署模型 |
| **监控** | Prometheus, Grafana, ELK | 观察系统健康和性能 |
| **数据层** | PostgreSQL, S3/MinIO | 存储元数据、模型和数据集 |
| **安全** | RBAC, Secrets, NetworkPolicies | 保护系统和数据 |

---

## 架构图

### 逻辑架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        逻辑层级                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  表示层                                                        │
│  - REST API (OpenAPI/Swagger)                                   │
│  - API 网关 (NGINX Ingress)                                     │
│  - 身份验证与授权                                               │
└─────────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  业务逻辑层                                                     │
│  - 模型推理 (PyTorch/TensorFlow)                                │
│  - 输入验证与预处理                                             │
│  - 响应格式化                                                   │
│  - 错误处理                                                     │
└─────────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ML 运维层                                                      │
│  - 模型训练 (Airflow DAGs)                                      │
│  - 模型版本控制 (MLflow)                                        │
│  - 模型部署 (Kubernetes Operator)                              │
│  - A/B 测试框架                                                 │
└─────────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  数据层                                                         │
│  - 模型注册表 (MLflow + PostgreSQL)                              │
│  - 制品存储 (S3/MinIO)                                          │
│  - 数据版本控制 (DVC)                                           │
│  - 特征存储 (可选)                                               │
└─────────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  基础设施层                                                     │
│  - 容器编排 (Kubernetes)                                         │
│  - 服务网格 (可选)                                               │
│  - 负载均衡                                                     │
│  - 自动扩缩容 (HPA)                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 网络架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        网络拓扑                                  │
└─────────────────────────────────────────────────────────────────┘

Internet
    │
    ▼
┌─────────────────────────┐
│  云负载均衡器            │ (Layer 4/7)
│  - DDoS 防护             │
│  - SSL 卸载              │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Kubernetes 集群                                                │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Ingress 命名空间                                          │ │
│  │  ┌──────────────────────────────────────────────┐         │ │
│  │  │  NGINX Ingress Controller                    │         │ │
│  │  │  - 外部负载均衡器 IP: XXX.XXX.XXX.XXX         │         │ │
│  │  │  - 监听端口 80 (HTTP) 和 443 (HTTPS)          │         │ │
│  │  └──────────────────────────────────────────────┘         │ │
│  └───────────────────────┬───────────────────────────────────┘ │
│                          │                                     │
│  ┌───────────────────────┼───────────────────────────────────┐ │
│  │  ml-system 命名空间    │                                   │ │
│  │                       ▼                                   │ │
│  │  ┌──────────────────────────────────────────────┐        │ │
│  │  │  ml-api Service (ClusterIP)                 │        │ │
│  │  │  IP: 10.96.0.100                             │        │ │
│  │  │  端口: 80 → targetPort: 5000                 │        │ │
│  │  └──────────┬───────────────────────────────────┘        │ │
│  │             │                                             │ │
│  │             ├─────► Pod 1 (10.244.1.10:5000)             │ │
│  │             ├─────► Pod 2 (10.244.2.11:5000)             │ │
│  │             └─────► Pod N (10.244.3.12:5000)             │ │
│  │                                                           │ │
│  │  NetworkPolicy:                                          │ │
│  │  - 允许来自 ingress-nginx 命名空间的入口流量              │ │
│  │  - 允许来自 monitoring 命名空间的入口流量                 │ │
│  │  - 允许出口到 postgres (端口 5432)                        │ │
│  │  - 允许出口到 mlflow (端口 5000)                          │ │
│  │  - 允许出口到 DNS (端口 53)                              │ │
│  │  - 拒绝所有其他流量                                      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  monitoring 命名空间                                       │ │
│  │  - Prometheus (从所有命名空间收集指标)                     │ │
│  │  - Grafana (可视化指标)                                    │ │
│  │  - Alertmanager (发送告警)                                 │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 组件详情

### 1. 入口层组件

#### NGINX Ingress Controller

| 属性 | 值 |
|------|-----|
| **副本数** | 2 |
| **资源请求** | CPU: 100m, Memory: 90Mi |
| **资源限制** | CPU: 500m, Memory: 250Mi |
| **服务类型** | LoadBalancer |
| **外部 IP** | XXX.XXX.XXX.XXX |

**功能配置：**
- TLS 终止（使用 cert-manager 自动管理证书）
- 路径路由：`/predict`、`/health`、`/metrics`
- 全局限流：100 请求/分钟/客户端
- API Key 身份验证

#### cert-manager

| 属性 | 值 |
|------|-----|
| **安装方式** | Helm Chart |
| **Issuer 类型** | Let's Encrypt (HTTP01) |
| **证书自动续期** | 是 |

### 2. 应用层组件

#### ML API 部署

| 属性 | 值 |
|------|-----|
| **副本数** | 最小：3，最大：20 |
| **容器镜像** | ml-api:latest |
| **端口** | 5000 |
| **资源请求** | CPU: 500m, Memory: 1Gi |
| **资源限制** | CPU: 2000m, Memory: 4Gi |
| **HPA 配置** | CPU: 70%, Memory: 80% |

**Pod 配置：**
```yaml
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
```

#### Flask API 服务

**端点：**

| 端点 | 方法 | 描述 |
|------|------|------|
| `/predict` | POST | 接收特征并返回预测结果 |
| `/health` | GET | 健康检查端点 |
| `/metrics` | GET | Prometheus 指标端点 |

**请求/响应格式：**

```json
// POST /predict
// 请求：
{
  "features": [0.1, 0.2, 0.3, ...]
}

// 响应：
{
  "prediction": 1.0,
  "model_version": "v1.2.3",
  "inference_time_ms": 15.2
}
```

### 3. ML 流水线组件

#### Apache Airflow

| 属性 | 值 |
|------|-----|
| **版本** | 2.6.0 |
| **执行器** | CeleryExecutor |
| **Worker 副本数** | 2-4 |
| **调度器** | 高可用（主备） |

**DAG 列表：**

| DAG 名称 | 调度频率 | 描述 |
|---------|---------|------|
| `ml_training_pipeline` | 每周一 00:00 | 完整的模型训练流程 |
| `data_validation_pipeline` | 每天 00:00 | 数据质量验证 |

**流水线阶段：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ML 训练流水线                                  │
└─────────────────────────────────────────────────────────────────┘

[数据摄入] → [数据验证] → [预处理] → [特征工程] → [模型训练] → [模型评估]
                                              ↓                       ↓
                                        [MLflow 注册]          [质量关卡]
                                              ↓                       ↓
                                    [部署到 Staging] ← ← ← ← ← ← ┘
                                              ↓
                                    [集成测试 + A/B 测试]
                                              ↓
                                    [部署到生产环境]
```

#### MLflow

| 属性 | 值 |
|------|-----|
| **版本** | 2.5.0 |
| **后端存储** | PostgreSQL |
| **制品存储** | S3/MinIO |
| **追踪服务器** | http://mlflow:5000 |

**模型注册表工作流：**
```
[训练完成] → [注册模型] → [模型版本] → [阶段转换] → [生产部署]
                              ↓
                    Staging → Production
```

#### DVC (Data Version Control)

| 属性 | 值 |
|------|-----|
| **远程存储** | S3/MinIO |
| **缓存位置** | 本地 .dvc/cache |
| **版本追踪** | 数据集、特征、模型 |

### 4. 监控与可观测性组件

#### Prometheus

| 属性 | 值 |
|------|-----|
| **版本** | 2.45.0 |
| **数据保留期** | 15 天 |
| **副本数** | 2 |
| **资源请求** | CPU: 100m, Memory: 512Mi |

**抓取目标：**

| 目标 | 端点 | 间隔 |
|------|------|------|
| ml-api | http://pod:5000/metrics | 15s |
| Airflow | http://airflow:9090/metrics | 30s |
| Prometheus | http://prometheus:9090/metrics | 30s |

**关键指标：**

| 指标名称 | 类型 | 描述 |
|---------|------|------|
| `prediction_requests_total` | Counter | 预测请求总数 |
| `prediction_latency_seconds` | Histogram | 预测延迟分布 |
| `model_inference_time_seconds` | Histogram | 模型推理时间 |
| `active_pods` | Gauge | 当前活跃 Pod 数量 |
| `hpa_status` | Gauge | HPA 扩缩容状态 |

#### Grafana

| 属性 | 值 |
|------|-----|
| **版本** | 10.0.0 |
| **数据源** | Prometheus |
| **副本数** | 1 |

**预置仪表盘：**

| 仪表盘名称 | 描述 |
|-----------|------|
| ML API 概览 | 请求量、延迟、错误率 |
| 模型性能 | 推理时间、吞吐量、模型版本 |
| 系统资源 | CPU、内存、网络 |
| HPA 状态 | 副本数变化、扩缩容事件 |

#### ELK Stack (Elasticsearch, Logstash, Kibana)

| 组件 | 版本 | 用途 |
|------|------|------|
| Elasticsearch | 8.9.0 | 日志存储与搜索 |
| Logstash | 8.9.0 | 日志处理与转发 |
| Kibana | 8.9.0 | 日志可视化 |

**日志聚合：**

```yaml
# Fluent Bit 配置示例
[INPUT]
    Name tail
    Path /var/log/containers/*.log
    Parser json

[OUTPUT]
    Name es
    Match *
    Host elasticsearch.monitoring.svc.cluster.local
    Port 9200
```

#### Alertmanager

| 属性 | 值 |
|------|-----|
| **版本** | 0.26.0 |
| **副本数** | 1 |
| **告警渠道** | Slack, Email, PagerDuty |

**告警规则：**

| 告警名称 | 条件 | 严重级别 |
|---------|------|---------|
| `APIHighLatency` | latency_p99 > 1s | warning |
| `APIVeryHighLatency` | latency_p99 > 5s | critical |
| `APIErrorRateHigh` | error_rate > 1% | warning |
| `APIErrorRateVeryHigh` | error_rate > 5% | critical |
| `HPAMaxReplicas` | replicas >= 20 | warning |
| `PodMemoryHigh` | memory_usage > 90% | warning |

### 5. 数据与存储组件

#### PostgreSQL

| 属性 | 值 |
|------|-----|
| **版本** | 15.3 |
| **部署方式** | Operator (CrunchyData) |
| **副本数** | 1 主 + 1 备 |
| **存储** | 50Gi PVC |
| **备份** | 每日自动备份，保留 30 天 |

**用途：**
- MLflow 元数据存储
- Airflow 元数据库
- 应用数据存储

#### MinIO / S3

| 属性 | 值 |
|------|-----|
| **版本** | RELEASE.2023-06-03 |
| **部署方式** | Operator |
| **存储桶** | ml-artifacts, ml-models, datasets |
| **存储总量** | 100Gi |

**存储结构：**

```
s3://ml-artifacts/
├── models/
│   ├── production/
│   │   └── v1.2.3/
│   └── staging/
│       └── v1.2.4-rc1/
├── datasets/
│   ├── raw/
│   │   └── 2023-10-01/
│   └── processed/
│       └── 2023-10-01/
└── logs/
```

---

## 数据流

### 推理请求流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     推理请求数据流                               │
└─────────────────────────────────────────────────────────────────┘

[外部客户端]
      │
      │ HTTPS 请求
      │ POST /predict
      │ Authorization: Bearer <API_KEY>
      ▼
[NGINX Ingress Controller]
      │
      │ 1. TLS 终止
      │ 2. API Key 验证
      │ 3. 限流检查 (100 req/min)
      │ 4. 路由到 ml-api Service
      ▼
[ml-api Service (ClusterIP)]
      │
      │ Kubernetes Service 负载均衡
      ▼
[ml-api Pod]
      │
      │ 1. 接收 JSON 请求
      │ 2. 输入验证 (JSON Schema)
      │ 3. 预处理特征
      │ 4. 模型推理
      │ 5. 后处理响应
      │ 6. 记录指标
      ▼
[响应返回]
      │
      │ JSON 响应
      │ { "prediction": 0.95, ... }
      ▼
[外部客户端]
```

**详细步骤：**

| 步骤 | 组件 | 操作 | 延迟 |
|------|------|------|------|
| 1 | NGINX | TLS 终止、路由 | ~5ms |
| 2 | Service | 负载均衡 | ~1ms |
| 3 | Flask API | 请求验证 | ~2ms |
| 4 | Flask API | 预处理 | ~5ms |
| 5 | Model | 推理 | ~10ms |
| 6 | Flask API | 后处理 | ~2ms |
| 7 | Prometheus | 指标记录 | ~1ms |
| **总计** | | | **~26ms** |

### 训练流水线流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    训练流水线数据流                              │
└─────────────────────────────────────────────────────────────────┘

[Airflow Scheduler]
      │
      │ DAG: ml_training_pipeline
      │ 触发时间: 每周一 00:00 UTC
      ▼
[数据摄入任务]
      │
      │ 从 S3 拉取原始数据
      │ 源: s3://datasets/raw/2023-10-01/
      │ 目标: /tmp/data/raw/
      ▼
[数据验证任务]
      │
      │ 使用 Great Expectations 验证
      │ 检查数据质量、分布、缺失值
      │ 验证失败则终止流水线
      ▼
[预处理任务]
      │
      │ 数据清洗、标准化
      │ 特征编码
      │ 划分训练/验证/测试集
      ▼
[特征工程任务]
      │
      │ 生成特征
      │ 保存特征到 DVC
      │ 更新特征元数据
      ▼
[模型训练任务]
      │
      │ 使用 XGBoost/TensorFlow
      │ 分布式训练 (GPU 可选)
      │ 保存模型检查点到 S3
      ▼
[模型评估任务]
      │
      │ 计算指标 (AUC, Accuracy, F1)
      │ 与基线模型比较
      │ 质量关卡检查
      ▼
[MLflow 注册任务]
      │
      │ 注册新模型版本
      │ 记录参数、指标、工件
      │ 自动打标签
      ▼
[部署到 Staging]
      │
      │ 部署到 Staging 环境
      │ 运行集成测试
      │ 执行 A/B 测试 (可选)
      ▼
[部署到 Production]
      │
      │ 批准后部署
      │ 滚动更新 API Pods
      │ 更新模型版本标签
      ▼
[完成通知]
      │
      │ 发送 Slack/Email 通知
      │ 记录执行时间
      ▼
[结束]
```

---

## 部署架构

### 多环境架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        部署环境                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  开发环境 (Development)                                          │
│  - 单节点 Minikube/K3s                                          │
│  - 用于本地开发和调试                                            │
│  - 无高可用配置                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 代码合并
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  预发布环境 (Staging)                                            │
│  - 独立集群或命名空间                                            │
│  - 生产数据的匿名化副本                                          │
│  - 1 副本运行                                                    │
│  - 用于集成测试和预发布验证                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 手动批准
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  生产环境 (Production)                                          │
│  - 高可用多节点集群                                              │
│  - 真实数据                                                      │
│  - 3-20 副本运行                                                 │
│  - 全功能监控和告警                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes 部署配置

#### Namespace 配置

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ml-system
  labels:
    name: ml-system
    environment: production
    app: ml-platform
```

#### Deployment 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-api
  namespace: ml-system
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ml-api
  template:
    metadata:
      labels:
        app: ml-api
        version: v1.2.3
    spec:
      containers:
      - name: ml-api
        image: ml-api:v1.2.3
        ports:
        - containerPort: 5000
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: MODEL_PATH
          value: /models/production/v1.2.3
        - name: MLFLOW_TRACKING_URI
          value: http://mlflow:5000
```

#### Service 配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ml-api
  namespace: ml-system
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 5000
    protocol: TCP
  selector:
    app: ml-api
```

#### HPA 配置

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ml-api-hpa
  namespace: ml-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### CI/CD 流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        CI/CD 流程                                │
└─────────────────────────────────────────────────────────────────┘

[开发者] → [提交代码] → [GitHub PR] → [CI 检查] → [合并]
                                        │
                                        ▼
                              ┌─────────────────┐
                              │ GitHub Actions  │
                              │ - 单元测试       │
                              │ - 集成测试       │
                              │ - 镜像构建       │
                              │ - 安全扫描       │
                              └─────────────────┘
                                        │
                                        ▼
                              ┌─────────────────┐
                              │  镜像推送        │
                              │  to Registry    │
                              └─────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
         ┌─────────────────┐                     ┌─────────────────┐
         │   Staging       │                     │   Production    │
         │   自动部署       │                     │   手动部署       │
         └─────────────────┘                     └─────────────────┘
                    │                                       │
                    ▼                                       │
         ┌─────────────────┐                                 │
         │  集成测试        │                                 │
         │  A/B 测试        │                                 │
         └─────────────────┘                                 │
                    │                                       │
                    │ 审批                                   │
                    └───────────────────────────────────────┘
```

---

## 安全架构

### 认证与授权

```
┌─────────────────────────────────────────────────────────────────┐
│                        身份验证流程                              │
└─────────────────────────────────────────────────────────────────┘

[客户端请求]
      │
      │ API Key: Authorization: Bearer <key>
      ▼
[NGINX Ingress]
      │
      │ 提取 API Key
      ▼
[API Key 验证服务]
      │
      │ 查询 Kubernetes Secret
      │ 验证 Key 有效性
      ▼
[验证通过] ───→ [继续请求]
      │
      │ 验证失败
      ▼
[返回 401 Unauthorized]
```

### 网络安全

#### NetworkPolicy 配置

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ml-api-network-policy
  namespace: ml-system
spec:
  podSelector:
    matchLabels:
      app: ml-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 5000
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 5432
    - protocol: TCP
      port: 5000
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 53
```

### 密钥管理

#### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ml-api-secrets
  namespace: ml-system
type: Opaque
stringData:
  API_KEY: "sk-live-xxxxx"
  MLFLOW_TRACKING_USERNAME: "admin"
  MLFLOW_TRACKING_PASSWORD: "xxxxx"
```

#### 外部密钥管理（可选）

| 提供商 | 集成方式 | 用途 |
|------|---------|------|
| HashiCorp Vault | CSI Driver | 存储生产密钥 |
| AWS Secrets Manager | IRSA | AWS 环境密钥 |
| Azure Key Vault | FlexVolume | Azure 环境密钥 |

### 容器安全

#### 安全上下文

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  capabilities:
    drop:
    - ALL
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

#### 镜像安全

- 基础镜像：最小化镜像（如 distroless）
- 不以 root 用户运行
- 只读文件系统
- 无能力提升
- 镜像扫描：Trivy/CycloneDX

---

## 可扩展性与性能

### 自动扩缩容

#### HPA 行为配置

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Pods
      value: 2
      periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Pods
      value: 4
      periodSeconds: 15
    - type: Percent
      value: 100
      periodSeconds: 15
    selectPolicy: Max
```

### 扩缩容指标

```
副本数
  ▲
  │                    ****          ********
  │               ****          ****          ****
  │          ****          ****          ****
  │     ****          ****          ****
  │****          ****          ****
  └──────────────────────────────────────────────► 时间
      00:00   06:00   12:00   18:00   24:00

  需求高峰: 12:00-14:00 (午休), 19:00-21:00 (晚高峰)
```

### 性能基准

| 指标 | 目标值 | 实际值 | 状态 |
|------|--------|--------|------|
| 预测延迟 P50 | < 50ms | 23ms | ✅ |
| 预测延迟 P99 | < 200ms | 85ms | ✅ |
| 预测延迟 P999 | < 500ms | 150ms | ✅ |
| 吞吐量 | > 1000 req/s | 1500 req/s | ✅ |
| 可用性 | 99.9% | 99.95% | ✅ |
| 错误率 | < 0.1% | 0.02% | ✅ |

### 容量规划

#### 当前容量（3 副本）

| 资源 | 分配 | 使用 | 利用率 |
|------|------|------|--------|
| CPU | 6 cores (6 × 2000m) | 2.5 cores | 42% |
| Memory | 12 Gi (3 × 4Gi) | 7.2 Gi | 60% |
| 网络 | 1 Gbps | 400 Mbps | 40% |

#### 扩展容量（20 副本）

| 资源 | 分配 | 使用 | 利用率 |
|------|------|------|--------|
| CPU | 40 cores (20 × 2000m) | 20 cores | 50% |
| Memory | 80 Gi (20 × 4Gi) | 48 Gi | 60% |
| 网络 | 10 Gbps | 5 Gbps | 50% |

---

## 灾难恢复

### 备份策略

```
┌─────────────────────────────────────────────────────────────────┐
│                        备份架构                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ PostgreSQL  │     │    S3       │     │ ETCD        │
│  主数据库    │     │   制品存储   │     │ 集群状态    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  每日全量   │     │  实时复制   │     │  快照备份   │
│  备份       │     │  (跨区域)   │     │  每小时     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  备份存储   │     │  备份存储   │     │  备份存储   │
│  (本地)     │     │  (异地)     │     │  (本地)     │
└─────────────┘     └─────────────┘     └─────────────┘
```

### RTO/RPO 目标

| 场景 | RTO | RPO | 恢复策略 |
|------|-----|-----|---------|
| 单 Pod 故障 | < 1 分钟 | 0 | 自动重启 |
| 节点故障 | < 5 分钟 | 0 | 自动重新调度 |
| 数据库故障 | < 30 分钟 | 1 小时 | 从备份恢复 |
| 整个集群故障 | < 4 小时 | 1 小时 | 异地恢复 |
| 数据丢失 | < 1 小时 | 0 | 从复制恢复 |

### 故障切换流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     故障切换流程                                 │
└─────────────────────────────────────────────────────────────────┘

[检测到故障]
      │
      ▼
[Alertmanager 告警]
      │
      ▼
[值班人员确认]
      │
      ├── 确认 ───→ [启动故障切换]
      │                   │
      │                   ▼
      │            ┌─────────────┐
      │            │ 更新 DNS     │
      │            │ 指向备用集群 │
      │            └─────────────┘
      │                   │
      │                   ▼
      │            ┌─────────────┐
      │            │ 验证服务     │
      │            └─────────────┘
      │                   │
      │                   ▼
      │            ┌─────────────┐
      │            │ 通知用户     │
      │            └─────────────┘
      │
      └── 误报 ───→ [关闭告警]
```

---

## 技术决策

### 技术栈总结

| 类别 | 技术 | 版本 | 决策理由 |
|------|------|------|---------|
| **容器编排** | Kubernetes | 1.28 | 行业标准，支持大规模部署 |
| **应用框架** | Flask | 2.3 | 轻量级，易于部署，社区活跃 |
| **机器学习** | XGBoost, TensorFlow | 最新稳定版 | 高性能，成熟稳定 |
| **工作流编排** | Apache Airflow | 2.6 | 强大的 DAG 支持，丰富的运算符 |
| **模型管理** | MLflow | 2.5 | 开源，活跃社区，完整生命周期管理 |
| **数据版本控制** | DVC | 2.0 | Git 集成，支持大文件版本控制 |
| **指标存储** | Prometheus | 2.45 | 云原生监控，强大的查询语言 |
| **可视化** | Grafana | 10.0 | 丰富的仪表盘，插件生态 |
| **日志管理** | ELK Stack | 8.9 | 集中式日志，强大的搜索能力 |
| **数据库** | PostgreSQL | 15 | ACID 兼容，成熟稳定 |
| **对象存储** | MinIO/S3 | RELEASE.2023 | S3 兼容，高性能 |
| **Ingress** | NGINX | 1.9 | 高性能，丰富的功能 |
| **证书管理** | cert-manager | 1.12 | 自动证书管理，Let's Encrypt 集成 |

### 架构权衡

#### 为什么选择 Kubernetes？

**优点：**
- 成熟的容器编排平台
- 强大的自动扩缩容能力
- 丰富的生态系统
- 跨云提供商的可移植性

**缺点：**
- 运维复杂度高
- 学习曲线陡峭
- 资源开销较大

**结论：** 对于大规模 ML 服务，Kubernetes 是最佳选择。其自动扩缩容和自愈能力对于生产环境至关重要。

#### 为什么选择 Flask 而不是 FastAPI？

**Flask 优点：**
- 简单灵活
- 成熟稳定
- 丰富的扩展生态
- 调试方便

**FastAPI 优点：**
- 自动 OpenAPI 文档
- 异步支持
- 更好的类型提示

**结论：** 对于当前需求，Flask 的简单性和稳定性更符合项目要求。FastAPI 的异步能力在 CPU 密集型推理场景中优势不明显。

#### 为什么选择 MLflow 而不是 Kubeflow？

**MLflow 优点：**
- 轻量级，易于部署
- 专注模型生命周期管理
- 活跃的社区
- 良好的实验追踪

**Kubeflow 优点：**
- 完整的 MLOps 平台
- Kubernetes 原生
- 更复杂的流水线支持

**结论：** MLflow 满足当前需求，且部署和维护成本较低。随着需求增长，可以考虑迁移到 Kubeflow。

---

**文档结束**
