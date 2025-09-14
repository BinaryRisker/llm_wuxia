# 部署指南

本目录包含《LLM武侠演义》项目的部署相关文档，为开发者和贡献者提供完整的部署解决方案。

## 目录结构

- `本地开发环境搭建.md` - 本地开发环境的安装和配置指南
- `GitHub Pages自动化部署.md` - GitHub Pages 的自动化部署配置

## 部署架构

### 开发环境
- **本地开发**：使用 mdBook 进行本地开发和预览
- **版本控制**：Git 管理源代码和协作
- **编辑工具**：VS Code 或其他 Markdown 编辑器

### 生产环境
- **GitHub Pages**：免费的静态网站托管
- **自动化部署**：GitHub Actions 自动构建和部署
- **CDN 加速**：GitHub 提供的全球 CDN 分发

## 部署流程

1. **本地开发** → 编写和测试内容
2. **提交代码** → Git 推送到 GitHub
3. **自动构建** → GitHub Actions 自动构建
4. **自动部署** → 发布到 GitHub Pages
5. **访问网站** → 通过域名访问最新版本

## 环境要求

- **操作系统**：Windows、macOS、Linux
- **Git**：版本控制工具
- **Rust**：用于安装 mdBook
- **浏览器**：现代浏览器（Chrome、Firefox、Safari、Edge）