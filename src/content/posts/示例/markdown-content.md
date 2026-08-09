---
title: "用 Markdown 管理博客内容"
description: "示例文章：介绍本博客的内容目录结构、frontmatter 字段与写作流程。"
date: "2025-12-05"
categories: 示例
tags:
  - Markdown
  - 内容管理
---

博客的内容分为两类：文章(`posts`)与随笔(`notes`)。两者都是 Markdown 文件，区别只在于字段：

## 文章(posts)

放在 `src/content/posts/`，按目录归类：

```
src/content/posts/
├── 示例/
├── AI/
└── 技术/
```

目录名会成为文章链接的一部分，例如本文的链接就是 `/posts/示例/markdown-content/`。

## 随笔(notes)

放在 `src/content/notes/`，以日期命名，字段更简：

```yaml
---
date: 2026-05-28
mood: 夜
---
```

随笔不需要标题和分类，适合记录碎片化的想法。

## 常用字段

| 字段 | 说明 | 必填 |
| --- | --- | --- |
| `title` | 文章标题 | 是 |
| `description` | 摘要，用于列表页和 SEO | 推荐 |
| `date` | 发布日期，决定排序 | 推荐 |
| `cover` | 封面图 URL | 否 |
| `categories` | 分类 | 否 |
| `tags` | 标签 | 否 |
| `sticky` | 置顶权重，数字越大越靠前 | 否 |

所有字段的完整定义见 `src/content.config.ts`。
