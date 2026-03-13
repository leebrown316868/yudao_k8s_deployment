# 附录 D：快速参考

本章节提供常用命令和操作的快速参考。

## Docker 命令

### 镜像操作

```bash
# 查看镜像
docker images

# 拉取镜像
docker pull <image>:<tag>

# 构建镜像
docker build -t <name>:<tag> <path>

# 打标签
docker tag <source> <target>

# 推送镜像
docker push <image>:<tag>

# 删除镜像
docker rmi <image>
```

### 容器操作

```bash
# 查看运行中的容器
docker ps

# 查看所有容器
docker ps -a

# 运行容器
docker run -d --name <name> -p <host_port>:<container_port> <image>

# 停止容器
docker stop <container>

# 启动容器
docker start <container>

# 重启容器
docker restart <container>

# 删除容器
docker rm <container>

# 查看日志
docker logs <container>
docker logs -f <container>  # 实时查看

# 进入容器
docker exec -it <container> /bin/bash

# 复制文件到容器
docker cp <src> <container>:<dest>

# 从容器复制文件
docker cp <container>:<src> <dest>
```

### 中间件部署

```bash
# MySQL
docker run -d -p 3306:3306 \
  --restart=unless-stopped \
  --name=yudao_mysql \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -v "/etc/localtime:/etc/localtime" \
  -v yc_mysql:/var/lib/mysql \
  mysql:8.3

# Redis
docker run -d \
  --restart=unless-stopped \
  --name=yudao_redis \
  -v "/etc/localtime:/etc/localtime" \
  -p 6379:6379 \
  redis

# Nacos
docker run -d \
  -p 8848:8848 -p 9848:9848 \
  --restart=unless-stopped \
  --name=yudao_nacos \
  -e MODE=standalone \
  -v "/etc/localtime:/etc/localtime" \
  nacos/nacos-server:v2.5.1
```

## Kubernetes 命令

### 集群管理

```bash
# 查看节点
kubectl get nodes

# 查看节点详情
kubectl describe node <node-name>

# 查看集群信息
kubectl cluster-info

# 查看 API 版本
kubectl api-version
```

### 资源操作

```bash
# 查看 Pod
kubectl get pods -A
kubectl get pods -n <namespace>

# 查看 Deployment
kubectl get deployments -A

# 查看 Service
kubectl get svc -A

# 查看 Ingress
kubectl get ingress -A

# 查看所有资源
kubectl get all -n <namespace>
```

### 创建和删除

```bash
# 从文件创建资源
kubectl apply -f <file.yaml>
kubectl create -f <file.yaml>

# 删除资源
kubectl delete -f <file.yaml>
kubectl delete pod <pod-name> -n <namespace>
kubectl delete deployment <deployment-name> -n <namespace>

# 强制删除
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

### 日志和调试

```bash
# 查看 Pod 日志
kubectl logs <pod-name> -n <namespace>

# 实时查看日志
kubectl logs -f <pod-name> -n <namespace>

# 查看之前的日志
kubectl logs <pod-name> -n <namespace> --previous

# 查看多个 Pod 的日志
kubectl logs -l app=<app-name> -n <namespace> --tail=100 -f

# 查看 Pod 详情
kubectl describe pod <pod-name> -n <namespace>

# 进入容器
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
```

### 配置管理

```bash
# 查看配置
kubectl config view

# 切换上下文
kubectl config use-context <context>

# 查看当前上下文
kubectl config current-context

# 查看所有上下文
kubectl config get-contexts
```

### 故障排查

```bash
# 查看 Pod 事件
kubectl describe pod <pod-name> -n <namespace>

# 查看 Node 事件
kubectl describe node <node-name>

# 查看资源使用
kubectl top nodes
kubectl top pods -n <namespace>

# 查看端点
kubectl get endpoints -n <namespace>
```

## Screen 命令

```bash
# 创建/恢复会话
screen -R <name>

# 分离会话
# Ctrl+a, 然后按 d

# 列出所有会话
screen -ls

