# 故障排查指南

## 排障原则

- 先看事件与日志，再定位到具体 Pod 与容器
- 先验证依赖组件是否可用，再排查业务服务
- 使用 `kubectl describe` 查看 Pod 详细状态
- 使用 `kubectl logs` 查看容器日志
- 遵循 OSI 七层模型从下到上排查（物理层 → 应用层）

---

## 一、数据库与中间件问题

### 1.1 应用数据库名配置问题

**问题**：应用无法连接到正确的数据库。

**分析**：硬编码在 jar 包中的数据库名无法灵活修改，违背配置外部化原则。

**通用解决方案**：
```bash
# 方法一：使用环境变量（推荐）
kubectl set env deployment/<app-name> DB_NAME=new_db_name -n <namespace>

# 方法二：使用 ConfigMap
kubectl create configmap db-config --from-literal=database=new_db_name
# 在 Deployment 中引用环境变量

# 方法三：使用配置文件挂载
# 将数据库配置写入外部配置文件，通过 ConfigMap/Secret 挂载
```

---

### 1.2 MySQL 标识符转义规则

**问题**：数据库名或表名包含特殊字符导致语法错误。

**分析**：MySQL 对标识符（数据库名、表名、列名）有严格命名规则。

**规则说明**：
- 允许字符：字母、数字、`$`、`_`
- 特殊字符（如 `-`、空格、关键字）必须用反引号 `` ` `` 包裹
- 反引号本身需要用 `` `` `` 转义

**最佳实践**：
```sql
-- 推荐使用下划线而非连字符
CREATE DATABASE my_app_db;         -- ✅ 推荐
CREATE DATABASE my-app-db;         -- ❌ 不推荐，需要用 `my-app-db`
CREATE DATABASE `my-app-db`;       -- ✅ 可用但需注意转义

-- 配置文件示例（Spring Boot）
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/my_app_db  # 使用下划线命名
```

---

### 1.3 MySQL StatefulSet 启动失败

**问题**：`kubectl get pod -n mysql` 显示 Pod 启动失败。

**通用排查步骤**：
```bash
# 1. 查看 Pod 状态和事件
kubectl describe pod mysql-0 -n mysql
kubectl get events -n mysql --sort-by='.lastTimestamp'

# 2. 查看容器日志
kubectl logs mysql-0 -n mysql
kubectl logs mysql-0 -c init-mysql -n mysql  # 查看 Init 容器

# 3. 常见问题排查
# 检查 PVC 绑定
kubectl get pvc -n mysql

# 检查资源限制
kubectl describe pod mysql-0 -n mysql | grep -A 5 "Limits"

# 检查配置挂载
kubectl exec -it mysql-0 -n mysql -- cat /etc/mysql/conf.d/

# 4. 如果是集群模式（如 MySQL Group Replication）
# 检查 Pod 间网络连通性
kubectl exec -it mysql-0 -n mysql -- ping mysql-1.mysql
```

**常见原因**：
| 原因 | 表现 | 解决 |
|------|------|------|
| PVC 绑定失败 | Pending、Volume 绑定错误 | 检查 StorageClass 和 PV 配置 |
| Init 容器失败 | Init:Error/ CrashLoopBackOff | 查看 Init 容器日志，检查脚本权限 |
| 资源不足 | OOMKilled | 调整 resources.requests/limits |
| 配置错误 | 启动后立即退出 | 检查 my.cnf 挂载和环境变量 |

---

### 1.4 数据库连接失败

**问题**：应用日志显示 `can not create connection to database server` 或 `Access denied`。

**通用排查流程**：
```bash
# 1. 验证数据库服务状态
kubectl exec -it mysql-0 -n mysql -- mysqladmin ping
kubectl get pods -n mysql

# 2. 验证网络连通性
kubectl exec -it <app-pod> -n <namespace> -- nc -zv mysql-service 3306

# 3. 验证认证信息
kubectl get secret db-credentials -n <namespace> -o yaml

# 4. 检查数据库用户权限
kubectl exec -it mysql-0 -n mysql -- mysql -u root -p -e "SELECT user, host FROM mysql.user;"

