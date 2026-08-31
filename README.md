# AI 博客 (AI Weblog)

基于 Hugo + GitHub Pages 构建的 AI 技术博客。

## 简介

本博客使用静态站点生成器 Hugo 构建，部署在 GitHub Pages 上。主要分享 AI 技术相关内容，包括大语言模型、机器学习、提示词工程等方面的文章。

## 技术栈

- **Hugo**: 静态站点生成器
- **GitHub Pages**: 静态网站托管
- **Markdown**: 文章内容格式

## 文章格式

所有文章使用 Markdown 格式，frontmatter 包含以下字段：

```yaml
---
title: "文章标题"
date: 2026-08-31
tags: [标签1, 标签2]
author: 作者名
---
```

## 发布流程

1. 新文章放在 `content/posts/` 目录下
2. 提交 Pull Request
3. 合并到 `main` 分支
4. GitHub Actions 自动部署

## 本地开发

```bash
# 安装 Hugo（需要 extended 版本）
# https://gohugo.io/getting-started/installing/

# 启动本地服务器
hugo server -D

# 构建站点
hugo
```

## 许可证

MIT License - 详见 LICENSE 文件
