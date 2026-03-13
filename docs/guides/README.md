# 芋道项目 K8S 部署指南

本指南详细介绍如何在 Kubernetes 集群中部署芋道管理系统，涵盖从环境准备到生产部署的完整流程。

## 指南结构

### 第一阶段：环境准备
- [01. 环境准备](./01-environment-setup.md)
  - 操作系统要求
  - 基础软件安装（JDK 17, Maven, Git, Node.js）
  - Maven 镜像源配置

### 第二阶段：中间件搭建
- [02. 中间件服务](./02-middleware-setup.md)
  - MySQL 8.3 部署与数据导入
  - Redis 缓存配置
  - Nacos 注册/配置中心

### 第三阶段：后端服务
- [03. 后端构建](./03-backend-build.md)
  - 源码下载与分支切换
  - Maven 构建配置
  - 配置文件批量修改
  - 服务启动（Screen/Docker）

### 第四阶段：前端服务
- [04. 前端构建](./04-frontend-build.md)
  - Vue3 管理后台下载
  - pnpm 依赖安装
  - 本地开发与构建

### 第五阶段：容器化
- [05. 服务容器化](./05-containerization.md)
  - 后端 Docker 镜像制作
  - 前端 Nginx 镜像制作
  - Harbor 私有仓库配置

### 第六阶段：K8S 集群搭建
- [06. 集群搭建](./06-cluster-setup.md)
  - 节点准备（Swap 关闭、Docker 安装）
  - cri-dockerd 安装
  - kubeadm 集群初始化
  - Flannel 网络配置

### 第七阶段：K8S 组件
- [07. 组件安装](./07-components-setup.md)
  - Helm 包管理器
  - Kubernetes Dashboard
  - Nginx Ingress
  - MetalLB 负载均衡

### 第八阶段：项目部署
- [08. 项目部署](./08-project-deployment.md)
  - 镜像推送到 Harbor
  - Deployment 创建
  - Service 与 Ingress 配置
  - 域名解析配置

### 附录
- [A. 常见问题](./appendix-troubleshooting.md)
- [B. 面试知识点](./appendix-interview.md)
- [C. 配置文件清单](./appendix-configs.md)
- [D. 快速参考](./appendix-quickref.md)

## 快速开始

如果您已经有一个运行中的 K8S 集群，可以直接跳转到：

- [08. 项目部署](./08-project-deployment.md) - 将芋道项目部署到现有集群
- [README](../../README.md) - 项目根目录的快速开始指南

## 环境要求

### 基础环境
- 操作系统：CentOS Stream 9 或 Ubuntu 22.04
- 内存：至少 8GB（推荐 16GB）
- CPU：至少 4 核

### K8S 集群
- 版本：1.29.x
- 节点数：至少 2 个（1 Master + 1 Node）
- 网络：Flannel

### 软件版本
| 软件 | 版本 |
|------|------|
| JDK | 17 |
| Maven | 3.6+ |
| Node.js | 18+ |
| Docker | 24.0+ |
| Kubernetes | 1.29.x |

## 贡献

如果您在部署过程中发现问题或有改进建议，欢迎提交 Issue 或 Pull Request。