# 5. 检查连接数限制
kubectl exec -it mysql-0 -n mysql -- mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"
kubectl exec -it mysql-0 -n mysql -- mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
```

**常见原因**：
- 服务未启动或端口未开放
- 网络策略（NetworkPolicy）阻止访问
- 凭证配置错误（用户名/密码/数据库名）
- 连接数达到上限
- DNS 解析失败

---

### 1.5 镜像仓库连接失败

**问题**：`can not create connection to registry` 或 ImagePullBackOff。

**通用排查**：
```bash
# 1. 验证镜像仓库服务
kubectl get pods -n harbor
kubectl get svc -n harbor

# 2. 验证证书信任
# 如果使用自签名证书，需要在 docker/containerd 中配置
openssl s_client -connect harbor.example.com:443 -showcerts

# 3. 验证认证凭证
kubectl get secret harbor-registry -n <namespace> -o yaml | grep .dockerconfigjson

# 4. 测试镜像拉取
docker pull harbor.example.com/project/image:tag
crictl pull harbor.example.com/project/image:tag

# 5. 检查镜像仓库存储
kubectl exec -it harbor-registry -n harbor -- df -h /data
```

---

## 二、前端构建与运行

### 2.1 前端无法连接后端 API

**问题**：前端登录报错，控制台显示网络错误或 CORS 错误。

**通用排查**：
```bash
# 1. 验证后端服务状态
kubectl get pods -n <backend-namespace>
kubectl get svc -n <backend-namespace>

# 2. 验证 Ingress 配置
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# 3. 验证环境变量配置
kubectl get configmap frontend-config -n <namespace> -o yaml
kubectl describe pod <frontend-pod> -n <namespace> | grep -i env

# 4. 测试后端连通性
kubectl exec -it <frontend-pod> -n <namespace> -- curl -v http://backend-service:8080/health

# 5. 检查 CORS 配置
kubectl exec -it <backend-pod> -n <namespace> -- cat /etc/nginx/conf.d/default.conf | grep -A 10 add_header
```

**配置最佳实践**：
```javascript
// .env.production
VITE_API_URL=https://api.example.com    // 生产环境使用绝对 URL
VITE_API_URL=                           // 开发环境可留空使用代理

// vite.config.js 代理配置
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://backend-service:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  }
})
```

---

### 2.2 前端构建失败或超时

**问题**：`npm run build` 卡住或失败。

**通用排查**：
```bash
# 1. 检查资源使用情况
kubectl top pods -n <namespace>
kubectl describe node | grep -A 5 "Allocated resources"

# 2. 清理缓存重新构建
rm -rf node_modules/.vite
rm -rf node_modules
npm install

# 3. 使用多阶段构建优化 Dockerfile
# 构建阶段使用更多资源，运行阶段使用最小资源
```

**资源规划建议**：
| 环境 | 推荐配置 | 说明 |
|------|----------|------|
| 开发构建 | 4C 8G | 单个容器构建 |
| 生产构建 | 8C 16G | CI/CD 构建节点 |
| 并发构建 | 按需扩展 | 使用构建队列 |

---

## 三、K8S 集群与网络

### 3.1 CoreDNS / Flannel 异常

**问题**：CoreDNS 或 Flannel Pod 未 Running，集群 DNS 解析失败。

**通用排查**：
```bash
# 1. 检查节点内核模块
lsmod | grep br_netfilter
modprobe br_netfilter

# 2. 配置网桥过滤参数
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 3. 检查 CoreDNS 日志
kubectl logs -n kube-system -l k8s-app=kube-dns

# 4. 检查 Flannel 日志
kubectl logs -n kube-flannel -l app=flannel

# 5. 测试 DNS 解析
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# 6. 检查节点网络接口
ip addr show flannel.1
ip route show | grep flannel
```

---

### 3.2 镜像拉取失败

**问题**：Pod 状态为 `Init:ErrImagePull` 或 `ImagePullBackOff`。

**通用解决方案**：

**方法一：配置镜像加速器（推荐）**
```bash
# containerd 配置
cat > /etc/containerd/config.toml << EOF
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://docker.m.daocloud.io"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."ghcr.io"]
      endpoint = ["https://ghcr.dockerproxy.com"]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
      endpoint = ["https://quay.dockerproxy.com"]
