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

### Support中的Behavior基类

CAR场景中一共出现了两个Behavior , AppBarLayout.Behavior和AppBarLayout.ScrollingViewBehavior , 前者应用于ABL , 后者应用于RV。这两个Behavior是我们这篇文章要分析的主要的类 , 但是在开始之前 , 我们要看一下它们的基类（职责分隔的很不错）。

#### ViewOffsetBehavior

使用ViewOffsetHelper工具类封装View的偏移量。View类支持对offset进行偏移 , 但是并不会保存偏移量。ViewOffsetHelper对Offset和Top/left进行缓存 , 使用ViewCompat工具类进行偏移处理。

```
private void updateOffsets() {
  ViewCompat.offsetTopAndBottom(mView, mOffsetTop - (mView.getTop() - mLayoutTop));
  ViewCompat.offsetLeftAndRight(mView, mOffsetLeft - (mView.getLeft() - mLayoutLeft));
}
```

ViewOffsetBehavior除了封装了对水平和垂直方向偏移的Setter和Getter方法 , 还覆写了`onLaoutChild()`方法 , 上一篇文章中有提到 , 实现这个方法可以代理CoL对子View的布局。不过ViewOffsetBehavior覆写这个方法的目的主要是创建ViewOffsetHelper , 获取真实偏移量并且将child偏移到正确位置。

> 说句题话外 , 当我们考虑一个滑动交互时 , 不要把滑动看做一个连续的过程 , 而是要拆分成多个单独的循环过程 , 连续的滑动只不过是单独循环过程在时间上不断重复而已 ； 而滑动的单个循环过程 , 说到底是对View进行偏移处理。当看到一个复杂交互效果的时候 , 要学会拆分 , 一个是刚说的时间上拆分 , 另一个方面就是要能拆分成多个单独效果的合成 , 能做到这一步 , 再加上牢固的基础 , 就没有什么交互效果是做不出来的。

#### HeaderBehavior

HeaderBehavior封装了经典Touch事件分发逻辑 , 主要是实现了Behavior的`onInterceptTouchEvent`方法和`onTouchEvent`方法　, 逻辑其实也很简单 :

 * 判断是否可以滑动
 * 当滑动超过阀值之后 , 标记滑动（mIsBeingDragged）并进行拦截
 * 处理ACTION_MOVE事件　, 调用ViewOffsetBehavior的方法进行偏移
 * 使用VelocityTracker计算滑动速度
 * 在ACTION_UP分之中停止并判断是否应该Fling
 * 实现scroll和fling方法

 HeaderBehavior的实现简单且清晰 , 都可以当作经典Touch事件实现活动的范例了 , 有这方面需求的同学不要错过。因为HeaderBehavior的定位很明确 , 实现类似AppBarLayout类似的Header功能 , 所以只处理了纵向滑动。

除了scroll和fling暴露给子类的方法主要是`setHeaderTopBoottimOffset` , 这个方法一共有两个重载声明 , 可以设置边界值避免滑动越界。

```
int setHeaderTopBottomOffset(CoordinatorLayout parent, V header, int newOffset) {
    return setHeaderTopBottomOffset(parent, header, newOffset,
            Integer.MIN_VALUE, Integer.MAX_VALUE);
}

int setHeaderTopBottomOffset(CoordinatorLayout parent, V header, int newOffset,
        int minOffset, int maxOffset) {
    final int curOffset = getTopAndBottomOffset();
    int consumed = 0;

    if (minOffset != 0 && curOffset >= minOffset && curOffset <= maxOffset) {
        // If we have some scrolling range, and we're currently within the min and max
        // offsets, calculate a new offset
        newOffset = MathUtils.constrain(newOffset, minOffset, maxOffset);

        if (curOffset != newOffset) {
            setTopAndBottomOffset(newOffset);
            // Update how much dy we have consumed
            consumed = curOffset - newOffset;
        }
    }

    return consumed;
}

```

> 这个方法是有返回值的 , 这个返回值在子类中处理嵌套滑动或者再次分发滑动是非常有用。

#### HeaderScrollingViewBehavior

同样继承自ViewOffsetBehavior , HeaderScrollingViewBehavior的职责主要是完成对ScrollingView的布局。 CoL的职责是给子类提供协调滚动的接口 , 并不会具体实现某种效果 , 所有子类需要完成的功能和效果 , 都需要通过统一接口Behavior完成。

在Header+ScrollingView的结构中 , HeaderScrollingViewBehavior就是对ScrollingView的控制。 **这两者结合要实现的就是MaterialDesign中经典的可收起Header的效果。**

