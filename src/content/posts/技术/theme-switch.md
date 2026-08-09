---
title: "浅色与深色主题切换的实践"
description: "示例文章：聊聊深色模式切换的实现思路，从 CSS 变量到过渡动画。"
date: "2026-06-12"
categories: 技术
tags:
  - CSS
  - 主题
---

深色模式如今几乎是博客的标配。这篇示例文章记录几种常见的实现思路。

## 用 CSS 变量管理颜色

把颜色抽象成一组语义变量，主题只是变量的不同取值：

```css
:root {
  --bg: #f8f7f4;
  --text: #1b1c1e;
}

html[data-theme="dark"] {
  --bg: #0e1116;
  --text: #eef1f6;
}
```

切换主题只需要改 `html` 上的一个 `data-theme` 属性，所有颜色自动跟随。

## 记住用户的选择

用 `localStorage` 保存偏好，首次访问时跟随系统设置：

```js
const theme = localStorage.getItem('theme') ?? (matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
```

## 过渡动画的坑

主题切换动画最容易出问题的地方在于：切换瞬间页面上大量元素的颜色过渡会同时触发，占满主线程，导致动画掉帧。实际项目中通常需要：

- 让动画值保持静态，避免在过渡伪元素里解析动态变量
- 动画期间冻结页面其他元素的过渡，保证主线程只服务一个动画

（本文为开源仓库示例文章。）
