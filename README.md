# Leon-Acco.github.io

Acho 的个人博客，基于 Jekyll 构建，托管于 GitHub Pages。内容来自个人 Obsidian 知识库（acho_barin）与语雀知识库的系统整理。

- 在线访问：https://leon-acco.github.io
- 四大分类：AI（ai）/ 车联网（iot）/ 大数据（bigdata）/ 工程效率（tools）
- 功能：知识系列卡片、文章 TOC、系列导航、Mermaid 图表、KaTeX 公式、全站搜索（search.json）、归档、RSS

## 目录结构

```
_posts/          文章（YYYY-MM-DD-slug.md）
_data/series.yml 系列定义（id/name/icon/desc）
_layouts/        布局（default/home/post/category）
assets/img/      文章图片（按文章 slug 分目录）
```

## 写文章

在 `_posts/` 下新建 `YYYY-MM-DD-slug.md`：

```markdown
---
layout: post
title: "文章标题"
date: 2026-01-01
categories: ai
series: ai-learning        # 可选,对应 _data/series.yml 的 id
series_order: 1            # 可选,系列内序号
tags: [AI, Agent]
original: https://www.yuque.com/...
mermaid: true              # 可选,页面含 mermaid 图时开启
toc: true
---

正文（标准 Markdown，支持 callout div、mermaid、图片 /assets/img/<slug>/...）
```

## 从 Obsidian 迁移

转换管道支持：`![[embed]]` 嵌入（含 excalidraw 源文件实时渲染 SVG）、wikilinks 自动转站内链接、Obsidian callout 转 HTML 卡片、`==高亮==`、`%%注释%%`、语雀 CDN 图片自动下载本地化。敏感内容（工作项目细节、个人数据）不出库，发布清单在管道中显式维护。
