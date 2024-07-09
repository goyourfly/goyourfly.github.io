---
title: "Flutter UI 绘制原理引导"
published: 2019-07-18
tags: [Flutter]
category: 技术
draft: false
---

> Flutter UI Framework 的核心是三棵树：
>
> 1. `Widget` 树：面向调用者
> 2. `Element` 树：中间处理件
> 3. `RenderObject` 树：面向布局和渲染
>
> 其中 `Widget` 被设计为不可变的，每次 UI 刷新都会重建，既无状态的    
> `RenderObject` 正如其名，负责 layout 和 render   
> `Element` 则是其中的抽象层，它像胶水一样粘合了 `Widget` 和    `RenderObject`，又像管家一样，接受一些外来的事件，分配一些工作。

> `Element` 由 `Widget.createElement` 创建    
> `RenderObject` 由 `Widget.createRenderObject` 创建    
> `Widget` 则由调用者创建     
> 这种设计简化了调用者使用逻辑，但是同时隐藏了一些实现细节，有利有弊把。

> 文中的 Native 指的是 Android 或 iOS  
> Flutter 版本：1.7.0    
> 阅读本文时一定要结合源码，源码在 packages/flutter/lib/src 文件夹下   [源码地址](https://github.com/flutter/flutter)    
> 这篇文章是我一边阅读源码一边写的，文中可能有些错误，甚至是原理性的错误，所以要抱着怀疑的态度阅读本文，觉得不合理的地方欢迎留言讨论，共同进步

### Text 的三棵树

很多介绍 Flutter 框架的文章都是从整体结构开始讲解，这种方式虽然条条框框较清晰，但是对于很多人来说有点老虎吃天---无从下口，所以我想从一个小的入口开始，看看能不能撕开一个口子。

从最常用的 `Text` 开始吧，通过源码我们看到 `Text` 继承自 `StatelessWidget`，而 `StatelessWidget` 则是直接继承 `Widget` 组件，按照上面说的，`Text` 应该有一个对应的 `Element` 和 `RenderObject`，然而找了好久只能找到 `StatelessElement`，并没有找到对应的 `RenderObject`，没有 `RenderObject`？那它怎么实现布局和渲染呢？
仔细看一下 `Text` 的源码你会发现，由于 `Text` 继承了 `StatelessWidget`，所以它有一个 `build` 方法：

````
  @override
  Widget build(BuildContext context) {
    ...
    Widget result = RichText(
      textAlign: textAlign ?? defaultTextStyle.textAlign ?? TextAlign.start,
      textDirection: textDirection, // RichText uses Directionality.of to obtain a default if this is null.
      locale: locale, // RichText uses Localizations.localeOf to obtain a default if this is null
      softWrap: softWrap ?? defaultTextStyle.softWrap,
      overflow: overflow ?? defaultTextStyle.overflow,
      textScaleFactor: textScaleFactor ?? MediaQuery.textScaleFactorOf(context),
      maxLines: maxLines ?? defaultTextStyle.maxLines,
      strutStyle: strutStyle,
      textWidthBasis: textWidthBasis ?? defaultTextStyle.textWidthBasis,
      text: TextSpan(
        style: effectiveTextStyle,
        text: data,
        children: textSpan != null ? <TextSpan>[textSpan] : null,
      ),
    );
    ...
    return result;
  }
````
观察 `build` 方法可以看到 `Text` 的实际绘制类是 `RichText`，也就是说 `Text` 只是一个容器，而实际显示的是 `RichText`，它的继承关系是：
`RichText -> MultiChildRenderObjectWidget -> RenderObjectWidget -> Widget `
> 低版本的可能继承 `LeafRenderObjectWidget`，不过最终都继承了 RenderObjectWidget

而其中的 `RenderObjectWidget` 中声明了   `RenderObject createRenderObject(BuildContext context);`  方法，

````
  @override
  RenderObjectElement createElement();
  @protected
  RenderObject createRenderObject(BuildContext context);
````
在 `RichText` 中实际实现的 `RenderObject` 是 `RenderParagraph`，`RenderParagraph` 在 render 包下，它继承了 `RenderBox -> RenderObject`
真正实现布局和渲染的逻辑分别在 `RenderParagraph` 的 `performLayout()` 和 `paint()` 方法中
现在整理一下：

|  Widget  |  Element  |  RenderObject  | 
| :-: | :-: | :-: | 
|  RichText  |  MultiChildRenderObjectElement  |  RenderParagraph  |

### 谁创建这三棵树
那么，是谁调用了 `createElement` 和 `createRenderObject` 从而创建了 `Element` 和 `RenderObject` 呢？
我们看一下 `MultiChildRenderObjectElement` 类，它有一个 `mount` 方法（每个 `Element` 都有一个 `mount` 方法）：

````
  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _children = List<Element>(widget.children.length);
    Element previousChild;
    for (int i = 0; i < _children.length; i += 1) {
      // 调用当前 Element 的 inflateWidget 方法
      final Element newChild = inflateWidget(widget.children[i], previousChild);
      _children[i] = newChild;
      previousChild = newChild;
    }
  }
  // super.mount
  // 父类的 mount 方法创建 _renderObject 并插入对应的插槽
  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _renderObject = widget.createRenderObject(this);
    assert(() { _debugUpdateRenderObjectOwner(); return true; }());
    assert(_slot == newSlot);
    attachRenderObject(newSlot);
    _dirty = false;
  }

  @protected
  Element inflateWidget(Widget newWidget, dynamic newSlot) {
    assert(newWidget != null);
    final Key key = newWidget.key;
    if (key is GlobalKey) {
      final Element newChild = _retakeInactiveElement(key, newWidget);
      if (newChild != null) {
        assert(newChild._parent == null);
        assert(() { _debugCheckForCycles(newChild); return true; }());
        newChild._activateWithParent(this, newSlot);
        final Element updatedChild = updateChild(newChild, newWidget, newSlot);
        assert(newChild == updatedChild);
        return updatedChild;
      }
    }
   // 调用子 Widget 的 createElement
    final Element newChild = newWidget.createElement();
    assert(() { _debugCheckForCycles(newChild); return true; }());
   // 调用子 Widget 的 mount
    newChild.mount(this, newSlot);
    assert(newChild._debugLifecycleState == _ElementLifecycle.active);
    return newChild;
  }
