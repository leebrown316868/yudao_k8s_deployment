# 05. 服务容器化

本章节介绍将前后端服务制作成 Docker 镜像并推送到 Harbor 私有仓库。

## 容器化概述

```
┌───────────────────────────────────────────────────────────────┐
│                        Harbor 仓库                             │
│  192.168.206.129:80/library/                                  │
│  ├── yudao_gateway                                            │
│  ├── yudao_system                                             │
│  ├── yudao_infra                                              │
│  └── yudao_ui_admin                                           │
└───────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Gateway   │      │   System    │      │    Infra    │
│  :48080     │      │  :48081     │      │  :48082     │
└─────────────┘      └─────────────┘      └─────────────┘
```

## 后端镜像制作

### Gateway 镜像

```bash
cd ~/gitcode/yudao-cloud/yudao-gateway
docker build -t yudao_gateway .
```

### System 镜像

```bash
cd ~/gitcode/yudao-cloud/yudao-module-system/yudao-module-system-biz
docker build -t yudao_system .
```

### Infra 镜像

```bash
cd ~/gitcode/yudao-cloud/yudao-module-infra/yudao-module-infra-biz
docker build -t yudao_infra .
```

### 验证镜像

```bash
docker images | grep yudao
```

预期输出：
```
yudao_gateway   latest   xxx   xxx   ago   xxx MB
yudao_system    latest   xxx   xxx   ago   xxx MB
yudao_infra     latest   xxx   xxx   ago   xxx MB
```

## 启动后端容器

### 停止之前的服务

```bash
# 查看 screen 会话
screen -ls

# 关闭所有 screen 会话
screen -ls | grep -E '[0-9]+' | awk '{print $1}' | xargs -I {} screen -S {} -X quit
```

### 启动容器（使用 host 网络）

使用 host 网络模式，使容器使用宿主机 IP，避免 K8S 集群通信问题。

```bash
# 启动 gateway
docker run -d \
  --network=host \
  --name yudao_gateway \
  -v "/etc/localtime:/etc/localtime" \
  yudao_gateway

# 启动 system
docker run -d \
  --network=host \
  --name yudao_system \
  -v "/etc/localtime:/etc/localtime" \
  yudao_system

# 启动 infra
docker run -d \
  --network=host \
  --name yudao_infra \
  -v "/etc/localtime:/etc/localtime" \
  yudao_infra
```

### 查看容器状态

```bash
docker ps | grep yudao
```

### 查看日志

```bash
# 实时查看日志（Ctrl+C 退出）
docker logs -f yudao_gateway
docker logs -f yudao_system
docker logs -f yudao_infra
```

## 前端镜像制作

### 环境要求

- 操作系统：Linux CentOS
- 内存：推荐 8GB+（构建过程消耗较大）

### 安装 Node.js

```bash
cd ~/yudao
wget https://nodejs.org/dist/v20.11.1/node-v20.11.1-linux-x64.tar.xz
tar -xf node-v20.11.1-linux-x64.tar.xz
```

配置环境变量，编辑 `~/.bashrc`：

```bash
export PATH=$PATH:~/yudao/node-v20.11.1-linux-x64/bin
```

使配置生效：
```bash
source ~/.bashrc
```

验证安装：
```bash
node --version
```

### 下载前端项目

```bash
cd ~/yudao
git clone https://gitee.com/yudaocode/yudao-ui-admin-vue3.git
cd yudao-ui-admin-vue3
```

### 配置后端地址

编辑 `.env.local`：

```env
# 使用负载均衡域名
VITE_API_BASE_URL=http://api.ymyw.net
```

### 构建项目

```bash
# 设置 npm 源
npm config set registry https://registry.npmmirror.com

# 安装依赖
npm install

# 构建项目
npm run build:local
```

构建完成后，静态资源在 `dist/` 目录。

### 制作 Docker 镜像

创建 `Dockerfile`：

```dockerfile
FROM nginx:alpine
COPY ./dist/ /usr/share/nginx/html/
```

构建镜像：
```bash
docker build -t yudao_ui_admin .
```

### 启动前端容器

```bash
docker run --name yudao_ui_admin -d -p 8080:80 yudao_ui_admin
```

验证：
浏览器访问 `http://192.168.206.129:8080`

## Harbor 私有仓库

### Harbor 简介

Harbor 是一个功能强大的私有 Docker 镜像仓库，提供：
- 镜像复制与同步
- 安全漏洞扫描
- 访问控制与审计
- 图形化管理界面

### 下载 Harbor

官网：https://github.com/goharbor/harbor/releases

下载并解压：
```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.2/harbor-offline-installer-v2.10.2.tgz
tar -xzf harbor-offline-installer-v2.10.2.tgz
cd harbor
```

### 配置 Harbor

```bash
# 复制配置文件
cp harbor.yml.tmpl harbor.yml

# 修改配置
vim harbor.yml
```

关键配置：
```yaml
hostname: 192.168.206.129

# 注释 https 部分（开发环境）
# https:
#   port: 443
#   certificate: ...
#   private_key: ...

http:
  port: 80
```

### 安装 Harbor

```bash
./install.sh
```

### 配置自动启动

创建 systemd 服务：

```bash
vim /etc/systemd/system/harbor.service
```

```ini
[Unit]
Description=Harbor Container Registry
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/opt/harbor
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down

[Install]
WantedBy=multi-user.target
```

启用服务：
```bash
systemctl daemon-reload
systemctl enable --now harbor
```

### 登录 Harbor

- 地址：`http://192.168.206.129`
- 账号：`admin`
- 密码：`Harbor12345`

首次登录后建议修改密码。

## 推送镜像到 Harbor

### 配置 Docker 信任 Harbor

编辑 `/etc/docker/daemon.json`：

```json
{
  "insecure-registries": ["192.168.206.129:80"]
}
```

重启 Docker：
```bash
systemctl restart docker
```

### 登录 Harbor

```bash
docker login 192.168.206.129:80
```

输入用户名和密码。

### 打标签

```bash
# 后端镜像
docker tag yudao_gateway 192.168.206.129:80/library/yudao_gateway
docker tag yudao_system 192.168.206.129:80/library/yudao_system
docker tag yudao_infra 192.168.206.129:80/library/yudao_infra

# 前端镜像
docker tag yudao_ui_admin 192.168.206.129:80/library/yudao_ui_admin
```

### 推送镜像

```bash
docker push 192.168.206.129:80/library/yudao_gateway
docker push 192.168.206.129:80/library/yudao_system
docker push 192.168.206.129:80/library/yudao_infra
docker push 192.168.206.129:80/library/yudao_ui_admin
```

### 验证推送

登录 Harbor 网页，进入 `library` 项目查看镜像。

## 常见问题

### Docker 构建复用缓存

修改 JAR 包后构建镜像未更新，禁用缓存重新构建：

```bash
docker build --no-cache -t 192.168.206.129:80/library/yudao_infra:v1 .
```

### Harbor 无法启动

```bash
# 查看日志
docker-compose logs -f

# 重启 Harbor
cd /opt/harbor
docker-compose down
docker-compose up -d
```

### 镜像推送失败

```bash
# 检查登录状态
docker login 192.168.206.129:80

# 检查镜像标签
docker images | grep 192.168.206.129
```

## 下一步

镜像推送完成后，继续阅读 [06. 集群搭建](./06-cluster-setup.md)。
