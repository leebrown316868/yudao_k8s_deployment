# 附录 A：常见问题

本章节汇总部署过程中的常见问题和解决方案。

## 问题排查原则

> "凡是看日志，日志没问题看下一个相关 pod 日志，不要以为没日志，有问题一定有日志"

## 数据库相关问题

### 问题：数据库取名包含冲突字符

**现象**：
```sql
ERROR 1064 (42000): You have an error in your SQL syntax;
```

**原因**：数据库名包含 `-` 等特殊字符

**解决**：
```sql
CREATE DATABASE `ruoyi-vue-pro`;
```

使用反引号包裹数据库名。

## 网络相关问题

### 问题：登录异常系统错误

**现象**：前端登录时提示"后端失联"

**原因**：
1. 后端服务未启动
2. 前端配置的后端地址错误
3. 防火墙阻止连接

**解决**：
1. 检查后端服务状态
2. 修改 `.env.local` 配置
3. 检查防火墙规则或关闭 VPN

```bash
# 检查后端服务
curl http://192.168.206.129:48080/actuator/health

# 检查防火墙
sudo firewall-cmd --list-all
sudo firewall-cmd --add-port=48080/tcp --permanent
sudo firewall-cmd --reload
```

## 镜像相关问题

### 问题：ImagePullBackOff

**现象**：
```bash
kubectl get pods
NAME                        READY   STATUS              RESTARTS   AGE
yudao-gateway-xxx           0/1     ImagePullBackOff    0          5m
```

**分析**：
```bash
kubectl describe pod yudao-gateway-xxx -n dev
```

**可能原因**：

#### 1. 镜像不存在
```bash
# 解决：检查镜像
docker images | grep yudao_gateway

# 或在 Harbor 中检查
# 登录 Harbor -> library 项目 -> 查看镜像
```

#### 2. Harbor 未配置信任
```bash
# 解决：配置 insecure-registries
vim /etc/docker/daemon.json

# 添加：
{
  "insecure-registries": ["192.168.206.129:80"]
}

# 重启 Docker
systemctl restart docker
```

#### 3. 镜像拉取密钥错误（生产环境）
```bash
# 检查密钥
kubectl get secret harbor-registry -n production -o yaml

# 重新创建密钥
kubectl delete secret harbor-registry -n production
kubectl create secret docker-registry harbor-registry \
  --docker-server=192.168.206.129:80 \
  --docker-username=admin \
  --docker-password=<password> \
  -n production
```

### 问题：镜像拉取缓慢

**解决**：使用国内镜像源或手动拉取后打标签

```bash
# 使用渡渡鸟等工具下载
# 然后打标签
docker tag <本地镜像> registry.cn-hangzhou.aliyuncs.com/google_containers/<镜像名>:<tag>

# 或重启 kubelet 清理缓存
systemctl restart kubelet
```

## 网络插件相关问题

### 问题：Flannel Pod 未 Running

**现象**：
```bash
kubectl get pods -A | grep flannel
NAME                READY   STATUS              RESTARTS   AGE
kube-flannel-ds-xxx 0/1     ErrImagePull        0          10m
```

**原因 1**：`br_netfilter` 模块未加载

**日志**：
```bash
kubectl logs -n kube-flannel <pod-name>
```

错误信息：`/proc/sys/net/bridge/bridge-nf-call-iptables not found`

**解决**：
```bash
# 加载模块
echo 'br_netfilter' >> /etc/modules-load.d/br_netfilter.conf
modprobe br_netfilter

# 重启 Flannel Pod
kubectl delete pod -n kube-flannel <pod-name>
```

**原因 2**：镜像拉取失败

**解决**：使用国内镜像源

```bash
# 手动拉取镜像
docker pull dockerproxy.com/flannel/flannel:v0.26.7
docker tag dockerproxy.com/flannel/flannel:v0.26.7 ghcr.io/flannel-io/flannel:v0.26.7
```

### 问题：ingress-nginx 未 Running

**分析**：
```bash
kubectl describe pod -n ingress-nginx <pod-name>
```

**可能原因**：

#### 1. 集群节点 Flannel 网络问题
先解决 Flannel 问题

#### 2. 镜像源配置错误
```bash
# 检查 yaml 文件中的镜像地址
grep "image:" nginx_ingress.yaml

# 确保使用国内镜像源
registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.10.0
```

## 负载均衡相关问题

### 问题：MetalLB 镜像拉取失败

**现象**：`Pulling image`

**解决**：更换镜像源

```bash
# 查看 Controller 使用的镜像
kubectl get pod -n metallb-system -l app=metallb-controller -o yaml | grep image:

# 手动拉取
docker pull quay.io/metallb/controller:v0.13.12
```

### 问题：MetalLB 无法创建 Pod

**现象**：`can not create sandbox`

**分析**：镜像无法下载

**解决**：
1. 检查网络连接
2. 使用本地镜像仓库
3. 查看 kubelet 日志：`journalctl -u kubelet`

