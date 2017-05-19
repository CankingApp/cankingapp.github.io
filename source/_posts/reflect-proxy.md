---
title: 知识总结之 插件化基础 java反射与代理
date: 2017-05-12 20:48:03
categories: android 
tags: Plugin
---

Java平台的反射机制是代码动态加载和调用的基本途径，在安卓系统源码中也用到了大量的反射动态加载类。反射也是安卓平台插件化实现的必要掌握的基础知识。代理是客户端灵活操作对象，间接的低耦合度操作对象的有效途径，也是插件化必要掌握知识。

![武汉·东湖](http://wx2.sinaimg.cn/mw690/70e618a5ly1ffe7fo685sj20ku0fmn1x.jpg)

<!--more-->

# 安卓插件化基础 java反射与代理

## 一、反射

java中反射机理比较常用，这里主要以代码实例展示其用法。

### 什么是反射？

指程序运行时 加载、获取一个未知类（已知类名）及类属性和方法的java技术机理。加载完成后会在虚拟机中产生一个class对象，一个类只有一个class对象，这个class对象包含了类的相关结构信息。
这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

### Class对象如何获取？

* 1. 运用Class.forName(String className)动态加载类；
* 2. .class 属性（轻量级）；
* 3. getClass()方法；

### 对象的创建

* 使用Class对象的newInstance()方法来创建该Class对象对应类的实例，这种方式要求该Class对象的对应类有默认构造器。
* 先使用Class对象获取指定的Constructor对象, 再调用Constructor对象的newInstance()。通过这种方式可以选择指定的构造器来创建实例。

```
//Class 类 newInstance源码
  @CallerSensitive
    public T newInstance()
        throws InstantiationException, IllegalAccessException
    {
        if (System.getSecurityManager() != null) {
            checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
        }

        // NOTE: the following code may not be strictly correct under
        // the current Java memory model.

        // Constructor lookup
        if (cachedConstructor == null) {
            if (this == Class.class) {
                throw new IllegalAccessException(
                    "Can not call newInstance() on the Class for java.lang.Class"
                );
            }
            try {
                Class<?>[] empty = {};
                final Constructor<T> c = getConstructor0(empty, Member.DECLARED);//默认构造器
                // Disable accessibility checks on the constructor
                // since we have to do the security check here anyway
                // (the stack depth is wrong for the Constructor's
                // security check to work)
                java.security.AccessController.doPrivileged(
                    new java.security.PrivilegedAction<Void>() {
                        public Void run() {
                                c.setAccessible(true);
                                return null;
                            }
                        });
                cachedConstructor = c;
            } catch (NoSuchMethodException e) {
                throw (InstantiationException)
                    new InstantiationException(getName()).initCause(e);
            }
        }
        Constructor<T> tmpConstructor = cachedConstructor;
        // Security check (same as in java.lang.reflect.Constructor)
        int modifiers = tmpConstructor.getModifiers();
        if (!Reflection.quickCheckMemberAccess(this, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            if (newInstanceCallerCache != caller) {
                Reflection.ensureMemberAccess(caller, this, null, modifiers);
                newInstanceCallerCache = caller;
            }
        }
        // Run constructor
        try {
            return tmpConstructor.newInstance((Object[])null);
        } catch (InvocationTargetException e) {
            Unsafe.getUnsafe().throwException(e.getTargetException());
            // Not reached
            return null;
        }
    }
```

### 相关Api相关实例

先定义一个bean类作为反射对象

```
 public class CankingBean extends Canking implements ICanking {
    private String name;
    private int id;

    public CankingBean() {
        //Nothing
    }

    public CankingBean(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String n) {
        this.name = n;
    }

    public int getId() {
        return id;
    }

    public void setId(int i) {
        this.id = i;

    }

    @Override
    public void createApp() {
        //nothing
    }
 }
```

#### 1.对象实例化


```
	 CankingBean bean = null, bean1 = null;

        Class test = null;
        try {
            test = Class.forName("com.canking.CankingBean");
        } catch (Exception e) {
            e.printStackTrace();
        }

        Constructor cons[] = test.getConstructors();
        System.out.println("cons.len:" + cons.length + " name:" + test.getName() + " simpleName:" + test.getSimpleName() + " Canonical:" + test.getCanonicalName() + " type:" + test.getTypeName());


        try {
            bean = (CankingBean) cons[0].newInstance();
            bean1 = (CankingBean) cons[1].newInstance(1, "canking");
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("bean.id:" + bean.getId() + " bean.name:" + bean.getName() + " b1.id:" + bean1.getId() + " b1.name:" + bean1.getName());

```


#### 2. 获取实现接口列表

```
 	Class<?> intes[]=test.getInterfaces();  
```

#### 3.获取父类

```
 Class<?> temp=demo.getSuperclass();  
```

#### 4.获取类属性

```
  Field[] fields = test.getDeclaredFields();//本类属性
        for (Field f : fields) {
            int modifier = f.getModifiers();//修饰符
            String modiStr = Modifier.toString(modifier);

            Class type = f.getType();
            System.out.println(f.getName() + " || modifier:" + modiStr + " type:" + type.getName());
        }
        System.out.println("\n");

        Field[] fieldsAll = test.getFields();//接口中属性
        for (Field f : fieldsAll) {
            int modifier = f.getModifiers();//修饰符
            String modiStr = Modifier.toString(modifier);

            Class type = f.getType();
            System.out.println(f.getName() + " || modifier:" + modiStr + " type:" + type.getName());
        }
```

想要获取父类中属性怎么办？getSuperclass()先获取父类实例，然后调用getDeclaredFields（）获取。可以依次遍历super类，获得全部继承关系中的属性。


#### 5.获取类方法


```
  Method method[] = test.getMethods(); //获取类及继承关系中的所有类方法
        for(Method m : method){
            Class<?> returnType = m.getReturnType();
            Class<?> para[] = m.getParameterTypes();
            int temp = m.getModifiers();

            System.out.println(m.getName()+" returnType:"+returnType+" modifiers:"+Modifier.toString(temp));
            for(Class c : para){
                System.out.println(" c name:"+c.getName());
            }
        }
```


#### 6.方法及属性的操作


```
 	   try {
            Method ms = test.getMethod("setName", String.class);
            Method mg = test.getMethod("getName");
            Object obj = test.newInstance();
            ms.invoke(obj, "New name");
            System.out.println("name:" + mg.invoke(obj));

            Field canking = test.getDeclaredField("name");
            canking.setAccessible(true);//禁用权限检测
            canking.set(obj, "Name Modified!");
            System.out.println("name:" + canking.get(obj));
        } catch (Exception e) {
            e.printStackTrace();
        }
```


## 二、代理

### 什么是java代理模式？

代理用到类反射的知识，这里进一步单独讲解加深理解。

对某个对象提供一个代理对象，使得我们可以通过代理对象间接的操作原对象。

> **静态代理**：代码层通过设计类获得一个中间层代理，是在编译时已经实现好的，可以说是一个class文件。
  **动态代理**：代理类是在运行时由java机理产生。起到动态调用和操作代码的目的。

由于静态代理比较简单，易于理解，本文主要讲解动态代理。

### 基本概念

**InvocationHandler接口**
代理类调用任何方法都会经过这个调用处理器类的invoke方法。

**Proxy**
主要用于产生代理类，通过 Proxy 类生成的代理类都继承了 Proxy 类。newProxyInstance方法封装了获取代理对象。

*newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h);*方法传如的参数可以看出，代理一个对象，必须要要满足这个对象实现一个接口。

