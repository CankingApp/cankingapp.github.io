---
title: 你最了解的 SharedPreference和ContentProvider 知多少？
date: 2018-04-01 11:35:17
categories: android 
tags: android
---

在技术学习的道路上，往往最常见、用的最多地方，却有着容易忽略的技术细节。某个时间点蓦然回首，才发现最应该了解和掌握的技术基础，却由于缺少总结和记录、或者是因为常态思维固化缺少场景去思考，却显得那么陌生。

这篇文章将从作者自身的角度，去重新认识SharedPreference和ContentProvider这两个控件，并且以后也会在博客中有意识的记录类似的技术细节，防止这些基础的技术细节问题再次被遗忘和忽略。

<!--more-->

# SharedPreference和ContentProvider 细节知识加固

开始前，先来自述下作者对安卓系统这两个控件以往一般的认识。

**SharedPreference**
* Android平台轻量级数据存储方式，底层系统封装了xml文件存储，基于key-value键值对数据。
* SharedPreference提供了多种读取模式，可支持多进程读取，但是并不保证*多进程*并发读写安全。
* 底层xml文件的初始化加载和Edit的apply方法是在子线程中进行。

**ContentProvider**
* Android平台四大组件之一，数据共享控件，多用于多个进程间的数据传递。
* 启动要优先于Application的OnCreat方法（查看Application初始化源码得之），尽量不要在App初次启动，（启动进程中）使用多个ContentProvider。
* ContentProvider进程间IPC调用，用到了Binder获取文件描述符（*FileDescriptor*），通过文件描述符实现了高效的共享内存，使得进程间可传递大量数据（Binder接口有1M大小限制）。


## 一、SharedPreference 知识加固

SharedPreference文件保存在App私有目录中，可以在root手机或者调试模式run-as进入查看。

![Shared Path](shared1.png)



### 1、SharedPreference在多进程并发读取时，数据是不安全，那单进程多个线程并发情况呢？

平时遇到SharedPreference的数据读取问题，几乎都是多进程并发场景。多进程问题项目中，基本都是封装一层ContentProvider，将数据的读写共享到其他进程，来保证数据的正确性（当然也可以自定义SharedPref的实现类，去保证多进程并发逻辑）。

那单进程情况下SharedPref多线程的并发，却是容易忽略的地方（虽然很久之前，看过SharedPref的部分源码，但是还是忘了系统是有针对并发加锁的）。

其实简单分析下，平时没有遇到单线程的并发问题，也没有针对单线程的SharedPref读写做相关锁操作，理论上应该是线程安全的，那么我们深入源码看看系统到底是如何保证并发安全的～

从ContextImpl开始去获取一个SharedPref实例：

```
 @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        checkMode(mode);
        SharedPreferencesImpl sp; //真正的系统默认实现类
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp); //首次加载缓存
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 || // 多进程模式去写，每次方法被调用时，会ReLoad一次xml文件。
                                                        // 这种机制在多进程并发严重时，数据是不安全的。
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
```
系统中默认的SharedPref实现类是SharedPreferenceImpl，找到这个类相关读写方法，会看到系统已经针对并发加类锁机制。

```
    public Editor putLong(String key, long value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
    }

    public long getLong(String key, long defValue) {
        synchronized (this) {
            awaitLoadedLocked();//注意：这里可能会wait
            Long v = (Long)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
```
由此可见，虽然系统默认实现的SharedPreferencesImpl支持的多进程的去写模式，但并不保证数据安全。方法读取时，系统已经帮我们针对并发加类锁机制。

### 2、Xml文件的初始化加载和Edit的apply方法是在子线程中进行，那么是否意味着，我们可以不用考虑卡UI，去放心的加载大Shared文件和apply的频繁调用？

**A、先来分析Xml的加载问题**

还是从源码开始，上文中SharedPreferencesImpl的获取是从下面这段代码开始的：

```
    private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;//这里居然是静态的（为了读取速度）
    synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();//入口
            sp = cache.get(file);
            if (sp == null) {
                sp = new SharedPreferencesImpl(file, mode);//开始加载
                cache.put(file, sp);
                return sp;
            }
        }
        
    private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
      if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
      }

      final String packageName = getPackageName();
      ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
      }

    return packagePrefs;
    }
```
这里看到，系统为了快速读取已经加载到内存的SharedPref文件，会将读进来的文件保存在一个**静态**的Map中。这样空间换时间，虽然读取速度有了一定优化，但是读到内存的对象，不会被释放。

SharedPreferencesImpl构造器中会开始加载目标file,如下：

```
    private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;//是否加载完成的标记
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                loadFromDisk();//	异步加载
            }
        }.start();
    }
```

这里主要看mloaded标记，用来标记文件是否已经加载完成，主要用在*awaitLoadedLocked()*方法中。

```
    private void awaitLoadedLocked() {
        ～～～
        ···
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {
            }
        }
    }
```