## 配置文件相关问题

### 问题：没有资源访问权限

**现象**：
```
Error from server (Forbidden): error when creating ...
```

**原因**：配置文件 API 版本与集群不匹配

**解决**：
```bash
# 更新 API 版本
kubeadm config migrate --old-config old.yaml --new-config new.yaml

# 或从官网下载对应版本的配置文件
```

## Kubelet 相关问题

### 问题：K8S 下载错误（GPG signature）

**现象**：
```
... GPG key retrieval failed: ...
```

**原因**：阿里云镜像源 GPG 密钥过期

**解决**：
```bash
# 方案 1：关闭 GPG 检查
vim /etc/yum.repos.d/kubernetes.repo
# 修改：gpgcheck=0

# 方案 2：更换镜像源
# 使用官方源或其他镜像源
```

### 问题：kubelet 启动失败

**查看日志**：
```bash
journalctl -u kubelet -f
```

**常见原因**：
1. Swap 未关闭
2. Docker 未启动
3. cri-dockerd 未启动
4. 配置文件错误

## 网络配置相关问题

### 问题：Win 浏览器更新 hosts 后需更新 DNS 缓存

**解决**：
```cmd
:: 管理员运行 cmd
ipconfig /flushdns
```

### 问题：IP 无输出网卡默认 DOWN

**检查网卡状态**：
```bash
nmcli device
```

**临时解决**：
```bash
sudo ip addr add 192.168.206.131/24 dev ens160
ip link set ens160 up
ip route add default via 192.168.206.2 dev ens160
```

**永久解决**：
```bash
# 重启 NetworkManager
systemctl restart NetworkManager
```

## Master 节点 IP 变更

### 问题：Master IP 改变后集群无法访问

**解决**：
```bash
# 1. 批量替换配置文件中的 IP
oldip=192.168.206.129
newip=192.168.100.129
grep -rl "$oldip" /etc/kubernetes ~/.kube/config 2>/dev/null | \
  xargs sed -i "s#$oldip#$newip#g"

# 2. 重新生成证书
kubeadm init phase certs apiserver --apiserver-advertise-address=$newip

# 3. 移除静态 Pod 清单
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 4. 重启 kubelet
systemctl restart kubelet

# 5. 将 yaml 移回来
mv /tmp/kube-*.yaml /etc/kubernetes/manifests/
systemctl restart kubelet
```

**预防措施**：在 kubeadm 配置中使用域名

```yaml
# kubeadm_init.yaml
controlPlaneEndpoint: "k8s-api.example.com:6443"
```

然后在 DNS 或 `/etc/hosts` 中映射域名。

## 命名空间问题

### 问题：命名空间 terminating 状态

**现象**：
```bash
kubectl get ns kube-flannel
NAME           STATUS        AGE
kube-flannel   Terminating   1d
```

**解决**：
```bash
# 1. 导出 namespace 定义
kubectl get namespace kube-flannel -o json > /tmp/kube-flannel.json

# 2. 编辑文件，删除 finalizers
vim /tmp/kube-flannel.json
# 找到 spec.finalizers，删除或设为空数组

# 3. 强制删除
kubectl replace --raw "/api/v1/namespaces/kube-flannel/finalize" -f /tmp/kube-flannel.json

# 或使用 jq 一行命令
kubectl get namespace kube-flannel -o json | \
  jq '.spec.finalizers=[]' | \
  kubectl replace --raw "/api/v1/namespaces/kube-flannel/finalize" -f -
```

## Docker 构建相关问题

### 问题：修改配置后构建镜像 Docker 复用前层

**现象**：JAR 文件已更新，但镜像中的文件未更新

**原因**：Docker 根据文件 hash 判断是否复用缓存层

**解决**：禁用缓存强制构建
```bash
docker build --no-cache -t 192.168.206.129:80/library/yudao_infra:v1 .
```

## 日志查看命令汇总

```bash
# Kubelet 日志
journalctl -u kubelet

# Docker 日志
journalctl -u docker

# Pod 日志
kubectl logs <pod-name> -n <namespace>

# 实时查看 Pod 日志
kubectl logs -f <pod-name> -n <namespace>

# 查看之前的日志（Pod 重启后）
kubectl logs <pod-name> -n <namespace> --previous

# 查看 Deployment 下所有 Pod 的日志
kubectl logs -l app=<app-name> -n <namespace> --tail=100 -f

# 查看 Pod 详情（包括事件）
kubectl describe pod <pod-name> -n <namespace>

# 查看 Node 详情
kubectl describe node <node-name>

# 查看 Service 详情
kubectl describe svc <service-name> -n <namespace>
```

## 更多帮助

如遇到其他问题：
1. 查看日志：`kubectl logs` 或 `journalctl`
2. 查看详情：`kubectl describe`
3. 搜索错误信息
4. 查阅官方文档
