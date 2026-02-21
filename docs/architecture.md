## 项目定位

面向 Kubernetes 环境的 Java 微服务系统部署与运维实践项目，重点覆盖集群搭建、镜像构建、私有仓库、服务编排与发布入口治理。

## 组件与职责

- Kubernetes 集群：承载所有业务与基础组件
- Flannel：提供 Pod 网络
- Nginx Ingress：对外统一流量入口
- MetalLB：为裸金属环境提供 LoadBalancer IP
- Harbor：私有镜像仓库
- MySQL、Redis、Nacos：业务依赖的基础中间件
- Gateway/System/Infra：后端微服务
- 前端管理系统：静态资源通过 Nginx 提供

## 访问路径

- 外部流量通过 Ingress 进入
- Ingress 根据域名与路径规则路由到 Gateway 或前端服务
- Gateway 通过服务发现访问后端业务服务
- 业务服务依赖 Nacos、MySQL、Redis

## 基础环境与版本

| 组件 | 版本/说明 |
| --- | --- |
| Kubernetes | 1.29 |
| 操作系统 | Ubuntu 22.04 / Rocky Linux 9 |
| 容器运行时 | Docker 26.0 + cri-dockerd |
| JDK | 17 |
| MySQL | 8.3 |
| Nacos | 2.5.1 |
| 前端 | Vue3 |

## 资源清单位置

- K8s 资源清单：deploy/k8s
- 集群初始化配置：deploy/k8s/kubeadm_init.yaml
- 网络与入口：deploy/k8s/kube-flannel.yml、deploy/k8s/nginx_ingress.yaml
- 负载均衡：deploy/k8s/metallb.yml
