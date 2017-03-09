---
layout: post
title: Maven使用随笔
---

{{ page.title }}
================

最近一段时间在给项目搞一个代码质量衡量report的功能, 这个功能大量的使用了maven提供的各种插件来实现. 有感于maven各种机制和插件的使用,在此做一个总结.
Mark: 本文基于maven 3.3.x版本

### Maven的生命周期机制(lifecycle)

要理解Maven的运行机制,必须熟悉Maven的生命周期机制.
