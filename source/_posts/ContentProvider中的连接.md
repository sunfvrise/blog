---
title: ContentProvider中的连接
date: 2022-10-23 10:00:00
toc: true
author: sunfvrise
categories: Android
tags:
    - Android
    - ContentProvider
summary: ContentProvider实现原理中体现出来的连接性质
---

ContentProvider 中常用的函数，除了 query 之外（异常情况下也会），在调用过程中都在 ActivityManagerService 侧有维护强连接的概念。一旦内容提供者进程死亡，使用者也会连坐被杀。

***

Android 应用进程会有因为 Provider 依赖关联杀死的情况，你可以看到以下日志：
`am_kill : [xx, xx, com.xx, xx, depends on provider com.xxx in dying proc com.xx]`
 
这会令你感到疑惑，是什么神秘机制导致我的应用被连坐呢？首先我们要按图索骥。

# 寻根

既然是 AM 的日志，那自然要去 Framework 寻找答案啦。
`ContentProviderHelper.java#removeDyingProviderLocked`
 
```java
ProcessRecord capp = conn.client;
final IApplicationThread thread = capp.getThread();
conn.dead = true;
if (conn.stableCount() > 0) {
    final int pid = capp.getPid();
    if (!capp.isPersistent() && thread != null
            && pid != 0 && pid != ActivityManagerService.MY_PID) {
        capp.killLocked(
                "depends on provider " + cpr.name.flattenToShortString()
                        + " in dying proc " + (proc != null ? proc.processName : "??")
                        + " (adj " + (proc != null ? proc.mState.getSetAdj() : "??") + ")",
                ApplicationExitInfo.REASON_DEPENDENCY_DIED,
                ApplicationExitInfo.SUBREASON_UNKNOWN,
                true);
    }
} else if (thread != null && conn.provider.provider != null) {
    try {
        thread.unstableProviderDied(conn.provider.provider.asBinder());
    } catch (RemoteException e) {
    }
    // In the protocol here, we don't expect the client to correctly
    // clean up this connection, we'll just remove it.
    cpr.connections.remove(i);
    if (conn.client.mProviders.removeProviderConnection(conn)) {
        mService.stopAssociationLocked(capp.uid, capp.processName,
                cpr.uid, cpr.appInfo.longVersionCode, cpr.name, cpr.info.processName);
    }
}
```
 
ContentProviderConnection 是 Framework 层里面记录应用 Provider 组件信息的数据结构。

```java
/**
 * Represents a link between a content provider and client.
 */
public final class ContentProviderConnection extends Binder {
    public final ContentProviderRecord provider;
    public final ProcessRecord client;
    public final String clientPackage;
    public AssociationState.SourceState association;
    public final long createTime;
    public int stableCount;
    public int unstableCount;
    // The client of this connection is currently waiting for the provider to appear.
    // Protected by the provider lock.
    public boolean waiting;
    // The provider of this connection is now dead.
    public boolean dead;
    // For debugging.
    public int numStableIncs;
    public int numUnstableIncs;
```
 
以下是几个调用到该函数的路径，总结一下，就是进程要死亡的时候会做这个动作。

`ActivityManagerService.java#forceStopPackageLocked
 ActivityManagerService.java#cleanupDisabledPackageComponentsLocked
 ActivityManagerService.java#processStartTimedOutLocked
 ProcessProviderRecord.java#onCleanupApplicationRecordLocked
`
 
那么从以上的源码分析中，我们可以得到以下信息：
进程死亡时，provider 有 stable 连接，依赖方也会被杀。
这个机制不会杀 persist 进程。（persist 进程被定义为系统的一部分，由系统保活）
 
那么问题又来了，什么是 provider 的 stable 连接。

# 探源

那么我们来找一下 stableCount 是在哪里被修改的。

`
ActivityManagerService.java#getContentProvider(..., boolean stable, ...)
ActivityManagerService.java#getContentProviderImpl(..., boolean stable, ...)
`
 
