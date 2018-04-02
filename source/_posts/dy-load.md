---
title: 安卓平台中的动态加载技术分析
date: 2018-01-07 10:20:21
categories: android 
tags: java
---

安卓平台的动态加载原理，本质其实还是利用java相关知识实现。然而java语言中，开发人员能通过程序进行动态操作class的，主要是字节码生成和类加载器这两部分的功能。本文中也主要是围绕这两方面的技术，展开在安卓平台上的应用分析。

阅读本文，一起宏观理解安卓插件化，热修复，模块化，AOP，Java类加载等知识。

<!--more-->

# 动态加载技术分析

## 一、Java基础知识

### 1、虚拟机类的加载剖析

Java虚拟机把描述类的数据从Class文件加载到内存，对数据进行校验、转化解析和初始化，最终形成被虚拟机直接使用的Java类型，完成了类的加载机制。

类型的加载、链接和初始化都是在程序运行时期间完成的，虽然会在初次加载耗一定性能，但是正是这个机制为Java提供高度的动态扩展灵活性。如可以编写一个面向接口的应用程序，运行时再决定实际的实现类，实现类可以从本地或是网络动态加载。

类从被加载到虚拟机内存到从内存中卸载，完整的生命周期包括：加载（loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）、和卸载（UnLoading）七个阶段，其中验证、准备、解析统称为连接（Linking）。
这里主要分析下加载阶段：

*‘加载’*是*‘类加载’*过程中的一个阶段，主要完成三件事：1）通过类的全限定名来获取定义此类的二进制字节流。2）将这个字节流所代表的静态储存结构转化为方法区的运行时数据结构。3）在内存中生成此类相应Class对象，作为方法区这个类的各种数据的访问入口。


虚拟机规定加载一个类在加载阶段需要能“通过一个类的全限定名获取此类二进制字节流”，但是并没有规定要从哪里、怎样获取，例如：

* 从ZIP中读取，如：jar，dex等。
* 网络流中获取，如Applet。
* 运行时计算生成，如动态代理。
* 有其他文件生成、数据库中读取等。
* ...

这样，开发人员就可以通过自定义类加载器（ClassLoader），或是hook系统类加载器来控制字节流的获取方式。

类的显式加载（Class.forName）和隐式加载（如：new）两种加载方式，最后都会调用类加载器的 loadClass 方法来完成类的实际加载工作。

![双亲委托模型](dl1.png)

类的加载器具有双亲委托模型，该特性保证了Java程序的安全性，但是并不是虚拟机强制规定的，我们也可以自定义类加载机制，来破坏这种机制。如安卓插件加载，热修及OSGI模块化动态加载中的类加载技术应用。

### 2、面向切面编程（AOP）

Java面对对象的语言，平时用的最多的就是OOP（面对对象编程），然后在很多技术研发中（如安卓动态加载，性能监控，日志等）慢慢的接触到一个新的名次**“AOP”**，Google过后，才发现这是一种很特别、很强大、很高级实用的编程方式。百度百科解释如下：

>AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

其实AOP编程模式已经存在很多年类，并且在Android项目开发中也有很多地方的应用，有很多优秀且流行的工具如APT，AspectJ, Javassist等，为我们AOP开发提供了便利实现。

不同的工具库作用在不同的阶段，这里引用网络上一张图，可以说明三者间的区别。

![AOP-引用自网络](dl2.jpeg)

**APT**

