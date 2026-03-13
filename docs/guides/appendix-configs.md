# 附录 C：配置文件清单

本章节汇总项目中涉及的配置文件及其用途。

## 环境变量配置

### `.env.local`（前端）

前端本地开发环境配置：

```env
# 后端 API 地址
VITE_API_BASE_URL=http://192.168.206.129:48080
```

### `.env.example`（项目根目录）

项目环境变量示例：

```bash
# Harbor 配置
HARBOR_REGISTRY=192.168.206.129:80
HARBOR_REGISTRY_USER=admin
HARBOR_REGISTRY_PASSWD=Harbor12345

# 镜像版本
VERSION=latest
```

## Maven 配置

### `/etc/maven/settings.xml`

Maven 全局配置，包含镜像源配置：

```xml
<mirrors>
  <mirror>
    <id>aliyunmaven</id>
    <mirrorOf>central</mirrorOf>
    <name>aliyun maven</name>
    <url>https://maven.aliyun.com/repository/public</url>
  </mirror>
</mirrors>
```

## Docker 配置

### `/etc/docker/daemon.json`

Docker 守护进程配置：

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
    "https://huecker.io/"
  ],
  "insecure-registries": ["192.168.206.129:80"],
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

## Kubernetes 配置

### `kubeadm_init.yaml`

Kubernetes 集群初始化配置：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- token: abcdef.0123456789abcdef
  ttl: 24h0m0s
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.206.129  # Master IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.29.1
networking:
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
imageRepository: registry.aliyuncs.com/google_containers
```

### `metallb.yaml`

MetalLB IP 地址池配置：

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.206.200-192.168.206.220  # 根据实际网段修改

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```

## 后端配置

### `application-local.yaml`

后端服务本地环境配置：

```yaml
spring:
  datasource:
    url: jdbc:mysql://192.168.206.129:3306/ruoyi-vue-pro?useSSL=false
    username: root
    password: "123456"

  data:
    redis:
      host: 192.168.206.129
      port: 6379

cloud:
  nacos:
    server-addr: 192.168.206.129:8848
    namespace: dev
```

## 前端配置

### `.env.production`

前端生产环境配置：

```env
VITE_API_BASE_URL=https://api.yudao.com
```

## Kubernetes 资源配置

### Deployment 配置模板

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yudao-gateway
  namespace: dev
  labels:
    app: yudao-gateway
    part-of: yudao
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yudao-gateway
  template:
    metadata:
      labels:
        app: yudao-gateway
        part-of: yudao
    spec:
      containers:
      - name: yudao-gateway
        image: 192.168.206.129:80/library/yudao_gateway:latest
        ports:
        - containerPort: 48080
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

### Service 配置模板

```yaml
apiVersion: v1
kind: Service
metadata:
  name: yudao-gateway
  namespace: dev
spec:
  ports:
  - port: 48080
    targetPort: 48080
  selector:
    app: yudao-gateway
  type: ClusterIP
```

### Ingress 配置模板

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yudao
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - host: api.ymyw.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: yudao-gateway
            port:
              number: 48080
```

## CI/CD 配置

### `.gitlab-ci.yaml`

GitLab CI 流水线配置：

```yaml
stages:
  - build-frontend
  - dockerize

build-frontend:
  stage: build-frontend
  image: node:18-alpine
  script:
    - npm config set registry https://registry.npmmirror.com
    - npm install
    - npm run build:local
  artifacts:
    paths:
      - dist/

dockerize:
  stage: dockerize
  image: docker:24.0.5
  services:
    - name: docker:24.0.5-dind
      command: ["--insecure-registry=192.168.206.129:80"]
  variables:
    DOCKER_HOST: tcp://docker:2375/
    DOCKER_TLS_CERTDIR: ""
  script:
    - docker login -u $HARBOR_REGISTRY_USER -p $HARBOR_REGISTRY_PASSWD $HARBOR_REGISTRY
    - docker build -t ${HARBOR_REGISTRY}/library/yudao_ui_admin:${VERSION} .
    - docker push ${HARBOR_REGISTRY}/library/yudao_ui_admin:${VERSION}
```

## Harbor 配置

### `harbor.yml`

Harbor 配置文件：

```yaml
hostname: 192.168.206.129

http:
  port: 80

# https:
#   port: 443
#   certificate: /your/certificate/path
#   private_key: /your/private/key/path

harbor_admin_password: Harbor12345

database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

data_volume: /data/harbor
```

## 系统配置

### `/etc/hosts`

本地域名解析配置：

```
# Kubernetes
192.168.206.129  k8s-master

# 芋道项目
192.168.206.200  api.ymyw.net
192.168.206.200  www.ymyw.net

# Harbor
192.168.206.129  harbor.xinxainghf.com
```

### `/etc/sysctl.conf`

内核参数配置：

```conf
# 网络转发
net.ipv4.ip_forward=1

# Bridge 网络过滤
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

应用配置：
```bash
sysctl -p
```

### `/etc/modules-load.d/bridge.conf`

内核模块加载配置：

```
br_netfilter
```

## 环境变量配置

### `~/.bashrc`

用户环境变量：

```bash
# Java 17
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.12-oracle-x64
export CLASSPATH=.:${JAVA_HOME}/lib

# Maven
export MAVEN_HOME=/usr/share/maven

# Node.js
export PATH=$PATH:~/yudao/node-v20.11.1-linux-x64/bin

# PATH
export PATH=${JAVA_HOME}/bin:${MAVEN_HOME}/bin:$PATH
```

## 配置文件位置汇总

| 配置文件 | 位置 |
|---------|------|
| Maven settings | `/etc/maven/settings.xml` |
| Docker daemon | `/etc/docker/daemon.json` |
| kubeadm | `~/kubeadm_init.yaml` |
| Harbor | `/opt/harbor/harbor.yml` |
| 前端环境变量 | 项目根目录 `.env.*` |
| 后端配置 | `src/main/resources/application-*.yaml` |
| K8s 资源 | `deploy/k8s/` |
| Kustomize | `deploy/overlays/` |
| CI 配置 | `ci/.gitlab-ci.yaml` |
| Hosts | `/etc/hosts` |

## 配置修改后操作

| 配置 | 修改后操作 |
|------|-----------|
| Maven settings | 无需操作 |
| Docker daemon | `systemctl restart docker` |
| Harbor | `cd /opt/harbor && docker-compose down && docker-compose up -d` |
| sysctl | `sysctl -p` |
| modules | `modprobe <module_name>` |
| bashrc | `source ~/.bashrc` |
| K8s 资源 | `kubectl apply -f <file>` |
| 前端 .env | 重新运行 `npm run dev` 或构建 |
| 后端配置 | 重新构建 JAR 并重启服务 |
