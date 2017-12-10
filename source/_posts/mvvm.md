---
title: Lifecycle+Retrofit+Room完美结合 领略架构之美
date: 2017-12-09 15:37:25
categories: android 
tags: android
---

安卓开发技术发展到现在已经非常成熟，有很多的技术专项如插件，热修，加固，瘦身，性能优化，自动化测试等已经在业界有了完善的或者开源的解决方案。
作为一枚多年的安卓研发，有必要学习或了解下这些优秀的解决方案，领略那些行业*开创者*的思想魅力，然后转化为自己的技术技能，争取应用到日常的开发中去，提高自己研发水平。

<!--more-->

# Lifecycle+Retrofit+Room 云端漫步飞一般的感觉

安卓项目的开发结构，有原来最初的mvc，到后来有人提出的mvp，然后到mvvm的发展，无非就是依着*六大设计原则*的不断解耦，不断演变，使得项目的开发高度组件化，满足日常复杂多变的项目需求。

* 依赖倒置原则－Dependency Inversion Principle (DIP) 
* 里氏替换原则－Liskov Substitution Principle (LSP) 
* 接口分隔原则－Interface Segregation Principle (ISP) 
* 单一职责原则－Single Responsibility Principle (SRP) 
* 开闭原则－The Open-Closed Principle (OCP)

