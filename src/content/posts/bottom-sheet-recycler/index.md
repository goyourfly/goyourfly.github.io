---
title: "BottomSheetBehavior+RecyclerView 的问题"
published: 2020-03-25
tags: [Android,View]
category: 技术
draft: false
---

在使用 BottomSheetBehavior+RecyclerView 的组合时出现一个 Bug，列表 Fling（惯性） 到底部或者顶部时，Item 会出现短暂的不能点击，具体时间长短跟滑动速度有关

监听 RecyclerView 的状态发现，在列表滑动到底部停止时，RecyclerView 的状态还是 SCROLL_STATE_SETTLING，并没有马上变成 SCROLL_STATE_IDLE，这是导致 Item 不能被点击的最直接原因

那么问题来了，列表都滑动到底，所有 View 都停止滑动了，为什么状态还没有被设置为 SCROLL_STATE_IDLE 呢？

通过断点调试 RecyclerView 内部处理 Fling 部分的类：ViewFlinger，发现它的 run 方法有一处行为比较异常：dispatchNestedScroll 返回 true，在所有滑动都停止后还是返回 true，导致 fling 没有被终止。

```
public void run() {
  ...
  // 由于这里其他 View 处理了 scroll，导致无法进入这个 if
  if (!dispatchNestedScroll(hresult, vresult, overscrollX, overscrollY, null,
TYPE_NON_TOUCH)
&& (overscrollX != 0 || overscrollY != 0)) {
    final int vel = (int) scroller.getCurrVelocity();

    int velX = 0;
    if (overscrollX != x) {
        velX = overscrollX < 0 ? -vel : overscrollX > 0 ? vel : 0;
    }

    int velY = 0;
    if (overscrollY != y) {
        velY = overscrollY < 0 ? -vel : overscrollY > 0 ? vel : 0;
    }

    if (getOverScrollMode() != View.OVER_SCROLL_NEVER) {
        absorbGlows(velX, velY);
    }
    if ((velX != 0 || overscrollX == x || scroller.getFinalX() == 0)
        && (velY != 0 || overscrollY == y || scroller.getFinalY() == 0)) {
        // 正常情况这里会停止 scroller
        scroller.abortAnimation();
    }
  }
  ...
}
```
如果返回了 true，证明 ViewParent 消耗了本次滑动，但是又没有做任何 UI 上的变动，被丢弃了？

那查一下 ViewParent 到底是谁？ 在 NestedScrollingChildHelper 里发现时通过一个 getNestedScrollingParentForType 返回那个处理 nestedScroll 的 Parent

```
private boolean dispatchNestedScrollInternal(int dxConsumed, int dyConsumed,
        int dxUnconsumed, int dyUnconsumed, @Nullable int[] offsetInWindow,
        @NestedScrollType int type, @Nullable int[] consumed) {
  if (isNestedScrollingEnabled()) {
    // 看这里
    final ViewParent parent = getNestedScrollingParentForType(type);
    if (parent == null) {
        return false;
    }
    ...
  }
}
```
而这个方法只是根据是触摸滑动还是惯性滑动返回对应的两个 Parent
```
private ViewParent getNestedScrollingParentForType(@NestedScrollType int type) {
    switch (type) {
        case TYPE_TOUCH:
        return mNestedScrollingParentTouch;
        case TYPE_NON_TOUCH:
        return mNestedScrollingParentNonTouch;
    }
    return null;
}
```
初始化 Parent 的地方则是在 startNestedScroll 方法，
> 这里会根据这个类是否支持 NestedScroll 以及 onStartNestedScroll(xx):Boolean 返回 true 来找到处理本次事件的 Parent，循环往上找到后调用 setNestedScrollingParentForType 将 Parent 暂存起来
```
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
    if (hasNestedScrollingParent(type)) {
        // Already in progress
        return true;
    }
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        while (p != null) {
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                // 暂存到全局变量
                setNestedScrollingParentForType(type, p);
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}
```
捋一下调用流程
```
onTouchEvent（通过事件判断是 fling） -> RecyclerView.fling() -> startNestedScroll()
                                                         |-> OverScroller.fling()
```
接着上面的层层往上找，处理本次 NestScroll 的 Parent 最终确定是 CoordinatorLayout

所以解决办法是重写一下自定义 BottomSheetBehavior 或者 CoordinatorLayout，然后重写一下 onStartNestedScroll
```
override fun onStartNestedScroll(child: View, target: View, axes: Int, type: Int): Boolean {
    // 告诉 RecyclerView Fling 时不接受 NestScroll
    if(target is RecyclerView && type == ViewCompat.TYPE_NON_TOUCH){
        return false
    }
    return super.onStartNestedScroll(child, target, axes, type)
}
```
至于为什么 fling 的时候 Parent 没有动？ 那是因为 BottomSheetBehavior 的 onNestedPreScroll 只处理了滑动类型不为 1 的情况 而 1 正式 Fling 类型到滑动 public static final int TYPE_NON_TOUCH = 1;
```
public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull V child, @NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {
    if (type != 1) {
        ...
    }
}
```
尝试重写一下 onNestedPreScroll，调 super 时 type 直接传 0，BottomSheetBehavior 就会处理 Fling 类型的 NestScroll 了，但是体验很糟糕，这应该就是为什么禁止在 Fling 时 NestScroll