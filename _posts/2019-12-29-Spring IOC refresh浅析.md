---
layout:     post
title:      Java Spring IOC 源码浅析（refresh 方法）
subtitle:   日常记录
date:       2019-12-29
author:     fyypumpkin
header-img: img/spring-ioc-bg.jpg
catalog: true
tags:
    - Java
    - Spring
    - IOC
---

## 正文

#### 好久没写博客了，最近看了下 Spring IOC 相关的内容，之前一直知道个大概，但是没有仔细的了解过相关的内容，最近有机会，了解了一下相关的内容，主要研究了下 AbstractApplicationContex 里面的 refresh 相关的代码，这里包含了整个 bean 的创建过程

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.