# 关闭会话
screen -S <session-id> -X quit

# 强制关闭所有会话
screen -ls | grep -E '[0-9]+' | awk '{print $1}' | xargs -I {} screen -S {} -X quit
```

## Git 命令

```bash
# 克隆仓库
git clone <url>

# 切换分支
git checkout -b <branch> origin/<branch>

# 查看状态
git status

# 添加文件
git add <file>
git add .

# 提交
git commit -m "message"

# 推送
git push

# 拉取
git pull
```

## Maven 命令

```bash
# 清理构建
mvn clean

# 编译
mvn compile

# 打包
mvn package

# 跳过测试
mvn package -Dmaven.test.skip=true

# 清理并打包
mvn clean package -Dmaven.test.skip=true

# 安装到本地仓库
mvn install
```

## npm/pnpm 命令

```bash
# 设置镜像源
npm config set registry https://registry.npmmirror.com
pnpm config set registry https://registry.npmmirror.com

# 安装依赖
npm install
pnpm install

# 运行开发服务器
npm run dev

# 构建
npm run build

# 构建（本地环境）
npm run build:local

# 全局安装
npm install -g <package>
```

## 系统操作

### 防火墙

```bash
# 查看状态
sudo firewall-cmd --state

# 查看规则
sudo firewall-cmd --list-all

# 开放端口
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --reload

# 关闭防火墙
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

### Swap

```bash
# 临时关闭
swapoff -a

# 永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 查看 swap 状态
free -h
```

### 系统服务

```bash
# 启动服务
systemctl start <service>

# 停止服务
systemctl stop <service>

# 重启服务
systemctl restart <service>

# 开机自启
systemctl enable <service>

# 查看状态
systemctl status <service>

# 查看日志
journalctl -u <service>
journalctl -u <service> -f  # 实时查看
```

## 网络操作

```bash
# 查看端口监听
netstat -tunlp | grep <port>

# 查看 IP 地址
ip addr show

# 查看路由
ip route show

# 测试连通性
ping <host>

# 测试端口
telnet <host> <port>
curl http://<host>:<port>
```

## Harbor 操作

```bash
# 登录
docker login <harbor-address>

# 推送镜像
docker push <harbor-address>/<project>/<image>:<tag>

# 拉取镜像
docker pull <harbor-address>/<project>/<image>:<tag>
```

## 快速部署流程

### 开发环境

```bash
# 1. 创建命名空间
kubectl create namespace dev

# 2. 部署应用
kubectl apply -k deploy/overlays/dev/

# 3. 配置 hosts
echo "<MetalLB-IP>  dev-api.yudao.com" >> /etc/hosts
echo "<MetalLB-IP>  dev-www.yudao.com" >> /etc/hosts

# 4. 验证
kubectl get pods -l part-of=yudao -n dev
```

### 生产环境

```bash
# 1. 创建命名空间
kubectl create namespace production

# 2. 创建镜像拉取密钥
kubectl create secret docker-registry harbor-registry \
  --docker-server=<harbor-address> \
  --docker-username=<username> \
  --docker-password=<password> \
  -n production

# 3. 部署应用
kubectl apply -k deploy/overlays/prod/

# 4. 配置 hosts（如需）
echo "<MetalLB-IP>  api.yudao.com" >> /etc/hosts
echo "<MetalLB-IP>  www.yudao.com" >> /etc/hosts

# 5. 验证
kubectl get pods -l part-of=yudao -n production
```

## 常用端口

| 服务 | 端口 |
|------|------|
| MySQL | 3306 |
| Redis | 6379 |
| Nacos | 8848, 9848 |
| Gateway | 48080 |
| System | 48081 |
| Infra | 48082 |
| Harbor | 80, 443 |
| Kubernetes API | 6443 |
| Nginx Ingress | 80, 443 |
| Dashboard | 8443 |

## 获取帮助

```bash
# Docker
docker --help
docker <command> --help

# Kubernetes
kubectl --help
kubectl <command> --help

# Screen
screen --help

# Git
git --help
git <command> --help
```