EOF
systemctl restart containerd
```

**方法二：私有镜像仓库 + ImagePullSecrets**
```bash
# 创建镜像拉取凭证
kubectl create secret docker-registry harbor-registry \
  --docker-server=harbor.example.com \
  --docker-username=<username> \
  --docker-password=<password> \
  -n <namespace>

# 在 Deployment 中引用
# imagePullSecrets:
# - name: harbor-registry
```

**方法三：修改 Deployment 镜像地址**
```bash
# 临时替换（不推荐生产环境）
kubectl set image deployment/<name> <container>=docker.m.daocloud.io/flannel/flannel:v0.26.7
```

---

### 3.3 K8s 软件源问题

**问题**：`repomd.xml GPG signature verification error` 或 404 错误。

**通用解决方案**：

**方法一：使用官方仓库（推荐）**
```bash
# 使用 Kubernetes 官方仓库
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
EOF
```

**方法二：使用国内镜像（注意同步延迟）**
```bash
# 阿里云镜像（注意密钥有效期）
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0  # 临时关闭验证
EOF
```

---

### 3.4 Master 节点 IP 变更

**问题**：Master 节点 IP 改变后集群不可用。

**标准修复流程**：
```bash
# 1. 备份现有配置
cp -r /etc/kubernetes /etc/kubernetes.backup