APT(Annotation Processing Too）应该都听过并且用过，概念比较好理解，主要作用在编译期，也是比较流行且常见的技术。

代码编译期解析注解后，结合square公司的开源项目javapoet项目，生成自定逻辑的java源码。有很多开源库都在用如：ButterKnife、EventBus等。

**AspectJ**

AspectJ支持编译期和加载时代码注入, 有一个专门的编译器用来生成遵守Java字节编码规范的Class文件。更多细节可以看[这里](https://www.jianshu.com/p/0fa8073fd144)。

**Javassist**

Javassist是一个开源的分析、编辑和创建Java字节码的类库。允许开发者自由的在一个已经编译好的类中添加新的方法，或者是修改已有的方法，允许开发者忽略被修改的类本身的细节和结构。

360开源插件项目[RePlugin](https://github.com/Qihoo360/RePlugin)中，为了减少对安卓系统的Hook点，又希望解耦开发层代码逻辑，其中就用到了*Javassist*技术。


## 二、插件化技术分析

安卓项目发展早起，可能最初只是为了解决65535问题及包大小，各家公司都在研究安卓平台动态加载，也就是现在所谓的插件化解决方案。

从14年的*Dynamic-load-apk*开源项目，到现在（17年）的*RePlugin*及*VirtualAPK*，分析下实现原理，个人理解，其实并不是很难，无非就是利用静态代理、动态代理，反射，HOOK及ClassLoader（dex合并，或双亲委托破坏）等相关技术对安卓四大组件的沙盒实现。

当然真正的难点在于对千万安卓机型及Rom的适配，及完整安卓功能的兼容实现，不可否认插件化大牛在安卓行业发展的巨大贡献（感谢各位开源大佬🙏）。

既然已经有很多优秀的开源库，那么我们是否还有必要学习它呢？答案是否定的。

个人认为，学习插件化是一名安卓开发，接触系统源码，牢固安卓知识，提升开发技能得良好途径。推荐可以好好学习下15年最早开源[DroidPlugin](https://github.com/Qihoo360/DroidPlugin)项目，虽然和今年开源的*RePlugin*和*VirtualAPK*比，有一定的复杂及兼容差，但是分析后者其实还是会有前者实现思想，基本上万变不离其综（如activity的插件实现，都是占坑思想；都是在startActivity后，利用一定技术手段在一定点替换targeActivity，等AMS回调回来，在找一合适点，换回原activity）。个人理解后者是从不同角度对DroidPlugin的升级及优化。

## 三、热修复

安卓技术发展中期，有很多大型APP，需要紧急解决线上问题，业界开始研究线上热修复技术。

期间，Google官方为了解决65535问题，发布了Multidex多dex方案，利用Application类加载及初始化的过程，从中attachContext时候，动态加载多个dex。并在Android Studio 2.0发布了Intant Run快速代码部署方案（热插拔）。


这不但给插件化给出类借鉴方案，也给热修改提供了Java层的可行方案（此文不展开Native层的实现分析）。具有代表性的项目如美团的[热更新方案Robust](https://tech.meituan.com/android_robust.html)。

这里简单分析下Google intant Run实现原理

通过阅读[官方文档](https://developer.android.com/studio/run/index.html?hl=zh-cn#cold-swap)及源码，可看出官方代码快速部分定义了三个概念：

* 热拔插：代码改变被应用、投射到APP上，不需要重启应用，不需要重建当前activity。场景：适用于多数的简单改变（包括一些方法实现的修改，或者变量值修改）
* 温拔插：activity需要被重启才能看到所需更改。场景：典型的情况是代码修改涉及到了资源文件，即res。
* 冷拔插：app需要被重启（但是仍然不需要重新安装）场景：任何涉及结构性变化的，比如：修改了继承规则、修改了方法签名等。

搜下Intant Run Server源码会发现代码更新方式逻辑如下：

```
    private int handlePatches(List<ApplicationPatch> changes,
            boolean hasResources, int updateMode) {
        if (hasResources) {
            FileManager.startUpdate();
        }
        for (ApplicationPatch change : changes) {
            String path = change.getPath();
            if (path.endsWith(".dex")) {
                handleColdSwapPatch(change);//冷拔插

                boolean canHotSwap = false;
                for (ApplicationPatch c : changes) {
                    if (c.getPath().equals("classes.dex.3")) {
                        canHotSwap = true;
                        break;
                    }
                }
                if (!canHotSwap) {
                    updateMode = 3;
                }
            } else if (path.equals("classes.dex.3")) {
                updateMode = handleHotSwapPatch(updateMode, change);//热拔插
            } else if (isResourcePath(path)) {
                updateMode = handleResourcePatch(updateMode, change, path);//温拔插
            }
        }
        if (hasResources) {
            FileManager.finishUpdate(true);
        }
        return updateMode;
    }
```

主要分析下涉及到代码更新的热插拔和冷插拔。分析代码过程主要是反编通过Intant Run为我们生成的Apk，然后顺藤摸瓜。

首先看这个壳子APK的Application都干了什么，分析下壳子APK是怎么启动我们真正的App的。

```
  protected void attachBaseContext(Context context) {
        if (!AppInfo.usingApkSplits) {
            String apkFile = context.getApplicationInfo().sourceDir;
            long apkModified = apkFile != null ? new File(apkFile)
                    .lastModified() : 0L;
            createResources(apkModified);
            setupClassLoaders(context, context.getCacheDir().getPath(),
                    apkModified);
        }
        createRealApplication();
        if (this.realApplication != null) {
            try {
                Method attachBaseContext = ContextWrapper.class
                        .getDeclaredMethod("attachBaseContext",
                                new Class[] { Context.class });
                attachBaseContext.setAccessible(true);
                attachBaseContext.invoke(this.realApplication,
                        new Object[] { context });
            } catch (Exception e) {
            }
        }
    }
    
    private void createResources(long apkModified) {
        File file = FileManager.getExternalResourceFile();
        this.externalResourcePath = (file != null ? file.getPath() : null);
       
        if (file != null) {
            try {
                long resourceModified = file.lastModified();
               
                if ((apkModified == 0L) || (resourceModified <= apkModified)) {
                    if (Log.isLoggable("InstantRun", 2)) {
                        Log.v("InstantRun",
                                "Ignoring resource file, older than APK");
                    }
                    this.externalResourcePath = null;
                }
            } catch (Throwable t) {
                Log.e("InstantRun", "Failed to check patch timestamps", t);
            }
        }
    }

    private static void setupClassLoaders(Context context, String codeCacheDir,
                                          long apkModified) {
        List dexList = FileManager.getDexList(context, apkModified);
        Class server = Server.class;
        Class patcher = MonkeyPatcher.class;
        if (!dexList.isEmpty()) {
            ClassLoader classLoader = BootstrapApplication.class
                    .getClassLoader();
            String nativeLibraryPath;
            try {
                nativeLibraryPath = (String) classLoader.getClass()
                        .getMethod("getLdLibraryPath", new Class[0])
                        .invoke(classLoader, new Object[0]);
                if (Log.isLoggable("InstantRun", 2)) {
                    Log.v("InstantRun", "Native library path: "
                            + nativeLibraryPath);
                }
            } catch (Throwable t) {
                nativeLibraryPath = FileManager.getNativeLibraryFolder()
                        .getPath();
            }
            IncrementalClassLoader.inject(classLoader, nativeLibraryPath,
                    codeCacheDir, dexList);
        }
    }
```

这里可以看到最后会有个IncrementalClassLoader，当前PathClassLoader委托IncrementalClassLoader加载dex。代码冷部署方案中，当APK进程重启时，IncrementalClassLoader会优先加载部署的变更代码。
分析源码看出它的结构图如下：

![引用网络图片](dl3.png)

createRealApplication方法创建出真正的Application，启动真正Dex方式和上面插件化思想有点类似，也是一个壳子动态加载代码的很好例子。

```
    private void createRealApplication() {
        if (AppInfo.applicationClass != null) {
            if (Log.isLoggable("InstantRun", 2)) {
                Log.v("InstantRun",
                        "About to create real application of class name = "
                                + AppInfo.applicationClass);
            }
            try {
                Class realClass = (Class) Class
                        .forName(AppInfo.applicationClass);
                if (Log.isLoggable("InstantRun", 2)) {
                    Log.v("InstantRun",
                            "Created delegate app class successfully : "
                                    + realClass + " with class loader "
                                    + realClass.getClassLoader());
                }
                Constructor constructor = realClass
                        .getConstructor(new Class[0]);
                this.realApplication = ((Application) constructor
                        .newInstance(new Object[0]));
                if (Log.isLoggable("InstantRun", 2)) {
                    Log.v("InstantRun",
                            "Created real app instance successfully :"
                                    + this.realApplication);
                }
            } catch (Exception e) {
                throw new IllegalStateException(e);
            }
        } else {
            this.realApplication = new Application();
        }
    }
```

真正的Application创建后，接下来再分析下变更的代码是如何热更新生效的？

```
  public void onCreate() {
        if (!AppInfo.usingApkSplits) {
            MonkeyPatcher.monkeyPatchApplication(this, this,
                    this.realApplication, this.externalResourcePath);
            MonkeyPatcher.monkeyPatchExistingResources(this,
                    this.externalResourcePath, null);
        } else {
            MonkeyPatcher.monkeyPatchApplication(this, this,//反射 ActivityThread 替换Application
                    this.realApplication, null);
        }
        if (AppInfo.applicationId != null) {
            try {
                boolean foundPackage = false;
                int pid = Process.myPid();
                ActivityManager manager = (ActivityManager) getSystemService("activity");
                List processes = manager
                        .getRunningAppProcesses();
                boolean startServer = false;
                if ((processes != null) && (processes.size() > 1)) {
                    for (ActivityManager.RunningAppProcessInfo processInfo : processes) {
                        if (AppInfo.applicationId
                                .equals(processInfo.processName)) {
                            foundPackage = true;
                            if (processInfo.pid == pid) {
                                startServer = true;
                                break;
                            }
                        }
                    }
                    if ((!startServer) && (!foundPackage)) {
                        startServer = true;
                    }
                } else {
                    startServer = true;
                }
                if (startServer) {
                    Server.create(AppInfo.applicationId, this);
                }
            } catch (Throwable t) {
                Server.create(AppInfo.applicationId, this);
            }
        }
        if (this.realApplication != null) {
            this.realApplication.onCreate();//真正的application回调onCreate
        }
    }
    
     public static void monkeyPatchApplication(Context context,
                                              Application bootstrap, Application realApplication,
                                              String externalResourceFile) {
        try {
            Class activityThread = Class
                    .forName("android.app.ActivityThread");
            Object currentActivityThread = getActivityThread(context,
                    activityThread);
            Field mInitialApplication = activityThread
                    .getDeclaredField("mInitialApplication");
            mInitialApplication.setAccessible(true);
            Application initialApplication = (Application) mInitialApplication
                    .get(currentActivityThread);
            if ((realApplication != null) && (initialApplication == bootstrap)) {
                mInitialApplication.set(currentActivityThread, realApplication);
            }
            if (realApplication != null) {
                Field mAllApplications = activityThread
                        .getDeclaredField("mAllApplications");
                mAllApplications.setAccessible(true);
                List allApplications = (List) mAllApplications
                        .get(currentActivityThread);
                for (int i = 0; i < allApplications.size(); i++) {
                    if (allApplications.get(i) == bootstrap) {
                        allApplications.set(i, realApplication);//完成application的替换
                    }
                }
            }
            Class loadedApkClass;
            try {
                loadedApkClass = Class.forName("android.app.LoadedApk");
            } catch (ClassNotFoundException e) {
                loadedApkClass = Class
                        .forName("android.app.ActivityThread$PackageInfo");
            }
            Field mApplication = loadedApkClass
                    .getDeclaredField("mApplication");
            mApplication.setAccessible(true);
            Field mResDir = loadedApkClass.getDeclaredField("mResDir");
            mResDir.setAccessible(true);
            Field mLoadedApk = null;
            try {
                mLoadedApk = Application.class.getDeclaredField("mLoadedApk");
            } catch (NoSuchFieldException e) {
            }
            for (String fieldName : new String[] { "mPackages",
                    "mResourcePackages" }) {
                Field field = activityThread.getDeclaredField(fieldName);
                field.setAccessible(true);
                Object value = field.get(currentActivityThread);
                for (Map.Entry> entry : ((Map>) value).entrySet()) {
                    Object loadedApk = ((WeakReference) entry.getValue()).get();
                    if (loadedApk != null) {
                        if (mApplication.get(loadedApk) == bootstrap) {
                            if (realApplication != null) {
                                mApplication.set(loadedApk, realApplication);
                            }
                            if (externalResourceFile != null) {
                                mResDir.set(loadedApk, externalResourceFile);//资源文件替换
                            }
                            if ((realApplication != null)
                                    && (mLoadedApk != null)) {
                                mLoadedApk.set(realApplication, loadedApk);
                            }
                        }
                    }
                }
            }
        } catch (Throwable e) {
            throw new IllegalStateException(e);
        }
    }
```
程序在调用realApplication前，替换调了自己进程中所有关于Application的实例，包括：

* ActivityThread的mInitialApplication为realApplication
* mAllApplications 中所有的Application为realApplication
* ActivityThread的mPackages,mResourcePackages中的mLoaderApk中的application为realApplication。

*monkeyPatchExistingResources*方法中替换资源问题,新建一个AssetManager对象newAssetManager，然后用newAssetManager对象替换所有当前Resource、Resource.Theme的mAssets成员变量。

到这里APK完成了启动操作，接下来重点分析下神奇的handleHotSwapPatch热部署，不用重启Activity，不用重启进程，就可以即时生效，这种方案对安卓的线上热修复有很大的思路借鉴。


```
    private int handleHotSwapPatch(int updateMode, ApplicationPatch patch) {
        try {
            String dexFile = FileManager.writeTempDexFile(patch.getBytes());
            if (dexFile == null) {
                return updateMode;//code update mode
            }

            String nativeLibraryPath = FileManager.getNativeLibraryFolder()
                    .getPath();
            DexClassLoader dexClassLoader = new DexClassLoader(dexFile,
                    this.mApplication.getCacheDir().getPath(),
                    nativeLibraryPath, getClass().getClassLoader());//加载dex

            Class<?> aClass = Class.forName(
                    "com.android.tools.fd.runtime.AppPatchesLoaderImpl", true,
                    dexClassLoader);
            try {
                PatchesLoader loader = (PatchesLoader) aClass.newInstance();
                String[] getPatchedClasses = (String[]) aClass
                        .getDeclaredMethod("getPatchedClasses", new Class[0])
                        .invoke(loader, new Object[0]);
 
                if (!loader.load()) {
                    updateMode = 3;
                }
            } catch (Exception e) {
                Log.e("InstantRun", "Couldn't apply code changes", e);
                e.printStackTrace();
                updateMode = 3;
            }
        } catch (Throwable e) {
            Log.e("InstantRun", "Couldn't apply code changes", e);
            updateMode = 3;
        }
        return updateMode;
    }
```

handleHotSwapPatch主要作用是反射调用AppPatchesLoaderImpl类的load方法,看下load方法具体实现：

```
public boolean load() {
        try {
            for (String className : getPatchedClasses()) {
                ClassLoader cl = getClass().getClassLoader();
                Class<?> aClass = cl.loadClass(className + "$override");
                Object o = aClass.newInstance();
                Class<?> originalClass = cl.loadClass(className);
                Field changeField = originalClass.getDeclaredField("$change");

                changeField.setAccessible(true);

                Object previous = changeField.get(null);
                if (previous != null) {
                    Field isObsolete = previous.getClass().getDeclaredField(
                            "$obsolete");
                    if (isObsolete != null) {
                        isObsolete.set(null, Boolean.valueOf(true));
                    }
                }
                changeField.set(null, o);
                if ((Log.logging != null)
                        && (Log.logging.isLoggable(Level.FINE))) {
                    Log.logging.log(Level.FINE, String.format("patched %s",
                            new Object[] { className }));
                }
            }
        } catch (Exception e) {
            if (Log.logging != null) {
                Log.logging.log(Level.SEVERE, String.format(
                        "Exception while patching %s",
                        new Object[] { "foo.bar" }), e);
            }
            return false;
        }
        return true;
    }
```
load方法主要加载了以*“Patch类名+$override”*命名的一个类，并初次加载时把该类**$change**中成员*$obsolete*设置成*true*，然后赋值给已经加载的原始类。

反编译原class文件，会发现每个类会多个*"IncrementalChange localIncrementalChange = $change"*成员，并会在每个方法内添加以下代码，已拦截patch执行逻辑。

```
   public void onCreate(Bundle paramBundle) {
        IncrementalChange localIncrementalChange = $change;
        if (localIncrementalChange != null) {//如上文中加载步骤，会用patch类给予赋值，然后走新代码逻辑
            localIncrementalChange.access$dispatch(
                    "onCreate.(Landroid/os/Bundle;)V", new Object[] { this,
                            paramBundle });//通过IncrementalChange接口，分派到对应的新方法逻辑
            return;
        }
        super.onCreate(paramBundle);
        ...
    }
```
这样，就可以不用重启，即时调用新类中的新方法逻辑。有兴趣的同学可以进一步看下*nuptboyzhb*同学反编译的intant run[源码](https://github.com/nuptboyzhb/AndroidInstantRun)。

## 四、模块化

安卓技术发展到成熟器，安卓客户度项目发展到一定规模，为了便于各个业务线低耦合开发、加快编译速度、按模块业务升级及测试等方面的高效进行，各家都陆续提出了模块化解决方案。

讲到模块化，还有个概念是“组件化”，网上有博客直接认为是一个概念，个人认为，两者还是有本质的不同的。

个人理解，组件化目标对象是代码，为了解耦功能模块代码，加大复用力度，按照代码功能模块抽离出组件模块，形成项目的“组件化”。而模块化目标对象是开发人员，更多的是以业务线为界限，解耦业务线间的代码调用，使得一个业务线内高内聚，可以独立完成开发到上线各阶段工作，而不受其他模块开发影响。也可以说组件化只是模块化项目中的一个子集概念。

模块化过程中，需要达到的开发目标要求，这引用阿里*Atlas*开源项目目标定义：

* 在工程期，实现工程独立开发，调试的功能，工程模块可以独立。
* 在运行期，实现完整的组件生命周期的映射，类隔离等机制。
* 在运维期，提供快速增量的更新修复能力，快速升级。

然而，在Java世界里，已经有了OSGI规范，是基于Java语言的动态模块化规范。安卓模块上，也可以参考*OSGI*的解耦方式，定制ClassLoader，动态加载模块代码。

有兴趣的可以研究的阿里的手机淘宝研发团队开源态组件化(Dynamic Bundle)框架项目[Atlas](http://atlas.taobao.org/)。


## 五、总结

本文主要以宏观视角，从Java动态加载基础知识，到最近比较流行的“模块化”进行了基本概念及技术基础的回顾，目的是对 有关Java动态加载技术在安卓平台上的应用 等相关的技术树的整理和梳理。希望能够对安卓平台的插件化、及模块化相关的动态加载技术有个整理宏观了解，并针对一定方向有一定深入研究。


------
欢迎转载，请标明出处：常兴E站 [canking.win](http://www.canking.win)











