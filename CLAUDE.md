# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **TFRobot Marketplace** documentation repository - a documentation-only repo for the TFRobot Marketplace platform. The deliverable is the documentation itself, built using MkDocs and deployed to both GitHub Pages and doc.turingfocus.cn.

**Current Version**: 0.1.0

## Repository Structure

```
tfrobot-marketplace/
├── .github/
│   └── workflows/
│       └── deploy-pages.yml  # GitHub Pages 自动部署
├── docs/                      # 文档源文件 (MkDocs docs_dir)
│   ├── index.md              # 首页
│   └── appendix/
│       └── faq.md            # 常见问题
├── scripts/                   # 部署脚本
│   └── docs/
│       ├── config.py         # 配置管理
│       ├── tasks.py          # Invoke 任务
│       └── version_utils.py  # 版本工具
├── mkdocs.yml                # MkDocs 配置
├── pyproject.toml            # 项目配置 (版本号单一来源)
├── tasks.py                  # Invoke 入口
└── README.md                 # 开发指南
```

## Working with This Repository

### Documentation Build & Deploy

This project uses [uv](https://github.com/astral-sh/uv) for dependency management.

```bash
# Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create venv and install dependencies
uv venv && source .venv/bin/activate
uv pip install -e ".[docs]"

# Local preview (hot reload)
inv docs.serve

# Build versioned docs
inv docs.build

# Deploy to GitHub Pages
inv docs.deploy-github

# Deploy to company server
inv docs.deploy

# Deploy to both platforms
inv docs.deploy-all
```

### Version Management
- Version number is defined in `pyproject.toml` (single source of truth)
- Use `bump-my-version` for version updates
- Multi-version docs managed by `mike`

### Deployment Targets
- **GitHub Pages**: `https://a2c-smcp.github.io/tfrobot-marketplace/`
- **Company server**: `https://doc.turingfocus.cn/tfrobot-marketplace/`

The deploy scripts under `scripts/docs/` (config.py, tasks.py, version_utils.py) mirror the layout of the
`a2c-smcp-protocol` repo and were initialized from it. Keep both repos in sync when fixing the build/deploy pipeline.