````
所以构建  `Element` `RenderObject` 这两棵树就在这里，通过层层传递的方式创建到树的末梢（看起来很像递归）：
`widget.mount -> widget.inflateWidget -> child.mount -> child. inflateWidget -> subchild.mount -> subchild. inflateWidget ...`

### 谁打响了第一枪
那么是谁第一个触发了这个调用链呢？
这得从 Flutter 程序的入口 `main` 方法说起，在 `main` 方法调用了 `runApp` 这个全局方法，该方法在 widgets/binding.dart 文件中，
````
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}
````
在这个方法中构建了一个 `WidgetsFlutterBinding` 单例并调用 `attachRootWidget(widget)`，将第一个 widget 传进去。
`WidgetsFlutterBinding` 类本身只实现类单例并暴露了一个接口 `WidgetsBinding`，但是这个类同时利用 mixin 机制实现了继承了一堆类：`BindingBase，GestureBinding，ServicesBinding，SchedulerBinding，PaintingBinding，SemanticsBinding，RendererBinding，WidgetsBinding`
> 根据 mixin 机制，权重最高的方法是在 extends 类中，如果 extends 类没有，则优先级最后的最高，最前的最低。
根据优先级，我们在 `WidgetsBinding` 中找到了 `attachRootWidget`，它首先创建一个 `RenderObjectToWidgetAdapter` 类并附带参数：`renderView，rootWidget`，其中 `renderView` 现在还不知道是什么，猜测是根容器。
然后马上调用 `attachToRenderTree(buildOwner, renderViewElement)`
````
  void attachRootWidget(Widget rootWidget) {
    _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget,
    ).attachToRenderTree(buildOwner, renderViewElement);
  }
````
`RenderObjectToWidgetAdapter` 本身也是一个 `Widget`，继承自 `RenderObjectWidget`，看看它的 `attacnToRenderTree`
````
  RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [ RenderObjectToWidgetElement<T> element ]) {
    if (element == null) {
      owner.lockState(() {
      // 如果上面传入的 element 是空的，则调用 RenderObjectToWidgetAdapter.createElement，并调用 element.mount()，终于看到了，第一个 mount 的地方，泪崩
        element = createElement();
        assert(element != null);
        element.assignOwner(owner);
      });
      owner.buildScope(element, () {
        element.mount(null, null);
      });
    } else {
     // 如果 element 不为 null，证明当前不是第一次创建，执行 element.markNeedsBuild() 方法，这个方法只是简单的把当前 element 标记为 dirty，通知 BuildOwner 我已经脏了，需要刷新，等待下次 VSync 信号时刷新即可。
      element._newWidget = this;
      element.markNeedsBuild();
    }
    return element;
  }
