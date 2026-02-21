## 部署目标

在 Kubernetes 集群中部署网关、后端服务与前端，并通过 Ingress 和 MetalLB 暴露访问入口。

## 前置条件

- 集群已通过 kubeadm 初始化并可用
- Flannel、Nginx Ingress、MetalLB 已安装
- Harbor 私有仓库可用，节点已信任仓库地址

## 镜像构建与推送

- 后端服务按模块构建镜像并推送 Harbor
- 前端构建静态资源后打包到 Nginx 镜像并推送 Harbor

## K8s 资源清单

- 应用部署清单：deploy/k8s/yudao_app.yaml
- Service：deploy/k8s/svc_yuao.yaml
- Ingress：deploy/k8s/ingress_yuao.yaml

## 典型部署流程

1. 配置私有仓库信任并登录 Harbor
2. 推送后端与前端镜像到 Harbor
3. 依次应用网络与入口组件清单
4. 应用应用部署清单与 Service、Ingress

## 访问方式

- 通过 Ingress 域名访问 Gateway 与前端
- 如需测试，可在客户端 hosts 中添加域名解析