为了让Header可收起 , 视觉上ScrollingView的高度被拉长了 , 但实际上ScrollingView的高度并没有变 , 变的是ScrollingView的位置。ScrollingView的测量和布局就是HeaderScrollingViewBehavior的实现内容。

```
final int childLpHeight = child.getLayoutParams().height;
if (childLpHeight == ViewGroup.Layout.MATCH_PARENT || childLpHeight == ViewGroup.LayoutParams.WRAP_CONTENT) {
  // If the menu's height is set match_parent/wrap_content then measure it
  // with the maximum visible height

      // {...}
      return true;
  }
}
return false;
```

`onMeasureChild`方法中的注释说明了只要child和LayoutParams是MATCH_PARENT或者WRAP_CONTENT , 就设置child的高度为**最大可见高度。** 这里的最大可见高度包含除了header之外的区域以及header收起时额外空出的区域 , 也就是**header的可滚动区域。**

```
int availableHeight = View.MeasureSpec.getSize(parentHeightMeasureSpec);
if (availableHeight == 0) {
    // If the measure spec doesn't specify a size, use the current height
    availableHeight = parent.getHeight();
}

final int height = availableHeight - header.getMeasuredHeight()
        + getScrollRange(header);
```

`onLayout` 中将ScrollingView置于header下方。

```
available.set(
  parent.getPaddingLeft() + lp.leftMargin,
  header.getBottom() + lp.topMargin,
  parent.getWidth() - parent.getPaddingRight() - lp.rightMargin,
  parent.getHeight() + header.getBottom() - parent.getPaddingBottom() - lp.bottomMargin
);
```

注意这里Rect的top值取`header.getBottom() + lp.topMargin` , 而不是 `getPaddingTop() + header.getHeight + lp.topMargin` , 这是因为header在onLayout时可能已经包含偏移量 , 不能**假定**header在初始位置 , 即便可能90%的情况均是如此。

说句题外话 , 项目开发过程中会遇到很多这类情况 , 有多种实现方式都能达到预期效果 , 但并不是所有的实现方案都是完整符合**预期逻辑**的。比如上面的例子 , ScrollingView的预期位置是header下方 , 而不是父控件中除header高度以外的区域。有的时候 , 需要转换角度看问题 , 体会下这其中的区别。

#### AppBarLayout.Behavior

AppBarLayout.ScrollingViewBehavior相对简单 , 这里略过。 AppbarLayout.Behavior继承自HeaderBehavior , 在其基础上 , 主要实现了以下功能：

1. 支持在布局文件中定义滚动效果：SCROLL/ECIT_UNTIL_COLLAPSED/ENTER_ALWAYLS/ENTER_ALWAYLS_COLLAPSED/SMAP

2. 实现NestedScrolling回调

滚动效果不是这篇文章的重点 , 我们主要看下NestedScrolling的相关实现。

**onStartNestedScroll**

判断是否是否为纵向滑动 , 并且AppBarLayout支持折叠并且ScrollingView的大小超出屏幕范围。

```
// Return true if we're nested scrolling vertically, and we have scrollable children
// and the scrolling view is big enough to scroll
final boolean started = (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0
        && child.hasScrollableChildren()
        && parent.getHeight() - directTargetChild.getHeight() <= child.getHeight();

```

**onNestedPreScroll**

这个方法回提前于 ScrollingView 消费滑动事件。AppBarLayout 的 scrollFlags , 也就是上面说的滚动效果会影响 onNestedPreScroll 方法的实现。抛开这个影响 , 这个方法中 , 首先确定 AppBarLayout 的可滑动范围 , 然后调用 `scroll()` 方法（继承自ViewOffsetBehavior）进行滚动 , 并将消费多少传递给 `consumed` 数组。

**onNestedScroll**

如果向下滚动时 , 在ScrollingVIew消费完滑动时间之后 , 还有剩余 , 说明 ScrollingView 已经滚动到顶部 , AppBarLayout 开始展开。

**onNestedFling**

这里并没有进行精确的消费 , 只是当ScrollingView触发flag时 , 对 AppbarLayout 执行动画 , 展开或者收起。**下篇文章实现支付宝首页效果时 , 实现了对 fling 的精确消费。**


#### 总结

自定义Behavior主要关心一下两个方面：

 1. 测量和布局

 2. 实现滑动效果

其中滑动效果有三种实现方式：

 1. 经典Touch事件

 2. NestedScrolling

 3. LayoutDependent

一般情况下 , CoL的child , 如果自身不可滚动 , 需要实现NestedScrolling来进行联动 , 或者实现Touch事件回调。如果自身可以滚动 , 通过 `onDependentViewChanged` 方法来响应其他View的偏移量改变事件。
