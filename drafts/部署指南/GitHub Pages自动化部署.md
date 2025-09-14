# GitHub Pages 自动化部署

## 概述

本文档详细说明如何配置 GitHub Actions 自动化部署 《LLM武侠演义》到 GitHub Pages，实现代码推送后自动构建和发布。

## GitHub Pages 设置

### 1. 启用 GitHub Pages

在 GitHub 仓库中：
1. 进入 **Settings** 选项卡
2. 滚动到 **Pages** 部分
3. 在 **Source** 下选择 **GitHub Actions**
4. 保存设置

### 2. 仓库权限配置

确保 GitHub Actions 有必要的权限：
1. 进入 **Settings** > **Actions** > **General**
2. 在 **Workflow permissions** 中选择：
   - **Read and write permissions**
   - **Allow GitHub Actions to create and approve pull requests**
3. 保存设置

## GitHub Actions 工作流配置

### 1. 创建工作流文件

在项目根目录创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to GitHub Pages

# 触发条件
on:
  # 当推送到 main 分支时触发
  push:
    branches: [ main ]
  
  # 允许手动触发
  workflow_dispatch:

# 设置权限
permissions:
  contents: read
  pages: write
  id-token: write

# 确保同时只有一个部署在运行
concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  # 构建作业
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 获取完整的提交历史

      - name: Setup Rust
        uses: dtolnay/rust-toolchain@stable

      - name: Cache Rust dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          restore-keys: |
            ${{ runner.os }}-cargo-

      - name: Install mdBook
        run: |
          cargo install --version 0.4.36 mdbook
          cargo install mdbook-mermaid

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v4

      - name: Build book
        run: |
          mdbook build
          
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./book

  # 部署作业
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 2. 高级配置选项

#### 多环境部署
```yaml
name: Deploy Multi-Environment

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [production, staging]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Rust
        uses: dtolnay/rust-toolchain@stable
      
      - name: Install mdBook
        run: cargo install mdbook
      
      - name: Build for ${{ matrix.environment }}
        run: |
          if [ "${{ matrix.environment }}" == "production" ]; then
            mdbook build --dest-dir book-prod
          else
            mdbook build --dest-dir book-staging
          fi
```

#### 添加构建优化
```yaml
      - name: Install mdBook with plugins
        run: |
          cargo install mdbook --version 0.4.36
          cargo install mdbook-mermaid
          cargo install mdbook-katex
          cargo install mdbook-linkcheck
          
      - name: Verify links
        run: mdbook-linkcheck
        
      - name: Build with optimizations
        run: |
          # 设置环境变量优化构建
          export MDBOOK_BUILD__CREATE_MISSING=false
          export MDBOOK_OUTPUT__HTML__PRINT_ENABLE=false
          mdbook build
```

### 3. 自定义域名配置

如果使用自定义域名：

#### 创建 CNAME 文件
在 `src/` 目录下创建 `CNAME` 文件：
```
your-domain.com
```

#### 修改工作流
```yaml
      - name: Add CNAME
        run: echo "your-domain.com" > book/CNAME
```

#### DNS 配置
在域名服务商处配置 DNS：
```
Type: CNAME
Name: www (or @)
Value: your-username.github.io
```

## 部署监控和通知

### 1. 部署状态徽章

在 README.md 中添加状态徽章：
```markdown
[![Deploy Status](https://github.com/your-username/llm_wuxia/workflows/Deploy%20to%20GitHub%20Pages/badge.svg)](https://github.com/your-username/llm_wuxia/actions)
```

### 2. 钉钉通知集成

```yaml
      - name: Notify DingTalk
        if: always()
        uses: zcong1993/actions-ding@master
        with:
          dingToken: ${{ secrets.DING_TOKEN }}
          body: |
            {
              "msgtype": "markdown",
              "markdown": {
                "title": "部署通知",
                "text": "## 《LLM武侠演义》部署${{ job.status }}\n\n- **仓库**: ${{ github.repository }}\n- **分支**: ${{ github.ref }}\n- **提交**: ${{ github.sha }}\n- **状态**: ${{ job.status }}\n- **时间**: $(date)\n\n[查看详情](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})"
              }
            }
```

### 3. 邮件通知

```yaml
      - name: Send Email Notification
        if: failure()
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 465
          username: ${{ secrets.EMAIL_USERNAME }}
          password: ${{ secrets.EMAIL_PASSWORD }}
          subject: "部署失败通知 - LLM武侠演义"
          body: |
            部署失败详情：
            - 仓库: ${{ github.repository }}
            - 分支: ${{ github.ref }}
            - 提交: ${{ github.sha }}
            - 查看日志: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          to: admin@example.com
          from: noreply@github.com
```

## 性能优化

### 1. 缓存策略
```yaml
      - name: Cache mdBook installation
        uses: actions/cache@v3
        with:
          path: ~/.cargo/bin/mdbook
          key: ${{ runner.os }}-mdbook-${{ hashFiles('book.toml') }}
          
      - name: Cache build output
        uses: actions/cache@v3
        with:
          path: book/
          key: ${{ runner.os }}-book-${{ hashFiles('src/**/*.md') }}
```

### 2. 并行构建
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        rust-version: [stable]
        node-version: [18]
    
    steps:
      - name: Setup Rust ${{ matrix.rust-version }}
        uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust-version }}
```

### 3. 增量构建
```yaml
      - name: Check for changes
        id: changes
        run: |
          if git diff --quiet HEAD~1 HEAD -- src/; then
            echo "changed=false" >> $GITHUB_OUTPUT
          else
            echo "changed=true" >> $GITHUB_OUTPUT
          fi
          
      - name: Build only if changed
        if: steps.changes.outputs.changed == 'true'
        run: mdbook build
```

## 故障排除

### 常见问题

**1. 部署失败 - 权限错误**
```yaml
# 确保工作流有正确的权限
permissions:
  contents: read
  pages: write
  id-token: write
```

**2. mdBook 安装失败**
```yaml
# 指定明确的版本
- name: Install mdBook
  run: |
    cargo install mdbook --version 0.4.36 --locked
```

**3. 构建超时**
```yaml
# 增加超时时间
jobs:
  build:
    timeout-minutes: 30
```

**4. 内存不足**
```yaml
# 使用更大的运行器
jobs:
  build:
    runs-on: ubuntu-latest-4-cores
```

### 调试技巧

**1. 启用调试输出**
```yaml
      - name: Debug build
        run: |
          export RUST_LOG=debug
          mdbook build --verbose
```

**2. 检查文件结构**
```yaml
      - name: List build output
        run: |
          ls -la book/
          find book/ -type f -name "*.html" | head -10
```

**3. 验证链接**
```yaml
      - name: Validate deployment
        run: |
          curl -f ${{ steps.deployment.outputs.page_url }} || exit 1
```

## 最佳实践

### 1. 版本管理
- 使用语义化版本标签
- 为重要发布创建 Release
- 保持构建脚本的版本兼容性

### 2. 安全考虑
- 使用 GitHub Secrets 存储敏感信息
- 定期更新 Actions 版本
- 限制工作流权限

### 3. 性能监控
- 监控构建时间
- 跟踪部署频率
- 优化构建流程

### 4. 文档维护
- 记录配置变更
- 更新部署流程文档
- 维护故障排除指南

通过以上配置，《LLM武侠演义》项目将实现完全自动化的部署流程，确保每次代码更新都能快速、可靠地发布到 GitHub Pages。