### 动态代理实现方法

个人学习习惯，学习新知识，先了解大概，然后就是代码，最后看原理，有精力的话就写点东西分享出来。这里就先看下代码如何实现一个动态代理。

只要一个类满足代理条件，即可实现代理。

```
 public class CankingProxyHandler implements InvocationHandler {
    private Object object;
    public CankingProxyHandler(Object obj){
        this.object= obj;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Proxy method in");
        if ("setName".equals(method.getName())) {
            return method.invoke(object, "Proxy");
        }
        return method.invoke(object, args);
    }
 }
```

```
        CankingProxyHandler proxyHandler = new CankingProxyHandler(bean1);
        ICanking cProxy = (ICanking) Proxy.newProxyInstance(CankingBean.class.getClassLoader(), CankingBean.class.getInterfaces(), proxyHandler);
        cProxy.createApp();
```

代理类的每次方法调用，都会触发proxyHandler的invoke方法，这样我们就可以在此时做一些自己的事情。

android系统中有些类也是满足代理条件的，这样我们同样可以做一个代理类，来实现hook系统部分方法的目的。


### 实现原理

首先来看下newProxyInstance源码

```
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs); //代理类生成关键方法

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```

可以看出getProxyClass0()方法为生成代理类关键方法

```
    /**
     * Generate a proxy class.  Must call the checkProxyAccess method
     * to perform permission checks before calling this.
     */
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```

看到源码注释，如果代理类在传入的classloader中已经加载则直接返回，否则会在ProxyClassFactory中生成。

proxyClassCache.get方法中有以下代码

```
        // create subKey and retrieve the possible Supplier<V> stored by that
        // subKey from valuesMap
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
        Supplier<V> supplier = valuesMap.get(subKey);
```

结合代码和注释可以大概看出*subKeyFactory.apply(key, parameter)*返回的就是代理类


那我们具体看看apply是怎么生产我们的代理类的：

```
        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);//加载原始接口
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }

            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
            for (Class<?> intf : interfaces) {//解析原始接口
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;//构造代理类名

            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);//生产代理类
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
```

最终ProxyGenerator.generateProxyClass方法中产生类代理类并写入磁盘。这种操作代码的代码java总称为**元编程**，java SOURCE级别的注解中也会大量用到。

代理类最终生产的样子如下：

```
public final class $Proxy1 extends Proxy implements Subject{
    private InvocationHandler h;
    private $Proxy1(){}
    public $Proxy1(InvocationHandler h){
        this.h = h;
    }
    public int request(int i){
        Method method = Subject.class.getMethod("method", new Class[]{int.class});    //创建method对象
        return (Integer)h.invoke(this, method, new Object[]{new Integer(i)}); //调用了invoke方法
    }
}
```

从代理来结构，可以看出为什么每次方法的调用都会掉到invocationHandler的invoke方法。

——————
欢迎转载，请标明出处：常兴E站 [www.canking.win](http://www.canking.win)