目前针对MVVM框架结构，[安卓官方](https://developer.android.google.cn/topic/libraries/architecture/adding-components.html)也给出了稳定的架构版本1.0。

本文也是围绕者官方思想，试着从*源码角度总结*学习经验，最后将这些控件二次封装，更加便于理解使用。其也会涉及到其他优秀的的库，如Gson,Glide,BaseRecyclerViewAdapterHelper等


## 一.起因
去年还在手机卫士团队做垃圾清理模块时候，赶上模块化二次代码重构技术需求，项目分为多个进程，其中后台进程负责从DB中获取数据（DB可云端更新升级），然后结合云端配置信息做相关逻辑操作，将数据回传到UI进程，UI进程中在后台线程中二次封装，最后post到主UI线程Update到界面展示给用户。

当时就和团队的同学沟通交流，面对这种数据多层次复杂的处理逻辑，设想可以做一种机制，将UI层的更新绑定到一个数据源，数据源数据的更新可以自动触发UI更新，实现与UI的解耦。
数据源控制着数据的来源，每种来源有着独立的逻辑分层，共享底层一些公共lib库。

后来想想，其实差不多就是MVVM思想，直到谷歌官方宣布[Android 架构组件 1.0 稳定版](https://mp.weixin.qq.com/s/9rC_5GhdAA_EMEbWKJT5vQ)的发布，才下决心学习下官方这一套思想，感受优秀的架构。


## 二.各组件库原理及基本用法

这里主要探究下主要组件库的基本用法和原理，以理解其优秀思想为主。

### 谷歌官方Android Architecture Components

**Lifecycle+LiveData+ViewMode+Room** 

>A collection of libraries that help you design robust, testable, and maintainable apps. Start with classes for managing your UI component lifecycle and handling data persistence.

#### Lifecyle

一个安卓组件生命周期感知回调的控件，可以感知activity或fragment的生命周期变化，并回调相应接口。

可以先从源码抽象类分析，如下：

```
public abstract class Lifecycle {
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract State getCurrentState();

    @SuppressWarnings("WeakerAccess")
    public enum Event {

        ON_CREATE,
 
        ON_START,

        ON_RESUME,

        ON_PAUSE,

        ON_STOP,

        ON_DESTROY,

        ON_ANY
    }

    @SuppressWarnings("WeakerAccess")
    public enum State {

        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}

/**
 * Marks a class as a LifecycleObserver. It does not have any methods, instead, relies on
 * {@link OnLifecycleEvent} annotated methods.
 * <p>
 * @see Lifecycle Lifecycle - for samples and usage patterns.
 */
@SuppressWarnings("WeakerAccess")
public interface LifecycleObserver {

}


```


*Lifecycle*抽象类有三个方法，可以添加、删除观察者,并且可以获取当前组件的生命周期的状态。
而*LifecycleObserver*接口只是个空接口做类型校验，具体事件交给了OnLifecycleEvent，通过注入回调相应事件。

二在最新的support V4包中，*Activity*和*Fragment*都实现了*LifecycleOwner*接口，意味着我们可以直接使用getLifecyle()获取当前*Activity*或*Fragment*的Lifecycle对象，很方便的添加我们的监听方法。

```
@SuppressWarnings({"WeakerAccess", "unused"})
public interface LifecycleOwner {

    @NonNull
    Lifecycle getLifecycle();
}
```

#### LiveData 
LiveData是一个持有范型类型的数据组件，将App生命组件与数据关联到一起，可以感知生命变更，规则回调数据变化。

```
public abstract class LiveData<T> {
	        private volatile Object mData = NOT_SET;//数据容器类

    //一个写死的处于Resume状态的LifecycleOwner，用于数据回调无限制情况
    private static final LifecycleOwner ALWAYS_ON = new LifecycleOwner() {

        private LifecycleRegistry mRegistry = init();

        private LifecycleRegistry init() {
            LifecycleRegistry registry = new LifecycleRegistry(this);
            registry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
            registry.handleLifecycleEvent(Lifecycle.Event.ON_START);
            registry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
            return registry;
        }

        @Override
        public Lifecycle getLifecycle() {
            return mRegistry;
        }
    };
    private SafeIterableMap<Observer<T>, LifecycleBoundObserver> mObservers =
            new SafeIterableMap<>();
   
    private void considerNotify(LifecycleBoundObserver observer) {
        if (!observer.active) {
            return;
        }

        if (!isActiveState(observer.owner.getLifecycle().getCurrentState())) {
            observer.activeStateChanged(false);
            return;
        }
        if (observer.lastVersion >= mVersion) {
            return;
        }
        observer.lastVersion = mVersion;
        //noinspection unchecked
        observer.observer.onChanged((T) mData);//最终的回调方法地方
    }

    private void dispatchingValue(@Nullable LifecycleBoundObserver initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<T>, LifecycleBoundObserver>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());//可以添加做个监听者，这里遍历分发数据到每个监听者
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }

    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        LifecycleBoundObserver existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && existing.owner != wrapper.owner) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }

    @MainThread
    public void observeForever(@NonNull Observer<T> observer) {
        observe(ALWAYS_ON, observer);
    }

    @MainThread
    public void removeObserver(@NonNull final Observer<T> observer) {
        assertMainThread("removeObserver");
        LifecycleBoundObserver removed = mObservers.remove(observer);
        if (removed == null) {
            return;
        }
        removed.owner.getLifecycle().removeObserver(removed);
        removed.activeStateChanged(false);
    }

    //实现类回调方法
    protected void onActive() {

    }

    //实现类回调方法
    protected void onInactive() {

    }

    class LifecycleBoundObserver implements GenericLifecycleObserver {

        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (owner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(observer);
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            activeStateChanged(isActiveState(owner.getLifecycle().getCurrentState()));
        }

        void activeStateChanged(boolean newActive) {
            if (newActive == active) {
                return;
            }
            active = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += active ? 1 : -1;
            if (wasInactive && active) {
                onActive();
            }
            if (LiveData.this.mActiveCount == 0 && !active) {
                onInactive();
            }
            if (active) {//只有生命组件处于前台时，才触发数据的变更通知
                dispatchingValue(this);
            }
        }
    }

    static boolean isActiveState(State state) {
        return state.isAtLeast(STARTED);
    }
}

```

看源码，会发现LiveData有个重要的方法**(observe(LifecycleOwner owner, Observer<T> observer)**, 在数据源数据有变更时，遍历分发数据到所有监听者，最后会回调onChanged()方法。


```
public interface Observer<T> {
    /**
     * Called when the data is changed.
     * @param t  The new data
     */
    void onChanged(@Nullable T t);
}
```

LiveData有两个实现类：*MediatorLiveData*和*MediatorLiveData*，继承关系如下：

[LiveData](arch2.png)

*MutableLiveData*类只是暴露了两个方法：postData()和setData()。
*MediatorLiveData*类有个**addSource()**方法，可以实现监听另一个或多个LiveData数据源变化，这样我们就可以比较便捷且底耦合的实现多个数据源的逻辑，并且关联到一个MediatorLiveData上，实现多数据源的自动整合。

```
    @MainThread
    public <S> void addSource(@NonNull LiveData<S> source, @NonNull Observer<S> onChanged) {
        Source<S> e = new Source<>(source, onChanged);
        Source<?> existing = mSources.putIfAbsent(source, e);
        if (existing != null && existing.mObserver != onChanged) {
            throw new IllegalArgumentException(
                    "This source was already added with the different observer");
        }
        if (existing != null) {
            return;
        }
        if (hasActiveObservers()) {
            e.plug();
        }
    }
```

#### ViewModel

LiveData和LiveCycle将数据与数据，数据与UI生命绑定到了一起，实现了数据的自动管理和更新，那边这些数据如何保存呢？能否在多个页面共享这些数据呢？答案是ViewMode。

>A ViewModel is always created in association with a scope (an fragment or an activity) and will be retained as long as the scope is alive. E.g. if it is an Activity, until it is finished.

ViewMode相当于一层数据隔离层，将UI层的数据逻辑全部抽离干净，管理制底层数据的获取方式和逻辑。


```
         ViewModel   viewModel = ViewModelProviders.of(this).get(xxxModel.class);
         
         ViewModel   viewModel = ViewModelProviders.of(this, factory).get(xxxModel.class);
```

可以通过以上方式获取ViewModel实例，如果有自定义ViewModel构造器参数，需要借助**ViewModelProvider.NewInstanceFactory**，自己实现create方法。

那么，ViewMode是怎么被保存的呢？ 可以顺着ViewModelProviders源码进去看看。

```
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            //noinspection unchecked
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }

        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
```

发现get方法会先从缓存中获取，没有的化就会通过*Factory*的create方法构造一个ViewModel，然后放入缓存，下次直接使用。

#### Room
Room是一种ORM（对象关系映射）模式数据库框架，对安卓SQlite的抽象封装，从此操作数据库提供了超便捷方式。

>The Room persistence library provides an abstraction layer over SQLite to allow fluent database access while harnessing the full power of SQLite.

同样基于ORM模式封装的数据库，比较有名还有*GreenDao*，而Room和其他ORM对比，具有编译时验证查询语句正常性，支持LiveData数据返回等优势。
我们选择room，更多的时完美的支持LiveData，可以动态的将数据变化自动更新到LiveData上，在通过LiveData自动刷新到UI上。

这里引用网络上的一张Room与其他同类性能对比图片：

[性能对比](arch3.png)

*用法：*

1. 继承RoomDatabase的抽象类, 暴露抽象方法getxxxDao()。

```
@Database(entities = {EssayDayEntity.class, ZhihuItemEntity.class}, version = 1)
@TypeConverters(DateConverter.class)
public abstract class AppDB extends RoomDatabase {
    private static AppDB sInstance;

    @VisibleForTesting
    public static final String DATABASE_NAME = "canking.db";
    public abstract EssayDao essayDao();
 }
```

2. 获取db实例

```
ppDatabase db = Room.databaseBuilder(getApplicationContext(),
        AppDatabase.class, "database-name").build();
```

3. 实现Dao层逻辑

```
@Dao
public interface ZhuhuDao {
    @Query("SELECT * FROM zhuhulist  order by id desc, id limit 0,1")
    LiveData<ZhihuItemEntity> loadZhuhu();

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    void insertItem(ZhihuItemEntity products);
}
```

4. 添加一张表结构

```
@Entity
public class User {
    @PrimaryKey
    private int uid;

    @ColumnInfo(name = "first_name")
    private String firstName;

    public String date;//默认columnInfo 为 date

}
```

就这么简单，就可以实现数据库的操作，完全隔离的底层复杂的数据库操作，大大节省项目研发重复劳动力。

从使用说明分析，UserDao和Db继承要不是接口，要不是抽象，那么这些逻辑的实现完全是由annotationProcessor依赖注入，帮我们实现的。
那么编译项目后，可以在build目录下看到生成相应的类xxx_impl.class。
[impl](arch4.png)

既然Room支持LiveData数据，那么有可以分析下源码,了解下具体原理，方便以后填坑。

先选Demo中Dao层的query方法，看看数据如何加载到内存的。我们的query方法如下：

```
    @Query("SELECT * FROM zhuhulist  order by id desc, id limit 0,1")
    LiveData<ZhihuItemEntity> loadZhuhu();
```

annotationProcessor帮我吗生成后的实现为：

```
 public LiveData<ZhihuItemEntity> loadZhuhu() {
        String _sql = "SELECT * FROM zhuhulist  order by id desc, id limit 0,1";
        final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire("SELECT * FROM zhuhulist  order by id desc, id limit 0,1", 0);
        return (new ComputableLiveData<ZhihuItemEntity>() {
            private Observer _observer;

            protected ZhihuItemEntity compute() {
                if(this._observer == null) {
                    this._observer = new Observer("zhuhulist", new String[0]) {
                        public void onInvalidated(@NonNull Set<String> tables) {
                            invalidate();
                        }
                    };
                    ZhuhuDao_Impl.this.__db.getInvalidationTracker().addWeakObserver(this._observer);
                }

                Cursor _cursor = ZhuhuDao_Impl.this.__db.query(_statement);

                ZhihuItemEntity var13;
                try {
                    int _cursorIndexOfId = _cursor.getColumnIndexOrThrow("id");
                    int _cursorIndexOfDate = _cursor.getColumnIndexOrThrow("date");
                    int _cursorIndexOfStories = _cursor.getColumnIndexOrThrow("stories");
                    int _cursorIndexOfTopStories = _cursor.getColumnIndexOrThrow("top_stories");
                    ZhihuItemEntity _result;
                    if(_cursor.moveToFirst()) {
                        _result = new ZhihuItemEntity();
                        int _tmpId = _cursor.getInt(_cursorIndexOfId);
                        _result.setId(_tmpId);
                        _result.date = _cursor.getString(_cursorIndexOfDate);
                        String _tmp = _cursor.getString(_cursorIndexOfStories);
                        _result.stories = DateConverter.toString(_tmp);
                        String _tmp_1 = _cursor.getString(_cursorIndexOfTopStories);
                        _result.top_stories = DateConverter.toString(_tmp_1);
                    } else {
                        _result = null;
                    }

                    var13 = _result;
                } finally {
                    _cursor.close();
                }

                return var13;
            }

            protected void finalize() {
                _statement.release();
            }
        }).getLiveData();
    }
```

实现类中通过