````

### setState
我们在修改 UI 后，一般都会调用 `setState` 方法通知 UI 数据有变化，然后 `build` 方法被执行，这是怎么一个过程？
`setState` 在 `State`（widget/framework.dart） 类下：

````
 @protected
  void setState(VoidCallback fn) {
	// 删掉一堆 assert，实际上只调用了 _element.markNeedsBuild()
    _element.markNeedsBuild();
  }

````
此处有三个问题：
1. `Element` 是谁？
2. 我们在实现 `StatefullWidget` 时，总是要重写 `createState` 方法，把自己的 `State` 传递给回去，那么这个 `createState` 在哪里调用呢？
3. markNeedsBuild 做了什么？

首先，`StatefulWidget` 对应的 `Element` 是 `StatefulElement`

````
  @override
  StatefulElement createElement() => StatefulElement(this);
````
我们猜测应该是在这个类里调用了 `createState` 方法，找呀找，找了一会果然找到了：

````
  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
        _state._element = this;
        _state._widget = widget;
  }
````
在它的构造函数中马上就调用了 `createState`，为什么不让用户直接通过构造函数传进去呢？可能是设计者觉得这样更优雅吧，我们看到构造函数里还将 `Element` 和 `Widget` 之间赋值给 `State` 对象，所以上面那两个问题找到答案了：`Element` 就是当前页面对应的 `StatefulElement`，`createState` 是在 `StatefulElement ` 的构造函数调用的。

现在再去找第三个问题 `markNeedsBuild 干了什么？` 的答案：
通过上面我们知道了调用的是 `StatefulElement.markNeedsBuild`，但是 `StatefulElement` 没有重写这个方法，所以只能去它的父类 `ComponentElement` 或 `Element` 中寻找，最后我们在 `Element` 类中找到了这个方法：
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
同样删掉一堆 assert 之后我们发现，它只是简单的将 `Element` 的全局变量 `_dirty` 标记为 true，然后又调了一下 `owner.scheduleBuildFor(this);`
dirty 和容易理解，标记这个 Widget 有变化，需要重绘，那 `owner` 是什么？
`owner` 全名叫 `BuildOwner`（在 widgets/framework.dart 下）

官方文档给的定义是：Manager class for the widgets framework.
这个类会记录需要 rebuilding 的 widgets，处理其他交付给 widget 树的任务统一管理，例如在 debug 时将所有 inactive element 记录在一个列表并在需要的时候触发 reassemble ，主 `BuildOwner` 被 `WidgetsBinding` （前面简单提到过）持有，build/layout/paint 都会用到。
`BuildOwner` 中有一个全局变量是：`_dirtyElements`，看名字大概能猜出它的用途，那我们看一下 `scheduleBuildFor`：

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

其实主要是把传进来的 `element` 塞进那个 dirty 列表中去，然后调用 `onBuildScheduled`，`onBuildScheduled` 是从构造函数传进来的一个方法，既然文档说它由 `WidgetsBinding`（widgets/binding.dart） 持有，那肯定是从 `WidgetsBinding` 传进来的，去看一下：

````
  void initInstances() {
    super.initInstances();
    _instance = this;
    buildOwner.onBuildScheduled = _handleBuildScheduled;
    ...
  }
  
  void _handleBuildScheduled() {
    ensureVisualUpdate();
  }
````
额，最后调到了 `ensureVisualUpdate`，这个类的具体实现是在 `SchedulerBinding`，但是由于 `WidgetsBinding ` mixin 继承了 `SchedulerBinding` 所以相当于有这个方法了：

````
  void ensureVisualUpdate() {
    switch (schedulerPhase) {
      case SchedulerPhase.idle:
      case SchedulerPhase.postFrameCallbacks:
        scheduleFrame();
        return;
      case SchedulerPhase.transientCallbacks:
      case SchedulerPhase.midFrameMicrotasks:
      case SchedulerPhase.persistentCallbacks:
        return;
    }
  }
