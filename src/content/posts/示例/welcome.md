---
title: "欢迎来到示例博客"
description: "这是一个开源的博客模板，本文是一篇示例文章，用于演示博客的布局、排版与各项功能。"
date: "2025-11-20"
cover: https://i.postimg.cc/KvTX6wFM/6f907a94a2afa830281dbfca13e22cb6.jpg
categories: 示例
sticky: 2
tags:
  - Astro
  - Markdown
  - 示例
---

> 本文是开源仓库自带的示例文章，不含真实个人信息。你可以把它删除，换成自己的内容。

这个博客基于 Astro 构建，内容全部使用 Markdown 管理。你只需要在 `src/content/posts/` 下新建 `.md` 文件，写入 frontmatter 和正文，重新构建后文章就会自动出现在首页、归档、分类、标签、搜索和 RSS 中。

## 文章能写什么

- 普通段落与 **加粗**、*斜体*、`行内代码`
- 多级标题、有序列表与无序列表
- 引用块、代码块、表格
- 图片(放在 `public/` 或外链均可)

## 一篇完整的 frontmatter

```yaml
---
title: "文章标题"
description: "列表页与搜索引擎看到的摘要"
date: "2025-11-20"
categories: 示例
tags:
  - Astro
  - 示例
---
```

正文从这里开始，支持完整的 Markdown 语法。

祝写作愉快。更多说明见仓库的 `README.md`。
