---
title: "Flutter 局部 build"
published: 2019-07-23
tags: [Flutter]
category: 技术
draft: false
---

在 Flutter 中用到最多的两个 Widget 就是 StatefulWidget 和 StatelessWidget，他们分别对应两个概念：有状态和无状态

有状态的意思是：它的父 Widget 刷新时，它不会被动的刷新，而它主动刷新也不会影响父 Widget
无状态的意思是：它的父 Widget 刷新是，它也会被动的刷新

比如 Text 就是一个无状态的 Widget，每次文字变化，你只需调用 `setState`，它就会刷新
而比如 Dialog 就是一类有状态的 Widget，如果在 AlertDialog 的外部调用 `setState`，它不会刷新。

首先，为什么不都用无状态的 Widget？
其实原因很简单，如果都是无状态的，那么每次刷新都从 Widge它树的根节点开始渲染太耗费性能，也没有必要

那怎么实现局部刷新的呢？
首先，我们看它的设计：

StatelessWidget 设计
````
class StatelessWidget extends Widget{
  @override
  StatelessElement createElement() => StatelessElement(this);
  @protected
  Widget build(BuildContext context);
}
````

StatefulWidget 设计
````
class StatefulWidget extends Widget{
  @override
  StatefulElement createElement() => StatefulElement(this);
  @protected
  State createState();
}

abstract class State<T extends StatefulWidget> extends Diagnosticable {
  void initState() {
    ...
  }
  void setState(VoidCallback fn) {
  ...
  }
  Widget build(BuildContext context);
  void dispose() {
   ...
  }
}
````
> 上面所说的有状态的其实是指这个 State 在父 Widget 刷新时没有被销毁重建，而 StatefulWidget 本身还是重新创建了。

从上面可以看到 StatelessWidget 的 build 方法在 Widget 中，而 StatefulWidget 则是剥离到 State 对象里，而 State 对象又由 StatefulElement 持有，由于之前讲的 Widget 的不可变性导致：StatelessWidget 和 StatefulWidget 的实例在每次父 Widget 刷新时都会销毁重建，所以 StatelessWidget.build 会及时刷新，而由于 StatefulWidget 的 build 方法被剥离到 State 对象中，而 State 对象又是由 StatefulElement 持有，所以在父 Widget 刷新时，State 对象并没有重建，State.build 方法并没有变化。

看到很多人文为什么不把 build 方法放在 StatefulWidget 中，非得创建一个 State 对象，写起来很费劲，这就是为什么。

那么，Flutter 如何实现局部刷新？
首先，当你在一个 StatefulWidget 中调用 setState，发生了下面这些事情：

````
  void setState(VoidCallback fn) {
    _element.markNeedsBuild();
  }
````
它会调用当前  StatefulElement.markNeedsBuild()`，`markNeedsBuild` 是由其父类 `markNeedsBuild` 实现的：
````
  void markNeedsBuild() {
    if (!_active)
      return;
    if (dirty)
      return;
    _dirty = true;
    owner.scheduleBuildFor(this);
  }
````
1. 把自己标记为 `_dirty`
2. 调用 BuildOwner.scheduleBuildFor(Element)
> BuildOwner 全局只有一个

那么 scheduleBuildFor 又做了哪些事情
````
   void scheduleBuildFor(Element element) {
    if (element._inDirtyList) {
      _dirtyElementsNeedsResorting = true;
      return;
    }
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
      _scheduledFlushDirtyElements = true;
      onBuildScheduled();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
  }
````

1. 把 StatefulElement 加入到 BuildOwner 全局变量：_dirtyElements 中
2. 标记需要在下次 VSync 时刷新界面

---------- 中间略去 VSync 信号处理过程 ----------

等下次 VSync 信号过来后，会调用 BuildOwner 的 buildScope 方法：

````
 void buildScope(Element context, [ VoidCallback callback ]) {
    try {
      _scheduledFlushDirtyElements = true;
      if (callback != null) {
        _dirtyElementsNeedsResorting = false;
          callback();
      }
      _dirtyElements.sort(Element._sort);
      _dirtyElementsNeedsResorting = false;
      int dirtyCount = _dirtyElements.length;
      int index = 0;
      while (index < dirtyCount) {
        _dirtyElements[index].rebuild();
        index += 1;
        if (dirtyCount < _dirtyElements.length || _dirtyElementsNeedsResorting) {
          _dirtyElements.sort(Element._sort);
          _dirtyElementsNeedsResorting = false;
          dirtyCount = _dirtyElements.length;
          while (index > 0 && _dirtyElements[index - 1].dirty) {
            // It is possible for previously dirty but inactive widgets to move right in the list.
            // We therefore have to move the index left in the list to account for this.
            // We don't know how many could have moved. However, we do know that the only possible
            // change to the list is that nodes that were previously to the left of the index have
            // now moved to be to the right of the right-most cleaned node, and we do know that
            // all the clean nodes were to the left of the index. So we move the index left
            // until just after the right-most clean node.
            index -= 1;
          }
        }
      }
    } finally {
      for (Element element in _dirtyElements) {
        assert(element._inDirtyList);
        element._inDirtyList = false;
      }
      _dirtyElements.clear();
      _scheduledFlushDirtyElements = false;
      _dirtyElementsNeedsResorting = null;
    }
  }
````

这个方法主要做的事情：
1. 遍历 _dirtyElements，调用每个 Element.rebuild()
2. rebuild 完成后，将这些 Element 的 _inDirtyList 标记为 false（此刻 Element 由于还没有执行 layout 和 paint，所以仍然是脏的，但是已经执行了 build，所以为了避免在本次 VSync 再被加入到 _dirtyElements 中，所以用此标识符标记）
3. 最后清空 _dirtyElements

致此，我们就知道了如何实现局部 build

