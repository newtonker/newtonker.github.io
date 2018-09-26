# RecyclerView+ViewPager实现轮播时的问题

## 问题描述
RecyclerView中嵌套ViewPager做轮播，最开始的思路是ViewPager要展示的数据作为一种type类型，在Adapter中展示数据的时候，按照类型展示出来。

但是在使用的过程中遇到了两个问题：

1. 每次启动后，当首次加载出整个页面时，不论是向左滑动还是向右滑动，动画都无法出现。由于轮播在ViewPager原始数据的首尾各加了一个页面，会出现往左滑动时，到达不了最后一个页面的情况。
2. 由于这个页面和另外一个页面存在关联性，当另外一个页面切换tab页的时候，返回当前页面时，有时会出现tab页动画未完成的情况，左右两个页面各出现了一半。

遇到这两个问题时，最开始怀疑是RecyclerView和ViewPager滑动冲突引起的。直到看到了下面[这篇文章](https://blog.csdn.net/u011002668/article/details/72884893)。这篇文章中列出的两个问题，跟我上面列出的两个问题完全一样。

### 问题一
#### 问题原因
正如上面的文章所介绍的那样，ViewPager在attachWindow的时候，有一个标志位是mFirstLayout，当这个标志位为true的时候，在setCurrentItem的时候并不会执行动画，而是直接进行set。详见下面的源码：

```java
@Override
protected void onAttachedToWindow() {
    super.onAttachedToWindow();
    mFirstLayout = true;
}

public void setCurrentItem(int item) {
    mPopulatePending = false;
    setCurrentItemInternal(item, !mFirstLayout, false);
}

void setCurrentItemInternal(int item, boolean smoothScroll, boolean always, int velocity) {
    ...
    if (mFirstLayout) {
        // We don't have any idea how big we are yet and shouldn't have any pages either.
        // Just set things up and let the pending layout handle things.
        mCurItem = item;
        if (dispatchSelected) {
            dispatchOnPageSelected(item);
        }
        requestLayout();
    } else {
        populate(item);
        scrollToItem(item, smoothScroll, velocity, dispatchSelected);
    }
}

```

从上面的代码中可以看出，当首次加载出页面的时候，mFirstLayout为true时，就不会执行动画操作。具体到项目中遇到的实际场景，我认真分析了一下，因为每次RecyclerView数据初次获取，页面初次创建的时候都会导致attach操作，所以会导致首次动画不执行的情况。

#### 解决方案
其中一个解决方案是按照上面的文章中提到的，由于mFirstLayout是private属性，所以自定义一个ViewPager，然后重写onAttachedToWindow，利用反射修改mFirstLayout的值。参考代码如下：

```java
@Override
protected void onAttachedToWindow() {
    super.onAttachedToWindow();
    try {
        Field mFirstLayout = ViewPager.class.getDeclaredField("mFirstLayout");
        mFirstLayout.setAccessible(true);
        mFirstLayout.set(this, false);
        getAdapter().notifyDataSetChanged();
        setCurrentItem(getCurrentItem());
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```


### 问题二

#### 问题原因
当ViewPager不可见之后，在另一个页面操作了数据，需要ViewPager切换tab的时候，出现两个tab页个出现一半的情况，这个问题的主要原因是，当ViewPager正在执行动画的时候，如果调用了detachWindow，则会导致动画终止，可能会出现两个tab页同时存在的情况。具体到项目中的场景，当回到viewpager页面的时候，如果正在执行tab切换的行为，又调用了RecyclerView的Adapter的notifyDataSetChanged的方法，可能会导致detachWindow，这样便会终止动画的执行。分析代码如下：

```java
 @Override
protected void onDetachedFromWindow() {
    removeCallbacks(mEndScrollRunnable);
    // To be on the safe side, abort the scroller
    if ((mScroller != null) && !mScroller.isFinished()) {
        mScroller.abortAnimation();
    }
    super.onDetachedFromWindow();
}
```

#### 解决方案
上述文章中给出的方案是一个参考，可以再detachwindow的时候，判断Activity是否已经销毁，如果没有销毁，则继续执行动画。

```java
@Override
protected void onDetachedFromWindow() {
    if (hasActivityDestroy) {
        super.onDetachedFromWindow();
    }
}

public void setHasDestroy(boolean hasDestroy) {
    hasActivityDestroy= hasDestroy;
}
```

## 另一种办法
从上面的解决方案中，问题一是利用了反射，问题二是利用强制不停止动画。这两个解决方案我都不太满意。一是因为8.0以上的系统Android官方已经不建议使用反射了，可能会有兼容性问题。第二个问题的解决方案也不是那么地道。所以考虑之后我还是把ViewPager移除了RecyclerView，不在作为一种type类型，改成单独维护了。

总结来说就是布局改成了：NestedScrollView+ViewPager+RecyclerView的方式来实现。这样便不会从在ViewPager被频繁的attach和detach的情况。但是在嵌套NestedScrolView和RecyclerView的时候遇到一个问题是，每次页面加载出来之后，RecyclerView都获取焦点并滑动到RecyclerView起始的位置。

解决方案参考了[这篇文章](https://juejin.im/post/5a2a04726fb9a045055e0993)，在xml中增加配置的属性descendantFocusability属性：

```xml
<android.support.v4.widget.NestedScrollView
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fillViewport="true"
    >

  <LinearLayout
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      android:descendantFocusability="blocksDescendants"
      android:orientation="vertical"
      >

    <com.netease.vbox.main.discover.widget.NaturalView
        android:id="@+id/natural_view"
        android:layout_width="match_parent"
        android:layout_height="432dp"
        />

    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_list"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        />

  </LinearLayout>

</android.support.v4.widget.NestedScrollView>
```

另外，为确保兼容性，在页面初始化的时候，增加如下代码：

```java
ViewCompat.setNestedScrollingEnabled(mBinding.rvList, false);
```