应用进程的 ActivityThread 中，`acquireProvider(..., boolean stable, ...)` 方法会调用获取目标 provider 的 binder 对象进行通讯。
 
显然我们日常使用的 provider 的时候，用的是 ContentResolver 呢。那么他们之间有什么联系呢？那显然是有调用关系的。
 
ActivityThread 是 ContextImpl 的成员，那巧了，ContentResolver 也是它的成员，可以愉快地调用 ActivityThread#acquireProvider 函数了。应用的 ContextImpl 对象是在 LoadedApk#makeApplication 函数中被创建的，这个函数会在 AM 和 App 通信的时候调用。
 
总结一下，provider 的连接数在 acquireProvider 系列函数中添加计数，在 releaseProvider 函数中消减计数。这个计数维护在 AM 中。
 
对了，ContextImpl 也是我们常用的 Context 对象的实现类，我们通常使用的 ContentResolver 是它内部预定义的一个实现类。

```java
private static final class ApplicationContentResolver extends ContentResolver {
    @UnsupportedAppUsage
    private final ActivityThread mMainThread;
    public ApplicationContentResolver(Context context, ActivityThread mainThread) {
        super(context);
        mMainThread = Objects.requireNonNull(mainThread);
    }
    @Override
    @UnsupportedAppUsage
    protected IContentProvider acquireProvider(Context context, String auth) {
        return mMainThread.acquireProvider(context,
                ContentProvider.getAuthorityWithoutUserId(auth),
                resolveUserIdFromAuthority(auth), true);
    }
    @Override
    protected IContentProvider acquireExistingProvider(Context context, String auth) {
        return mMainThread.acquireExistingProvider(context,
                ContentProvider.getAuthorityWithoutUserId(auth),
                resolveUserIdFromAuthority(auth), true);
    }
    @Override
    public boolean releaseProvider(IContentProvider provider) {
        return mMainThread.releaseProvider(provider, true);
    }
    @Override
    protected IContentProvider acquireUnstableProvider(Context c, String auth) {
        return mMainThread.acquireProvider(c,
                ContentProvider.getAuthorityWithoutUserId(auth),
                resolveUserIdFromAuthority(auth), false);
    }
    @Override
    public boolean releaseUnstableProvider(IContentProvider icp) {
        return mMainThread.releaseProvider(icp, false);
    }
    @Override
    public void unstableProviderDied(IContentProvider icp) {
        mMainThread.handleUnstableProviderDied(icp.asBinder(), true);
    }
    @Override
    public void appNotRespondingViaProvider(IContentProvider icp) {
        mMainThread.appNotRespondingViaProvider(icp.asBinder());
    }
    /** @hide */
    protected int resolveUserIdFromAuthority(String auth) {
        return ContentProvider.getUserIdFromAuthority(auth, getUserId());
    }
}
```
 
既然了解了他们之间的调用链，那么我们日常使用的 API 中，有哪些不是 stable 的，也就是不会导致连坐呢。答案很遗憾，没有绝对安全的方法。
 
我们常用的 provider 相关的函数有：query，insert，update，delete，call。只有 query 来说是“相对安全”的。其他的函数均获取 stable provider 进行通信。
 
query 过程会优先获取 unstable provider，通信失败时会获取 stable provider 重试。

