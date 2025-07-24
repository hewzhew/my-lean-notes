---
tags:
  - 网站搭建
  - GitHub Pages
  - MkDocs
---

# 使用 GitHub Pages 和 MkDocs 搭建笔记网站的经验

本文档记录了在使用 GitHub Pages 和 MkDocs 搭建个人网站过程中遇到的问题及其解决方案。

## 问题一：GitHub Actions 显示成功，但网站无法访问 (404 Not Found)

**现象:**
推送代码后，GitHub Actions 显示构建流程成功，但访问对应的 GitHub Pages 网址时出现 404 错误。

**原因分析:**
Actions 成功表示网站的静态文件已成功生成并推送到 `gh-pages` 分支。然而，GitHub Pages 服务未被配置为使用此分支作为网站来源。

**解决方案:**
需手动设置仓库的部署源。

1.  进入仓库的 **Settings -> Pages** 菜单。
2.  在 **Build and deployment** 部分，将 **Source** 设置为 **Deploy from a branch**。
3.  在 **Branch** 部分，选择 `gh-pages` 分支，文件夹保持 `/(root)`，然后点击 **Save**。

此设置完成后，GitHub Pages 方可发布 `gh-pages` 分支的内容。

## 问题二：“git clone”后出现“cloned an empty repository”警告

**现象:**
执行 `git clone` 命令后，终端显示 `warning: You appear to have cloned an empty repository`。

**原因分析:**
此警告为预期行为。在 GitHub 创建仓库时未选择初始化文件（如 `README.md`），导致仓库为空。Git 工具克隆时检测到此情况，故给出提示。

**解决方案:**
无需操作。可直接进入新创建的文件夹，开始添加项目文件（如 `mkdocs.yml` 和 `docs` 目录）。

## 问题三：“mkdocs.yml”文件的正确位置

**现象:**
执行 `mkdocs serve` 或部署时出现错误，提示找不到配置文件或 `docs` 目录。

**原因分析:**
`mkdocs.yml` 文件位置不正确。作为 MkDocs 项目的核心，其必须位于特定位置才能被识别。

**解决方案:**
`mkdocs.yml` 文件必须放置在项目的**根目录**下，与 `docs` 文件夹处于同一级别。

正确的目录结构示例：
my-lean-notes/
├── docs/
│   └── index.md
└── mkdocs.yml