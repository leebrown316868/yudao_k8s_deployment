# 附录 B：面试知识点

本章节汇总项目中可能涉及的面试问题和知识点。

## 项目部署相关

### Q：项目中怎么部署前后端镜像的，为什么这么部署？

**答案要点**：

**前后端服务容器化**：
- 前端：本地构建静态资源，再用 Nginx 镜像打包
- 后端：每个服务有自己的 Dockerfile

**推送到 Harbor 私有镜像仓库**：
```bash
docker tag <image> <harbor-address>/library/<image-name>
docker push <harbor-address>/library/<image-name>
```

**Kubernetes 部署**：
- 使用 Deployment 管理应用
- 为前端和 Gateway 创建 Service
- 配置 Ingress 实现基于域名的流量分发

**为什么这么部署**：

1. **基于域名更安全** - 可以方便地配置 SSL 证书
2. **前后端解耦** - 统一入口，前端不用关心后端 IP 变化
3. **易于管理** - 不同域名对应不同的访问策略，便于安全管理
4. **易于扩展** - 增加新服务，只需增加域名和路由规则

## MetalLB 相关

### Q：MetalLB 的工作原理是什么？

**L2 模式**：

1. **IP 分配**：从配置的地址池分配一个局域网 IP
2. **ARP 响应**：监听 ARP/NDP 请求并主动响应，将 IP "绑定" 到运行 Pod 的节点
3. **流量转发**：流量发送到该节点的 MAC 地址，kube-proxy 转发到实际 Pod

**优点**：配置简单，适合裸金属环境

**BGP 模式**：

1. **BGP Peer 建立**：与网络中的 BGP 路由器建立邻居关系
2. **路由通告**：通过 BGP 协议向路由器通告 Service 的外部 IP
3. **流量转发**：路由器根据 BGP 路由将流量转发到对应节点
4. **高可用**：节点故障时自动停止通告，切换到其他健康节点

### Q：为什么需要 MetalLB？

云厂商（AWS、阿里云）会自动为 LoadBalancer 类型的 Service 分配外部 IP，但裸金属环境没有这个能力。MetalLB 填补了这个空白。

## Helm 相关

### Q：Helm 在若依部署中带来的便利是什么？

1. **简化部署** - 一条命令完成复杂应用安装
2. **版本管理** - 方便进行版本升级和回滚
3. **参数化配置** - 通过 values.yaml 自定义配置

### Q：Helm 的核心概念是什么？

- **Chart**：Helm 的包格式，包含一组 K8s 资源定义
- **Release**：Chart 的运行实例
- **Repository**：Chart 的存储仓库
- **Values**：配置参数，用于自定义 Chart

## Ingress 相关

### Q：为什么使用 Ingress 而不是 LoadBalancer？

1. **成本更低** - 多个服务共享一个 LoadBalancer
2. **基于域名** - 可以使用 SSL 安全证书
3. **前后端解耦** - 统一入口，前端不用关心后端 IP 变化
4. **灵活路由** - 支持基于路径、域名的路由规则

### Q：Ingress-Nginx Controller 的工作原理？

```
客户端请求 → Ingress → Ingress Controller (Nginx) → Service → Pod
```

1. Ingress Controller 监听 Ingress 资源变化
2. 动态生成 Nginx 配置
3. Nginx 根据配置将请求路由到对应 Service

## K8S 基础概念

### Q：Kubernetes 的核心组件有哪些？

**Master 节点**：
- **API Server**：集群统一入口，处理 REST 操作
- **etcd**：键值数据库，存储集群状态
- **Scheduler**：负责 Pod 调度
- **Controller Manager**：维护集群状态

**Node 节点**：
- **Kubelet**：与 API Server 通信，管理容器生命周期
- **Kube-proxy**：维护网络规则，实现 Service 负载均衡
- **Container Runtime**：运行容器（Docker/containerd）

### Q：Pod、Deployment、Service 的区别？

| 资源 | 作用 | 特点 |
|------|------|------|
| Pod | 最小部署单元 | 一个或多个容器，共享网络和存储 |
| Deployment | 管理 Pod | 声明式更新，支持滚动更新、回滚 |
| Service | 服务发现 | 负载均衡，稳定的访问入口 |

### Q：Kubernetes 网络模型是什么？

1. **Pod 网络**：所有 Pod 在同一个扁平网络中，可以直接通信
2. **Service 网络**：ClusterIP，虚拟 IP，通过 iptables/IPVS 实现
3. **外部网络**：NodePort、LoadBalancer、Ingress

## 项目实战问题

### Q：如何保证服务高可用？

1. **多副本部署**：Deployment 设置 replicas > 1
2. **健康检查**：livenessProbe 和 readinessProbe
3. **资源限制**：requests 和 limits
4. **反亲和性**：Pod 分散到不同节点
5. **HPA**：Horizontal Pod Autoscaler 自动扩缩容

### Q：如何实现服务的滚动更新？

```yaml
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1        # 最多多出 1 个 Pod
      maxUnavailable: 0  # 最多 0 个 Pod 不可用
    type: RollingUpdate
```

### Q：如何实现配置管理？

1. **ConfigMap**：存储非敏感配置
2. **Secret**：存储敏感信息（密码、密钥）
3. **环境变量**：容器注入配置
4. **配置中心**：Nacos、Apollo

## CI/CD 相关

### Q：GitLab CI/CD 的流程是什么？

```
代码提交 → GitLab Runner → 执行 Pipeline → 构建镜像 → 推送到 Harbor → 部署到 K8S
```

### Q：GitLab CI 的 stage 是什么？

将流水线分为多个阶段：
- **lint**：代码检查
- **test**：单元测试
- **build**：构建镜像
- **deploy**：部署应用

## Docker 相关

### Q：Docker 的网络模式有哪些？

| 模式 | 说明 |
|------|------|
| bridge | 默认模式，通过 veth pair 连接到 docker0 |
| host | 使用宿主机网络 |
| none | 无网络配置 |
| container | 共享其他容器的网络 |

### Q：Dockerfile 的最佳实践？

1. 使用 `.dockerignore` 排除不必要文件
2. 合理利用缓存，将不常变化的层放前面
3. 使用多阶段构建减小镜像大小
4. 使用具体版本标签而非 latest
5. 最小化镜像层数

## 排障问题

### Q：Pod 启动失败的排查思路？

1. 查看 Pod 状态：`kubectl get pods`
2. 查看详情：`kubectl describe pod <pod-name>`
3. 查看日志：`kubectl logs <pod-name>`
4. 检查镜像是否存在
5. 检查资源是否充足
6. 检查配置是否正确

### Q：Service 无法访问的排查思路？

1. 确认 Pod 运行正常
2. 检查 Service 选择器是否匹配
3. 检查 Service 端口配置
4. 检查网络策略
5. 检查 kube-proxy 运行状态

## 项目亮点

### Q：这个项目的技术亮点是什么？

1. **微服务架构**：前后端分离，服务独立部署
2. **容器化部署**：Docker + Kubernetes
3. **自动化 CI/CD**：GitLab CI 自动构建部署
4. **高可用设计**：多副本、健康检查、资源限制
5. **配置管理**：Kustomize 多环境配置
6. **私有镜像仓库**：Harbor 镜像管理与安全扫描
7. **服务网格准备**：Ingress 统一入口，便于后续集成 Istio

## 学习路径建议

1. **基础**：Docker、Kubernetes 核心概念
2. **进阶**：Kubernetes 网络、存储、安全
3. **实践**：微服务部署、CI/CD、监控告警
4. **深入**：服务网格、云原生应用设计
