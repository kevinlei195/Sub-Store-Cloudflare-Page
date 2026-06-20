# Sub-Store Workers 开发指南

## 项目概述

Sub-Store Workers 是将 [Sub-Store](https://github.com/sub-store-org/Sub-Store) 后端部署到 Cloudflare Workers/Pages 的移植版本。

**核心特点**：
- 零服务器成本，运行在 Cloudflare 边缘网络
- KV 持久化存储
- 复用原始后端全部业务逻辑
- 预编译 PEG 文法，避免运行时 eval()

## 目录结构

```
sub-store-workers/
├── src/                          # Workers 适配层
│   ├── index.js                  # Workers 入口
│   ├── core/app.js              # OpenAPI 封装
│   ├── vendor/
│   │   ├── express.js           # 路由处理
│   │   ├── open-api.js          # HTTP/KV 适配层
│   │   └── open-api.js          # Workers fetch 替换
│   ├── restful/                 # REST API 路由
│   │   ├── token.js
│   │   └── miscs.js
│   └── utils/
│       └── env.js
├── Sub-Store/backend/src/        # 原始后端源码（上游）
├── dist/                         # 构建输出
├── pages-dist/                   # Pages 部署目录
├── esbuild.js                    # 构建脚本
├── wrangler.toml                # Workers 配置
└── package.json
```

## 架构说明

### Workers 适配层职责

| 文件 | 作用 |
|------|------|
| `src/vendor/open-api.js` | KV 替换 fs，fetch 替换 undici |
| `src/vendor/express.js` | Workers fetch handler 替换 Node express |
| `src/core/app.js` | 导入 Workers 版 OpenAPI |
| `src/index.js` | Workers 入口，路径前缀处理 |

### esbuild 插件

| 插件 | 作用 |
|------|------|
| `aliasPlugin` | 解析 `@/` 导入，优先 Workers 覆盖 |
| `evalRewritePlugin` | 将 eval() 调用替换为静态表达式 |
| `peggyPrecompilePlugin` | 构建时编译 PEG 文法 |
| `nodeStubPlugin` | 存根 fs/crypto 等不可用模块 |

## 本地开发

### 1. 安装依赖

```bash
npm install
```

### 2. 启动开发服务器

```bash
npm run dev
```

访问 `http://localhost:8787`。

### 3. 配置环境变量

编辑 `wrangler.toml`：

```toml
[vars]
SUB_STORE_FRONTEND_BACKEND_PATH = "/你的密码"
```

### 4. 导入生产 KV 数据（可选）

用于本地测试已有订阅：

```bash
# 导出
npx wrangler kv key get "sub-store" --namespace-id <KV_ID> --remote > sub-store-value.json

# 导入到本地
npx wrangler kv key put "sub-store" --path sub-store-value.json --namespace-id <KV_ID> --local --persist-to .wrangler/kvstate
```

### 5. 测试订阅

```bash
# 测试订阅
curl "http://localhost:8787/kevinlei/download/111"

# 带格式参数
curl "http://localhost:8787/kevinlei/download/111?target=ClashMeta&prettyYaml=true"
```

## 构建

```bash
npm run build
```

输出：`dist/worker.js`

## 部署

### Workers 部署（含 Cron）

```bash
npm run deploy
```

### Pages 部署（国内可直连）

```bash
npm run deploy:pages
```

### 部署后验证

```bash
curl "https://你的域名/kevinlei/api/utils/worker-status"
```

应返回：
```json
{"kv":{"bound":true},"auth":{"backendPathConfigured":true}}
```

## 常见问题排查

### 问题：订阅下载返回"不含有效节点"

**可能原因 1**：`yaml` 模块被错误 stub

检查 `esbuild.js` 的 `nodeStubPlugin`：
```javascript
// 不应该包含 'yaml'
const stubs = [
    'dotenv',
    // ... 其他 stubs
    // 'yaml',  // 不应该在这里
];
```

**修复**：将 `yaml` 从 stub 列表中移除，并确保安装了 `yaml` 包：
```bash
npm install yaml
```

**可能原因 2**：本地 KV 数据未导入

确保已将生产数据导入本地：
```bash
npx wrangler kv key list --namespace-id <ID> --local
```

### 问题：路径前缀不生效

检查 `wrangler.toml`：
```toml
[vars]
SUB_STORE_FRONTEND_BACKEND_PATH = "/kevinlei"
```

### 问题：wrangler dev 自动退出

使用后台启动：
```bash
nohup npx wrangler dev dist/worker.js --port 8790 --local > wrangler.log 2>&1 &
```

## Git 工作流

### 提交规范

```bash
git add <files>
git commit -m "fix: description"
git push origin main
```

### 本次修复提交记录

```
fix: remove yaml from stub list and install yaml package for proper YAML parsing
```

修改的文件：
- `esbuild.js` - 移除 yaml stub
- `package.json` - 添加 yaml 依赖
- `package-lock.json` - 依赖锁定

### 不应提交的文件

```
DEPLOY.md
KV_Namespace_ID
sub-store-value.json
wrangler.log
.wrangler/
```

建议添加到 `.gitignore`。

## Cloudflare KV 操作

### 查看 KV 命名空间

```bash
npx wrangler kv namespace list
```

### 列出所有 keys

```bash
npx wrangler kv key list --namespace-id <ID>
```

### 获取 key 值

```bash
# 远程
npx wrangler kv key get <key> --namespace-id <ID> --remote

# 本地
npx wrangler kv key get <key> --namespace-id <ID> --local --persist-to .wrangler/kvstate
```

### 写入 key

```bash
npx wrangler kv key put <key> --path <file> --namespace-id <ID> --local --persist-to .wrangler/kvstate
```

## 环境变量

| 变量 | 说明 | 建议 |
|------|------|------|
| `SUB_STORE_FRONTEND_BACKEND_PATH` | API 路径前缀密码 | 使用 Worker Secret |
| `SUB_STORE_PUSH_SERVICE` | HTTP 推送通知 | 可选 |

## 已知限制

| 功能 | 状态 |
|------|------|
| 脚本操作（Script Operator） | 不支持 |
| jsrsasign TLS 指纹 | 不支持 |
| 前端静态文件托管 | 不支持 |
| 自定义 HTTP/SOCKS5 代理 | 不支持 |

## 相关链接

- [Sub-Store 原始仓库](https://github.com/sub-store-org/Sub-Store)
- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
- [Wrangler 文档](https://developers.cloudflare.com/workers/wrangler/)
