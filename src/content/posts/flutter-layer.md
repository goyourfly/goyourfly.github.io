---
title: "Flutter Layer"
published: 2019-07-25
tags: [Flutter]
category: 技术
draft: false
---

在 Flutter 渲染的过程中，有一步是 `paint`，实现将 `RenderObject` 绘制到画布上，那画布是谁呢？

随便找一个 `RenderObject` 实现类，在 `paint` 方法中我们发现是用 `PaintContext.canvas` 进行绘制和处理
````
 @override
  void paint(PaintingContext context, Offset offset) {
      ...
      context.canvas.save();
      ...
      context.canvas.drawRect(Offset.zero & size, paint);
      context.canvas.restore();
    }
  }

````
`PaintContext` 是什么？它怎么会持有 `canvas`，最终又画到哪里了呢？
带着这些问题，我们接着往下看

首先打开 rendering/object.dart 这个文件，找到 `PaintContext` 的所在地
构造方法：
````
PaintingContext(this._containerLayer, this.estimatedBounds)
````
我们看到它包含一个 `_containerLayer` 和一个 `estimatedBounds`，一个听起来像图层（就像是 PS 中的图层），一个像是边界约束，先不深究

上面的 `paint` 方法用到了 `PaintingContext.canvas`，马上去找出这个熟悉的又陌生的 `canvas`，遗憾的是 `canvas` 不在 `PaintingContext` 类，继续向上寻找：`ClipContext`
果然在这里找到了：
````
Canvas get canvas;
````
虽然在这里声明的，但初始化却不在这里，唉，还是回头找吧，在 `PaintingContext` 类中，我们发现了它的创造地：
````
  @override
  Canvas get canvas {
    if (_canvas == null)
      _startRecording();
    return _canvas;
  }

  void _startRecording() {
    assert(!_isRecording);
    // 在这里，先根据边界创建一个 PictureLayer
    _currentLayer = PictureLayer(estimatedBounds);
    // 创建一个叫 PictureRecorder 的东西，现在还不知道是什么
    _recorder = ui.PictureRecorder();
    // 在这里创建了 Canvas
    _canvas = Canvas(_recorder);
    // 最后我们再把 PictureLayer 添加到构造函数传进来的 _currentLayer 中，作为它的子 layer
    _containerLayer.append(_currentLayer);
  }
````

> 记得在 Android 中创建 `Canvas` 传入的是 `Bitmap`，那这里的 PictureRecorder 应该也有类似的功能吧

先看一下 `PictureRecorder` 这个东东
````
class PictureRecorder extends NativeFieldWrapperClass2 {
  @pragma('vm:entry-point')
  PictureRecorder() { _constructor(); }
  void _constructor() native 'PictureRecorder_constructor';

  bool get isRecording native 'PictureRecorder_isRecording';

  Picture endRecording() native 'PictureRecorder_endRecording'; // 注意这里返回一个 Picture 对象
}
````
这个类好生简单，就三个方法，并且都是 native 方法，俺看不懂也懒得看对应的 native 方法，所以猜一下：
- 可能1：PictureRecorder 应该是开辟了一块内存区域，将来的 Canvas 绘制将会在这块内存中
- 可能2：PictureRecorder 对应 Skia 引擎的一个处理图像的类，Canvas 的操作映射到 Skia

接下来我们慢慢通过观察其他类的表现，从侧面证明这种猜想。

先看一下 `PictureRecorder.endRecording` 方法返回的对象 `Picture`：
````
class Picture extends NativeFieldWrapperClass2 {
  Future<Image> toImage(int width, int height) {
    return _futurize(
      (_Callback<Image> callback) => _toImage(width, height, callback)
    );
  }
  String _toImage(int width, int height, _Callback<Image> callback) native 'Picture_toImage';
  ...
}
````
注释有一句：An object representing a sequence of recorded graphical operations.
意思是：记录一些图形操作的类，现在对他的了解就是酱紫

再看一下 `Canvas`，`Canvas` 基本也是一个 native 的代理类：

````
  void _drawLine(double x1,
                 double y1,
                 double x2,
                 double y2,
                 List<dynamic> paintObjects,
                 ByteData paintData) native 'Canvas_drawLine';
  void _drawRect(double left,
                 double top,
                 double right,
                 double bottom,
                 List<dynamic> paintObjects,
                 ByteData paintData) native 'Canvas_drawRect';
  void _drawCircle(double x,
                   double y,
                   double radius,
                   List<dynamic> paintObjects,
                   ByteData paintData) native 'Canvas_drawCircle';