````
其实这个方法就是确保页面会被刷新，怎么确保呢？
让 VSync 信号能及时的回调过来

那 `build` 回调在哪里触发的呢？答案是在 BuildOwner.buildScope 方法中，完整的调用链是这样的：

`BuildOwner.buildScope -> _dirtyElements.rebuild() -> ComponentElement.performRebuild() -> StatefullWidget.build() -> State.build(StatefullWidget.build)`

那么 `buildScope` 又是在哪里触发的呢？
在 `Element.markNeedsBuild` 的注释中我们看到这样一段话：

- Marks the element as dirty and adds it to the global list of widgets to
- rebuild in the next frame.
- Since it is inefficient to build an element twice in one frame,
- applications and widgets should be structured so as to only mark
- widgets dirty during event handlers before the frame begins, not during
- the build itself.

也就是说会不会马上执行 `build`，只会在下一个 VSync 到来时 `build`
而 VSync 相关的代码在 `SchedulerBinding `（下面会讲到）类中，具体在哪里调用了 `BuildOwner.buildScope`，耐心往下看。


### 假 VSync 信号
最开始 `WidgetsFlutterBinding` 调完 `attachRootWidget` 后，还马上调用了 `scheduleWarmUpFrame`，这个方法在 `SchedulerBinding`（scheduler/binding.dart）下，看一下这个方法的注释：
Schedule a frame to run as soon as possible, rather than waiting for the engine to request a frame in response to a system "Vsync" signal. This is used during application startup so that the first frame (which is likely to be quite expensive) gets a few extra milliseconds to run.
也就是说，这是一个假的 VSync 信号，目的是让第一帧马上显示，不用等下一次 VSync 信号来，相当于预热。

````
  void scheduleWarmUpFrame() {
    Timer.run(() {
      assert(_warmUpFrame);
      handleBeginFrame(null);
    });
    Timer.run(() {
      assert(_warmUpFrame);
      handleDrawFrame();
      resetEpoch();
      _warmUpFrame = false;
      if (hadScheduledFrame)
        scheduleFrame();
    });
    lockEvents(() async {
      await endOfFrame;
      Timeline.finishSync();
    });
  }
````

在这个方法内部调用了 `handleBeginFrame()` 和 `handleDrawFrame()`，而 `handleBeginFrame()` 和 `handleDrawFrame()` 会检查回调列表并调用 `_invokeFrameCallback(callback, _currentFrameTimeStamp) `，`_invokeFrameCallback` 内部直接调用 `callback` 方法，至于 `handleBeginFrame, handleDrawFrame callback` 是什么，在后面讲

### 真 VSync 信号
那真的 VSync 信号什么时候执行呢？
`SchedulerBinding` 初始化时会向 `ui.Window` 注册 `window.onBeginFrame = _handleBeginFrame` 和 `window.onDrawFrame _handleDrawFrame`，而在 VSync 信号来的时候会分别调用这两个方法（native 直接调用），一前一后，具体这俩方法有什么区别呢？看官方文档的意思是：
`_handleBeginFrame` 处理一些临时的任务，比如动画完成后，它的回调应该就结束了，专门又一个 `_removedIds` 负责取消任务
` _handleDrawFrame` 主要处理绘制，它是每次都会触发，伴随着应用的生命周期，还有一类特殊的回调是一次性的，只执行一次，它是在 `handleDrawFrame` 中执行，在绘制 UI 之后执行。

那 `handleDrawFrame` 里的 `callback` 什么时候注册的呢？
全局搜索了以下发现在 `RenderBinding`（rendering/binding.dart）的 `initInstances()` 方法中，注册了一个持久回调 `_handlePersistentFrameCallback`，而这个方法的内部也很是简单，直接调用 `drawFrame()`：

````
  @protected
  void drawFrame() {
    assert(renderView != null);
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // 等上面的布局和渲染完成后，发送给 Native
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
  }
````
但是这里有一个非常绕的写法请注意，最开始我们提到 `WidgetsFlutterBinding` 是一个单例，并且所有以 Binding 为结尾的类好像它都继承了，也就是说这里的 `SchedulerBinding` 实际是 `WidgetsFlutterBinding`，那这里调用 `drawFrame` 会不会调用到其他的 `drawFrame()` 呢？

我们看一下这个 WidgetsFlutterBinding 究竟继承了哪些类：

````
class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
````
> 多继承真的会把人绕晕

