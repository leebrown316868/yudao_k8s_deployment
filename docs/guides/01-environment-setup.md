# 01. 环境准备

本章节介绍 CentOS Stream 9 环境的基础软件安装配置。

## 操作系统要求

- **操作系统**：CentOS Stream 9
- **内存**：至少 8GB（推荐 16GB）
- **CPU**：至少 4 核
- **磁盘**：至少 50GB 可用空间

## 基础软件安装

### 1. 安装 Git

```bash
dnf install -y git
```

验证安装：
```bash
git --version
```

### 2. 安装 JDK 17

#### 2.1 下载 JDK

```bash
mkdir -p ~/yudao && cd ~/yudao
wget https://download.oracle.com/java/17/archive/jdk-17.0.12_linux-x64_bin.rpm
```

#### 2.2 安装 JDK

```bash
sudo rpm -ivh jdk-17.0.12_linux-x64_bin.rpm
```

#### 2.3 配置环境变量

编辑 `~/.bashrc`，添加以下内容：

```bash
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.12-oracle-x64
export CLASSPATH=.:${JAVA_HOME}/lib
export PATH=${CLASSPATH}:${JAVA_HOME}/bin:$PATH
```

使配置生效：
```bash
source ~/.bashrc
```

#### 2.4 验证安装

```bash
java --version
```

预期输出：
```
java 17.0.12 2024-07-16 LTS
Java(TM) SE Runtime Environment (build 17.0.12+8-LTS-286)
Java HotSpot(TM) 64-Bit Server VM (build 17.0.12+8-LTS-286, mixed mode, sharing)
```

### 3. 安装配置 Maven

#### 3.1 安装 Maven

```bash
yum install -y maven
```

#### 3.2 配置国内镜像源

编辑 `/etc/maven/settings.xml`，在 `<mirrors>` 标签内添加：

```xml
<mirror>
    <id>aliyunmaven</id>
    <mirrorOf>central</mirrorOf>
    <name>aliyun maven</name>
    <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

验证安装：
```bash
mvn --version
```

### 4. 下载后端源码

```bash
mkdir -p ~/gitcode && cd ~/gitcode
git clone https://gitee.com/zhijiantianya/yudao-cloud.git
cd yudao-cloud/
```

#### 4.1 切换到 JDK17 分支

```bash
git checkout -b master-jdk17 origin/master-jdk17
```

#### 4.2 查看项目版本

查看 `pom.xml` 确认环境版本要求：

```bash
grep -A 5 "java.version" pom.xml
```

## 环境变量汇总

将以下内容添加到 `~/.bashrc`：

```bash
# Java 17
export JAVA_HOME=/usr/lib/jvm/jdk-17.0.12-oracle-x64
export CLASSPATH=.:${JAVA_HOME}/lib

# Maven
export MAVEN_HOME=/usr/share/maven

# PATH
export PATH=${JAVA_HOME}/bin:${MAVEN_HOME}/bin:$PATH
```

重新加载：
```bash
source ~/.bashrc
```

## 验证清单

| 软件 | 命令 | 预期版本 |
|------|------|----------|
| Git | `git --version` | git version 2.x |
| Java | `java --version` | 17.0.12 |
| Maven | `mvn --version` | Apache Maven 3.6+ |

## 常见问题

### JDK 安装后版本不正确

```bash
# 查看已安装的 Java 版本
alternatives --list | grep java

# 设置默认 Java 版本
alternatives --config java
```

### Maven 下载依赖缓慢

确保已配置阿里云镜像源，检查配置：

```bash
grep -A 5 "aliyunmaven" /etc/maven/settings.xml
```

## 下一步

环境准备完成后，继续阅读 [02. 中间件服务](./02-middleware-setup.md)。
