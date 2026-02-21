# CI/CD 流水线文档

## 概述

本项目的 CI/CD 流水线基于 GitLab CI/CD 构建，当前实现了完整的构建、测试与部署能力。

**注意**：本文档描述了流水线的完整配置，当前实际使用的是基础版本（仅前端构建），完整配置已就绪，待 GitLab Runner 配置完成后即可启用。

## 流水线架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           GitLab CI/CD Pipeline                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────┐    ┌──────┐    ┌─────────────────────────────────────────┐   │
│  │ Lint │ -> │ Test │ -> │                 Build                    │   │
│  └──────┘    └──────┘    ├─────────┬─────────┬─────────┬───────────┤   │
│                           │         │         │         │           │   │
│                           │ Frontend│  Gateway│  System │   Infra   │   │
│                           │         │         │         │           │   │
│                           └─────────┴─────────┴─────────┴───────────┘   │
│                                          │                               │
│                                          v                               │
│                           ┌─────────────────────────┐                    │
│                           │        Deploy           │                    │
│                           ├───────────┬─────────────┤                    │
│                           │    Dev    │    Prod     │                    │
│                           └───────────┴─────────────┘                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 当前状态

### 已启用（基础版本）

- **build-frontend**：前端静态资源构建
- **dockerize**：前端镜像构建与推送

### 已配置（待启用）

以下配置已完成，待 GitLab Runner 配置完成后即可使用：

- **lint-frontend**：前端代码规范检查（ESLint）
- **lint-backend**：后端代码规范检查（Checkstyle）
- **test-backend**：后端单元测试
- **dockerize-gateway**：Gateway 镜像构建与推送
- **dockerize-system**：System 镜像构建与推送
- **dockerize-infra**：Infra 镜像构建与推送
- **deploy-dev**：部署到开发环境
- **deploy-prod**：部署到生产环境

## 流水线阶段详解

### 1. Lint 阶段（待启用）

代码规范检查：

- **lint-frontend**：前端代码 ESLint 检查
- **lint-backend**：后端代码 Checkstyle 检查

### 2. Test 阶段（待启用）

单元测试阶段：

- **test-backend**：运行后端单元测试，生成覆盖率报告

### 3. Build 阶段

#### 前端构建（已启用）

- **build-frontend**：构建前端静态资源
  - 使用 Node 18 镜像
  - 产物保留为 artifacts
- **dockerize**：构建并推送前端镜像
  - 使用 Docker-in-Docker
  - 推送到 Harbor 私有仓库

#### 后端构建（待启用）

- **dockerize-gateway**：构建并推送 Gateway 镜像
- **dockerize-system**：构建并推送 System 镜像
- **dockerize-infra**：构建并推送 Infra 镜像

**特点**：
- 使用 Git 提交短 SHA 作为版本号
- 同时推送版本标签和 latest 标签
- 支持增量构建（仅在相关文件变化时触发）

### 4. Deploy 阶段（待启用）

部署阶段：

- **deploy-dev**：部署到开发环境（develop 分支，手动触发）
- **deploy-prod**：部署到生产环境（main/master 分支，手动触发）

部署方式：
- 使用 kubectl 滚动更新
- 等待 rollout 完成确认

## 环境变量

流水线需要以下变量在 GitLab 项目中配置：

### 必需变量

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `HARBOR_REGISTRY` | Harbor 仓库地址 | `192.168.163.135` |
| `HARBOR_NAMESPACE` | Harbor 命名空间 | `library` |
| `HARBOR_REGISTRY_USER` | Harbor 用户名 | `admin` |
| `HARBOR_REGISTRY_PASSWD` | Harbor 密码 | `******` |

### 自动变量

| 变量名 | 说明 | 来源 |
|--------|------|------|
| `VERSION` | 镜像版本号 | `${CI_COMMIT_SHORT_SHA}` |
| `IMAGE_TAG_LATEST` | latest 标签 | `latest` |

## 配置 GitLab Runner

### 安装 Runner

```bash
# 在 Linux 节点上安装 GitLab Runner
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner

# 注册 Runner
sudo gitlab-runner register
```

### Runner 配置建议

- **Executor**：Docker
- **Docker Image**：docker:24.0.5
- **Privileged**：true（用于 Docker-in-Docker）
- **Volumes**：挂载 /var/run/docker.sock

## 使用示例

### 触发流水线

```bash
# 推送代码自动触发
git push origin feature/new-feature

# 创建 Merge Request
git push origin feature/new-feature
# 在 GitLab UI 中创建 MR
```

### 手动部署（待启用）

1. 进入 GitLab 项目 → CI/CD → Pipelines
2. 找到对应的流水线
3. 点击 "Deploy" 阶段右侧的播放按钮
4. 选择目标环境（dev/prod）

## 后续扩展

- [ ] 配置 GitLab Runner 启用完整流水线
- [ ] 添加 Kustomize 构建阶段
- [ ] 集成部署与 Kustomize
- [ ] 添加安全扫描阶段（Trivy、Snyk）
- [ ] 添加性能测试阶段
- [ ] 实现金丝雀发布
