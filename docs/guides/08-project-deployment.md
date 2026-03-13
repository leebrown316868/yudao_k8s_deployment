# 08. 项目部署

本章节介绍将芋道项目部署到 Kubernetes 集群的完整流程。

## 部署架构

```
                        ┌─────────────────────────────┐
                        │      外部用户访问             │
                        └─────────────┬───────────────┘
                                      │
                        ┌─────────────▼───────────────┐
                        │    Nginx Ingress Controller │
                        │     (LoadBalancer IP)       │
                        └─────────────┬───────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
            ┌───────▼────────┐  ┌────▼─────┐  ┌────────▼────────┐
            │ www.ymyw.net   │  │          │  │ api.ymyw.net    │
            │   (前端)        │  │          │  │   (API网关)      │
            │ yudao-ui-admin │  │          │  │ yudao-gateway    │
            └────────┬────────┘  │          │  └────────┬────────┘
                     │           │          │           │
                     └───────────┼──────────┴───────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │   芋道微服务集群           │
                    │  ├─ yudao-system         │
                    │  ├─ yudao-infra          │
                    │  └─ ...                  │
                    └──────────────────────────┘
```

## 部署流程

### 1. 配置 Harbor 信任

在所有 K8S 节点上编辑 `/etc/docker/daemon.json`：

```json
{
  "insecure-registries": ["192.168.206.129:80"]
}
```

重启 Docker：
```bash
systemctl restart docker
```

### 2. 创建命名空间

```bash
kubectl create namespace dev
kubectl create namespace production
```

### 3. 创建镜像拉取密钥

```bash
kubectl create secret docker-registry harbor-registry \
  --docker-server=192.168.206.129:80 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  -n production
```

### 4. 部署后端服务

#### 方式一：使用命令行

```bash
# Gateway
kubectl create deployment yudao-gateway \
  --image=192.168.206.129:80/library/yudao_gateway:latest \
  -n dev

# System
kubectl create deployment yudao-system \
  --image=192.168.206.129:80/library/yudao_system:latest \
  -n dev

# Infra
kubectl create deployment yudao-infra \
  --image=192.168.206.129:80/library/yudao_infra:latest \
  -n dev
```

#### 方式二：使用 YAML（推荐）

创建 `yudao-deployments.yaml`：

```yaml
---
# Gateway Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yudao-gateway
  namespace: dev
  labels:
    app: yudao-gateway
    part-of: yudao
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yudao-gateway
  template:
    metadata:
      labels:
        app: yudao-gateway
        part-of: yudao
    spec:
      containers:
      - name: yudao-gateway
        image: 192.168.206.129:80/library/yudao_gateway:latest
        ports:
        - containerPort: 48080
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 48080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 48080
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"

---
# System Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yudao-system
  namespace: dev
  labels:
    app: yudao-system
    part-of: yudao
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yudao-system
  template:
    metadata:
      labels:
        app: yudao-system
        part-of: yudao
    spec:
      containers:
      - name: yudao-system
        image: 192.168.206.129:80/library/yudao_system:latest
        ports:
        - containerPort: 48081

---
# Infra Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yudao-infra
  namespace: dev
  labels:
    app: yudao-infra
    part-of: yudao
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yudao-infra
  template:
    metadata:
      labels:
        app: yudao-infra
        part-of: yudao
    spec:
      containers:
      - name: yudao-infra
        image: 192.168.206.129:80/library/yudao_infra:latest
        ports:
        - containerPort: 48082

---
# UI Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yudao-ui-admin
  namespace: dev
  labels:
    app: yudao-ui-admin
    part-of: yudao
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yudao-ui-admin
  template:
    metadata:
      labels:
        app: yudao-ui-admin
        part-of: yudao
    spec:
      containers:
      - name: yudao-ui-admin
        image: 192.168.206.129:80/library/yudao_ui_admin:latest
        ports:
        - containerPort: 80
```

应用配置：
```bash
kubectl apply -f yudao-deployments.yaml
```

### 5. 创建 Service

