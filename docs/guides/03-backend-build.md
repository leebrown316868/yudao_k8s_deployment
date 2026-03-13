# 03. 后端构建

本章节介绍芋道后端服务的构建、配置和启动。

## 项目结构

```
yudao-cloud/
├── yudao-gateway/                    # 网关服务
│   └── target/yudao-gateway.jar
├── yudao-module-system/              # 系统模块
│   └── yudao-module-system-biz/
│       └── target/yudao-module-system-biz.jar
└── yudao-module-infra/               # 基础设施模块
    └── yudao-module-infra-biz/
        └── target/yudao-module-infra-biz.jar
```

## 构建服务

### 1. 进入项目目录

```bash
cd ~/gitcode/yudao-cloud/
```

### 2. 执行 Maven 构建

```bash
mvn clean install package '-Dmaven.test.skip=true'
```

> **说明**：
> - 构建时间取决于网络和机器性能
> - 构建完成后，各服务目录下会生成 `target/*.jar` 文件
> - 跳过测试以加快构建速度

### 3. 验证构建结果

```bash
find ./ -name "*.jar" -path "*/target/*" | grep -v sources | grep -v javadoc
```

预期输出：
```
./yudao-gateway/target/yudao-gateway.jar
./yudao-module-system/yudao-module-system-biz/target/yudao-module-system-biz.jar
./yudao-module-infra/yudao-module-infra-biz/target/yudao-module-infra-biz.jar
```

## 服务架构

```
                        ┌─────────────────┐
                        │   Nginx Ingress │
                        └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │     Gateway     │
                        │   Port: 48080   │
                        └────────┬────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
    ┌───────▼───────┐  ┌────────▼────────┐  ┌───────▼───────┐
    │    System     │  │      Infra      │  │     ...       │
    │  Port: 48081  │  │   Port: 48082   │  │              │
    └───────────────┘  └─────────────────┘  └───────────────┘
```

## 启动服务

### 使用 Screen 管理

**概述**：Screen 允许在多个虚拟终端中运行程序，终端关闭后程序继续运行。

#### 基本操作

```bash
# 创建/恢复会话
screen -R <name>

# 分离会话（回到终端）
Ctrl+a, 然后按 d

# 列出所有会话
screen -ls

# 关闭会话
screen -S <ID> -X quit
```

### 1. 启动 Gateway 服务

**功能**：API 网关，提供用户认证、服务路由、灰度发布、访问日志、异常处理

```bash
# 创建 screen 会话
screen -R gateway

# 启动服务
cd ~/gitcode/yudao-cloud/
java -jar yudao-gateway/target/yudao-gateway.jar
```

启动成功示例：
```
  ____                   _   __      _
 / ___| _   _ _ __   __| | | \ \    | |
| |  _ | | | | '_ \ / _` | | |\ \   | |
| |_| || |_| | | | | (_| | | | \ \  | |
 \____|\__, |_| |_|\__,_|_|_|  \_\ |_|
       |___/
```

分离 screen：`Ctrl+a`，然后按 `d`

验证服务：
```bash
curl http://localhost:48080/actuator/health
```

### 2. 启动 System 服务

**功能**：系统模块，包括用户管理、角色权限、系统设置

```bash
# 创建 screen 会话
screen -R system

# 启动服务
cd ~/gitcode/yudao-cloud/
java -jar yudao-module-system/yudao-module-system-biz/target/yudao-module-system-biz.jar
```

验证服务：
```bash
# 直连服务
curl http://localhost:48081/admin-api/system

# 通过网关访问
curl http://localhost:48080/admin-api/system
```

### 3. 启动 Infra 服务

**功能**：基础设施模块，包括日志、任务调度、资源监控、消息队列

```bash
# 创建 screen 会话
screen -R infra

# 启动服务
cd ~/gitcode/yudao-cloud/
java -jar yudao-module-infra/yudao-module-infra-biz/target/yudao-module-infra-biz.jar
```

验证服务：
```bash
# 直连服务
curl http://localhost:48082/admin-api/infra/

# 通过网关访问
curl http://localhost:48080/admin-api/infra/
```

## 服务端口清单

| 服务 | 端口 | 说明 |
|------|------|------|
| Gateway | 48080 | API 网关入口 |
| System | 48081 | 系统模块 |
| Infra | 48082 | 基础设施模块 |

## 停止服务

```bash
# 查看 screen 会话
screen -ls

# 强制关闭指定会话
screen -S <ID> -X quit

# 或在 screen 会话中按 Ctrl+c 停止服务
```

## 配置文件位置

服务配置文件在各模块的 `src/main/resources/` 目录下：

- `application.yaml` - 主配置
- `application-local.yaml` - 本地环境配置（包含数据库、Redis、Nacos 地址）

## 常见问题

### 服务启动失败

```bash
# 检查端口占用
netstat -tunlp | grep -E "48080|48081|48082"

# 查看 screen 日志
screen -r gateway
```

### 无法连接 Nacos

1. 确认 Nacos 服务运行正常
2. 检查 `application-local.yaml` 中的 Nacos 地址
3. 确认命名空间已创建

### 无法连接数据库

1. 确认 MySQL 容器运行正常
2. 检查数据库地址和密码配置
3. 确认数据库已初始化

## 下一步

后端服务启动后，继续阅读 [04. 前端构建](./04-frontend-build.md)。
