---
title: "getApplication和getApplicationContext"
published: 2019-04-08
tags: [Android]
category: 技术
---

- getApplication 和 getApplicationContext 返回的是不是同一个对象？
- 通过日志打出来发现，是同一个对象
- ````
  2019-04-08 13:56:20.381 MainActivity: Application:android.app.Application@8559fbb
  2019-04-08 13:56:20.381 MainActivity: ApplicationContext:android.app.Application@8559fbb
- 但是为什么呢？
- getApplication 直接返回 Activity 中持有的一个 mApplication 属性
- getApplicationContext 则是调用 mBase.getApplicationContext
- ````
    @Override
    public Context getApplicationContext() {
        return mBase.getApplicationContext();
    }
- 第一个 mApplication 是通过 Activity 的 attach 方法传入，而调用该方法的地方是在
- ActivityThread.performLaunchActivity 中，而 Application 则是通过 LoadedApk.makeApplication 获取的
- ````
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }
       ...
      // 底下执行创建方法
    }
- 也就是说，这里的 Application 由 LoadedApk 创建的。
- 再看看 getApplicationContext，它的 Application 是 mBase.getApplication 
- 而 mBase 是在 attachBaseContext 传入，而最终也是 performLaunchActivity 中创建
- 最后会创建一个 ContextImpl 实例，它继承 Context，也就是那个 mBase 就是 ContextImpl，看看它的 getApplicationContext
- ````
    @Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
- 而此处 mPackageInfo 就是 `LoadedApk mPackageInfo; ` 上面说的那个 LoadApk
- 所以一般情况下，它们是相同的
- 那为什么还需要 getApplicationContext 呢？
- 首先，getApplication 只存在于 Activity 和 Service 中，而 getApplicationContext 存在于 Context 中
- 比如在广播中想获取 Application，则必须调用 getApplicationContext
- 其次，在特定的情况下，它们俩不是一个对象