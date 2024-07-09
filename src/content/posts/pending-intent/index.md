---
title: "使用 PendingIntent 的过程中遇到的问题"
published: 2019-03-27
tags: [Android]
category: 技术
---

> 需求是这样的，一个 Android 应用要求弹出一个通知，同时通知上还有两个按钮，通知本身点击和按钮点击都对应不同的响应，但是在实现后发现不管点哪个按钮最后都响应同一个事件, 既返回的参数都是同一个。

我的实现方式是这样的，BroadcastReceiver 接收到通知后根据 key 作出不同的响应：

````
val pendingIntent1 = PendingIntent.getBroadcast(context, 0, Intent(BROADCAST_ACTION).apply {
            setClass(context, BroadcastReceiver::class.java)
            putExtra(key, value1)
        }, PendingIntent.FLAG_UPDATE_CURRENT)

val pendingIntent2 = PendingIntent.getBroadcast(context, 0, Intent(BROADCAST_ACTION).apply {
            setClass(context, BroadcastReceiver::class.java)
            putExtra(key, value2)
        }, PendingIntent.FLAG_UPDATE_CURRENT)

val pendingIntent3 = PendingIntent.getBroadcast(context, 0, Intent(BROADCAST_ACTION).apply {
            setClass(context, BroadcastReceiver::class.java)
            putExtra(key, value3)
        }, PendingIntent.FLAG_UPDATE_CURRENT)

````


一开始每个响应PendingIntent都是全局变量，只实例化一次，我怀疑是因为后续更新通知导致，所以改成每次更改通知时都新建 一个 PendingIntent，结果还是不行。

后来我把每个 PendingIntent 打印出来如下：

````
PendingIntent{2fd4457: android.os.BinderProxy@2f2cb44}
PendingIntent{5e7742d: android.os.BinderProxy@2f2cb44}
PendingIntent{c05d62: android.os.BinderProxy@2f2cb44}
````

而PendingIntent对象都toString() 方法是这样的：

````
 @Override
    public String toString() {
        StringBuilder sb = new StringBuilder(128);
        sb.append("PendingIntent{");
        sb.append(Integer.toHexString(System.identityHashCode(this)));
        sb.append(": ");
        sb.append(mTarget != null ? mTarget.asBinder() : null);
        sb.append('}');
        return sb.toString();
    }
````

从上面两张图可以得出结论，虽然 PendingIntent 每次都是一个新的对象，但是他们都指向一个 IIntentSender，呐泥，why？

首先, 我们看看 PendingIntent 是如何创建的？
PendingIntent 提供几个静态方法来实例化自己:
````
PendingIntent.getActivity()
PendingIntent.getBroadcast()
PendingIntent.getService()
...
````

