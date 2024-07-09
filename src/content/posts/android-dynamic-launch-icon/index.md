---
title: "Android 动态切换应用图标"
published: 2019-07-24
tags: [Android]
category: 技术
draft: false
---

- Android 动态切换应用图标
- 如支付宝和淘宝经常在节日：比如 618/双11 时修改应用图标
- ````
        <activity-alias
            android:name="BicycleActivity"
            android:icon="@mipmap/ic_launcher_2"
            android:enabled="false"
            android:targetActivity=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity-alias>
    ````
- 先在 manifest 中创建一个 `activity-alias`，将 `targetActivity` 设置为启动页
- 另外记得将 `enabled` 设置为 `false`，否则将会创建两个图标
- 然后在代码中：
- ````
        val packageManager = packageManager
        packageManager.setComponentEnabledSetting(
            ComponentName(this, "$packageName.BicycleActivity"),
            PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
            PackageManager.DONT_KILL_APP
        )
        packageManager.setComponentEnabledSetting(
            ComponentName(this, "$packageName.MainActivity"),
            PackageManager.COMPONENT_ENABLED_STATE_ENABLED,
            PackageManager.DONT_KILL_APP
        )
    ````
- 其中第三个参数如果是 `PackageManager.DONT_KILL_APP` ，则会在应用退出后修改应用图标，如果设置为 `0` 则会马上杀死当前应用
- 已知问题，如果修改了图标，在 Android Studio 安装的时候提示：
- ![image](./img.png)
- 1. adb 启动应用时找不到 MainActivity，因为启动页已经被改成 BicycleActivity，不影响实际用户使用
- 2. 上面的那个参数传成 `0` 后，我在模拟器又一次发现图标都没有了，导致用户找到不应用图标了，可能是 Launcher 的 bug