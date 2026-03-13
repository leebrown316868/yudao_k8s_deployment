# 07. 组件安装

本章节介绍 Kubernetes 集群常用组件的安装，包括 Helm、Dashboard、Nginx Ingress 和 MetalLB。

## Helm 安装

### 概述

Helm 是 Kubernetes 的包管理器，类似于 yum/apt，用于简化应用的部署和管理。

- **Chart**：Helm 的包格式，包含一组 Kubernetes 资源定义
- **Release**：Chart 的运行实例
- **Repository**：Chart 的仓库

### 安装 Helm

```bash
wget https://get.helm.sh/helm-v3.14.2-linux-amd64.tar.gz
tar xvf helm-v3.14.2-linux-amd64.tar.gz
mv linux-amd64/helm /usr/bin/
```

验证：
```bash
helm version
```

### 常用命令

```bash
# 添加仓库
helm repo add <name> <url>

# 更新仓库
helm repo update

# 搜索 Chart
helm search repo <keyword>

# 安装/升级
helm upgrade --install <release> <chart>

# 卸载
helm uninstall <release>

# 列出 Release
helm list
```

## Kubernetes Dashboard

### 概述

Dashboard 是 Kubernetes 的 Web UI，提供可视化的集群管理界面。

### 安装 Dashboard

```bash
# 添加存储库
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# 安装
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace --namespace kubernetes-dashboard

# 暴露服务
kubectl patch svc kubernetes-dashboard-kong-proxy -n kubernetes-dashboard \
  -p '{"spec": {"type": "NodePort"}}'

# 查看服务
kubectl get svc -n kubernetes-dashboard kubernetes-dashboard-kong-proxy
```

### 创建管理员账户

创建 `dashboard-adminuser.yaml`：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

应用配置：
```bash
kubectl apply -f dashboard-adminuser.yaml
```

### 获取访问令牌

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

复制输出的令牌，用于登录 Dashboard。

### 访问 Dashboard

```bash
# 获取访问地址
kubectl get svc -A | grep dashboard
```

访问：`https://<NodeIP>:<NodePort>`

## Nginx Ingress

### 概述

Ingress 是集群的统一入口，负责将外部 HTTP/HTTPS 请求路由到内部 Service。

```
                    ┌─────────────────────────────────────┐
                    │         外部流量                      │
                    └─────────────────┬───────────────────┘
                                      │
                    ┌─────────────────▼───────────────────┐
                    │      Nginx Ingress Controller       │
                    │         (NodePort/LoadBalancer)      │
                    └─────────────────┬───────────────────┘
                                      │
           ┌──────────────────────────┼──────────────────────────┐
           │                          │                          │
    ┌──────▼──────┐           ┌───────▼───────┐          ┌──────▼──────┐
    │  Service 1  │           │   Service 2   │          │  Service 3  │
    │   :8080     │           │    :8081      │          │   :8082     │
    └─────────────┘           └───────────────┘          └─────────────┘
```

### 下载部署文件

访问：https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/cloud/deploy.yaml

保存为 `nginx_ingress.yaml`。

### 替换镜像源

```bash
# 替换 controller 镜像
sed -ri 's$registry.k8s.io/ingress-nginx/controller:(v?[0-9]+\.[0-9]+\.[0-9]+([-a-zA-Z0-9.]*)?)$registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:\1$' nginx_ingress.yaml

# 替换 kube-webhook-certgen 镜像
sed -ri 's$registry.k8s.io/ingress-nginx/kube-webhook-certgen:(v?[0-9]+\.[0-9]+\.[0-9]+([-a-zA-Z0-9.]*)?)$registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:\1$' nginx_ingress.yaml

# 删除镜像摘要
sed -i 's/@sha.*//' nginx_ingress.yaml
```

### 部署 Ingress

```bash
kubectl apply -f nginx_ingress.yaml
```

### 验证部署

```bash
kubectl get pod -n ingress-nginx
```

预期输出：
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-xxx          0/1     Completed   0          5m
ingress-nginx-admission-patch-xxx           0/1     Completed   0          5m
ingress-nginx-controller-xxx                1/1     Running     0          5m
```

## MetalLB

### 概述

MetalLB 为裸机环境 Kubernetes 集群提供 LoadBalancer 类型的 Service 支持。

### L2 模式工作原理

```
┌──────────────────────────────────────────────────────────────┐
│                        MetalLB L2 模式                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   1. 从 IP 地址池分配一个 IP (如 192.168.206.200)            │
│                                                              │
│   2. 响应 ARP/NDP 请求，将 IP "绑定" 到运行 Pod 的节点        │
│                                                              │
│   3. 流量发送到该节点的 MAC 地址                             │
│                                                              │
│   4. kube-proxy 将流量转发到实际 Pod                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 安装 MetalLB

```bash
# 添加仓库
helm repo add metallb https://metallb.github.io/metallb

# 安装
helm install metallb metallb/metallb --namespace metallb-system --create-namespace
```

### 配置 IP 地址池

创建 `metallb.yaml`（注意修改 IP 网段）：

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.206.200-192.168.206.220

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

应用配置：
```bash
kubectl apply -f metallb.yaml
```

### 验证

```bash
# 查看 Pod
kubectl get pod -n metallb-system

# 测试 LoadBalancer
kubectl expose deployment kubernetes-dashboard --name=test-lb --port=443 --target-port=8443 -n kubernetes-dashboard --type=LoadBalancer

# 查看 Service
kubectl get svc test-lb -n kubernetes-dashboard

# 应该看到 EXTERNAL-IP 被分配
```

清理测试 Service：
```bash
kubectl delete svc test-lb -n kubernetes-dashboard
```

## 组件端口清单

| 组件 | 端口 | 用途 |
|------|------|------|
| Dashboard Kong | 8443/NodePort | Web UI |
| Ingress Controller | 80/443 | HTTP/HTTPS 入口 |

## 常见问题

### Dashboard 无法访问

```bash
# 检查 Service 类型
kubectl get svc -n kubernetes-dashboard

# 改为 NodePort
kubectl patch svc kubernetes-dashboard-kong-proxy -n kubernetes-dashboard -p '{"spec": {"type": "NodePort"}}'
```

### Ingress Controller 镜像拉取失败

```bash
# 查看 Pod 状态
kubectl describe pod -n ingress-nginx <pod-name>

# 手动拉取镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.10.0
```

### MetalLB 未分配 IP

```bash
# 检查 IP 地址池
kubectl get ipaddresspool -n metallb-system

# 检查 L2Advertisement
kubectl get l2advertisement -n metallb-system

# 查看 Controller 日志
kubectl logs -n metallb-system -l app=metallb-controller
```

## 下一步

组件安装完成后，继续阅读 [08. 项目部署](./08-project-deployment.md)。
