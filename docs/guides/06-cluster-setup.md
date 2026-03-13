# 06. 集群搭建

本章节介绍使用 kubeadm 搭建 3 节点 Kubernetes 集群。

## 集群规划

### 架构图

```
                    ┌──────────────────────────────┐
                    │      Kubernetes 集群          │
                    └──────────────────────────────┘
         ┌────────────────────────────────────────────────┐
         │                                                │
    ┌────▼────┐                                       ┌────▼────┐
    │ Master  │                                       │  Node1  │
    │  节点    │                                       │  节点    │
    │         │                                       │         │
    │ API     │                                       │ Kubelet │
    │ Control │                                       │ Proxy   │
    │ Manager │                                       │ Runtime │
    │ Scheduler│                                      │         │
    │ etcd    │                                       │         │
    └─────────┘                                       └─────────┘
         │                                                │
         │                 (可选)                         │
    ┌────▼────┐                                       ┌────▼────┐
    │  Node2  │                                       │  Node3  │
    │  节点    │                                       │  节点    │
    └─────────┘                                       └─────────┘
```

### 节点规划

| 角色 | 主机名 | IP 地址 | 配置 |
|------|--------|---------|------|
| Master | k8s-master | 192.168.206.129 | 4C 5G |
| Node | k8s-node1 | 192.168.206.130 | 4C 5G |
| Node | k8s-node2 | 192.168.206.131 | 4C 5G |

### 软件版本

| 软件 | 版本 |
|------|------|
| 操作系统 | Ubuntu 22.04 |
| Kubernetes | 1.29.1 |
| Docker | 26.0 |
| cri-dockerd | 0.3.12 |

## 节点准备（所有节点）

### 1. 关闭 Swap

**概述**：Kubernetes 要求系统只使用物理内存，确保资源管理的准确性。

```bash
# 临时关闭
swapoff -a

# 永久关闭（注释 /etc/fstab 中的 swap 行）
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

验证：
```bash
free -h
```

预期输出 `Swap: 0 0 0`。

### 2. 安装 Docker

#### 配置 Docker daemon

编辑 `/etc/docker/daemon.json`：

```json
{
  "data-root": "/data/docker",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "100"
  },
  "registry-mirrors": [
    "https://docker.m.daocloud.io/",
    "https://huecker.io/",
    "https://dockerhub.timeweb.cloud"
  ],
  "insecure-registries": ["harbor.xinxainghf.com", "192.168.206.129:80"],
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

#### 启动 Docker

```bash
systemctl daemon-reload
systemctl enable --now docker
```

验证：
```bash
docker info | grep "Cgroup Driver"
```

预期输出 `Cgroup Driver: systemd`。

### 3. 安装 cri-dockerd

**概述**：cri-dockerd 是 Kubernetes 的 CRI（容器运行时接口）适配器，让 Kubernetes 能够管理 Docker 容器。

#### 下载 cri-dockerd

官网：https://github.com/Mirantis/cri-dockerd/releases

```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.12/cri-dockerd-0.3.12.amd64.tgz
tar xvf cri-dockerd-0.3.12.amd64.tgz
install -o root -g root -m 0755 ./cri-dockerd/cri-dockerd /usr/bin/cri-dockerd
```

#### 配置 systemd 服务

创建 `/etc/systemd/system/cri-docker.service`：

```ini
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
```

创建 `/etc/systemd/system/cri-docker.socket`：

```ini
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
```

#### 启动 cri-dockerd

```bash
systemctl daemon-reload
systemctl enable --now cri-docker.socket
systemctl status cri-docker.socket
```

### 4. 安装 Kubernetes 软件包

#### 配置软件仓库

编辑 `/etc/yum.repos.d/kubernetes.repo`：

```ini
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

#### 安装软件

```bash
yum install -y kubeadm kubectl kubelet
systemctl enable kubelet.service
```

验证：
```bash
kubectl version --client
kubeadm version
```

## Master 节点配置

### 1. 创建 kubeadm 配置文件

创建 `kubeadm_init.yaml`：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.206.129
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock
  imagePullPolicy: IfNotPresent
  taints: null

---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.29.1
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
scheduler: {}
```

### 2. 预拉取镜像

```bash
kubeadm config images pull --config ./kubeadm_init.yaml
```

### 3. 初始化集群

```bash
kubeadm init --config ./kubeadm_init.yaml
```

成功后输出包含 `--discovery-token-ca-cert-hash`，保存用于 Node 节点加入。

### 4. 配置 kubectl

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

验证：
```bash
kubectl get nodes
```

## 配置集群网络

### 安装 Flannel

**概述**：Flannel 为集群内所有 Pod 提供统一的扁平网络，使不同节点上的 Pod 可以直接通信。

#### 加载内核模块

```bash
yum install -y epel-release bridge-utils
modprobe br_netfilter
echo 'br_netfilter' >> /etc/modules-load.d/bridge.conf
echo 'net.bridge.bridge-nf-call-iptables=1' >> /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1' >> /etc/sysctl.conf
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p
```

#### 部署 Flannel

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

#### 验证部署

```bash
kubectl get pods -A
```

等待 `coredns` Pod 变为 `Running` 状态。

## Node 节点配置（所有 Node 节点）

### 1. 加载内核模块

```bash
echo 'br_netfilter' >> /etc/modules-load.d/br_netfilter.conf
modprobe br_netfilter
```

### 2. 加入集群

在 Master 上获取 join 命令：

```bash
kubeadm token create --print-join-command
```

在 Node 节点执行输出命令：

```bash
kubeadm join 192.168.206.129:6443 \
  --cri-socket /var/run/cri-dockerd.sock \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:46ba901fffcd09ca181959bf14711827f8bc718bb68da5e4856daa19fd7f603f
```

### 3. 验证节点加入

在 Master 上执行：

```bash
kubectl get nodes
```

预期输出：
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   10m   v1.29.1
k8s-node1      Ready    <none>          5m    v1.29.1
k8s-node2      Ready    <none>          5m    v1.29.1
```

## 常见问题

### Flannel Pod 启动失败

**错误**：`/proc/sys/net/bridge/bridge-nf-call-iptables not found`

**原因**：未加载 `br_netfilter` 模块

**解决**：
```bash
echo 'br_netfilter' >> /etc/modules-load.d/br_netfilter.conf
modprobe br_netfilter
kubectl delete pod -n kube-flannel <flannel-pod-name>
```

### 镜像拉取失败

**解决**：使用国内镜像源或手动拉取后重新打标签

```bash
# 拉取镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9

# 打标签
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9 registry.k8s.io/pause:3.9
```

### 节点 NotReady

```bash
# 查看节点事件
kubectl describe node <node-name>

# 查看 kubelet 日志
journalctl -u kubelet -f
```

### 重置集群

如需重新初始化：

```bash
# Master 节点
sudo kubeadm reset --force --cri-socket unix:///var/run/cri-dockerd.sock

# Node 节点
sudo kubeadm reset --force --cri-socket unix:///var/run/cri-dockerd.sock

# 清理残留
rm -rf /etc/cni/net.d
rm -rf $HOME/.kube/config
```

## 下一步

集群搭建完成后，继续阅读 [07. 组件安装](./07-components-setup.md)。
