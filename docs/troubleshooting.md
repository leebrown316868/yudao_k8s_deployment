# 故障排查指南

## 排障原则

- 先看事件与日志，再定位到具体 Pod 与容器
- 先验证依赖组件是否可用，再排查业务服务
- 使用 `kubectl describe` 查看 Pod 详细状态
- 使用 `kubectl logs` 查看容器日志

---

## 一、数据库与中间件问题

### 1.1 MySQL 数据库名修改限制

**问题**：在 jar 包中无法用 sed 修改数据库名。

**解决**：必须使用 MySQL 创建名为 `ruoyi-vue-pro` 的数据库。

---

### 1.2 MySQL 语法错误

**问题**：数据库命名或命令中包含了特殊字符。

**解决**：
- 不允许使用反引号（`）避免语法错误
- 若数据库取名包含 `-` 等冲突字符，需加反引号消除歧义

---

### 1.3 MySQL 集群启动失败

**问题**：`kubectl get pod -n mysql` 发现启动失败。

**解决步骤**：
```bash
# 1. 使用 kubectl describe 找问题
kubectl describe pod mysql-0 -n mysql

# 2. 查看日志
kubectl logs -f mysql-0 -n mysql

# 3. 检查初始化状态
# MySQL 容器需要初始化，若 /var/lib/mysql 下无初始文件则不需要初始化

# 4. 检查 sock 冲突
# 指定 sock 时发生 sock 冲突，需检查脚本
```

---

### 1.4 数据库连接失败

**问题**：后端服务日志显示 `can not create connection to database server`。

**分析**：可能跟服务器开机自启动 Docker 有关。

**解决**：重启对应的数据库容器。

---

### 1.5 注册中心连接失败

**问题**：`can not create connection to registry`。

**分析**：集群下载后端服务错误，私有镜像仓库（Harbor）未运行。

**解决**：
```bash
docker start <harbor_container>
```

---

## 二、前端构建与运行

### 2.1 前端登录报错

**问题**：登录报错（系统错误）。

**分析**：后端失联（防火墙、VPN 问题或者前端连接后端的配置错误）。

**解决**：更改配置（`.env.local`）后保存文件。

---

### 2.2 前端构建卡住

**问题**：构建项目卡住。

**分析**：内存或 CPU 不够。

**解决**：
- 推荐使用 16G 内存
- 若无其他容器运行推荐 8G，否则 11G

---

## 三、K8S 集群搭建与网络

### 3.1 CoreDNS Pod 未运行 / Flannel 状态错误

**问题**：CoreDNS 甚至 Flannel 未处于 Running 状态。

**分析**：
- 查看日志发现模块 `br_netfilter` 未加载
- `/proc/sys/net/bridge/bridge-nf-call-iptables` 文件不存在

**解决步骤**：
```bash
# 1. 加载模块
echo 'br_netfilter' >> /etc/modules-load.d/br_netfilter.conf
modprobe br_netfilter

# 2. 设置 sysctl 参数
sysctl net.bridge.bridge-nf-call-iptables=1
sysctl net.bridge.bridge-nf-call-ip6tables=1
sysctl net.bridge.bridge-nf-call-arptables=1

# 3. 重建 Flannel Pod
kubectl delete pod -n kube-flannel <pod-name>

# 或者等待 Pod 自动恢复
```

---

### 3.2 Flannel 镜像拉取失败 (Init:ErrImagePull)

**问题**：
```
Warning Failed ... kubelet Failed to pull image "ghcr.io/flannel-io/flannel:v0.26.7":
rpc error: code = Canceled desc = context canceled
```

**分析**：拉取 `ghcr.io` 镜像缓慢或超时。

**解决**：使用国内加速地址（如"渡渡鸟"）拉取，然后 tag 为 `ghcr.io/flannel-io/flannel:v0.26.7`。

```bash
docker pull docker.m.daocloud.io/flannel/flannel:v0.26.7
docker tag docker.m.daocloud.io/flannel/flannel:v0.26.7 ghcr.io/flannel-io/flannel:v0.26.7
```

---

### 3.3 K8S 下载元数据失败

**问题**：`repomd.xml GPG signature verification error: Bad PGP signature`。

**分析**：阿里云镜像源内密钥有效期 23 天，转换 unix 时间戳比对发现已过期。

**解决**：
- 更换镜像源（cnk8s 官网）
- 或将配置中的 `gpgcheck=0`

---

### 3.4 Master IP 变更

**问题**：Master 节点 IP 改变导致集群不可用。

**解决步骤**：
```bash
# 1. 批量替换配置中的旧 IP 为新 IP

# 2. 修改 kubeadm-config.yaml 并重新生成配置
kubeadm init phase kubeconfig all --config kubeadm-config.yaml

# 3. 重新生成证书
kubeadm init phase certs apiserver --config kubeadm-config.yaml

# 4. 移动 manifests 文件到临时目录
cd /etc/kubernetes/manifests
mv *.yaml /tmp/

