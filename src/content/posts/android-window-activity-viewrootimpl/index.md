---
title: "Window，Activity，DecorView，ViewRootImpl 的关系"
published: 2020-09-17
tags: [Android]
category: 技术
---

- 捋一下 Window，Activity，DecorView，ViewRootImpl 的关系
- 1.Activity 创建了 PhoneWindow，在 Activity.attach 中
- 2.PhoneWindow 创建了 DecorView
- 3.WindowManager.addView(decorView) 创建了 ViewRootImpl 然后马上调用 ViewRootImpl.setView 将 decorView 绑定到 ViewRootImpl
- 4.Window 不是一个 View，是界面相关数据的集合
- 5.ViewRootImpl 虽然继承了 ViewParent，但它不是一个 View，但是持有 decorView，Activity 上所有 View 的祖宗，继承 ViewParent 应该是为了实现一些 View 相关的接口
- 6.屏幕刷新，窗口尺寸变化，都是从 ViewRootImpl 发生，然后通过 decorView 层层往下传
- 7.触摸事件传递很绕，先到 ViewRootImpl，然后传递给 DecorView，DecorView 传递给 Activity，Activity 再传给 Window，Window 传递回 DecorView，DecorView 再往子 View 传递下去
- `ViewRootImpl -> DecorView -> Activity -> Window -> DecorView`
- 8.上面说的都是 Activity 这套逻辑，自定义 Window 不适用