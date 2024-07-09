---
title: "WebView 渲染进程崩溃"
published: 2023-01-17
tags: [Android,WebView]
category: 技术
draft: false
---

- `[FATAL:crashpad_client_linux.cc(404)] Render process (28925)'s crash wasn't handled by all associated  webviews, triggering application crash.`
- WebView 在渲染打的网页，或者是因为手机内存不足，可能会导致渲染进程崩溃，然后紧接着就触发应用崩溃
- 那怎么避免应用崩溃呢，其实 Android 提供了一个方法专门处理渲染进程崩溃
- `onRenderProcessGone(WebView view,RenderProcessGoneDetail detail)`
- 这里也有官方文档告诉你怎么做：https://developer.android.com/guide/webapps/managing-webview#termination-handle
- return true 表示客户端会处理，不要崩溃
- 所以在接收到这个会掉以后，要将当前 WebView 销毁，重新创建一个 WebView，重新加载之前的网页
- 也可以先提示用户，就像 Chrome 那样
- ![img.png](img.png)