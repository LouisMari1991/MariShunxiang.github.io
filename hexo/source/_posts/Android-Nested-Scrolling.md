---
title: Android Nested Scrolling
date: 2017-08-27 19:34:18
tags: NestedScrolling
categories: Android
---

![NestedScrollHead](http://otiwf3ulm.bkt.clouddn.com/1301a18845a986101aa7bbbbca4ff8c0.png)

Android常规的Touch事件机制是自顶向下 , 由外向内的, 一旦确定了时间消费者View , 随后的时间都将传递到该View。 因为是自顶向下 , 父控件可以随时拦截事件 , 下拉刷新、拖拽排序、折叠等交互效果都可以通过这套机制完成。Touch事件传递机制是Android开发必须掌握的基本内容。但是这套机制存在一个缺陷: **子View 无法通知父View处理事件** NestedScrolling就是为了这个场景设计的。

### NestedScrollingChild和NestedScrollingParent

NestedScrolling是指存在嵌套滑动的场景 , 常见用于下拉刷新、展开/收起标题栏等。Support包中的`CoordinatorLayout`和`ScrollRefreshLayout`就是基于NestedScrolling机制实现的。

`NestedScrollingChild`和`NestedScrollParent`分别定义了嵌套子View和嵌套父View需要实现的接口,方法列表分别如下 , 可以先略过 , 后面会把这些方法串起来。另外这些方法基本都是通过`NestedScrollChildHelper`和`NestedScrollParentHelper`来实现的 , 一般并不需要手动编写多少逻辑。

```
// NestedScrollingChild
void setNestedScrollingEnabled(boolean enabled);
boolean isNestedScrollingEnabled();
boolean startNestedScroll(int axes);
void stopNestedScroll();
boolean hasNestedScrollingParent();
boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);
boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);
boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed);
boolean dispatchNestedPreFling(float velocityX, float velocityY);
```

```
// NestedScrollingParent
boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);
void onStopNestedScroll(View target);
void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed);
void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);
boolean onNestedPreFling(View target, float velocityX, float velocityY);
int getNestedScrollAxes();
```

通过方法名可以看出,`NestedScrollingChild`的方法均为主动方法 , 而`NestedScrollingParent`的方法基本都是回调方法。这也是`NestedScrolling`机制的一个体现 , 子View作为NestedScrolling事件传递的主动方。

NestedScrolling机制生效的前提条件是子View作为Touch事件的消费者 , 在消费过程中向父View发送NestedScrolling事件（**注意这里不是Touch事件 , 而是NestedScrolling事件**）。

### NestedScrolling事件传递

NestedScrolling机制中 , NestedScrolling事件使用`dx,dy`表示 , 分别表示子View Touch 事件处理方法中判定的x和y方向上的滚动偏移量。

NestedScrolling事件的传递：
1. 由子View产生NestedScrolling事件 ;
2. 发送给父View进行处理 , 父View处理之后 , 返回消费的偏移量 ;
3. 子View根据View消费的偏移量计算NestedScrolling事件剩余偏移量 ;
4. 根据剩余偏移量判断是否能处理滚动事件 ; 如果处理事件 , 同时将自身滚动情况通知父View ;
5. 处理完成 , 事件传递完成。

> 1. 这里只说明了一层嵌套的情况 , 事实上NestedScrolling很可能出现在多重嵌套的场景。对于多重嵌套 , 步骤 2、3、4将事件自底向上进行传递 , 步骤2中消费的偏移量将记录所有嵌套父View消费偏移量的总和。这里不再重复。
> 2. Fling事件的传递和Scroll类似 , 也不再赘述。

### 方法调用流程

我们可以上面的方法根据NestedScrolling事件传递的不同阶段进行分组（Fling跟随Scrolling发生）。

**初始阶段：** 确认开启NestedScrolling , 关联父View和子View。

```
// NestedScrollingChild
void setNestedScrollingEnabled(boolean enabled);
boolean startNestedScroll(int axes);
```

```
// NestedScrollingParent
int getNestedScrollAxes();
boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);
```

**预滚动阶段：** 子View将事件分发到父View

```
// NestedScrollingChild
boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);
```

```
// NestedScrollingParent
void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
```

**滚动阶段：** 子View处理滚动事件

```
// NestedScrollingChild
boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);
```

```
// NestedScrollingParent
void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed);
```

**结束阶段：** 结束

```
// NestedScrollingChild
void stopNestedScroll();
```

```
// NestedScrollingParent
void onStopNestedScroll(View target);
```

下面是一次嵌套滚动（三级嵌套）从开始到结束的方法调用时序图：

![NestedScrollingThree](http://otiwf3ulm.bkt.clouddn.com/38b0dbd0d45c8dc0ba9c09b17df82467.png)

> 金色是`NestedScrollingChild`的方法 , 为View主动调用。

> 紫色是`NestedScrollingParent`回调的方法 , 由View的相关方法调用。

> 橙色是**滚动事件被消费的时机**

### 划重点

最重要的一点：**pre-scroll过程是子View向父View传递事件 , 而scroll过程才是子View消耗滚动事件的过程** , 也就是说父View拥有优先消费事件的权利。

从事件消耗的优先级来看 , 可以画这样一张图。

![NestedScrollEvent](http://otiwf3ulm.bkt.clouddn.com/89ad50e20fd57910556a37b5cfbcb689.png)

dispatchNestedPreScroll传给父View的是没有被消费的滚动事件 , 父View消费完之后通过consumed数组返回 , 如果还有剩余 , 由子View进行消费 , 并将消费多少和剩余多少再次发给父View。

如果一个View同时作为NestedScrollChild和NestedScrollParent , 那么在处理 onNestedPreScrolling和onNestedScrolling的时候也要按照自底向上的规则 , 先让父View处理事件。

### 实例分析以及Q&A

这里通过对**CoordinatorLayout -> SwipeRefreshLaout -> RecyclerView**这个常用的三级嵌套实例进行分析 , 以便深入理解NestedScrolling事件传递的机制。

嗯 , 其实上面那张时序图基本就通过方法调用顺序 , 理清了传递过程。

这里通过几个Q&A , 来答疑解惑。

> 如果你还不清楚SwipeRefreshLaout的原理 , 建议先去看一下我的另外一篇文章：SwipeRefreshLaout源码解析
> CL代表CoordinatorLayout , SRL 代表SwipeRefreshLaout , RV 表示RecyclerView

**Q1：SwipeRefreshLaout在Touch事件分发过程中 , 为什么SwipeRefreshLaout没有作为Touch事件的消费者?**

**A1：**Touch事件流从**ACTION_DOWN**开始：

1. 先经过SRL的`onInterceptTouchEvent()` , 返回false
2. 进入RV的`onInterceptTouchEvent()` , 进入ACTION_DOWN分支 , RV调用`startNestedScroll()`方法
3. 根据上面的时序图 , 会调用SRL的`onNestedScrollAccepted()` , 而这个方法里面 , 会将SRL的`mNestedScrollInProgress`设置为true。实际上到此为止已经进入了NestedScrolling事件的分发流程。
4. 后续事件 , SRL的`onInterceptTouchEvent()`反复会根据`mNestedScrollInProgress`属性返回false , 也就不会拦截事件了。
5. CL的部分根据时序图可以清楚理解。

**Q2：**接Q1 , 既然没有拦截 , 为什么还能处理事件？

**A2：**首先 , 要注意SRL主力的不是Touch事件 , 而是NestedScrolling事件 , 还记得吗 , 实际上是以(dx,dy)偏移量的形式存在的。A1中可以看到 , 一旦触发NestedScrolling机制 , 作为父View的SRL , 就有优先处理NestedScrolling事件的权利 , 所以当然能处理事件（当然优先级比CL低 , 所以只能处理CL处理剩下的部分）。

**Q3：**为什么CL能消费事件进行滚动?

**A3：**NestedScrolling机制决定NestedScrolling事件是自底向上传播的 , 并且通过pre-scroll和scoll两个过程划分 , 越上层的View , 处理NestedScrolling事件的优先级越高 , CL在最上层 , 自然优先处理事件。

**Q4：**对于SwipeRefreshLaout来说 , 什么时候通过TouchEvent处理事件 , 什么时候通过NestedScrolling机制处理事件?

**A4：**NestedScrolling机制由实现了NestedScrollingChild接口的子View触发 , 所以事实上 , 当SRL的子View实现了NestedScrollingChild接口时 , 均会使用NestedScrolling机制分发事件给SRL。比如RecyclerView作为子View将通过NestedScrolling处理事件 , 如果是ListView作为子View , 将通过Touch机制处理事件。

### 总结

读到这里你会发现 , 要理解NestedScrolling , 实际上就是要理解NestedScrolling事件分发过程。
