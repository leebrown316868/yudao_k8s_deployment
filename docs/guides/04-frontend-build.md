# 04. 前端构建

本章节介绍芋道前端管理项目的下载、构建和运行。

## 环境要求

- **操作系统**：Windows（开发）/ Linux（构建）
- **Node.js**：18+
- **包管理器**：pnpm

## Windows 环境准备

### 1. 安装基础软件

下载并安装以下软件：

| 软件 | 下载地址 | 用途 |
|------|----------|------|
| Git | https://git-scm.com/ | 代码下载 |
| Node.js | https://nodejs.org/ | 运行环境 |
| VSCode | https://code.visualstudio.com/ | 代码编辑 |

### 2. 配置 Git 环境变量

1. `Win + R` 输入 `sysdm.cpl`
2. 进入「高级」→「环境变量」
3. 在系统变量中找到 `Path`，点击编辑
4. 添加 Git 安装路径的 `bin` 目录，例如：`C:\Program Files\Git\bin`

验证安装：
```cmd
git --version
```

## 下载前端项目

### 1. 创建工作目录

在 D 盘下创建 `gitcode` 目录。

### 2. 使用 Git Bash 下载

打开 Git Bash，执行：

```bash
cd /d/gitcode
git clone https://gitee.com/yudaocode/yudao-ui-admin-vue3.git
```

### 3. 用 VSCode 打开项目

```
File -> Open Folder -> 选择 yudao-ui-admin-vue3 目录
```

## 项目结构

```
yudao-ui-admin-vue3/
├── src/                    # 源代码
├── public/                 # 静态资源
├── .env.local             # 本地环境配置
├── .env.production        # 生产环境配置
├── package.json           # 依赖配置
└── dist/                  # 构建输出（生成）
```

## 配置项目

### 1. 配置 npm 镜像源

```bash
npm config set registry https://registry.npmmirror.com
```

### 2. 安装 pnpm

```bash
npm install -g pnpm
```

### 3. 配置 pnpm 镜像源

```bash
pnpm config set registry https://registry.npmmirror.com
```

### 4. 安装项目依赖

```bash
cd yudao-ui-admin-vue3
pnpm install
```

> **说明**：pnpm 使用硬链接和符号链接，避免重复下载和存储依赖包

## 配置后端连接

编辑 `.env.local` 文件：

```env
# 后端 API 地址
VITE_API_BASE_URL=http://192.168.206.129:48080
```

> **注意**：将 IP 地址替换为你的后端服务实际地址

## 本地开发

### 启动开发服务器

```bash
npm run dev
```

开发服务器启动后，浏览器访问：`http://localhost:5173`

### 登录测试

默认账号密码：
- 用户名：`admin`
- 密码：`admin123`

### 登录失败处理

如果出现"后端失联"错误：

1. 检查 `.env.local` 中的 `VITE_API_BASE_URL` 配置
2. 确认后端服务正常运行
3. 检查防火墙或 VPN 连接

登录成功示例：
```
┌──────────────────────────────────────┐
│         芋道管理后台                   │
│  ┌────────────────────────────────┐  │
│  │  用户名：[admin      ]         │  │
│  │  密  码：[•••••••••••]         │  │
│  │         [ 登 录 ]              │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

## 生产构建

### 执行构建

```bash
npm run build:local
```

构建完成后，静态资源输出到 `dist/` 目录。

### 验证构建结果

```bash
ls -la dist/
```

预期包含：
- `index.html` - 入口文件
- `assets/` - 静态资源目录
- 其他静态文件

## 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue | 3.x | 前端框架 |
| TypeScript | 5.x | 类型系统 |
| Vite | 5.x | 构建工具 |
| Element Plus | - | UI 组件库 |
| Pinia | - | 状态管理 |

## 常用命令

```bash
# 安装依赖
pnpm install

# 启动开发服务器
npm run dev

# 构建生产版本
npm run build:local

# 预览构建结果
npm run preview

# 代码检查
npm run lint

# 代码格式化
npm run format
```

## 常见问题

### pnpm install 失败

```bash
# 清除缓存
pnpm store prune

# 重新安装
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

### 构建内存不足

如构建卡住或内存溢出，建议：
- 使用 16GB+ 内存的机器
- 或增加 Node.js 内存限制：`NODE_OPTIONS="--max-old-space-size=4096" npm run build:local`

### 代理配置问题

在 `vite.config.ts` 中配置代理：

```typescript
server: {
  proxy: {
    '/admin-api': {
      target: 'http://192.168.206.129:48080',
      changeOrigin: true,
    }
  }
}
```

## 下一步

前端构建完成后，继续阅读 [05. 服务容器化](./05-containerization.md)。
