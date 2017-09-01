---
title: '轻量级自定义圆角ImageView-适配-Android-L'
date: 2017-07-25 22:46:42
tags: View
categories: Android
---

最近在看Google的一个开源项目 Topeka ,想研究一下,了解大神都怎么写代码的.
官方介绍只有一句话：一部有趣的问答应用！
[传送门](https://github.com/googlesamples/android-topeka)


在app的登录页面有一个头像选择,实现了圆形头像,选择头像时ImageView外圈增加一个圆圈,而且适配了5.0以上的版本.实现也很优雅.于是我就仿照写了一个圆角的 ImageView.

效果如下：屏幕中间的就是自定义的 圆角ImageView

![RoundImageView](http://otiwf3ulm.bkt.clouddn.com/c739a9fae63484169161c2cf6d5fd795.png)

布局文件如下:

使用方法也很简单,在布局文件中使用就可以了:
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    >


  <com.sync.customviewstudy.view.RoundImageView
      android:layout_width="80dp"
      android:layout_height="80dp"
      android:layout_centerInParent="true"
      android:id="@+id/img"
      android:src="@mipmap/avatar_12_raster"
      />

</RelativeLayout>
```

先上自定义ImageVIew的代码:

AvatarView.java
```
/**
 * Description: 圆形头像
 * Author：Mari on 2017-07-25 22:30
 * Contact：289168296@qq.com
 */
public class RoundImageView extends ImageView {

  public RoundImageView(Context context) {
    super(context);
  }

  public RoundImageView(Context context, @Nullable AttributeSet attrs) {
    super(context, attrs);
  }

  public RoundImageView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
  }

  @Override public void setImageDrawable(@Nullable Drawable drawable) {
    if (ApiLevelHelper.isAtLeast(Build.VERSION_CODES.LOLLIPOP)) {
      setClipToOutline(true);
      super.setImageDrawable(drawable);
    } else {
      BitmapDrawable bitmapDrawable = (BitmapDrawable) drawable;
      RoundedBitmapDrawable roundedDrawable =
          RoundedBitmapDrawableFactory.create(getResources(), bitmapDrawable.getBitmap());
      roundedDrawable.setCircular(true);
      super.setImageDrawable(roundedDrawable);
    }
  }

  // onLayout() 之后调用
  @Override protected void onSizeChanged(int w, int h, int oldw, int oldh) {
    super.onSizeChanged(w, h, oldw, oldh);
    if (ApiLevelHelper.isLowerThan(Build.VERSION_CODES.LOLLIPOP)) {
      return;
    }
    if (w > 0 && h > 0) {
      setOutlineProvider(new RoundOutlineProvider(Math.min(w, h)));
    }
  }
}
```

代码很少,在 `setImageDrawable`,`onSizeChanged` 两个方法中区分 Android 5.0 即以上版本和5.0一下2个版本不同的处理方式, 5.0 以下版本使用 `RoundedBitmapDrawable` 实现圆形 `drawable` 资源,
而在 5.0 以上版本使用 `RoundOutlineProvider` 实现圆角 `drawable` 资源. 'RoundOutlineProvider' 继承 `ViewOutlineProvider` 这个类是在 `android.view.ViewOutlineProvider` 包下面,大家想要了解更多可以自己去找下资料看看哈.

RoundOutlineProvider.java

```
/**
 * Description: Creates round outlines for views.
 * Author：Mari on 2017-07-25 22:30
 * Contact：289168296@qq.com
 */
@RequiresApi(api = Build.VERSION_CODES.LOLLIPOP) public class RoundOutlineProvider extends ViewOutlineProvider {

  private final int mSize;

  public RoundOutlineProvider(int size) {
    mSize = size;
  }

  @Override public void getOutline(View view, Outline outline) {
    outline.setOval(0, 0, mSize, mSize);
  }
}
```

ApiLevelHelper.java

```
/**
 * Encapsulates checking api levels.
 */
public class ApiLevelHelper {

    private ApiLevelHelper() {
        //no instance
    }

    /**
     * Checks if the current api level is at least the provided value.
     *
     * @param apiLevel One of the values within {@link Build.VERSION_CODES}.
     * @return <code>true</code> if the calling version is at least <code>apiLevel</code>.
     * Else <code>false</code> is returned.
     */
    public static boolean isAtLeast(int apiLevel) {
        return Build.VERSION.SDK_INT >= apiLevel;
    }

    /**
     * Checks if the current api level is at lower than the provided value.
     *
     * @param apiLevel One of the values within {@link Build.VERSION_CODES}.
     * @return <code>true</code> if the calling version is lower than <code>apiLevel</code>.
     * Else <code>false</code> is returned.
     */
    public static boolean isLowerThan(int apiLevel) {
        return Build.VERSION.SDK_INT < apiLevel;
    }
}
```



源代码地址: https://github.com/MariShunxiang/AndroidSamples/blob/master/widget/customviewstudy/src/main/java/com/sync/customviewstudy/view/RoundOutlineProvider.java
