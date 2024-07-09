---
title: Start一个Activity
published: 2016-05-17
tags: [Android,Activity]
category: 技术
draft: false
---


#### Context，Activity继承关系

首先，Activity是继承自ContextThemeWrapper，而ContextThemeWrapper继承自ContextWrapper，而ContextWrapper继承自Context，具体关系如下图：

![Activity关系图](../_res/Activity关系图.png)

Context 是一abstract类，基本没有实现，而真正的实现从名字就能看出来：
`ContextImpl`。ContextWrapper也是继承自Context，但是它同时持有一个Context对象，这个对象是通过attachBaseContext(base:Context)这个方法传入ContextImpl，ContextWrapper中对Context的实现都是用如下这种方法实现：

````java
 	/**
     * @return the base context as set by the constructor or setBaseContext
     */
    public Context getBaseContext() {
        return mBase;
    }

    @Override
    public AssetManager getAssets() {
        return mBase.getAssets();
    }

    @Override
    public Resources getResources() {
        return mBase.getResources();
    }

    @Override
    public PackageManager getPackageManager() {
        return mBase.getPackageManager();
    }
    //... 省略号下面还有一万个这样的方法
````
也就是说，其实ContextWrapper虽然继承了Context，但是它只是一个代理类，实现都是通过ContextImpl。

>启动默认Activity和通过Context.startActivity()在Android中是不一致的

#### LaunchActivity();


#### Activity.startActivity(...)

````java
	// 一号startActivity
    public void startActivity(Intent intent) {
    	//调用 二号startActivity
        this.startActivity(intent, null);
    }
    
    // 二号startActivity
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
    
    // 三号startActivity
	public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }
    
    // 四号startActivity，这个是最终的Boss
    public void startActivityForResult(Intent intent, int requestCode,Bundle options) {
        // mParent是Activity类型，通过attach(...)方法初始化，而attact方法在哪里调用呢？
        // TODO
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
			// 如果说没有Parent，则调用Instrumentation.execStartActivity
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
        	// 如果有mParent，则调用Parent.startActivityFromChild(...)
        	// Parent也是Activity，所以看看startActivityFromChild(...)这个方法
            if (options != null) {
            		// 其中，child是this，当前类
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
    
    // 五号方法，有父Activity时候，startActivity会触发这个方法来启动Activity
    // 跟上面那个方法差不多，只是多了一个child Activity
    public void startActivityFromChild(@NonNull Activity child, @RequiresPermission Intent intent,
            int requestCode, @Nullable Bundle options) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, child,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, child.mEmbeddedID, requestCode,
                ar.getResultCode(), ar.getResultData());
        }
        cancelInputsAndStartExitTransition(options);
    }
````

在调用startActivity后，最终都会触发下面这段代码，只不过child可能换成this：

````java
// 其中的child，如果startActivity是由Activit发起，
// 则child为目标Activity，否则为自身
 Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, child,
                intent, requestCode, options);
````

#### Instrumentation.execStartActivity(...)
接下来进入这个方法看一看

````java
    public ActivityResult execStartActivity(
            Context who,//发起者，既谁调用的startActivity
             IBinder contextThread, //TODO 认识ApplicationThread
             IBinder token, 
             Activity target,// 要启动的目标Activity
            Intent intent, // 用户传入的Intent
            int requestCode, 
            Bundle options, 
            UserHandle user) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            // 看这里，最终跳转到ActivityManagerNative.getDefault()
            int result = ActivityManagerNative.getDefault()
                .startActivityAsUser(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options, user.getIdentifier());
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
````

然后我们在ActivityManagerNative找到该方法

````java
    public int startActivityAsUser(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options,
            int userId) throws RemoteException {
        ...
        // 最后的最后，发现它居然发了一个Binder消息，然后就不管了，那是谁接收和处理这个消息呢？
        mRemote.transact(START_ACTIVITY_AS_USER_TRANSACTION, data, reply, 0);
        ...
    }
````

#### ActivityManagerService
Dengdengdengdeng，当然是我们牛逼的ActivityManagerService来接收这个消息了，