# 2. 更新所有配置文件中的旧 IP
sed -i 's/<OLD_IP>/<NEW_IP>/g' /etc/kubernetes/*.conf
sed -i 's/<OLD_IP>/<NEW_IP>/g' ~/.kube/config

# 3. 生成新证书
kubeadm init phase certs all --config /etc/kubernetes/kubeadm-config.yaml

# 4. 生成新配置
kubeadm init phase kubeconfig all --config /etc/kubernetes/kubeadm-config.yaml

# 5. 更新静态 Pod 清单
# 方法一：使用 kubeadm
kubeadm init phase control-plane all --config /etc/kubernetes/kubeadm-config.yaml

# 方法二：手动编辑（如果上述方法失败）
cd /etc/kubernetes/manifests
sed -i 's/<OLD_IP>/<NEW_IP>/g' *.yaml

# 6. 重启 kubelet
systemctl restart kubelet

# 7. 更新所有节点上的 kubeconfig
scp /etc/kubernetes/admin.conf <user>@<node>:~/.kube/config
```

**注意**：IP 变更属于重大变更，建议在维护窗口执行，并提前备份 etcd 数据。

---

## 四、K8S 组件问题

### 4.1 Ingress-Nginx 镜像问题

**问题**：`registry.k8s.io` 镜像无法拉取。

**推荐方案**：
```bash
# 使用官方维护的镜像源
# 国内用户可使用阿里云镜像
kubectl set image deployment/ingress-nginx-controller \
  ingress-nginx-controller=registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.9.4 \
  -n ingress-nginx
```

**生产环境建议**：
- 部署私有镜像仓库（Harbor）
- 同步官方镜像到私有仓库
- 定期更新镜像版本

---

### 4.2 MetalLB 镜像问题

**问题**：`quay.io` 镜像拉取失败。

**解决方案**：
```bash
# 修改 ConfigMap 中的镜像地址
kubectl edit configmap config -n metallb-system

# 将 quay.io 替换为可访问的镜像源
# 或使用镜像同步工具同步到私有仓库
```

---

### 4.3 Dashboard 访问权限问题

**问题**：Dashboard 显示无权限访问资源。

**标准方案**：
```bash
# 创建具有适当权限的 ServiceAccount
cat > dashboard-admin.yaml << EOF
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
EOF

kubectl apply -f dashboard-admin.yaml

# 获取登录 Token
kubectl -n kubernetes-dashboard create token admin-user
```

**注意**：生产环境应遵循最小权限原则，避免使用 cluster-admin。

---

### 4.4 命名空间卡在 Terminating

**问题**：Namespace 长时间处于 Terminating 状态。

**原因分析**：
- Namespace 中有资源未完全删除
- finalizers 阻止删除流程
- API Server 无法与控制器通信

**解决方案**：
```bash
# 1. 查看 Namespace 状态
kubectl get namespace <name> -o yaml

# 2. 查找未删除的资源
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get -n <name>

# 3. 手动删除剩余资源
kubectl delete all --all -n <name>
kubectl delete secret,configmap --all -n <name>

# 4. 如果仍卡住，移除 finalizers（谨慎使用）
kubectl get namespace <name> -o json > tmp.json
# 编辑 tmp.json，删除 spec.finalizers 字段
kubectl replace --raw /api/v1/namespaces/<name>/finalize -f tmp.json

# 5. 强制删除 API 对象（危险！）
kubectl delete namespace <name> --force --grace-period=0
```

---

### 4.5 镜像拉取超时或失败

**问题**：镜像拉取失败，但镜像仓库可访问。

**排查与解决**：
```bash
# 1. 检查镜像大小
crictl images | grep <image>

# 2. 检查磁盘空间
df -h /var/lib/containerd
df -h /var/lib/docker

# 3. 检查网络带宽
iftop -i <interface>
speedtest-cli

# 4. 调整镜像拉取超时
# containerd 配置
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"

# /etc/containerd/certs.d/<registry>/hosts.toml
[host."https://<registry>"]
  capabilities = ["pull", "resolve"]
  [host."https://<registry>".header]
    scope = ["repository"]

# 5. 使用 pre-pulled 镜像（离线环境）
# 在所有节点预先拉取镜像
crictl pull <image>
```

---

## 五、运维最佳实践

### 5.1 镜像构建缓存管理

**问题**：构建镜像时缓存导致更新不生效。

**优化方案**：
```dockerfile
# ❌ 不推荐：每次都禁用缓存
# docker build --no-cache ...

# ✅ 推荐：优化 Dockerfile 层级结构
# 1. 变化少的层放前面
FROM python:3.11-slim

# 2. 依赖安装层（变化少）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 3. 应用代码层（变化多）
COPY . .

# 4. 使用多阶段构建减少最终镜像大小
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o app

FROM alpine:latest
COPY --from=builder /app/app /app
```

**高级技巧**：
```dockerfile
# 使用 --mount 缓存依赖
RUN --mount=type=cache,target=/root/.cache go build ...
RUN --mount=type=cache,target=/var/cache/apt apt-get update
```

---

### 5.2 网络接口配置

**问题**：网络接口异常或配置丢失。

**持久化配置方案**：

**NetworkManager（推荐）**：
```bash
# 创建连接配置
nmcli connection add type ethernet con-name eth0 ifname eth0
nmcli connection modify eth0 ipv4.addresses 192.168.1.100/24
nmcli connection modify eth0 ipv4.gateway 192.168.1.1
nmcli connection modify eth0 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify eth0 ipv4.method manual
nmcli connection up eth0

# 保存配置
nmcli connection save
```

**systemd-networkd**：
```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=8.8.4.4
```

```bash
systemctl enable systemd-networkd
systemctl restart systemd-networkd
```

---

### 5.3 DNS 解析问题

**问题**：Windows/Linux DNS 缓存导致解析不生效。

**系统清理方案**：

**Windows**：
```cmd
REM 管理员运行
ipconfig /flushdns
ipconfig /registerdns
nbtstat -RR
```

**Linux**：
```bash
# systemd-resolved
systemctl restart systemd-resolved

# 传统方法
rm /etc/resolv.conf
systemctl restart NetworkManager

# 清理 nscd 缓存
nscd -i hosts
```

**检查 DNS 配置**：
```bash
# 查看 DNS 服务器
cat /etc/resolv.conf

# 测试 DNS 解析
nslookup kubernetes.default.svc.cluster.local
dig kubernetes.default.svc.cluster.local

# 检查 CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

---

## 六、常用排查命令

### 6.1 Pod 状态查看

```bash
# 查看 Pod 状态
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -o wide

# 查看 Pod 详细信息
kubectl describe pod <pod-name> -n <namespace>

# 查看 Pod 日志
kubectl logs <pod-name> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>              # 实时跟踪
kubectl logs --tail=100 <pod-name> -n <namespace>      # 最近 100 行

# 多容器 Pod
kubectl logs <pod-name> -c <container-name> -n <namespace>

# 查看所有容器日志
kubectl logs <pod-name> --all-containers -n <namespace>

# 查看之前的容器日志（CrashLoopBackOff）
kubectl logs <pod-name> --previous -n <namespace>
```

---

### 6.2 Service 与网络调试

```bash
# 查看 Service
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>

# 查看 Endpoints
kubectl get endpoints -n <namespace>

# 端口转发测试
kubectl port-forward <pod-name> <local-port>:<pod-port> -n <namespace>

# 临时运行调试 Pod
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never

# 在已有 Pod 中执行命令
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl exec -it <pod-name> -n <namespace> -- sh

# 测试 Pod 连通性
kubectl exec -it <pod-name> -n <namespace> -- ping <target-ip>
kubectl exec -it <pod-name> -n <namespace> -- curl -v <target-url>
kubectl exec -it <pod-name> -n <namespace> -- telnet <target-host> <port>
kubectl exec -it <pod-name> -n <namespace> -- nc -zv <target-host> <port>
```

---

### 6.3 事件与日志查看

```bash
# 查看命名空间事件
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl get events -n <namespace> --field-selector type=Warning

# 查看所有事件
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# 持续监控事件
kubectl get events -n <namespace> --watch

# 查看 kubelet 日志
journalctl -u kubelet -f
journalctl -u kubelet --since "1 hour ago"

# 查看容器运行时日志
journalctl -u containerd -f
journalctl -u docker -f
```

---

### 6.4 资源分析

```bash
# 查看节点资源
kubectl top nodes
kubectl describe nodes

# 查看 Pod 资源使用
kubectl top pods -n <namespace>

# 查看资源配额
kubectl describe quota -n <namespace>

# 查看限制范围
kubectl describe limitrange -n <namespace>

# 查看持久卷
kubectl get pv
kubectl get pvc -n <namespace>
```

---

## 七、快速诊断决策树

```
问题发生
    ↓
检查 Pod 状态
    ↓
┌─────────────┬─────────────┬──────────────┬──────────────┐
│             │             │              │              │
ImagePullBackOff │ CrashLoopBackOff │ Pending │ Running 异常
    ↓             │              │              ↓
    │             ↓              ↓          业务层排查
    │         查看日志        检查资源/调度
    ↓         查看详情        检查污点/容忍
检查镜像配置    ↓              ↓
    │     检查环境变量      检查 PVC
    │     检查挂载配置      检查节点状态
    │         ↓              ↓
    └─────→ 修复/验证  ←────┘
```

---

## 八、常见错误码速查

| 状态 | 含义 | 常见原因 |
|------|------|----------|
| `Pending` | Pod 未调度 | 资源不足、调度器异常、污点/容忍、PVC 未绑定 |
| `ImagePullBackOff` | 镜像拉取失败 | 镜像不存在、凭证错误、网络问题、仓库不可达 |
| `CrashLoopBackOff` | 容器反复重启 | 启动命令错误、配置错误、OOM、健康检查失败 |
| `RunContainerError` | 容器运行错误 | 镜像不兼容、安全策略限制、Volume 挂载失败 |
| `ErrImageNeverPull` | 镜像策略错误 | imagePullPolicy: Always 但本地无镜像 |
| `OOMKilled` | 内存溢出 | 内存限制过低、内存泄漏 |
| `Init:Error` | Init 容器失败 | Init 脚本错误、依赖服务不可用 |
| `Terminating` | 删除中 | finalizers 阻塞、资源未清理 |
| `Unknown` | 状态未知 | 节点失联、kubelet 异常 |

---

*文档版本：v3.0*
*最后更新：2025-02-22*
*优化说明：修正了不准确的描述，添加了通用解决方案和最佳实践*
