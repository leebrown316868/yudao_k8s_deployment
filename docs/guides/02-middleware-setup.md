# 02. 中间件服务

本章节介绍芋道项目所需的中间件服务部署，包括 MySQL、Redis 和 Nacos。

## 架构概述

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│      MySQL      │      │      Redis      │      │      Nacos      │
│   (数据存储)     │      │   (缓存/Token)   │      │  (注册/配置)     │
│   Port: 3306    │      │   Port: 6379    │      │  Port: 8848    │
└─────────────────┘      └─────────────────┘      └─────────────────┘
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   │
                          ┌────────▼─────────┐
                          │  芋道微服务集群    │
                          └──────────────────┘
```

## MySQL 部署

### 作用说明

- 存储用户信息、业务数据、配置信息
- 支持事务和关系型数据查询

### 部署 MySQL

```bash
docker run -d -p 3306:3306 \
  --restart=unless-stopped \
  --name=yudao_mysql \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -v "/etc/localtime:/etc/localtime" \
  -v yc_mysql:/var/lib/mysql \
  mysql:8.3
```

### 导入数据

#### 1. 复制 SQL 文件到容器

```bash
docker cp /tmp/ruoyi-vue-pro.sql yudao_mysql:/tmp/ruoyi-vue-pro.sql
```

#### 2. 创建数据库

```bash
docker exec -it yudao_mysql mysql -uroot -p123456
```

在 MySQL 命令行中执行：

```sql
CREATE DATABASE `ruoyi-vue-pro`;
EXIT;
```

> **注意**：数据库名包含 `-` 字符，必须使用反引号

#### 3. 导入数据

```bash
docker exec -it yudao_mysql mysql -uroot -p123456 ruoyi-vue-pro < /tmp/ruoyi-vue-pro.sql
```

或在 MySQL 命令行中：

```sql
USE `ruoyi-vue-pro`;
SOURCE /tmp/ruoyi-vue-pro.sql;
```

### 修改后端配置

查找所有 `application-local.yaml` 文件：

```bash
find ~/gitcode/yudao-cloud/ -name application-local.yaml
```

批量修改数据库地址（替换为你的 IP）：

```bash
cd ~/gitcode/yudao-cloud/
find ./ -name application-local.yaml -print0 | xargs -0 sed -i 's|jdbc:mysql://127.0.0.1:3306|jdbc:mysql://192.168.206.129:3306|g'
```

验证修改：

```bash
find ./ -name application-local.yaml -exec grep 'jdbc:mysql://.*:3306' {} +
```

## Redis 部署

### 作用说明

- 缓存数据，提升响应速度
- 存储 JWT Token 或 Session ID，实现会话共享
- 实现访问限流功能

### 部署 Redis

```bash
docker run -d \
  --restart=unless-stopped \
  --name=yudao_redis \
  -v "/etc/localtime:/etc/localtime" \
  -p 6379:6379 \
  redis
```

### 修改后端配置

批量修改 Redis 地址：

```bash
cd ~/gitcode/yudao-cloud/
find ./ -name application-local.yaml -print0 | xargs -0 sed -i 's|host: 127.0.0.1 # 地址|host: 192.168.206.129 # 地址|g'
```

验证修改：

```bash
find ./ -name application-local.yaml -exec grep 'host:.*# 地址' {} +
```

## Nacos 部署

### 作用说明

**注册中心功能**：
- 服务注册与发现
- 动态服务路由
- 健康检查

**配置中心功能**：
- 集中管理配置
- 动态配置更新
- 配置版本管理

### 部署 Nacos

```bash
docker run -d \
  -p 8848:8848 \
  -p 9848:9848 \
  --restart=unless-stopped \
  --name=yudao_nacos \
  -e MODE=standalone \
  -v "/etc/localtime:/etc/localtime" \
  nacos/nacos-server:v2.5.1
```

### 访问 Nacos 控制台

浏览器访问：`http://<你的IP>:8848/nacos`

默认账号密码：
- 用户名：`nacos`
- 密码：`nacos`

### 创建命名空间

1. 登录 Nacos 控制台
2. 进入「命名空间」页面
3. 点击「新建命名空间」
4. 创建以下命名空间：
   - `dev` - 开发环境
   - `test` - 测试环境
   - `prod` - 生产环境

### 修改后端配置

批量修改 Nacos 地址：

```bash
cd ~/gitcode/yudao-cloud/
find ./ -name application-local.yaml -print0 | xargs -0 sed -i 's|server-addr: 127.0.0.1:8848|server-addr: 192.168.206.129:8848|g'
```

验证修改：

```bash
find ./ -name application-local.yaml -exec grep 'server-addr:.*:8848' {} +
```

## 中间件端口清单

| 中间件 | 端口 | 用途 |
|--------|------|------|
| MySQL | 3306 | 数据库连接 |
| Redis | 6379 | 缓存连接 |
| Nacos | 8848 | 控制台/HTTP API |
| Nacos | 9848 | gRPC 客户端 |

## 验证清单

```bash
# 检查容器状态
docker ps | grep -E "yudao_mysql|yudao_redis|yudao_nacos"

# 检查 MySQL 连接
docker exec yudao_mysql mysql -uroot -p123456 -e "SELECT 1"

# 检查 Redis 连接
docker exec yudao_redis redis-cli ping

# 检查 Nacos
curl http://localhost:8848/nacos/
```

## 常见问题

### MySQL 连接失败

```bash
# 查看日志
docker logs yudao_mysql

# 检查端口
netstat -tunlp | grep 3306
```

### Nacos 无法启动

```bash
# 查看日志
docker logs yudao_nacos

# 确保单机模式
docker inspect yudao_nacos | grep MODE
```

### 配置修改不生效

修改配置后需要重新构建后端服务，详见 [03. 后端构建](./03-backend-build.md)。

## 下一步

中间件部署完成后，继续阅读 [03. 后端构建](./03-backend-build.md)。