````
想象也可以理解，如果直接用 Dart 画几何图形，那就不会用 Skia 二维图形引擎了

OK，基本元素了解的差不多了，捋一下他们之间的关系：

- PictureLayer 是图层，这里就当它是张桌子吧
- Canvas 是画笔
- PictureRecorder 就像是画纸
- Picture 是画，它是 canvas 在 PictureRecorder 上的记录
- PaintingContext 则像是一个画室，把这些东西放在一起

那在 Flutter 中具体怎么一个流程呢，还是以 `Text` 为例：
1. VSync 信号来的时候执行了 `drawFrame` 方法其中触发了 `pipelineOwner.flushPaint()` 和 `renderView.compositeFrame()` 方法
2. 先看 `flushPaint()` 方法，所有标记（markNeedsPaint）需要绘制的 RenderObject 都进行如下判断：
````
  void flushPaint() {
    try {
      final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
      _nodesNeedingPaint = <RenderObject>[];
      // Sort the dirty nodes in reverse order (deepest first).
      for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
        // 这里就是标记为需要重绘
        if (node._needsPaint && node.owner == this) {
          // 这里又是什么意思？
          if (node._layer.attached) {
            PaintingContext.repaintCompositedChild(node);
          } else {
            node._skippedPaintingOnLayer();
          }
        }
      }
    } 
  }
````
attached 的判断条件是：
````
 bool get attached => _owner != null;
````
也就是这个 Layer 和 `PipelineOwner` 绑定了的意思，表示该 Layer 参与构建
如果条件成立，则执行 `PaintingContext.repaintCompositedChild(node);`

然后一系列调用到
````
child._paintWithContext(childContext, Offset.zero);
PaintingContext.stopRecordingIfNeeded();
````
其中 `_paintWithContext` 会调用 `RenderObject.paint` 方法，在 `paint` 方法中利用 `canvas `开始绘制自己内容，这里由于文字比较特殊，使用了 `TextPainter` 进行绘制的，此时如果发现 `canvas` 是空的，则构建一个新的 `canvas` 和对应的 `PictureLayer` 和 `PictureRecorder`，`canvas` 所有绘制的内容都被保存到 `PictureRecorder` 中

> 注意：此时 `PictureLayer` 和 `PictureRecorder` 还没有关联

`_paintWithContext` 完成后马上调用了 `stopRecordingIfNeeded`：
````
  void stopRecordingIfNeeded() {
    if (!_isRecording)
      return;
    _currentLayer.picture = _recorder.endRecording(); // 将 Picture 赋值给 _currentLayer
    _currentLayer = null;
    _recorder = null;
    _canvas = null;
  }
````
这个方法主要作用是：停止绘制，获取当前画布（`PictureRecorder`）中的内容 `Picture` 赋值给 `_currentLayer`，其他参数清零，其中最重要的是把 `Picture` 赋值给 `_currentLayer`，而 `_currentLayer` 就是 `PictureLayer`

此时  `pipelineOwner.flushPaint()` 的工作总算完成了，马上执行的是 `renderView.compositeFrame()`：

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
其中主要方法在 `layer.buildScene(builder);` 中：
````
  ui.Scene buildScene(ui.SceneBuilder builder) {
    List<PictureLayer> temporaryLayers;
    updateSubtreeNeedsAddToScene();
    // 看这里
    addToScene(builder);
    final ui.Scene scene = builder.build();
      return true;
    }());
    return scene;
  }

  @override
  ui.EngineLayer addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
    final ui.EngineLayer engineLayer = builder.pushOffset(layerOffset.dx + offset.dx, layerOffset.dy + offset.dy);
    addChildrenToScene(builder);
    builder.pop();
    return engineLayer;
  }
  // PictureLayer.addToSence
  @override
  ui.EngineLayer addToScene(ui.SceneBuilder builder, [ Offset layerOffset = Offset.zero ]) {
   // 看这里看这里，终于把 picture 添加到 builder 中了
    builder.addPicture(layerOffset, picture, isComplexHint: isComplexHint, willChangeHint: willChangeHint);
    return null; // this does not return an engine layer yet.
  }
````
最后 ` _window.render(scene);` 将 `scene` 发送给 native 进行显示

### Over