继承了这么多，哪些实现了 `drawFrame()` ？
找了一圈我们发现，除了 `RenderBinding`（rendering/binding.dart），还有 `WidgetsBinding`（widgets/binding.dart）。
再根据 mixin 优先级原则，首先触发的应该是 `WidgetsBinding` 的 `drawFrame`：

````
  @override
  void drawFrame() {
    try {
      if (renderViewElement != null)
        buildOwner.buildScope(renderViewElement);
      super.drawFrame();
      buildOwner.finalizeTree();
    } finally {
      assert(() {
        debugBuildingDirtyElements = false;
        return true;
      }());
    }
    if (!kReleaseMode) {
      if (_needToReportFirstFrame && _reportFirstFrame) {
        developer.Timeline.instantSync('Widgets completed first useful frame');
        developer.postEvent('Flutter.FirstFrame', <String, dynamic>{});
        _needToReportFirstFrame = false;
      }
    }
  }
````
果然在这里 `buildOwner.buildScope`，并马上调用 `super.drawFrame()`，这里的 super 就是指 `RenderBinding`


### 布局与渲染

`PipelineOwner`（rendering/object.dart） 又是什么东西？

````
  void flushLayout() {
      while (_nodesNeedingLayout.isNotEmpty) {
        final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
        _nodesNeedingLayout = <RenderObject>[];
        for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
          if (node._needsLayout && node.owner == this)
            // 直接调用了 RenderObject 的 _layoutWithoutResize
            node._layoutWithoutResize();
        }
      }
  }

  void flushPaint() {
      final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
      _nodesNeedingPaint = <RenderObject>[];
      // Sort the dirty nodes in reverse order (deepest first).
      for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
        assert(node._layer != null);
        if (node._needsPaint && node.owner == this) {
          if (node._layer.attached) {
            PaintingContext.repaintCompositedChild(node);
          } else {
            node._skippedPaintingOnLayer();
          }
        }
      }
  }
````
看了一下它的 `flushLayout` 和 `flushPaint` 方法，很像是布局和绘制

前面讲 `RenderBox` 时说它的布局和渲染方法分别是 `performLayout` 和 `paint`，那是谁在调这两个方法呢？
首先，`RenderObject` 虽然是布局和渲染的基类，它没有实现任何布局和渲染的功能，只是声明了对应的接口并维护了几个状态并，甚至没有约定具体使用哪个坐标系，而它的子类 `RenderBox` 规定了使用笛卡尔坐标系，并强制它的子类必须重写 `performLayout()` 进行布局，布局方式和 Android
很像，首先父类给子类限制范围，子类在告诉父类自己使用了具体的位置和宽高。
还是以 `Text` 为例，前面讲它的 `RenderObject` 是 `RenderParagraph`，在它的 `performLayout` 中根据 `constraints` 实现布局，而 `paint` 则是绘制到一个传进来的 `canvas` 上。

捋一下两条调用链：

`PipelineOwner.flushLayout -> RenderObject._layoutWithoutResize -> 具体实现类.performLayout -> child.layout() -> child.performLayout() -> ...`

`PipelineOwner.flushPaint -> PaintingContext.repaintCompositedChild -> RenderObject._paintWithContext -> 具体实现类.paint -> PaintingContext.paintChild -> child._paintWithContext -> child.paint -> ...`

### 提交给 GPU

`RenderView.compositeFrame` 又是什么？
`RenderView`（根容器） 的 `compositeFrame` 方法是构建一个 `ui.Scene`（有点像游戏😄）并调用 `window.render(scene)` 传递给 native，native 再传递给 GPU 显示

````
  void compositeFrame() {
      final ui.SceneBuilder builder = ui.SceneBuilder();
      final ui.Scene scene = layer.buildScene(builder);
      if (automaticSystemUiAdjustment)
        _updateSystemChrome();
      _window.render(scene);
      scene.dispose();
  }
````

### 最后

- Flutter UI 流程：`VSync -> build -> layout -> paint -> compositing -> GPU`
- Flutter 中是根据 VSync 事件驱动刷新 UI 的
- 只有标记为 dirty 的 Element 和它的 child 才参与刷新
- Widget 参与 build 阶段
- RenderObject 参与 layout/paint 阶段
- Flutter 中一切皆 Widget 是因为 Widget 足够抽象

就写这么多吧