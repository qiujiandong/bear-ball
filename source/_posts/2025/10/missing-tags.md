---
title: hexo本地预览缺少tags和categories
date: 2025-10-04 23:36:21
tags:
  - hexo
categories:
  - hexo
---

今天遇到一个问题, 用`hexo`生成的网页部署在Github显示时没有问题,
但是在本地预览 (`npx hexo server`) 时却缺少`tags`和`categories`.

比如用`hexo server`预览的是很, 没有`tags`和`categories`,
而且相应的`标签`和`分类`页面也显示没有内容. 显示效果如下:

![no-tags-and-categories](2025-10-05-00-00-40.png)

如果用`hexo server --static`, 预览的网页就没有这个问题.

![with-tags-and-categories](2025-10-05-00-01-28.png)
