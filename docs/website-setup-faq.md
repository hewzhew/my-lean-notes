---
tags:
  - 网站搭建
  - GitHub Pages
  - MkDocs
---

# 使用 GitHub Pages 和 MkDocs 搭建笔记网站的经验

本文档记录了在使用 GitHub Pages 和 MkDocs 搭建个人网站过程中遇到的问题及其解决方案。

### **MkDocs + GitHub Pages 网站搭建流程**

1.  在 GitHub 创建一个新的公开 (Public)仓库，可以不勾选初始化文件（如 `README`）。
2.  在本地电脑上，使用 `git clone` 命令克隆这个仓库，然后 `cd` 进入项目文件夹。
3.  运行 `pip install mkdocs mkdocs-material` 命令，安装建站工具和主题。
4.  在项目根目录，手动创建三个核心部分：配置文件 `mkdocs.yml`，用于自动部署的工作流文件 `.github/workflows/ci.yml`，以及存放所有笔记的 `docs` 文件夹（需在 `docs` 内再创建一个首页 `index.md`）。
5.  完成文件创建后，使用 `git add .`、`git commit -m "Initial setup"` 和 `git push` 将网站的基础框架推送到 GitHub。
6.  在 GitHub 仓库的 **`Settings > Pages`** 页面，将部署源 (Source) 设置为 **`Deploy from a branch`**，然后从下拉菜单中选择 **`gh-pages`** 分支并点击保存。
7.  等待 `Actions` 自动运行成功后，即可在 `Settings > Pages` 页面顶部找到并访问网站公开地址。

完成以上步骤，网站即搭建完毕并已配置好自动化部署。之后的所有更新，都只需在本地修改 `docs` 文件夹内的文件，然后重复第五步的 `add`, `commit`, `push` 即可。

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