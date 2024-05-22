---
title: next主题边框圆角化
tags:
  - hexo
  - next
categories: hexo
abbrlink: 47864
date: 2024-02-06 10:03:38
---

# next主题边框完全圆角化

问题描述：next主题边框圆角化后作者所属的边框有一层阴影，圆角化不明显
<!--more-->
## 问题解决
  
打开文件 “/themes/next/source/css/_variables/Gemini.styl”，
在

    // Settings for some of the most global styles.
    // --------------------------------------------------
    // $body-bg-color           = #eee;
    $body-bg-color           = transparent;

    // Borders.
    // --------------------------------------------------
    $box-shadow-inner        = 0 2px 2px 0 rgba(0, 0, 0, .12), 0 3px 1px -2px rgba(0, 0, 0, .06), 0 1px 5px 0 rgba(0, 0, 0, .12);
    $box-shadow              = 0 2px 2px 0 rgba(0, 0, 0, .12), 0 3px 1px -2px rgba(0, 0,   0, .06), 0 1px 5px 0 rgba(0, 0, 0, .12), 0 -1px .5px 0 rgba(0, 0, 0, .09);

中将 `$body-bg-color` 赋值为透明 transparent