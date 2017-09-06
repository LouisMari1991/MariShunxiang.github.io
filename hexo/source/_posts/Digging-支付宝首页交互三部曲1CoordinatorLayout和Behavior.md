---
title: '[Digging]支付宝首页交互三部曲1CoordinatorLayout和Behavior'
date: 2017-08-30 21:34:01
tags: NestedScrolling
categories: Android
---

![zhifubao_first](http://otiwf3ulm.bkt.clouddn.com/2272923f2d28ef82c3680eea371e8eff.png)

### 前言

自己动手实现支付宝首页效果 , 用三篇文章记录并分享给大家。

* CoordinatorLayout和Behavior
* 自定义CoordiantorLayout和Behavior
* 支付宝首页效果实现

> 文中 : Col 表示 CoordiantorLayout , ABL 表示AppBarLayout , CTL 表示 CollapsingToolbarLayout , SRL 表示 SwipeRefreshLaout , RV 表示 RecyclerVIew。

第一篇文章主要讨论Behavior的结构、CoordianatorLayout的实现以及Coordianator和Behavoir之间的通信。除了Behavior相关的内容 , CoordinarotLayout作为官方实现的一个ViewGroup , 也有一些自定义ViewGroup时可以借鉴的内容 , 这些也穿插在这篇文章中。

### Behavior结构

使用 CoordinatorLayout 结合 ABL , CTL , SWL/RV 可以方便的实现各种 MaterialDesign ToolBar 效果 , 还有 FloatingActionButton , SnackBar 等控件 , 可以直接使用。 Col的主要功能是为其子View提供 **协调滚动** 的统一接口 , 让子View可以方便的实现诸如嵌套滚动 , 跟随滚动等效果 , 让界面更加灵动 ; 而这个统一接口 , 就是Behavior。

我们先来看一下 Behavior 提供的方法 , 大致可以分为4组:

**布局相关** , 这类方法用来重载child的Mesure、 Layout相关的回调。

```
public boolean onMeasureChild(CoordinatorLayout parent, V child,
                int parentWidthMeasureSpec, int widthUsed,
                int parentHeightMeasureSpec, int heightUsed);
public boolean onLayoutChild(CoordinatorLayout parent, V child, int layoutDirection);
public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency);
public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency);
public void onDependentViewRemoved(CoordinatorLayout parent, V child, View dependency)
```

**Touch事件相关** , 这组方法用来拦截和处理Touch事件传递。

```
public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev);
public boolean onTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev);
public boolean blocksInteractionBelow(CoordinatorLayout parent, V child);
```

**NestedScrolling相关** , 这组方法用来响应NestedScrolling , 更多的NestedScrolling的讨论可以查看[另一篇博客](https://marishunxiang.github.io/2017/08/27/Android-Nested-Scrolling/)

```
public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout,
                V child, View directTargetChild, View target, int nestedScrollAxes);
public void onNestedScrollAccepted(CoordinatorLayout coordinatorLayout, V child,
                                View directTargetChild, View target, int nestedScrollAxes);
public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target);
public void onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target,
                int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed);
public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target,
                              int dx, int dy, int[] consumed);                      
public boolean onNestedFling(CoordinatorLayout coordinatorLayout, V child, View target,
                          float velocityX, float velocityY, boolean consumed);      
public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target,
                                          float velocityX, float velocityY)
```

**其他辅助方法**

```
// 关联/取消关联LayoutParams的时候回调
public void onAttachedToLayoutParams(@NonNull CoordinatorLayout.LayoutParams params);
public void onDetachedFromLayoutParams();
// 控制Scrim效果 , 只有当getScrimOpaticy返回值不为0时才绘制。
public int getScrimColor(CoordinatorLayout parent, V child);
public float getScrimOpacity(CoordinatorLayout parent, V child);
// 暂时没有发现用到的地方
public static void setTag(View child, Object tag);
public static Object getTag(View child);
public boolean onRequestChildRectangleOnScreen(CoordinatorLayout coordinatorLayout,
                V child, Rect rectangle, boolean immediate);
public void onRestoreInstanceState(CoordinatorLayout parent, V child, Parcelable state);
public Parcelable onSaveInstanceState(CoordinatorLayout parent, V child);
// 防止Col子View间出现遮挡 , 获取child应避免遮挡部分的Rect。
public boolean getInsetDodgeRect(@NonNull CoordinatorLayout parent, @NonNull V child,
                @NonNull Rect rect);
```

多数情况下我们只需要关注前三组方法。通过这些方法我们看到Behavior可以做的事情不仅仅是 "依赖某个View的变化并且在其变化后进行相应" 这么单一 , **Behavior实际上可以控制child在Col中的mesure , layout以及拦截touch事件 , 支持NestedScrolling等等** , 基本上是除了Draw之外的全部自定义VIew需要关注的内容了。

Behavior的使用和自定义我们下一篇文章进行讨论 , 这篇文章我们继续关注Col如何操作Behavior。

### 设置Behavior

给View设置Behavior有两种方法 : 在布局文件中指定 ; 使用`@CoordinatorLayout.DefaultBehavior`注解。

第一种方法适用于多种情况 , 需要注意的是 ,　如果使用自定义Behavior , 需要覆写２个参数的构造方法 ;

```
public Behavior(Context context, AttributeSet attrs);
```

因为Col在解析时是通过反射调用Behavior的这个构造方法创建Behavior对象的 :

```
static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
        if (TextUtils.isEmpty(name)) {
            return null;
        }

        final String fullName;
        if (name.startsWith(".")) {
            // Relative to the app package. Prepend the app package name.
            fullName = context.getPackageName() + name;
        } else if (name.indexOf('.') >= 0) {
            // Fully qualified package name.
            fullName = name;
        } else {
            // Assume stock behavior in this package (if we have one)
            fullName = !TextUtils.isEmpty(WIDGET_PACKAGE_NAME)
                    ? (WIDGET_PACKAGE_NAME + '.' + name)
                    : name;
        }

        try {
            Map<String, Constructor<Behavior>> constructors = sConstructors.get();
            if (constructors == null) {
                constructors = new HashMap<>();
                sConstructors.set(constructors);
            }
            Constructor<Behavior> c = constructors.get(fullName);
            if (c == null) {
                final Class<Behavior> clazz = (Class<Behavior>) Class.forName(fullName, true,
                        context.getClassLoader());
                c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
                c.setAccessible(true);
                constructors.put(fullName, c);
            }
            return c.newInstance(context, attrs);
        } catch (Exception e) {
            throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
        }
    }
```

第二种方法 , 适用于自定义View并且自定义Behavior的情况 , 比如AppBarLayout :

```
@CoordinatorLayout.DefaultBehavior(AppBarLayout.Behavior.class)
public class AppBarLayout extends LinearLayout {
  // {...}
}
```

如果在布局文件中不另外指定 , 这里将调用Behavior的无参构造方法创建对象。

### 事件分发

上面看到的Behavior的各个方法 , 其调用者基本都是CoL。CoL在自己的回调方法中通过子View Behavior的相关方法 , 将事件向下分发。

在讨论分发方法之前 , 有一点需要注意 : **Behavior随人影响的是子View的布局和行为 , 但实际是对CoL本身事件处理的代理。**

基本的模式大家可以想到 , 就是遍历子View , 获取Bahavior , 然后调用子Bahavior对应的方法。这里对几个有意思的地方进行讨论 :

#### Touch事件分发

Touch事件分发分两个方法onInterceptTouchEvent和onTouchEvent。 CoL的视线中 , 这两个方法都调用`performIntercept`方法将是否拦截事件的判断交给Behavior处理。每个CoordinatorLayout的子View都有机会拦截事件并响应 , 注意这里子View并不是在自己的onTouch相关方法中进行处理, 而是Behavior子类 , 有机会代理CoL对事件进行拦截并处理。

处于篇幅考虑这里不贴源码了 , 关键的地方这里解释一下 :

- 在遍历子View之前 , 使用`getTopSortedChildren(topmostChildList);`获取按照显示顺序自上至下排效果的子View列表。 ViewGroup可以复写`getChildDrawingOrder`自定义子View的绘制顺序 , getTopSortedChildren方法会按照绘制顺序获取子View ; 在5.0以上版本中 , 还要考虑Z轴次序 , 也就是elevation , 会再进行一次排序 , 最终得到真实可靠的自顶之下的子View分发顺序。这对让子View合理响应Touch事件很重要 , 如果自定义ViewGroup需要有类似功能 , 可以参考CoL的实现。

- Behavior通过覆写`onInterceptTouch`或者`onTouchEvent`并返回true来声明拦截事件 , Col会将该View缓存到mBehaviorTouchView属性 , 后续事件将直接分发到该View。直到该View的onTouchEvent方法返回false。

- 在确定mBehaviorTouchView之后 , CoL会将该View（Z轴）下面View的事件流终止 , 具体操作是向这些View分发一个CANCEL事件。

>Behavior可以通过覆写blocksInteractionBelow方法block下方View的事件。 在自己不需要处理事件但同时不希望子View处理事件时 , 可以简单的覆写这个方法。

> 默认实现逻辑是判断getScrimOpaticy的值>0

#### NestedScrolling

NestedScrolling是Behavior实现滑动的重要支撑。前文提到Behavior是对CoL自身事件的代理 , 所以Behavior对NestedScrolling的支持 , 就是在代理CoL的NestedScrollingParent接口的方法。

> 更多NestedScrolling相关信息参见更早的博客 : [Android Nested Scrolling](https://marishunxiang.github.io/2017/08/27/Android-Nested-Scrolling/)

需要注意的是关于NestedScrolling机制中"消费量"的处理

```
@Override
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    int xConsumed = 0;
    int yConsumed = 0;
    boolean accepted = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        if (view.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            mTempIntPair[0] = mTempIntPair[1] = 0;
            viewBehavior.onNestedPreScroll(this, view, target, dx, dy, mTempIntPair);

            xConsumed = dx > 0 ? Math.max(xConsumed, mTempIntPair[0])
                    : Math.min(xConsumed, mTempIntPair[0]);
            yConsumed = dy > 0 ? Math.max(yConsumed, mTempIntPair[1])
                    : Math.min(yConsumed, mTempIntPair[1]);

            accepted = true;
        }
    }

    consumed[0] = xConsumed;
    consumed[1] = yConsumed;

    if (accepted) {
        onChildViewsChanged(EVENT_NESTED_SCROLL);
    }
}
```

注意这里是取了所有Behavior消费掉偏移量的最大值。因为Behavior是代理的角色 , 而各个代理的消费对于NestedScrolling机制来说 , 都会被看作是CoL这个NestedScrollingParent的消费。各Behavior之间是同级的 , 所以他们对事件的消费是"重叠"的（可以重复消费） , 所以这里返回的consumed是取最大值。

#### LayoutDependence

这里是其他博客讲的比较多的地方 , 确定View依赖的dependency变化之后 , 会将变化广播给所有依赖这个View的兄弟View。这里我要说的有两点 : 1.onDependentViewChanged回调的时机。2.依赖关系的存贮。

1. onDependentViewChanged在某个View的大小或者位置发生变化的时候都会进行回调。并且真正变化之后才会回调。
2. 子View之间的依赖关系通过非循环有向图数据结构进行存储。具体到数据结构上就是通过一个`Map<Node, List<Node>>`存储（这个Map并不是JDK的实现 , 感兴趣的可以看下源码）。
3. 既然存在依赖关系 , 那么在涉及到对子View遍历的时候 , 就要考虑到子View之间的依赖关系。CoL的实现中`prepareChildren`方法构建依赖图并根据依赖图进行DFS搜索得到依赖链列表 , 这个列表用在了分发布局、 NestedScrolling、 LayoutDependentChanged的过程中。

### 自定义LayoutParams

自定义ViewGroup很常见 , 但是多数情况下用不到自定义LayoutParams。LayoutParams正如其名 , 用了设置布局参数 , 也就是控制ViewGroup如何 mesure 和 layout 子View。 如果一个自定义ViewGroup提供了额外的布局参数 , 那就需要自定义LayoutParams了。自定义LayoutParams并没有多复杂 , 这里列出几点需要注意的地方。

#### 基类

如果自定义LayoutParams需要支持margin , 继承自 `ViewGroup.MarginLayoutParams`即可 , 默认的ViewGroup.LayoutParams并不支持margin。

#### 构造方法

LayoutParams默认有多个不同参数的构造方法 :

* LayoutParams(Context context, AttrbuteSet attr) 使用于解析布局文件时生成LayoutParams , layout相关的xml属性 , 就是在这个构造方法里面解析的
* LayoutParams(int width, int height) 代码构建时只传入宽高
* LayoutParams(LayoutParams source) LayoutParams转换
* LayoutParams() 无参构造函数

自定义LayoutParams也要覆写这些构造方法并做相应的转换。

#### ViewGroup方法

使用自定义 LayoutParams 的 ViewGroup , 也需要实现几个相关方法 , 主要是在解析 、 addView 的时候生成适合的 LayoutParams。

```
protected LayoutParams generateDefaultLayoutParams();
protected LayoutParams generateLayoutParams(ViewGroup.LayoutParams p);
public LayoutParams generateLayoutParams(AttributeSet attrs);
// 检查LayoutParams是否自定义的LayoutParams类型
protected boolean checkLayoutParams(ViewGroup.LayoutParams p);
```

具体可以参考 CoL。

#### CoordinatorLayout.LayoutParams

CoL.LP 主要实现了基于 anchorView 的布局和 keylines（纵向基准线 , 子View可以对齐到keylines）的布局、 保存Bahavior、 存储滑动过程中的标记位 , 具体实现这里就不展开了 , 逻辑比较简单。