```java
@Override
public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
@Nullable String[] projection, @Nullable Bundle queryArgs,
@Nullable CancellationSignal cancellationSignal) {
    android.util.SeempLog.record_uri(13, uri);
    Objects.requireNonNull(uri, "uri");
    try {
        if (mWrapped != null) {
            return mWrapped.query(uri, projection, queryArgs, cancellationSignal);
        }
    } catch (RemoteException e) {
        return null;
    }
    IContentProvider unstableProvider = acquireUnstableProvider(uri);
    if (unstableProvider == null) {
        return null;
    }
    IContentProvider stableProvider = null;
    Cursor qCursor = null;
    try {
        long startTime = SystemClock.uptimeMillis();
        ICancellationSignal remoteCancellationSignal = null;
        if (cancellationSignal != null) {
            cancellationSignal.throwIfCanceled();
            remoteCancellationSignal = unstableProvider.createCancellationSignal();
            cancellationSignal.setRemote(remoteCancellationSignal);
        }
        try {
            qCursor = unstableProvider.query(mPackageName, mAttributionTag, uri, projection,
                    queryArgs, remoteCancellationSignal);
        } catch (DeadObjectException e) {
            // The remote process has died...  but we only hold an unstable
            // reference though, so we might recover!!!  Let's try!!!!
            // This is exciting!!1!!1!!!!1
            unstableProviderDied(unstableProvider);
            stableProvider = acquireProvider(uri);
            if (stableProvider == null) {
                return null;
            }
            qCursor = stableProvider.query(mPackageName, mAttributionTag, uri, projection,
                    queryArgs, remoteCancellationSignal);
        }
        if (qCursor == null) {
            return null;
        }
        // Force query execution.  Might fail and throw a runtime exception here.
        qCursor.getCount();
        long durationMillis = SystemClock.uptimeMillis() - startTime;
        maybeLogQueryToEventLog(durationMillis, uri, projection, queryArgs);
        // Wrap the cursor object into CursorWrapperInner object.
        final IContentProvider provider = (stableProvider != null) ? stableProvider
                : acquireProvider(uri);
        final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
        stableProvider = null;
        qCursor = null;
        return wrapper;
    } catch (RemoteException e) {
        // Arbitrary and not worth documenting, as Activity
        // Manager will kill this process shortly anyway.
        return null;
    } finally {
        if (qCursor != null) {
            qCursor.close();
        }
        if (cancellationSignal != null) {
            cancellationSignal.setRemote(null);
        }
        if (unstableProvider != null) {
            releaseUnstableProvider(unstableProvider);
        }
        if (stableProvider != null) {
            releaseProvider(stableProvider);
        }
    }
}
```
 
其他的几个函数都会获取 stable provider 进行通信。

```java
@Override
public final @Nullable Uri insert(@RequiresPermission.Write @NonNull Uri url,
        @Nullable ContentValues values, @Nullable Bundle extras) {
    android.util.SeempLog.record_uri(37, url);
    Objects.requireNonNull(url, "url");
    try {
        if (mWrapped != null) return mWrapped.insert(url, values, extras);
    } catch (RemoteException e) {
        return null;
    }
    IContentProvider provider = acquireProvider(url);
    if (provider == null) {
        throw new IllegalArgumentException("Unknown URL " + url);
    }
    try {
        long startTime = SystemClock.uptimeMillis();
        Uri createdRow = provider.insert(mPackageName, mAttributionTag, url, values, extras);
        long durationMillis = SystemClock.uptimeMillis() - startTime;
        maybeLogUpdateToEventLog(durationMillis, url, "insert", null /* where */);
        return createdRow;
    } catch (RemoteException e) {
        // Arbitrary and not worth documenting, as Activity
        // Manager will kill this process shortly anyway.
        return null;
    } finally {
        releaseProvider(provider);
    }
}
```

# 思考和启示
通过机制的挖掘，我们了解到使用 provider 通讯时，除了 query 函数一般情况下不会建立强连接，其他函数的 IPC 过程都是被定义为强连接的。那么在通讯过程中，如果对端死亡，我们自己的进程也会发生裙带效应。
 
单次调用结束后，由于引用计数的消减，所以并不会导致发生连坐现象。但持续时间过长的调用，更容易因为对端进程的稳定性而引发连带的问题。
 
结合数据的 CRUD 操作，只有 query 查询是读操作，其他的都是写操作。通常我们需要保证写操作的一致性。那么 Android 对于写时异常的处理策略是这样的，你觉得它是好是坏呢？