以 getBroadcast() 为例
````
    public static PendingIntent getBroadcast(Context context, int requestCode,
            Intent intent, @Flags int flags) {
        return getBroadcastAsUser(context, requestCode, intent, flags, context.getUser());
    }

    /**
     * @hide
     * Note that UserHandle.CURRENT will be interpreted at the time the
     * broadcast is sent, not when the pending intent is created.
     */
    public static PendingIntent getBroadcastAsUser(Context context, int requestCode,
            Intent intent, int flags, UserHandle userHandle) {
        String packageName = context.getPackageName();
        String resolvedType = intent != null ? intent.resolveTypeIfNeeded(
                context.getContentResolver()) : null;
        try {
            intent.prepareToLeaveProcess(context);
            IIntentSender target =
                ActivityManager.getService().getIntentSender(
                    ActivityManager.INTENT_SENDER_BROADCAST, packageName,
                    null, null, requestCode, new Intent[] { intent },
                    resolvedType != null ? new String[] { resolvedType } : null,
                    flags, null, userHandle.getIdentifier());
            return target != null ? new PendingIntent(target) : null;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
````

而上面打印出来的不同的 PendingIntent 对应同一个 target 就是上面通过 ActivityManager.getService().getIntentSender(...) 创建的，这里 ActivityManager.getService 返回的是 ActivityManagerService，我们去看看这个方法：

````

    @Override
    public IIntentSender getIntentSender(int type,
            String packageName, IBinder token, String resultWho,
            int requestCode, Intent[] intents, String[] resolvedTypes,
            int flags, Bundle bOptions, int userId) {
        //...删掉部分不重要的代码
        synchronized(this) {
            int callingUid = Binder.getCallingUid();
            int origUserId = userId;
            userId = mUserController.handleIncomingUser(Binder.getCallingPid(), callingUid, userId,
                    type == ActivityManager.INTENT_SENDER_BROADCAST,
                    ALLOW_NON_FULL, "getIntentSender", null);
            if (origUserId == UserHandle.USER_CURRENT) {
                // We don't want to evaluate this until the pending intent is
                // actually executed.  However, we do want to always do the
                // security checking for it above.
                userId = UserHandle.USER_CURRENT;
            }
            try {
                if (callingUid != 0 && callingUid != SYSTEM_UID) {
                    final int uid = AppGlobals.getPackageManager().getPackageUid(packageName,
                            MATCH_DEBUG_TRIAGED_MISSING, UserHandle.getUserId(callingUid));
                    if (!UserHandle.isSameApp(callingUid, uid)) {
                        String msg = "Permission Denial: getIntentSender() from pid="
                            + Binder.getCallingPid()
                            + ", uid=" + Binder.getCallingUid()
                            + ", (need uid=" + uid + ")"
                            + " is not allowed to send as package " + packageName;
                        Slog.w(TAG, msg);
                        throw new SecurityException(msg);
                    }
                }
                // ... 最终会调用和返回这个方法, 继续往下看看
                return getIntentSenderLocked(type, packageName, callingUid, userId,
                        token, resultWho, requestCode, intents, resolvedTypes, flags, bOptions);

            } catch (RemoteException e) {
                throw new SecurityException(e);
            }
        }
    }

````

````
    IIntentSender getIntentSenderLocked(int type, String packageName,
            int callingUid, int userId, IBinder token, String resultWho,
            int requestCode, Intent[] intents, String[] resolvedTypes, int flags,
            Bundle bOptions) {
         // 删掉部分不重要的代码
        final boolean noCreate = (flags&PendingIntent.FLAG_NO_CREATE) != 0;
        final boolean cancelCurrent = (flags&PendingIntent.FLAG_CANCEL_CURRENT) != 0;
        final boolean updateCurrent = (flags&PendingIntent.FLAG_UPDATE_CURRENT) != 0;
        flags &= ~(PendingIntent.FLAG_NO_CREATE|PendingIntent.FLAG_CANCEL_CURRENT
                |PendingIntent.FLAG_UPDATE_CURRENT);
        
// 这里根据传经来的一些参数构建来一个 key, 这个里很关键
        PendingIntentRecord.Key key = new PendingIntentRecord.Key(type, packageName, activity,
                resultWho, requestCode, intents, resolvedTypes, flags,
                SafeActivityOptions.fromBundle(bOptions), userId);
        WeakReference<PendingIntentRecord> ref;
        ref = mIntentSenderRecords.get(key);
        PendingIntentRecord rec = ref != null ? ref.get() : null;
        if (rec != null) {
            if (!cancelCurrent) {
                if (updateCurrent) {
                    if (rec.key.requestIntent != null) {
                        rec.key.requestIntent.replaceExtras(intents != null ?
                                intents[intents.length - 1] : null);
                    }
                    if (intents != null) {
                        intents[intents.length-1] = rec.key.requestIntent;
                        rec.key.allIntents = intents;
                        rec.key.allResolvedTypes = resolvedTypes;
                    } else {
                        rec.key.allIntents = null;
                        rec.key.allResolvedTypes = null;
                    }
                }
                return rec;
            }
            makeIntentSenderCanceledLocked(rec);
            mIntentSenderRecords.remove(key);
        }
        if (noCreate) {
            return rec;
        }
        rec = new PendingIntentRecord(this, key, callingUid);
        mIntentSenderRecords.put(key, rec.ref);
        if (type == ActivityManager.INTENT_SENDER_ACTIVITY_RESULT) {
            if (activity.pendingResults == null) {
                activity.pendingResults
                        = new HashSet<WeakReference<PendingIntentRecord>>();
            }
            activity.pendingResults.add(rec.ref);
        }
        return rec;
    }

````

上面的代码中可以看到最终返回的是PendingIntentRecord对象（既target = PendingIntentRecord），它是 IIntentSender.IIntentSender 的子类，调用者传入的参数会构建成一个对象 PendingIntentRecord.Key，而 mIntentSenderRecords 则是一个 HashMap 缓存所有的 PendingIntentRecord。

上面用新建的 key 去 mIntentSenderRecords 查询是否已经存在对应的 PendingIntentRecord，如存在则直接返回，否则新建一个 PendingIntentRecord 塞入 mIntentSenderRecords 并返回。

到这里稍微有一点豁然开朗，但是别急，还有一点点迷雾等待我们去拨开。

既然 HashMap 觉得 PendingIntentRecord 已经存在，那肯定是 PendingIntentRecord.Key 存在，那我们看看这个对象，它是 PendingIntentRecord 的内部类:

````
    final static class Key {
        final int type;
        final String packageName;
        final ActivityRecord activity;
        final String who;
        final int requestCode;
        final Intent requestIntent;
        final String requestResolvedType;
        final SafeActivityOptions options;
        Intent[] allIntents;
        String[] allResolvedTypes;
        final int flags;
        final int hashCode;
        final int userId;

        private static final int ODD_PRIME_NUMBER = 37;

        Key(int _t, String _p, ActivityRecord _a, String _w,
                int _r, Intent[] _i, String[] _it, int _f, SafeActivityOptions _o, int _userId) {
       //  删掉部分无用代码
            requestIntent = _i != null ? _i[_i.length-1] : null; // 最后一个Intent
            requestResolvedType = _it != null ? _it[_it.length-1] : null;
       // 着重看一下 hash 生成的部分
            int hash = 23;
            // _f 是在 PendingIntent.getBroadcast 时传入的 flags
            hash = (ODD_PRIME_NUMBER*hash) + _f;
            // _r 是在 PendingIntent.getBroadcast 时传入的 requestCode
            hash = (ODD_PRIME_NUMBER*hash) + _r;
            hash = (ODD_PRIME_NUMBER*hash) + _userId;
            if (_w != null) {
                hash = (ODD_PRIME_NUMBER*hash) + _w.hashCode();
            }
            if (_a != null) {
                hash = (ODD_PRIME_NUMBER*hash) + _a.hashCode();
            }
            // requestIntent 就是你传入的 Intent 数组里的最后一个 Intent,
            // 一般使用的时候只有一个 Intent
            // 这里为里生成 hashcode, 调用里 Intent.filterHashCode()
            if (requestIntent != null) {
                hash = (ODD_PRIME_NUMBER*hash) + requestIntent.filterHashCode();
            }
            if (requestResolvedType != null) {
                hash = (ODD_PRIME_NUMBER*hash) + requestResolvedType.hashCode();
            }
            hash = (ODD_PRIME_NUMBER*hash) + (_p != null ? _p.hashCode() : 0);
            hash = (ODD_PRIME_NUMBER*hash) + _t;
            hashCode = hash;
        }

        public int hashCode() {
            return hashCode;
        }
    }

````

接着看一下 Intent.filterHashCode 的生成算法
````
    public int filterHashCode() {
        int code = 0;
        if (mAction != null) {
            code += mAction.hashCode();
        }
        if (mData != null) {
            code += mData.hashCode();
        }
        if (mType != null) {
            code += mType.hashCode();
        }
        if (mPackage != null) {
            code += mPackage.hashCode();
        }
        if (mComponent != null) {
            code += mComponent.hashCode();
        }
        if (mCategories != null) {
            code += mCategories.hashCode();
        }
        return code;
    }
````
那我们捋一下 PendingIntentRecord.key 的 hashcode 和我们传入的哪些参数有关系：
记得我们上面是如何创建 PendingIntent 的吗？

````
val pendingIntent1 = PendingIntent.getBroadcast(context, 0, Intent(BROADCAST_ACTION).apply {
            setClass(context, BroadcastReceiver::class.java)
            putExtra(key, value1)
        }, PendingIntent.FLAG_UPDATE_CURRENT)
````

我们传入的参数是：
````context, requestCode, Intent, flags````
而 Intent 又包含哪些参数呢？
````
    private String mAction;
    private Uri mData;
    private String mType;
    private String mPackage;
    private ComponentName mComponent;
    private int mFlags;
    private ArraySet<String> mCategories;
    private Bundle mExtras;
    private Rect mSourceBounds;
    private Intent mSelector;
    private ClipData mClipData;
    private int mContentUserHint = UserHandle.USER_CURRENT;
    private String mLaunchToken;
````
大概是上面这些, 其中参与了生成 hashcode 就是上面提到的 filterHashCode 这个方法里的参数 所以总共参与生成 hashcode 的和调用中有关的参数是:

````context, requestCode, flags, Intent.action, Intent.data, Intent.package, Intent.component, Intent.categories````

再回过头来看我是如何创建 PendingIntent 的
````
val pendingIntent1 = PendingIntent.getBroadcast(context, 0, Intent(BROADCAST_ACTION).apply {
            setClass(context, BroadcastReceiver::class.java)
            putExtra(key, value1)
        }, PendingIntent.FLAG_UPDATE_CURRENT)

val pendingIntent2 = PendingIntent.getBroadcast(context, 0, Intent(BROADCAST_ACTION).apply {
            setClass(context, BroadcastReceiver::class.java)
            putExtra(key, value2)
        }, PendingIntent.FLAG_UPDATE_CURRENT)

val pendingIntent3 = PendingIntent.getBroadcast(context, 0, Intent(BROADCAST_ACTION).apply {
            setClass(context, BroadcastReceiver::class.java)
            putExtra(key, value3)
        }, PendingIntent.FLAG_UPDATE_CURRENT)

````

😭 唯一不同的是 Intent.extras，却没有参与 hashcode 计算，不知道是 google 故意为止还是一个小 bug 😄。

### 总结：
- PendingIntent 中 的 target 是 PendingIntentRecord，它们都被缓存到 ActivityManagerService 中的一个HashMap中，是弱引用类型；
- 影响 PendingIntent 复用的参数主要是: requestCode, flags, Intent.action, Intent.data, Intent.package, Intent.component, Intent.categories
  所以在我的做法基础上，只要修改一下 requestCode 为不同的值就解决了。

参考：[说说PendingIntent的内部机制](https://my.oschina.net/youranhongcha/blog/196933?utm_source=tuicool&utm_medium=referral)
### 完！