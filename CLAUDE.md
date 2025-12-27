# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

**sync-your-cookie** 是一个浏览器扩展（支持 Chrome 和 Firefox），可以将 cookies 和 localStorage 同步到 Cloudflare KV 或 GitHub Gist，以便在不同设备间共享。该扩展以「Sync Your Cookie」的名称在 Chrome Web Store 和 Microsoft Edge Add-ons 上发布。

## 开发命令

```bash
# 开发（Chrome）
pnpm dev              # 启动所有开发服务器，支持热重载
pnpm dev:apps         # 仅启动应用程序（不包含 packages）
pnpm dev-server       # 启动 HMR 开发服务器

# 开发（Firefox）
pnpm dev:firefox      # 启动 Firefox 开发构建

# 构建
pnpm build            # Chrome 生产环境构建
pnpm build:firefox    # Firefox 生产环境构建
pnpm zip              # 构建并打包扩展用于分发

# 代码质量
pnpm test             # 运行测试
pnpm type-check       # TypeScript 类型检查
pnpm lint             # 运行 ESLint
pnpm lint:fix         # 修复 lint 问题
pnpm prettier         # 使用 Prettier 格式化代码
pnpm clean            # 清理构建产物
```

**环境要求**：Node.js >= 20.12.0，pnpm（v9.1.1）

## 架构设计

这是一个基于 **Turborepo** 的 monorepo，结构如下：

```
sync-your-cookie/
├── chrome-extension/      # 扩展核心（service worker、manifest）
├── pages/                 # UI 页面（popup、options、sidepanel、content scripts）
├── packages/
│   ├── @sync-your-cookie/shared    # 核心同步逻辑（GitHub API、Cloudflare API）
│   ├── @sync-your-cookie/storage   # 存储抽象层（cookies、域名配置、设置）
│   ├── @sync-your-cookie/ui        # 共享 React 组件
│   ├── @sync-your-cookie/protobuf  # Protocol Buffer 定义
│   ├── @sync-your-cookie/hmr       # 开发时热模块替换
│   └── @sync-your-cookie/zipper    # 扩展打包工具
└── dist/                 # 构建输出（所有包都构建到这里）
```

### 核心架构概念

**Service Worker（`chrome-extension/`）**：负责协调 cookie 同步的后台脚本，主要处理：
- 通过 `chrome.cookies` API 读取/写入 cookies
- 通过 content script 注入读取 localStorage
- 使用 Protocol Buffers 编码/解码数据
- 推送/拉取数据到 Cloudflare KV 或 GitHub Gist

**存储后端**：两种主要存储选项，按账户配置：
- **Cloudflare KV**：通过 Cloudflare API 的键值存储
- **GitHub Gist**：通过 Octokit 的 Gist 存储

**同步流程**：
1. **推送**：收集当前域名的 cookies/localStorage → protobuf 编码 → gzip 压缩 → 上传到存储后端
2. **拉取**：从存储后端下载 → gunzip 解压 → protobuf 解码 → 与现有 cookies/localStorage 合并
3. **自动合并/推送**：按域名规则控制自动同步行为

**多账户支持**：使用「存储密钥（Storage Key）」区分不同的同步账户。每个账户有独立的配置（后端类型、凭据、域名规则）。

**数据编码**：Cookie 数据使用 Protocol Buffers（`packages/shared/lib/protobuf/proto/`）编码，并用 pako（gzip）压缩以高效传输。

### UI 页面

- **popup/**：主扩展弹出窗口，用于快速同步操作
- **options/**：完整设置页面，用于账户配置和域名规则
- **sidepanel/**：Cookie 管理面板，用于查看/复制/管理同步的数据
- **content/**：注入到网页中的 content scripts，用于读取 localStorage

## 构建系统

- **Turborepo**：跨 monorepo 协调构建，管理依赖关系
- **Vite**：每个包的主要构建工具
- **TypeScript**：v5.2.2，启用严格类型检查
- **热模块替换**：自定义 HMR 服务器用于快速开发（`@sync-your-cookie/hmr`）

构建产物统一放在仓库根目录的 `dist/` 目录中。

## 环境变量

- `__DEV__`：设为 `true` 启用开发构建（启用 HMR、调试功能）
- `__FIREFOX__`：设为 `true` 启用 Firefox 构建（调整 manifest、API）

## 关键文件

- `chrome-extension/background/service-worker.ts`：主同步协调逻辑
- `packages/shared/lib/cloudflare/api.ts`：Cloudflare KV 集成
- `packages/shared/lib/github/api.ts`：GitHub Gist 集成
- `packages/shared/lib/cookie/withCloudflare.ts` 和 `withStorage.ts`：各后端的同步逻辑
- `packages/storage/`：扩展数据的存储层（设置、域名配置）
