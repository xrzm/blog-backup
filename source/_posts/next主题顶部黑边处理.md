---
title: next主题顶部黑边处理
tags:
  - hexo
  - next
categories: hexo
abbrlink: 937
date: 2024-02-06 10:03:09
---

# next主题顶部黑边处理

问题描述：next主题优化后仍然有一条黑边在顶部出现
<!--more-->
## 问题解决

### 方法一：  
打开文件 “themes\next\layout\_layout.swig” ,在 

    <body itemscope itemtype="http://schema.org/WebPage">
      <div class="container{%- if theme.motion.enable %} use-motion{%- endif %}">
        <div class="headband"></div>

中删除 “`<div class="headband"></div>`”

### 方法二：  
打开文件 “themes\next\source\css\_variables\base.styl” ，
在

    // Headband
    // --------------------------------------------------
    $headband-height                = 3px;
    $headband-bg                    = $black-deep;

中找到“`$headband-height`”，把3px改成0px；