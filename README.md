# yudao-k8s

## 项目简介

基于 kubeadm 搭建 3 节点 Kubernetes 集群，完成 Java 微服务容器化与 Ingress 入口配置，配套 Harbor 私有仓库与 CI 流水线，用于本科阶段的运维与云计算实践。

## 项目范围

### 已完成

- kubeadm 初始化集群与 Flannel 网络
- Harbor 私有镜像仓库
- 后端与前端容器化并部署到 K8s
- Nginx Ingress 与 MetalLB 入口配置
- 基础 CI（前端构建与镜像推送）
- 健康检查与资源配置
- Kustomize 多环境配置（已完善）
- 命名空间隔离（dev/production）
- ConfigMap/Secret 配置管理
- 排障记录与操作手册

### 未完成

- HPA 与 PDB
- 监控与告警接入
- GitOps（ArgoCD/Flux）
- 安全扫描

## 架构与文档

- 架构说明：docs/architecture.md
- 部署说明：docs/deployment.md
- CI/CD：docs/cicd.md
- 排障：docs/troubleshooting.md
- 选型理由：docs/decisions.md
- 审视与优化：docs/optimization-summary.md

## 目录结构

```
.
├── deploy/               # K8s 资源清单与配置
│   ├── base/            # Kustomize 基础配置
│   │   ├── kustomization.yaml
│   │   └── namespaces.yaml
│   ├── overlays/        # Kustomize 环境覆盖
│   │   ├── dev/        # 开发环境
│   │   └── prod/       # 生产环境
│   └── k8s/            # 原始 K8s 资源清单
├── docs/               # 项目文档与排障
├── ci/                 # CI 配置
└── .env.example        # 环境变量示例
```

## 快速开始

### 详细指南

如需从头开始部署，请查看完整指南：

- [完整部署指南](docs/guides/README.md) - 从环境准备到生产部署的完整流程
- [01. 环境准备](docs/guides/01-environment-setup.md) - JDK、Maven、Git 安装
- [02. 中间件服务](docs/guides/02-middleware-setup.md) - MySQL、Redis、Nacos
- [03. 后端构建](docs/guides/03-backend-build.md) - 源码构建与服务启动
- [04. 前端构建](docs/guides/04-frontend-build.md) - Vue3 管理后台
- [05. 服务容器化](docs/guides/05-containerization.md) - Docker 镜像与 Harbor
- [06. 集群搭建](docs/guides/06-cluster-setup.md) - K8S 集群初始化
- [07. 组件安装](docs/guides/07-components-setup.md) - Helm、Dashboard、Ingress、MetalLB
- [08. 项目部署](docs/guides/08-project-deployment.md) - 部署到 K8S

### 前置条件

- 已完成集群初始化并安装 Flannel、Nginx Ingress、MetalLB
- 已配置 Harbor 私有仓库信任
- 已安装 kubectl 和 kustomize

### 部署方式

#### 方式一：直接应用清单（快速验证）

```bash
# 1. 创建命名空间
kubectl apply -f deploy/base/namespaces.yaml

# 2. 创建镜像拉取密钥（生产环境）
kubectl create secret docker-registry harbor-registry \
  --docker-server=192.168.163.135 \
  --docker-username=admin \
  --docker-password=your-password \
  -n production

# 3. 应用基础清单
kubectl apply -f deploy/k8s/svc_yuao.yaml
kubectl apply -f deploy/k8s/yudao_app.yaml
kubectl apply -f deploy/k8s/ingress_yuao.yaml
```

#### 方式二：使用 Kustomize（推荐）

```bash
# 开发环境
kubectl apply -k deploy/overlays/dev/

# 生产环境（需先创建镜像拉取密钥）
kubectl apply -k deploy/overlays/prod/
```

### 验证部署

```bash
# 查看 Pod 状态
kubectl get pods -l part-of=yudao -A

# 查看 Service
kubectl get svc -l part-of=yudao -A

# 查看 Ingress
kubectl get ingress yudao -A

# 查看日志
kubectl logs -l app=yudao-gateway -n dev --tail=100 -f
```

## 访问方式

### 开发环境

配置本地 hosts 文件：

```
<MetalLB-IP>  dev-api.yudao.com
<MetalLB-IP>  dev-www.yudao.com
```

### 生产环境

```
<MetalLB-IP>  api.yudao.com
<MetalLB-IP>  www.yudao.com
```

## 环境管理

### 开发环境

```bash
kubectl apply -k deploy/overlays/dev/
```

特点：
- 使用 `latest` 镜像标签
- 1 副本部署
- DEBUG 日志级别
- 开发域名（dev-api.yudao.com）

### 生产环境

```bash
# 先创建镜像拉取密钥
kubectl create secret docker-registry harbor-registry \
  --docker-server=192.168.163.135 \
  --docker-username=<username> \
  --docker-password=<password> \
  -n production

# 部署
kubectl apply -k deploy/overlays/prod/
```

特点：
- 使用 `v1.0.0` 镜像标签
- 2 副本部署
- INFO 日志级别
- 生产域名（api.yudao.com）
- 更高的资源配置
- SSL 重定向启用

## CI/CD

流水线配置见 `ci/.gitlab-ci.yaml`，当前包含：

1. **build-frontend**：前端构建
2. **dockerize**：前端镜像构建与推送

完整流水线（待 GitLab Runner 配置后启用）：
- Lint：代码规范检查
- Test：单元测试
- Build：所有服务镜像构建
- Deploy：自动部署

详细说明见 [docs/cicd.md](docs/cicd.md)

## 项目特色

1. **渐进式优化**：保持项目简单性，符合学习定位
2. **配置分离**：Kustomize 实现多环境配置
3. **健康检查**：完整的 liveness 和 readiness probes
4. **资源管理**：合理的 requests 和 limits 配置
5. **标签规范**：统一的标签体系便于管理
6. **命名空间隔离**：dev 和 production 环境完全隔离

## 常见问题

### 镜像拉取失败

```bash
# 检查 Harbor 信任配置
cat /etc/docker/daemon.json

# 确保包含：
# { "insecure-registries": ["192.168.163.135"] }

# 重启 Docker
sudo systemctl restart docker
```

### Pod 启动失败

```bash
# 查看 Pod 详情
kubectl describe pod <pod-name> -n <namespace>

# 查看日志
kubectl logs <pod-name> -n <namespace> --previous
```

### Kustomize 构建失败

```bash
# 验证 Kustomize 配置
kustomize build deploy/overlays/dev/

# 预览部署（不实际执行）
kubectl apply -k deploy/overlays/dev/ --dry-run=client
```

更多问题见 [docs/troubleshooting.md](docs/troubleshooting.md)

## 参考附录

- [附录 A：常见问题](docs/guides/appendix-troubleshooting.md) - 问题排查与解决
- [附录 B：面试知识点](docs/guides/appendix-interview.md) - 技术面试准备
- [附录 C：配置文件清单](docs/guides/appendix-configs.md) - 配置文件汇总
- [附录 D：快速参考](docs/guides/appendix-quickref.md) - 常用命令速查

## 贡献指南

本项目聚焦本科阶段可复现的运维实践，强调"可跑、可解释、可复盘"的工程能力。

欢迎提交 Issue 和 Pull Request！

## 许可证

MIT License
