# TFRobot Marketplace

TFRobot Marketplace 是 TFRobot 生态的市场平台。本仓库存放与之相关的文档与规范。

## 在线文档

| 平台 | 地址 |
|------|------|
| GitHub Pages | [https://a2c-smcp.github.io/tfrobot-marketplace/](https://a2c-smcp.github.io/tfrobot-marketplace/) |
| 公司服务器 | [https://doc.turingfocus.cn/tfrobot-marketplace/](https://doc.turingfocus.cn/tfrobot-marketplace/) |

## 版本

当前版本：**0.3.2**

---

## 开发指南

### 环境准备

本项目使用 [uv](https://github.com/astral-sh/uv) 管理依赖。

```bash
# 安装 uv（如果尚未安装）
curl -LsSf https://astral.sh/uv/install.sh | sh
# 或 macOS: brew install uv

# 创建虚拟环境并安装依赖
uv venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

# 安装文档依赖
uv pip install -e ".[docs]"
```

> **备选方案**：如果不使用 uv，也可以用标准 pip：
> ```bash
> python -m venv .venv && source .venv/bin/activate
> pip install -e ".[docs]"
> ```

### 本地预览

```bash
# 启动开发服务器（实时热重载）
inv docs.serve

# 访问 http://127.0.0.1:8000
```

### 构建文档

```bash
# 构建当前版本（版本号从 pyproject.toml 自动读取）
inv docs.build

# 构建指定版本
inv docs.build --version 0.3.2

# 预览多版本文档
inv docs.serve-versioned
```

### 部署文档

```bash
# 部署到公司服务器（构建 + 推送 + 服务器 Git pull）
inv docs.deploy

# 部署到 GitHub Pages
inv docs.deploy-github

# 同时部署到两个平台
inv docs.deploy-all

# 仅构建，不推送
inv docs.deploy --push=false
```

> **自动部署**：推送到 `main` 分支的文档变更会自动触发 GitHub Actions 部署到 GitHub Pages。

### 可用命令

```bash
inv --list                  # 查看所有可用任务
inv docs.serve              # 本地预览（实时热重载）
inv docs.serve-versioned    # 多版本预览
inv docs.build              # 构建文档
inv docs.deploy             # 部署到公司服务器
inv docs.deploy-github      # 部署到 GitHub Pages
inv docs.deploy-all         # 同时部署到两个平台
inv docs.clean              # 清理构建产物
inv docs.server-setup       # 查看服务器初始化步骤
```

---

## 部署配置

### GitHub Pages（自动）

推送到 `main` 分支的文档变更会自动触发 GitHub Actions 部署：

- **触发条件**：`docs/**`、`mkdocs.yml` 或 workflow 文件变更
- **手动触发**：在 GitHub Actions 页面点击 "Run workflow"
- **自定义版本**：手动触发时可指定版本号和别名

仓库需要启用 GitHub Pages：
1. 进入 Settings → Pages
2. Source 选择 "Deploy from a branch"
3. Branch 选择 `gh-pages`，路径选择 `/ (root)`

### 公司服务器（环境变量）

部署脚本通过环境变量配置，建议创建 `.env` 文件（已在 .gitignore 中排除）：

```bash
# .env 文件示例

# ========== 必需配置 ==========

# 文档服务器地址
DOCS_SERVER_HOST=<YOUR_SERVER_IP>

# SSH 认证（二选一）
DOCS_SERVER_PASSWORD=<YOUR_PASSWORD>
# 或使用密钥文件
# DOCS_SERVER_KEY_FILE=~/.ssh/id_rsa

# ========== 可选配置 ==========

# SSH 端口（默认 22）
DOCS_SERVER_PORT=22

# SSH 用户名（默认 root）
DOCS_SERVER_USER=root

# 部署路径（默认 /var/www/doc.turingfocus.cn/tfrobot-marketplace）
DOCS_DEPLOY_PATH=/var/www/doc.turingfocus.cn/tfrobot-marketplace

# 企业微信通知（可选）
# WECOM_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
```

### 加载环境变量

```bash
# 方式1: 使用 dotenv
pip install python-dotenv
# 在脚本中会自动加载 .env 文件

# 方式2: 手动 export
source .env

# 方式3: 临时设置
DOCS_SERVER_HOST=xxx inv docs.deploy
```

### 首次部署

首次部署前需要在服务器上初始化环境：

```bash
# 查看初始化步骤
inv docs.server-setup
```

服务器端操作：

```bash
# 1. SSH 登录服务器
ssh root@<YOUR_SERVER_IP>

# 2. 创建文档目录
cd /var/www/doc.turingfocus.cn/
git clone -b gh-pages https://github.com/A2C-SMCP/tfrobot-marketplace.git tfrobot-marketplace
chown -R nginx:nginx /var/www/doc.turingfocus.cn/tfrobot-marketplace
chmod -R 755 /var/www/doc.turingfocus.cn/tfrobot-marketplace

# 3. 更新 Nginx 配置
# 在 /etc/nginx/conf.d/doc.turingfocus.cn.conf 添加:
#
# location /tfrobot-marketplace/ {
#     alias /var/www/doc.turingfocus.cn/tfrobot-marketplace/;
#     try_files $uri $uri/ /tfrobot-marketplace/latest/index.html;
#     index index.html;
# }

# 4. 测试并重载 Nginx
nginx -t && systemctl reload nginx
```

---

## 版本管理

本项目使用 [mike](https://github.com/jimporter/mike) 进行多版本文档管理：

- 每个版本独立部署到 `gh-pages` 分支
- `latest` 别名指向最新版本
- 文档内置版本切换器

### 版本发布流程

```bash
# 1. 更新 pyproject.toml 中的版本号
# 2. 构建并部署
inv docs.deploy

# 或使用 bump-my-version 自动更新版本
pip install bump-my-version
bump-my-version bump patch  # 0.3.2 -> 0.3.3
bump-my-version bump minor  # 0.3.2 -> 0.4.0
bump-my-version bump major  # 0.3.2 -> 1.0.0
```

---

## 项目结构

```
tfrobot-marketplace/
├── docs/                      # 文档源文件
│   ├── index.md              # 首页
│   └── appendix/
│       └── faq.md            # 常见问题
├── scripts/                   # 部署脚本
│   └── docs/
│       ├── __init__.py
│       ├── config.py         # 配置管理
│       ├── tasks.py          # Invoke 任务
│       └── version_utils.py  # 版本工具
├── mkdocs.yml                # MkDocs 配置
├── pyproject.toml            # 项目配置
├── tasks.py                  # Invoke 入口
├── CLAUDE.md                 # Claude Code 指南
└── README.md                 # 本文件
```

---

## License

MIT