而awaitLoadedLocked()方法会在每次读和保存时调用，以为着在加载过程中，如果主线程有读写操作，依然会卡UI。

So，Xml的加载本身虽然不卡UI，但是读取操作会等待加载过程，依然有可能卡UI的。并且SharedPref尽量分类去保存，避免一次加载很大的无用文件到内存（可能引起GC，依然会卡UI），耗时又耗内存。

**B、Apply方法是否会引起卡顿？**

> Show me the code

```
    public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);//关键：新加一个任务到队列

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);//当保存任务执行后会从队列中移除
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);//在线程中执行

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }
        
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {//支持并发
                        writeToFile(mcr);//真正的写入文件
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };

        final boolean isFromSyncCommit = (postWriteRunnable == null);

        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }

        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);//单线程池中执行
    }

```

然后只到发现Activity的一段代码（[思路及代码参考见这里](http://weishu.me/2016/10/13/sharedpreference-advices/)）

```
  private void handleStopActivity(IBinder token, boolean show, int configChanges, int seq) {
	···
    // Make sure any pending writes are now committed.
    if (!r.isPreHoneycomb()) {
        QueuedWork.waitToFinish();
    }
    ···
  }
  
  public static void waitToFinish() {
    Runnable toFinish;
    while ((toFinish = sPendingWorkFinishers.poll()) != null) {
        toFinish.run();
    }
}
```
如果Activity在stop时，apply到QueuedWork中任务未执行完，就会在引起主线的等待造成卡顿。

所以，尽量不要频繁的调用apply，可以将多个提交合成一个apply，并且注意规避activity stop时候的SharedPref保存逻辑，避免卡顿。


## 二、ContentProvider 知识加固


### 1、ContentProvider为多进程数据的读取提供类机制，那么单进程下使用ContentProvider,能保证线程并发安全吗？

平时项目中常用到ContentProvider的地方见的最多的可能就是DB的和SharedPreference的数据封装类，由于Sqlite和SharedPref都是自身可以支持线程并发问题的，并没有注意到ContentProvider自身方法的并发特性。

提高ContentProvider，工程中更多的注意力放在类多进程的交互方式上，而忽略了自身方法调用的细节问题。


```
    ContentResolver resolver = content.getContentResolver();
```

通常，在使用ContentProvider时，会通过以上代码去获取ContentResolver引用，然后调用相关接口方法。

```
    private final ApplicationContentResolver mContentResolver;
    
    @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```

同样，接口方法依然在ContextImpl中，可以看到ContentResolver的实现类是ApplicationContentResolver。
这里以*query*方法为例，深入分析下：

```
     public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
            @Nullable String[] projection, @Nullable String selection,
            @Nullable String[] selectionArgs, @Nullable String sortOrder,
            @Nullable CancellationSignal cancellationSignal) {
        IContentProvider unstableProvider = acquireUnstableProvider(uri);//抽象方法，ApplicationContentResolver中实现
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
                qCursor = unstableProvider.query(mPackageName, uri, projection,//缓存Binder处理
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                // The remote process has died...  but we only hold an unstable
                // reference though, so we might recover!!!  Let's try!!!!
                // This is exciting!!1!!1!!!!1
                unstableProviderDied(unstableProvider);//Provider进程死掉逻辑处理
                stableProvider = acquireProvider(uri);//抽象方法，ApplicationContentResolver中实现
                if (stableProvider == null) {
                    return null;
                }
                qCursor = stableProvider.query(mPackageName, uri, projection,//再次query
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
            if (qCursor == null) {
                return null;
            }

            return wrapper;
        } catch (RemoteException e) {
            // Arbitrary and not worth documenting, as Activity
            // Manager will kill this process shortly anyway.
            return null;
        } finally {
           //···
        }
    }
```

主要跟踪下IContentProvider接口如何在*acquireProvider*和*acquireUnstableProvider*中的实现，最终会委托给ActivityThread的*acquireProvider*方法，如下：

```
    public final IContentProvider acquireProvider(
        Context c, String auth, int userId, boolean stable) {
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);//缓存中读取
        if (provider != null) {
        return provider;
        }

        IActivityManager.ContentProviderHolder holder = null;
        try {
        holder = ActivityManagerNative.getDefault().getContentProvider(
                getApplicationThread(), auth, userId, stable);//缓存没有读到，则跨进程AMS中读取
            } catch (RemoteException ex) {
        }
        if (holder == null) {
        Slog.e(TAG, "Failed to find provider info for " + auth);
        return null;
        }

        holder = installProvider(c, holder, holder.info,
            true /*noisy*/, holder.noReleaseNeeded, stable);//安装
        return holder.provider;
    }
```
方法的最后一个参数stable代表着ContentProvider能否可靠拿到一个进程存活的Provider。首先调用了acquireExistingProvider，去mProviderMap中获取一次接口。

```
    public final IContentProvider acquireExistingProvider(
            Context c, String auth, int userId, boolean stable) {
        synchronized (mProviderMap) {
            final ProviderKey key = new ProviderKey(auth, userId);
            final ProviderClientRecord pr = mProviderMap.get(key);//从缓存中先读取
            if (pr == null) {
                return null;
            }

            IContentProvider provider = pr.mProvider;
            IBinder jBinder = provider.asBinder();
            if (!jBinder.isBinderAlive()) {
                // The hosting process of the provider has died; we can't
                // use this one.
                Log.i(TAG, "Acquiring provider " + auth + " for user " + userId
                        + ": existing object's process dead");
                handleUnstableProviderDiedLocked(jBinder, true);
                return null;
            }

            // Only increment the ref count if we have one.  If we don't then the
            // provider is not reference counted and never needs to be released.
            ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
            if (prc != null) {
                incProviderRefLocked(prc, stable);
            }
            return provider;
        }
    }
```

mProviderMap中，这个map的类型是ArrayMap，那么mProviderMap中的数据是哪里的呢？

查下mProvider的put方法，只有*installProviderAuthoritiesLocked*中调用，并且它是在又是在**installProvider**的调用的，那么重的分析下*installProvider*方法。

installProvider方法主要会在*handleBindApplication*和*acquireProvider*方法中调用，意味着：本进程和AMS获取到的Provider都会调用installProvider最后会放到mProviderMap缓存中，

```
    private IActivityManager.ContentProviderHolder installProvider(Context context,
            IActivityManager.ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
        if (holder == null || holder.provider == null) {
            //本地进程provider安装
            try {
                final java.lang.ClassLoader cl = c.getClassLoader();
                localProvider = (ContentProvider)cl.
                    loadClass(info.name).newInstance();
                provider = localProvider.getIContentProvider();
                if (provider == null) {
                    Slog.e(TAG, "Failed to instantiate class " +
                          info.name + " from sourceDir " +
                          info.applicationInfo.sourceDir);
                    return null;
                }
               
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
               
                return null;
            }
        } else {
            provider = holder.provider;
           	//跨进程
        }

        IActivityManager.ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
			//获取可跨进程的Binder对象
            IBinder jBinder = provider.asBinder();
            if (localProvider != null) {
                ComponentName cname = new ComponentName(info.packageName, info.name);
                ProviderClientRecord pr = mLocalProvidersByName.get(cname);
                if (pr != null) {

                    provider = pr.mProvider;
                } else {
                    holder = new IActivityManager.ContentProviderHolder(info);
                    holder.provider = provider;
                    holder.noReleaseNeeded = true;
                    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }
                retHolder = pr.mHolder;
            } else {
                ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
                if (prc != null) {
                    if (!noReleaseNeeded) {
                        incProviderRefLocked(prc, stable);
                        try {
                            ActivityManagerNative.getDefault().removeContentProvider(
                                    holder.connection, stable);
                        } catch (RemoteException e) {
                            //do nothing content provider object is dead any way
                        }
                    }
                } else {
                    ProviderClientRecord client = installProviderAuthoritiesLocked(
                            provider, localProvider, holder);
                    if (noReleaseNeeded) {
                        prc = new ProviderRefCount(holder, client, 1000, 1000);
                    } else {
                        prc = stable
                                ? new ProviderRefCount(holder, client, 1, 0)
                                : new ProviderRefCount(holder, client, 0, 1);
                    }
                    mProviderRefCountMap.put(jBinder, prc);
                }
                retHolder = prc.holder;
            }
        }

        return retHolder;
    }
```

分析到这里可以看出，如何Provider是在同一个进程，会通过Binder进口先查下本地对象，然后返回本地Provider实例。

回到起初的query方法，就是直接调用目标对象的query，然后系统对query方法的调用并没有做并发处理，所以可得出结论，ContentProvider的方法调用是非并发安全的。

### 2、ContentProvider启动时机？

上文分析中在App进程启动时，通过thread.attach()方法会调用到handleBindApplication，方法内初始化当前进程的application和provider。

```
  private void handleBindApplication(AppBindData data) {
  //···
        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);//注：这里第二个参数是null，意味着Application只会调用attachBaseContext方法
            mInitialApplication = app;

            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    installContentProviders(app, data.providers); //本进程内Provider的启动安装
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            try {
                mInstrumentation.callApplicationOnCreate(app);// Provider启动后才会继续回调Application的onCreate
            } catch (Exception e) {

            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
  }
```

由此可见，本进程的Provider的安装是在Application初始化调用attact方法后，调用onCreate前启动。所以在App启动时注意规避本进程内的Provider，避免启动耗时。

## 三、总结

技术成长的道路上，不要忽略基础知识的积累，特别是一些细节问题，是不同阶段程序员修炼内功需要坚持的习惯，也是深入技术研究的基础之路。


------
欢迎转载，请标明出处：常兴E站 [canking.win](http://www.canking.win)

