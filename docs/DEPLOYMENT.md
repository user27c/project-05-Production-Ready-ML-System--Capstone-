# 部署指南：生产级 ML 系统

**最后更新：** 2025 年 10 月 18 日
**版本：** 1.0

---

## 目录

1. [前置条件](#前置条件)
2. [本地开发设置](#本地开发设置)
3. [Staging 部署](#staging-部署)
4. [生产部署](#生产部署)
5. [回滚程序](#回滚程序)
6. [故障排除](#故障排除)

---

## 前置条件

### 必需工具

部署前请安装以下工具：

```bash
# kubectl（Kubernetes CLI）
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Helm（Kubernetes 包管理器）
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# kustomize（可选，已内置于 kubectl）
kubectl version --client

# Docker
# 按照说明安装：https://docs.docker.com/get-docker/
```

### 所需访问权限

确保您拥有：

- [ ] Kubernetes 集群访问权限（kubeconfig 文件）
- [ ] 容器仓库凭证（ghcr.io、ECR、GCR）
- [ ] 已配置域名（用于生产环境）
- [ ] DNS 访问权限（创建 A 记录）
- [ ] GitHub 仓库访问权限
- [ ] 密钥（API 密钥、凭证）

### 基础设施要求

**开发环境：**
- Minikube 或 kind
- 最少 8GB RAM
- 20GB 磁盘空间

**Staging：**
- Kubernetes 集群（3 个节点）
- 节点规格：8GB RAM、4 CPU
- 支持 LoadBalancer
- 持久存储（50GB）

**生产环境：**
- Kubernetes 集群（5+ 个节点）
- 节点规格：16GB RAM、8 CPU
- 多可用区部署
- 带有 DDoS 防护的 LoadBalancer
- 持久存储（200GB）

---

## 本地开发设置

### 步骤 1：启动本地 Kubernetes

```bash
# 使用 Minikube
minikube start --memory=8192 --cpus=4

# 使用 kind
kind create cluster --config kind-config.yaml
```

### 步骤 2：构建 Docker 镜像

```bash
# 构建镜像
docker build -t ml-api:dev .

# 加载镜像到 Minikube（如果使用 Minikube）
minikube image load ml-api:dev

# 或推送到仓库
docker tag ml-api:dev ghcr.io/your-org/ml-api:dev
docker push ghcr.io/your-org/ml-api:dev
```

### 步骤 3：部署到本地 Kubernetes

```bash
# 使用 kustomize 部署
kubectl apply -k kubernetes/overlays/dev/

# 验证部署
kubectl get pods -n ml-system-dev
kubectl get svc -n ml-system-dev

# 等待 pod 就绪
kubectl wait --for=condition=Ready pods -l app=ml-api -n ml-system-dev --timeout=5m
```

### 步骤 4：访问本地应用

```bash
# 端口转发以进行本地访问
kubectl port-forward -n ml-system-dev svc/ml-api 5000:80

# 在另一个终端中测试
curl http://localhost:5000/health
```

### 步骤 5：查看日志

```bash
# 查看 pod 日志
kubectl logs -n ml-system-dev -l app=ml-api --tail=100 -f

# 描述 pod 以进行故障排除
kubectl describe pod -n ml-system-dev -l app=ml-api
```

---

## Staging 部署

Staging 部署在代码合并到 `develop` 分支时通过 CI/CD **自动**进行。

### 手动 Staging 部署

如需手动部署：

#### 步骤 1：配置访问权限

```bash
# 设置 staging 集群的 kubeconfig
export KUBECONFIG=/path/to/staging-kubeconfig.yaml

# 验证集群访问
kubectl cluster-info
kubectl get nodes
```

#### 步骤 2：创建命名空间

```bash
# 创建命名空间（如果不存在）
kubectl create namespace ml-system-staging

# 验证
kubectl get namespace ml-system-staging
```

#### 步骤 3：创建 Secrets

```bash
# 创建 secrets（绝对不要提交到 Git！）
kubectl create secret generic api-keys \
  --from-literal=API_KEY=staging-api-key-here \
  -n ml-system-staging

kubectl create secret generic mlflow-credentials \
  --from-literal=password=mlflow-password-here \
  -n ml-system-staging
```

**更好的方法：使用 SealedSecrets**

```bash
# 安装 SealedSecrets 控制器
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# 创建 sealed secret
kubectl create secret generic api-keys \
  --from-literal=API_KEY=staging-api-key-here \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# 将 sealed-secret.yaml 提交到 Git（已加密）
kubectl apply -f sealed-secret.yaml
```

#### 步骤 4：安装依赖

```bash
# 安装 NGINX Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# 安装 cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# 验证安装
kubectl get pods -n ingress-nginx
kubectl get pods -n cert-manager
```

#### 步骤 5：配置 DNS

```bash
# 获取 LoadBalancer IP
kubectl get svc ingress-nginx-controller -n ingress-nginx

# 创建 DNS A 记录，将 staging.example.com 指向 LoadBalancer IP
#（在您的 DNS 提供商处完成：GoDaddy、Cloudflare、Route53 等）
```

#### 步骤 6：部署应用

```bash
# 使用 kustomize 部署
kubectl apply -k kubernetes/overlays/staging/

# 观察滚动更新
kubectl rollout status deployment/ml-api -n ml-system-staging

# 验证 pod
kubectl get pods -n ml-system-staging
```

#### 步骤 7：验证证书

```bash
# 等待证书就绪
kubectl wait --for=condition=Ready certificate/staging-tls \
  -n ml-system-staging --timeout=5m

# 检查证书状态
kubectl get certificate -n ml-system-staging
kubectl describe certificate staging-tls -n ml-system-staging

# 检查 ingress
kubectl get ingress -n ml-system-staging
```

#### 步骤 8：测试部署

```bash
# 测试健康端点
curl https://staging.example.com/health

# 使用 API 密钥测试
curl -H "X-API-Key: staging-api-key-here" \
  https://staging.example.com/info

# 运行集成测试
export API_URL=https://staging.example.com
export API_KEY=staging-api-key-here
pytest tests/integration/test_e2e.py -v
```

---

## 生产部署

生产部署需要**手动批准**并使用**金丝雀部署**策略。

### 部署前检查清单

部署到生产环境之前：

- [ ] 所有测试在 staging 环境下通过
- [ ] 安全扫描已完成（无严重/高危漏洞）
- [ ] 负载测试成功完成
- [ ] 回滚程序已测试
- [ ] 灾难恢复计划已审查
- [ ] 监控和告警已配置
- [ ] 值班团队已通知
- [ ] 变更请求已批准（如需要）
- [ ] 当前生产备份已验证
- [ ] 团队准备好部署时间窗口

### 生产部署步骤

#### 步骤 1：触发 CD 流水线

生产部署通过 GitHub Actions 工作流触发：

```bash
# 导航到仓库中的 GitHub Actions
# 选择 "CD Pipeline" 工作流
# 点击 "Run workflow"
# 选择：
#   - environment: production
#   - image_tag: v1.2.3（特定版本，绝对不要用 'latest'）
# 点击 "Run workflow"
```

**需要手动批准：**
- GitHub 将暂停并请求批准
- 团队负责人/经理必须批准
- 批准门控确保生产安全

#### 步骤 2：监控金丝雀部署

一旦批准，流水线将：

1. 部署金丝雀 pod（10% 流量）
2. 监控 10 分钟：
   - 错误率 < 0.1%
   - P95 延迟 < 500ms
   - 无崩溃

#### 步骤 3：升级或回滚

如果金丝雀健康：
- 流量逐渐转移：10% → 25% → 50% → 100%
- 完全滚动更新完成

如果金丝雀不健康：
- 自动回滚
- 向团队发送告警
- 创建事件

### 手动生产部署

如需手动部署（紧急情况）：

#### 步骤 1：配置访问权限

```bash
# 设置生产集群的 kubeconfig
export KUBECONFIG=/path/to/production-kubeconfig.yaml

# 验证集群访问
kubectl cluster-info
kubectl get nodes
```

#### 步骤 2：备份当前状态

**关键：生产变更前务必备份！**

```bash
# 备份当前部署
kubectl get all -n ml-system-production -o yaml > \
  backup-production-$(date +%Y%m%d-%H%M%S).yaml

# 备份 ConfigMaps 和 Secrets
kubectl get configmap -n ml-system-production -o yaml >> \
  backup-production-$(date +%Y%m%d-%H%M%S).yaml
kubectl get secret -n ml-system-production -o yaml >> \
  backup-production-$(date +%Y%m%d-%H%M%S).yaml

# 将备份存储到安全位置
aws s3 cp backup-production-*.yaml s3://your-backup-bucket/
```

#### 步骤 3：部署金丝雀

```bash
# 部署金丝雀（10% 流量）
helm upgrade ml-system ./helm/ml-system \
  --namespace ml-system-production \
  --values ./helm/ml-system/values-production.yaml \
  --set api.image.tag=v1.2.3 \
  --set api.canary.enabled=true \
  --set api.canary.weight=10 \
  --wait \
  --timeout 10m
```

#### 步骤 4：监控金丝雀

```bash
# 观察金丝雀 pod 日志
kubectl logs -n ml-system-production -l version=canary -f

# 监控 Grafana 仪表板
# 检查错误率、延迟、吞吐量

# 查询 Prometheus 获取错误率
# 错误率应 < 0.1%
```

**监控 10-15 分钟**

如果错误率增加或延迟激增，**立即回滚**。

#### 步骤 5：升级金丝雀

如果指标健康：

```bash
# 完全滚动更新
helm upgrade ml-system ./helm/ml-system \
  --namespace ml-system-production \
  --values ./helm/ml-system/values-production.yaml \
  --set api.image.tag=v1.2.3 \
  --set api.canary.enabled=false \
  --wait \
  --timeout 15m

# 验证滚动更新
kubectl rollout status deployment/ml-api -n ml-system-production
```

#### 步骤 6：验证生产环境

```bash
# 运行冒烟测试
curl https://api.example.com/health

export API_URL=https://api.example.com
export API_KEY=$PRODUCTION_API_KEY
pytest tests/integration/test_e2e.py -v --skip-slow

# 至少监控 1 小时
# 观察 Grafana，检查日志，验证指标
```

#### 步骤 7：标记部署

```bash
# 标记 Git 提交
git tag -a "prod-v1.2.3" -m "Production deployment v1.2.3"
git push origin "prod-v1.2.3"

# 更新 MLflow（将模型标记为 Production）
#（运行您的模型标记脚本）
```

---

## 回滚程序

### 何时回滚

立即回滚如果：
- 错误率 > 1%
- P95 延迟 > 1 秒
- Pod 崩溃
- 关键功能损坏
- 发现安全问题

### 自动回滚

如果金丝雀部署失败，CD 流水线会自动回滚。

### 手动回滚（Helm）

```bash
# 列出发布
helm list -n ml-system-production

# 回滚到上一个发布
helm rollback ml-system -n ml-system-production

# 回滚到特定版本
helm rollback ml-system 5 -n ml-system-production

# 验证回滚
kubectl rollout status deployment/ml-api -n ml-system-production
```

### 手动回滚（kubectl）

```bash
# 回滚部署
kubectl rollout undo deployment/ml-api -n ml-system-production

# 回滚到特定版本
kubectl rollout undo deployment/ml-api --to-revision=3 -n ml-system-production

# 检查滚动更新历史
kubectl rollout history deployment/ml-api -n ml-system-production
```

### 从备份回滚

如果 Helm/kubectl 回滚失败：

```bash
# 从备份恢复
kubectl apply -f backup-production-YYYYMMDD-HHMMSS.yaml

# 验证恢复
kubectl get pods -n ml-system-production
```

### 回滚后

回滚后：

1. **通知团队** - 通过 Slack/PagerDuty 告警
2. **创建事件** - 记录问题
3. **检查日志** - 找出根本原因
4. **修复问题** - 解决问题
5. **在 staging 测试** - 验证修复
6. **计划重新部署** - 安排下一次尝试

---

## 故障排除

### Pod 未启动

```bash
# 检查 pod 状态
kubectl get pods -n ml-system-production

# 描述 pod
kubectl describe pod <pod-name> -n ml-system-production

# 检查事件
kubectl get events -n ml-system-production --sort-by='.lastTimestamp'
```

**常见原因：**
- 镜像拉取错误（标签错误、仓库认证）
- 资源限制（CPU/内存不足）
- 配置错误（环境变量错误）
- Secret 缺失

### 证书未颁发

```bash
# 检查证书状态
kubectl describe certificate <cert-name> -n ml-system-production

# 检查 cert-manager 日志
kubectl logs -n cert-manager deploy/cert-manager -f

# 检查 challenges
kubectl get challenges -n ml-system-production
```

**常见原因：**
- DNS 未指向 LoadBalancer
- 防火墙阻止端口 80（HTTP-01 挑战）
- 速率限制（请先使用 staging！）

### 高错误率

```bash
# 检查应用日志
kubectl logs -n ml-system-production -l app=ml-api --tail=200

# 检查 Grafana 仪表板
# 打开浏览器访问 Grafana URL

# 查询 Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# 打开 http://localhost:9090
```

**常见原因：**
- 模型加载失败
- 数据库连接问题
- 资源耗尽
- 依赖不可用

### 部署卡住

```bash
# 检查滚动更新状态
kubectl rollout status deployment/ml-api -n ml-system-production

# 如果卡住则强制回滚
kubectl rollout undo deployment/ml-api -n ml-system-production

# 删除卡住的 pod
kubectl delete pod <pod-name> -n ml-system-production --force --grace-period=0
```

---

## 最佳实践

1. **绝不在周五部署**（或长假前）
2. **生产变更前务必备份**
3. **在 staging 测试后再到生产**
4. **部署期间和之后持续监控**
5. **部署前准备好回滚计划**
6. **生产环境使用金丝雀或蓝绿部署**
7. **在 Git 中标记部署**
8. **在发布说明中记录变更**
9. **部署前通知团队**
10. **部署后 24 小时内审查指标**

---

## 支持

**部署期间的问题：**
- 查看 #ml-infrastructure Slack 频道
- 呼叫值班工程师（PagerDuty）
- 查看 `/docs/runbooks/` 中的运行手册

**紧急生产问题：**
- 按照事件响应程序操作
- 联系：incidents@example.com

---

**文档版本：** 1.0
**最后审查：** 2025 年 10 月 18 日
**下次审查：** 2026 年 1 月 18 日
