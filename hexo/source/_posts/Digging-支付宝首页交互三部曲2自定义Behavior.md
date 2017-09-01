---
title: '[Digging]支付宝首页交互三部曲2自定义Behavior'
date: 2017-09-01 20:52:13
tags: NestedScrolling
categories: Android
---

![digging2](http://otiwf3ulm.bkt.clouddn.com/7fa7fd062bad4bd7d01c3d8bcd65672d.png)

### 前言

自己动手实现支付宝首页效果 , 用三篇文章记录并分享给大家。

* CoordinatorLayout和Behavior
* 自定义CoordiantorLayout和Behavior
* 支付宝首页效果实现

> 文中 : Col 表示 CoordinatorLayout , ABL 表示AppBarLayout , CTL 表示 CollapsingToolbarLayout , SRL 表示 SwipeRefreshLaout , RV 表示 RecyclerVIew。

第二篇文章主要用经典的CoordinatorLayout、AppBarLayout、RecyclerView的联动场景（CAR场景）来分析一下自定义Behavior需要关注的内容 , 以及如何自定义一个Behavior。同时 , 支付宝首页效果和AppBarLayout的效果有相似之处 , 分析CAR场景 , 也有意于后文实现支付宝首页效果。

>这篇文章适合同时阅读源码 , 如果已经读过源码 , 可以直接诶跳到最后总结。
