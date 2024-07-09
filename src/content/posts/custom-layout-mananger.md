---
title: 写出你自己的 RecyclerView LayoutManager
published: 2015-06-06
tags: [Android,RecyclerView]
category: 技术
draft: false
---

[原文链接](http://wiresareobsolete.com/2014/09/building-a-recyclerview-layoutmanager-part-1/)
如果你是Android开发者，到现在为止，你应该至少听说过RecyclerView；一个新的组件，将被加入Support库，它致力于便捷自定义、高性能、可回收的View集合。关于RecyclerView怎么使用的文章在网上已经有一大片就不多说了，这里有几个Demo：
[A First Glance at Android's RecyclerView](http://www.grokkingandroid.com/first-glance-androids-recyclerview/)
[The new TwoWayView](http://lucasr.org/2014/07/31/the-new-twowayview/)
[RecyclerViewItemAnimators](https://github.com/gabrielemariotti/RecyclerViewItemAnimators)
在这一系列的文章中，我将关注于底层的实现，来帮助你建立你自己的较复杂LayoutManager实现，而不是一个简单的横向或者竖向的滑动列表。
>在介绍之前，需要注意一下，LayoutManager API 支持强大且复杂的布局回收，因为它没有实现很多，大部分的代码需要你自己去实现。就像在其他的项目中所涉及的自定义View，不要过度的封装和过度的优化你的代码，只需实现在这个应用需要的特性即可。

#RecyclerView Playground
在该系列的文章中，所有的代码片段都是来自于我之前在GitHub中提交的代码[RecyclerView Playground sample](https://github.com/devunwired/recyclerview-playground)，这个示例包含了几乎各种类型的RecyclerView的用法，从创建一个简单的列表到自定义LayoutManagers。
这篇文章中的代码是从 FixedGridLayoutManager的示例中摘出来的，一个二维的Grid Layout，并且支持水平和垂直方向的滑动。
#The Recycler
首先，看一下API的结构。LayoutManager被赋予一个可访问的`Recycler`实例在需要的地方，当你需要回收旧的views时，或者需要从可能之前已经被回收的之前的子view中获取新的views。

Recycler同时屏蔽了直接操作当前Adapter中的views，当你的LayoutManager需要请求一个新的子view是，只需要简单的调用`getViewForPosition()`，然后Recycler将会把绑定正确数据的View返回给你。Recycler注重确定是否需要创建一个新的View还是将已存在的，报废的View复用。在LayoutManager中，你的职责就是确定那些不再显示的View及时传递给Recycler，这将使Recycler保证不会创造多于需求的View。
#Detach vs.Remove

有两种方式可以在更新布局的适合传递退出的子View，分离和移除。分离是一种比较轻量级的重新排序的方式。在你的代码被返回之前，被分离的View将被重新附加，这将被用于改变附加子View指数而不需要通过Recycler重新绑定或者重新创建这些View。

移除意味着View将不在需要，任何被永久移除的将被放置在Recycler用于以后的复用，但是API没有强制这样做，这取决于你是否需要回收它们。
#Scrap vs. Recycle
Recycler是两级缓存系统：废料堆和回收池。废料堆是一个轻量级的可回收的集合，这里的View可以被直接返回到LayoutManager而不需要经过Adapter。
View通常被放在这里当它们被暂时废弃的时候，但在同一次布局中马上复用。回收池由一些被认定为绑定不正确的数据（从其他位置类的数据）的View，所以在传递到LayoutManager之前，它们将被传递到Adapter去重新绑定。

当尝试着给LayoutManager提供新的View时，Recycler首先会检查废料堆去匹配位置/ID，如果存在，它将被直接返回，而不需要重新通过Adapter绑定数据。如果没有匹配的View，Recycler将会从回收池中取一个合适的View并通过Adapter绑定必要的数据（i.e. `RecyclerView.Adapter.bindViewHolder()` 会被执行）在返回之前。万一回收池中没有可用的View，一个新的View将会被创建（i.e. `RecyclerView.Adapter.createViewHolder()`将会被执行）在绑定之前，然后返回。
#Rule of Thumb
如果你愿意，LayoutManager API可以让你独立做很多完美的事情，所以可能的组合是多种多样的，一般来说，对你想要临时改编并期望在同一操作中使用的可以直接调用`detachAndScrapView()`，如果你不需要马上复用，则可以调用`removeAndRecyclerView()`
#Building The Core
LayoutManager实例主要负责在需要的时候，实时的添加、测量和铺设所有的子View，当用户在滑动View的时候，它将取决于LayoutManager去决定什么时候，子View需要被添加，什么时候旧的View可以被分离和报废。

>你将需要重写和继承以下的方法，去创建一个最小的可用的LayoutManager.

#generateDefaultLayoutParams()
事实上，这个方法是唯一一个必须被重写的方法。实现非常的简单，只要返回一个RecyclerView.LayoutParams的实例用于设置子View的高度，在返回`getViewForPosition()`之前，这些参数将被赋予每个子View。

    @Override
    public RecyclerView.LayoutParams generateDefaultLayoutParams(){
        return new RecyclerView.LayoutParams(
            RecyclerView.LayoutParams.WRAP_CONTENT,
            RecyclerView.LayoutParams.WRAP_CONTENT);
    }

#onLayoutChildren()
这是LayoutManager的主要入口，当view需要被初始化或者Adapter的数据有变化时会调用该方法。不管是初始化还是数据源变化，对于重置子view，这是个好时机。

在下段，我们将学习当Adapter更新时，这个方法如何实现基于当前显示的元素布局。现在，我们简单的认为它是实现初始化子布局。底下是一个简单的例子，在`FixedGridLayoutManager`示例：

    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        //报废的view，用于计算宽高
        View scrap = recycler.getViewForPosition(0);
        addView(scrap);
        measureChildWithMargins(scrap, 0, 0);

        /*
         * 在这里我们假设每个Item的高度是一致的
         * 这使我们可以计算Item的大小，因为Item的大小是不变的
         */
        mDecoratedChildWidth = getDecoratedMeasuredWidth(scrap);
        mDecoratedChildHeight = getDecoratedMeasuredHeight(scrap);
        detachAndScrapView(scrap, recycler);
    
        updateWindowSizing();
        int childLeft;
        int childTop;

        /*
         * 重置第一个item的位置
         */
        mFirstVisiblePosition = 0;
        childLeft = childTop = 0;
    
        //Clear all attached views into the recycle bin
        detachAndScrapAttachedViews(recycler);
        //Fill the grid for the initial layout of views
        fillGrid(DIRECTION_NONE, childLeft, childTop, recycler);
    }

我们设定并保证所有view的退出都保持到废料堆里，我们抽象了一部分工作到`fillGrid()`以便于复用，不久之后我们将看到这个方法中执行了一大堆的更新view是否显示当滑动发生时。

>就像是对ViewGroup的自定义实例化，你的任务就是触发测量布局每个从Recycler中获取到的子view。这些工作Api是无法直接实现的。

总体来说，初步完成这个方法需要执行的步骤如下：
- 当最后一次滑动时间发生时，检查所有附加view当前位置偏移；
- 判断滑动是否导致空白的空间产生，是否需要添加新的view，如果有，从Recycler中获取；
- 判断是否有不再需要显示的view，如果有，移除并将他们返回Recycler；
- 检查是否有需要呗重新绘制的view，可能是上层请求移动item的位置。

注意我们在讨论的`FixedGridLayoutManager.fillGrid()`。这个Manager调整从右到左的位置，当到达最大列时，包裹他们：
- 取到我们拥有的所有view的清单，分离他们以便于我们可以重新添加。

    SparseArray<View> viewCache = new SparseArray<View>(getChildCount());
    //...
    if (getChildCount() != 0) {
        //...
        //Cache all views by their existing position, before     updating counts
        for (int i=0; i < getChildCount(); i++) {
            int position = positionOfIndex(i);
            final View child = getChildAt(i);
            viewCache.put(position, child);
        }

        //Temporarily detach all views.
        // Views we still need will be added back at the  proper index.
        for (int i=0; i < viewCache.size(); i++) {
            detachView(viewCache.valueAt(i));
        }
    }


by goyourfly
****
**未完待续...**



