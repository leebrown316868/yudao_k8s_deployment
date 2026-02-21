## 排障原则

- 先看事件与日志，再定位到具体 Pod 与容器
- 先验证依赖组件是否可用，再排查业务服务

## 常见问题与处理

### ImagePullBackOff

- 现象：Pod 一直处于 ImagePullBackOff
- 排查：查看 Pod 事件与拉取日志
- 处理：确认仓库地址可达、凭证有效、镜像存在

### Flannel 异常

- 现象：Pod 网络不通或 kube-flannel 异常
- 排查：检查 br_netfilter 是否加载、查看 kube-flannel 日志
- 处理：加载模块并重建异常 Pod

### K8s 软件源问题

- 现象：安装组件时报 repomd.xml GPG 校验失败
- 排查：确认镜像源密钥状态
- 处理：更换镜像源或关闭 gpg 校验

### Namespace 卡在 Terminating

- 现象：命名空间长时间无法删除
- 排查：导出命名空间 JSON 查看 finalizers
- 处理：移除 finalizers 后强制替换

### Docker 构建缓存未更新

- 现象：镜像更新后 Pod 仍运行旧版本
- 排查：确认镜像 tag 与构建缓存
- 处理：使用 --no-cache 重新构建镜像