创建 `svc_yudao.yaml`：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: yudao-gateway
  namespace: dev
  labels:
    app: yudao-gateway
    part-of: yudao
spec:
  ports:
  - port: 48080
    protocol: TCP
    targetPort: 48080
  selector:
    app: yudao-gateway
  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: yudao-ui-admin
  namespace: dev
  labels:
    app: yudao-ui-admin
    part-of: yudao
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: yudao-ui-admin
  type: ClusterIP
```

应用配置：
```bash
kubectl apply -f svc_yudao.yaml
```

### 6. 配置 Ingress

创建 `ingress_yudao.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yudao
  namespace: dev
  labels:
    part-of: yudao
spec:
  ingressClassName: nginx
  rules:
  - host: api.ymyw.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: yudao-gateway
            port:
              number: 48080
  - host: www.ymyw.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: yudao-ui-admin
            port:
              number: 80
```

应用配置：
```bash
kubectl apply -f ingress_yudao.yaml
```

### 7. 配置域名解析

#### 获取 Ingress IP

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

查看 `EXTERNAL-IP` 列，假设为 `192.168.206.200`。

#### 配置本地 hosts

Windows：编辑 `C:\Windows\System32\drivers\etc\hosts`

```
192.168.206.200  api.ymyw.net
192.168.206.200  www.ymyw.net
```

Linux/Mac：编辑 `/etc/hosts`

```bash
echo "192.168.206.200  api.ymyw.net" >> /etc/hosts
echo "192.168.206.200  www.ymyw.net" >> /etc/hosts
```

#### 刷新 DNS 缓存

Windows（管理员运行 cmd）：
```cmd
ipconfig /flushdns
```

### 8. 验证部署

```bash
# 查看 Pod 状态
kubectl get pods -l part-of=yudao -n dev

# 查看 Service
kubectl get svc -l part-of=yudao -n dev

# 查看 Ingress
kubectl get ingress yudao -n dev

# 查看日志
kubectl logs -l app=yudao-gateway -n dev --tail=100 -f
```

浏览器访问：
- 前端：`http://www.ymyw.net`
- API：`http://api.ymyw.net`

## 快速部署命令

### 开发环境

```bash
# 创建命名空间
kubectl create namespace dev

# 部署应用
kubectl apply -f deploy/k8s/yudao-deployments.yaml
kubectl apply -f deploy/k8s/svc_yudao.yaml
kubectl apply -f deploy/k8s/ingress_yudao.yaml

# 配置本地 hosts（使用 MetalLB IP）
echo "<MetalLB-IP>  api.ymyw.net" >> /etc/hosts
echo "<MetalLB-IP>  www.ymyw.net" >> /etc/hosts
```

### 生产环境

```bash
# 创建命名空间
kubectl create namespace production

# 创建镜像拉取密钥
kubectl create secret docker-registry harbor-registry \
  --docker-server=192.168.206.129:80 \
  --docker-username=<username> \
  --docker-password=<password> \
  -n production

# 使用 Kustomize 部署
kubectl apply -k deploy/overlays/prod/
```

## 常见问题

### Pod 启动失败（ImagePullBackOff）

```bash
# 查看详情
kubectl describe pod <pod-name> -n dev

# 确认镜像存在
docker images | grep yudao

# 检查镜像拉取密钥（生产环境）
kubectl get secret harbor-registry -n production -o yaml
```

### 无法访问后端 API

1. 确认 Pod 运行正常
2. 检查 Service 配置
3. 验证 Ingress 规则
4. 确认 hosts 文件配置正确

### 日志查看

```bash
# 实时查看日志
kubectl logs -f <pod-name> -n dev

# 查看多个 Pod 的日志
kubectl logs -l app=yudao-gateway -n dev --tail=100 -f

# 查看之前的日志（Pod 重启后）
kubectl logs <pod-name> -n dev --previous
```

## 下一步

部署完成后，查看：
- [附录 A：常见问题](./appendix-troubleshooting.md)
- [附录 B：面试知识点](./appendix-interview.md)