# 5. 重启 kubelet
systemctl restart kubelet

# 6. 移回文件，再次重启 kubelet
mv /tmp/*.yaml .
systemctl restart kubelet
```

---

## 四、K8S 组件问题

### 4.1 Ingress-Nginx 状态 ImagePullBackOff

**问题**：Pod 状态为 `ImagePullBackOff`。

**分析**：`registry.k8s.io` 国内无法访问。

**解决**：
```bash
# 1. 使用 sed 替换镜像源为阿里云镜像
sed -i 's/registry.k8s.io/registry.cn-hangzhou.aliyuncs.com/g' ingress-controller.yaml

# 2. 删除镜像摘要 (@sha...)
sed -i 's/@sha256:.*//g' ingress-controller.yaml

# 3. 如果未 Running，检查是否受 Flannel 网络问题影响
```

---

### 4.2 MetalLB 镜像问题

**问题**：
- `Pulling image "quay.io/frrouting/frr:9.1.0"` 拉取慢
- 或 `can not create sandbox`

**分析**：镜像无法下载。

**解决**：
- 更换镜像源
- 查看本地镜像仓库是否有目的内容

---

### 4.3 Dashboard 资源访问权限

**问题**：没有资源访问权限。

**分析**：配置文件问题。

**解决**：官网更换对应的配置文件。

---

### 4.4 命名空间 Terminating

**问题**：Namespace 卡在 `terminating` 状态。

**解决步骤**：
```bash
# 1. 导出 JSON
kubectl get namespace <namespace-name> -o json > temp.json

# 2. 编辑文件删除 finalizers 字段内容
# 将 "spec": {"finalizers": [...] } 改为 "spec": {"finalizers": []}

# 3. 强制清除
kubectl replace --raw /api/v1/namespaces/<namespace-name>/finalize -f temp.json
```

---

### 4.5 ImagePullErr / 拉取慢

**问题**：镜像拉取错误或缓慢。

**解决**：
```bash
# 1. 重新拉取（"第一次可能占线，第二次可能接通"）
kubectl delete pod <pod-name>

# 2. 重启 kubelet
systemctl restart kubelet

# 3. 使用加速拉取方式
```

---

## 五、运维与环境

### 5.1 Docker 镜像未更新

**问题**：修改配置后构建镜像，Docker 复用前层（Cache），导致新 jar 包未被更新至镜像。

**解决**：禁用缓存强制构建。
```bash
docker build --no-cache -t <image-name>:<tag> .
```

---

### 5.2 网络接口 Down

**问题**：
- `ip r` 没输出
- 网卡默认 DOWN
- `nmcli` 显示 `unmanaged`

**解决**：
```bash
# 临时添加 IP 和路由
ip addr add <ip>/<mask> dev <interface>
ip route add default via <gateway>

# 重启 NetworkManager
systemctl restart NetworkManager
```

---

### 5.3 Windows Hosts 解析

**问题**：Windows 更新 hosts 后浏览器未生效。

**解决**：
```cmd
# 管理员运行 cmd 输入
ipconfig /flushdns

# 关闭 VPN（防止劫持 hosts）
# 关闭浏览器 DNS over HTTPS
```

---

## 六、常用排查命令

### 6.1 Pod 状态查看

```bash
# 查看 Pod 状态
kubectl get pods -n <namespace>

# 查看 Pod 详细信息
kubectl describe pod <pod-name> -n <namespace>

# 查看 Pod 日志
kubectl logs <pod-name> -n <namespace>
kubectl logs -f <pod-name> -n <namespace>  # 实时跟踪

# 查看容器日志（多容器 Pod）
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

### 6.2 Service 查看与调试

```bash
# 查看 Service
kubectl get svc -n <namespace>

# 查看 Service 详情
kubectl describe svc <service-name> -n <namespace>

# 端口转发测试
kubectl port-forward <pod-name> <local-port>:<pod-port> -n <namespace>
```

### 6.3 网络排查

```bash
# 查看网络策略
kubectl get networkpolicies -n <namespace>

# 测试 Pod 连通性
kubectl exec -it <pod-name> -n <namespace> -- ping <target-ip>
kubectl exec -it <pod-name> -n <namespace> -- curl <target-url>
```

### 6.4 事件查看

```bash
# 查看命名空间事件
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# 查看所有事件
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

---

## 七、快速诊断流程图

```
问题发生
    ↓
查看 Pod 状态 (kubectl get pods)
    ↓
Pod 状态异常？
    ├─ ImagePullBackOff → 检查镜像仓库/网络
    ├─ CrashLoopBackOff → 查看日志 (kubectl logs)
    ├─ Pending → 检查资源/调度/污点
    └─ Running 但业务异常 → 端口转发测试
    ↓
查看 describe 输出
    ↓
查看事件和日志
    ↓
定位问题 → 修复 → 验证
```

---

*文档版本：v2.0*
*最后更新：2025-02-22*
