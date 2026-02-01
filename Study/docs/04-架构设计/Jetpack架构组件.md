# Jetpack 架构组件深度解析

## 1. 概述

Jetpack 架构组件是 Google 推出的一套帮助开发者构建健壮、可测试、可维护应用的库集合。核心组件包括：

| 组件 | 作用 | 核心类 |
|------|------|--------|
| **ViewModel** | 管理界面相关数据，配置变更时保持数据 | ViewModelStore、ViewModelProvider |
| **LiveData** | 生命周期感知的可观察数据持有者 | MutableLiveData、MediatorLiveData |
| **Lifecycle** | 生命周期感知组件的基础 | LifecycleOwner、LifecycleObserver |
| **SavedStateHandle** | 进程死亡后恢复数据 | SavedStateRegistry |
| **DataBinding** | 声明式绑定 UI 组件与数据源 | ViewDataBinding |
| **Room** | SQLite 抽象层 | RoomDatabase、DAO |
| **Paging** | 分页加载数据 | PagingSource、Pager |

### 架构组件关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         UI Layer                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │  Activity   │    │  Fragment   │    │  Compose Screen     │  │
│  │(LifecycleOwner)  │(LifecycleOwner)  │  (LifecycleOwner)   │  │
│  └──────┬──────┘    └──────┬──────┘    └──────────┬──────────┘  │
│         │                  │                      │              │
│         └──────────────────┼──────────────────────┘              │
│                            │ observe                             │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    ViewModel Layer                          ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ ││
│  │  │  ViewModel  │  │  LiveData   │  │  SavedStateHandle   │ ││
│  │  │             │──│             │──│                     │ ││
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘ ││
│  └─────────────────────────────────────────────────────────────┘│
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Data Layer                               ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ ││
│  │  │ Repository  │──│    Room     │──│      Paging         │ ││
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘ ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```


---

## 2. 核心原理

### 2.1 Lifecycle 原理

Lifecycle 是整个架构组件的基石，提供了生命周期感知能力。

#### 2.1.1 核心类关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    Lifecycle 核心类关系                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐         ┌─────────────────────────────┐    │
│  │ LifecycleOwner  │         │    LifecycleObserver        │    │
│  │   (接口)        │         │       (接口)                │    │
│  │                 │         │                             │    │
│  │ + getLifecycle()│         │ 标记方法监听生命周期事件     │    │
│  └────────┬────────┘         └──────────────┬──────────────┘    │
│           │                                 │                    │
│           │ 实现                            │ 实现               │
│           ▼                                 ▼                    │
│  ┌─────────────────┐         ┌─────────────────────────────┐    │
│  │ComponentActivity│         │  DefaultLifecycleObserver   │    │
│  │   Fragment      │         │  LifecycleEventObserver     │    │
│  └────────┬────────┘         └─────────────────────────────┘    │
│           │                                                      │
│           │ 持有                                                 │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              LifecycleRegistry                          │    │
│  │                                                         │    │
│  │  - mState: State (当前状态)                             │    │
│  │  - mObserverMap: FastSafeIterableMap (观察者集合)       │    │
│  │                                                         │    │
│  │  + addObserver(observer)                                │    │
│  │  + removeObserver(observer)                             │    │
│  │  + handleLifecycleEvent(event)                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 生命周期状态与事件

```
                    ON_CREATE      ON_START       ON_RESUME
    ┌───────────┐ ──────────► ┌─────────┐ ────────► ┌─────────┐ ────────► ┌─────────┐
    │INITIALIZED│             │ CREATED │           │ STARTED │           │ RESUMED │
    └───────────┘ ◄────────── └─────────┘ ◄──────── └─────────┘ ◄──────── └─────────┘
                   ON_DESTROY    ON_STOP            ON_PAUSE
    
    ┌───────────┐
    │ DESTROYED │
    └───────────┘
```

**状态与事件对应关系：**
- `INITIALIZED` → `ON_CREATE` → `CREATED`
- `CREATED` → `ON_START` → `STARTED`
- `STARTED` → `ON_RESUME` → `RESUMED`
- `RESUMED` → `ON_PAUSE` → `STARTED`
- `STARTED` → `ON_STOP` → `CREATED`
- `CREATED` → `ON_DESTROY` → `DESTROYED`


### 2.2 ViewModel 原理

#### 2.2.1 ViewModel 存储机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ViewModel 存储架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────┐                                                   │
│   │ ViewModelProvider│ ◄─── 创建/获取 ViewModel 的入口                  │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            │ 使用                                                        │
│            ▼                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    ViewModelStore                                │   │
│   │  ┌─────────────────────────────────────────────────────────┐   │   │
│   │  │  HashMap<String, ViewModel> mMap                        │   │   │
│   │  │                                                         │   │   │
│   │  │  Key: "androidx.lifecycle.ViewModelProvider.DefaultKey" │   │   │
│   │  │       + ViewModel.class.getCanonicalName()              │   │   │
│   │  │                                                         │   │   │
│   │  │  Value: ViewModel 实例                                  │   │   │
│   │  └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                           │
│                              │ 存储于                                    │
│                              ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              ViewModelStoreOwner                                 │   │
│   │  (ComponentActivity / Fragment 实现此接口)                       │   │
│   │                                                                  │   │
│   │  + getViewModelStore(): ViewModelStore                          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 配置变更时 ViewModel 保留机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    配置变更时 ViewModel 保留流程                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  配置变更前 (如屏幕旋转)                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Activity (旧实例)                                               │    │
│  │    │                                                            │    │
│  │    ├── ViewModelStore ──► ViewModel (保留)                      │    │
│  │    │         │                                                  │    │
│  │    │         │ 通过 NonConfigurationInstances 传递              │    │
│  │    │         ▼                                                  │    │
│  │    │   ActivityClientRecord.lastNonConfigurationInstances       │    │
│  │    │                                                            │    │
│  └────┼────────────────────────────────────────────────────────────┘    │
│       │                                                                  │
│       │ onRetainNonConfigurationInstance()                              │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Activity (新实例)                                               │    │
│  │    │                                                            │    │
│  │    ├── getLastNonConfigurationInstance()                        │    │
│  │    │         │                                                  │    │
│  │    │         ▼                                                  │    │
│  │    ├── ViewModelStore (恢复) ──► ViewModel (同一实例)           │    │
│  │    │                                                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```


### 2.3 LiveData 原理

#### 2.3.1 LiveData 核心机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LiveData 观察者分发机制                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      LiveData<T>                                 │   │
│   │                                                                  │   │
│   │  - mData: Object (数据)                                         │   │
│   │  - mVersion: int (数据版本号，初始 -1)                          │   │
│   │  - mObservers: SafeIterableMap<Observer, ObserverWrapper>       │   │
│   │                                                                  │   │
│   │  + observe(owner, observer)  // 生命周期感知                    │   │
│   │  + observeForever(observer)  // 永久观察                        │   │
│   │  + setValue(value)           // 主线程设值                      │   │
│   │  + postValue(value)          // 任意线程设值                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                           │
│                              │ 包装                                      │
│                              ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │              LifecycleBoundObserver                              │   │
│   │                                                                  │   │
│   │  - mOwner: LifecycleOwner                                       │   │
│   │  - mLastVersion: int (观察者版本号，初始 -1)                    │   │
│   │                                                                  │   │
│   │  + shouldBeActive(): boolean                                    │   │
│   │    // 返回 owner.lifecycle.currentState.isAtLeast(STARTED)      │   │
│   │                                                                  │   │
│   │  + onStateChanged(source, event)                                │   │
│   │    // 生命周期变化时检查是否需要分发数据                        │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2.3.2 LiveData 数据分发流程

```
setValue(value) 调用流程：

┌──────────────────┐
│  setValue(value) │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│ 1. 检查是否在主线程          │
│    assertMainThread()        │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ 2. mVersion++ (版本号自增)   │
│    mData = value             │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ 3. dispatchingValue(null)    │
│    遍历所有观察者            │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. considerNotify(observer)                                  │
│                                                              │
│    if (!observer.mActive) return;  // 非活跃状态不分发       │
│    if (!observer.shouldBeActive()) {                         │
│        observer.activeStateChanged(false);                   │
│        return;                                               │
│    }                                                         │
│    if (observer.mLastVersion >= mVersion) return; // 已分发  │
│    observer.mLastVersion = mVersion;                         │
│    observer.mObserver.onChanged(mData);  // 回调观察者       │
└──────────────────────────────────────────────────────────────┘
```


#### 2.3.3 LiveData 粘性事件问题

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LiveData 粘性事件问题分析                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  问题场景：                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. ViewModel 中 LiveData 已有数据 (mVersion = 0)                │    │
│  │ 2. 新 Fragment 订阅该 LiveData                                  │    │
│  │ 3. 新观察者 mLastVersion = -1 < mVersion = 0                    │    │
│  │ 4. 立即收到"旧"数据 → 粘性事件                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  根本原因：                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ observer.mLastVersion >= mVersion 判断                          │    │
│  │                                                                  │    │
│  │ 新观察者 mLastVersion 初始值为 START_VERSION(-1)                │    │
│  │ 而 LiveData 的 mVersion 可能已经 > -1                           │    │
│  │ 导致 considerNotify() 会分发数据给新观察者                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  解决方案：                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 方案1: SingleLiveEvent (Google 官方示例)                        │    │
│  │        - 使用 AtomicBoolean 标记是否已消费                      │    │
│  │        - 缺点：只支持一个观察者                                 │    │
│  │                                                                  │    │
│  │ 方案2: Event 包装类                                             │    │
│  │        - 包装数据，标记是否已处理                               │    │
│  │        - 支持多观察者                                           │    │
│  │                                                                  │    │
│  │ 方案3: 反射修改 mLastVersion                                    │    │
│  │        - 订阅时将 observer.mLastVersion = mVersion              │    │
│  │        - 不推荐，依赖内部实现                                   │    │
│  │                                                                  │    │
│  │ 方案4: 使用 Flow (推荐)                                         │    │
│  │        - SharedFlow/StateFlow 可配置 replay                     │    │
│  │        - replay = 0 即无粘性                                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```


### 2.4 SavedStateHandle 原理

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SavedStateHandle 工作原理                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    SavedStateHandle                              │    │
│  │                                                                  │    │
│  │  - regular: Map<String, Object> (普通数据)                      │    │
│  │  - savedStateProviders: Map<String, SavedStateProvider>         │    │
│  │  - liveDatas: Map<String, MutableLiveData>                      │    │
│  │                                                                  │    │
│  │  + get<T>(key): T?                                              │    │
│  │  + set(key, value)                                              │    │
│  │  + getLiveData<T>(key): MutableLiveData<T>                      │    │
│  │  + getStateFlow<T>(key, initialValue): StateFlow<T>             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              │ 由 SavedStateHandleController 管理        │
│                              ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │              SavedStateRegistry                                  │    │
│  │                                                                  │    │
│  │  - components: Map<String, SavedStateProvider>                  │    │
│  │  - restoredState: Bundle? (恢复的状态)                          │    │
│  │                                                                  │    │
│  │  + registerSavedStateProvider(key, provider)                    │    │
│  │  + consumeRestoredStateForKey(key): Bundle?                     │    │
│  │  + performSave(outBundle)                                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              │ 关联                                      │
│                              ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │              SavedStateRegistryOwner                             │    │
│  │              (ComponentActivity / Fragment 实现)                 │    │
│  │                                                                  │    │
│  │  + getSavedStateRegistry(): SavedStateRegistry                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  数据保存恢复流程：                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  onSaveInstanceState()                                          │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  SavedStateRegistry.performSave(outBundle)                      │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  遍历所有 SavedStateProvider，收集数据到 Bundle                 │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  进程被杀死...                                                   │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  onCreate(savedInstanceState)                                   │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  SavedStateRegistry.performRestore(savedInstanceState)          │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  SavedStateHandle 从 restoredState 恢复数据                     │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```


### 2.5 DataBinding 原理

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DataBinding 编译时处理流程                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  编译阶段：                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  activity_main.xml (含 <layout> 标签)                           │    │
│  │         │                                                        │    │
│  │         │ DataBinding 编译器处理                                 │    │
│  │         ▼                                                        │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │ 1. 生成 activity_main-layout.xml (纯布局，无绑定表达式) │    │    │
│  │  │ 2. 生成 ActivityMainBinding.java (绑定类)               │    │    │
│  │  │ 3. 生成 BR.java (绑定变量 ID)                           │    │    │
│  │  │ 4. 生成 DataBinderMapperImpl.java (映射器)              │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  运行时绑定：                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  DataBindingUtil.setContentView(activity, layoutId)             │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  DataBinderMapper.getDataBinder(bindingComponent, view, layoutId)│   │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  ActivityMainBindingImpl 实例化                                 │    │
│  │         │                                                        │    │
│  │         ├── 通过 tag 找到所有绑定的 View                        │    │
│  │         ├── 建立 View 与变量的绑定关系                          │    │
│  │         └── 注册 Observable 监听器                              │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  数据更新流程：                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  binding.setVariable(BR.user, user)                             │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  mDirtyFlags |= 对应位标记                                      │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  requestRebind() → mRebindRunnable                              │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  executeBindings() (在 Choreographer 下一帧执行)                │    │
│  │         │                                                        │    │
│  │         ▼                                                        │    │
│  │  根据 mDirtyFlags 更新对应 View                                 │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```


### 2.6 Room 数据库原理

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Room 架构设计                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    @Database                                     │    │
│  │              AppDatabase (抽象类)                                │    │
│  │                      │                                           │    │
│  │                      │ 编译时生成                                │    │
│  │                      ▼                                           │    │
│  │              AppDatabase_Impl                                    │    │
│  │                      │                                           │    │
│  │          ┌───────────┼───────────┐                              │    │
│  │          ▼           ▼           ▼                              │    │
│  │    ┌─────────┐ ┌─────────┐ ┌─────────┐                         │    │
│  │    │UserDao  │ │OrderDao │ │ProductDao│                         │    │
│  │    │(接口)   │ │(接口)   │ │(接口)    │                         │    │
│  │    └────┬────┘ └────┬────┘ └────┬────┘                         │    │
│  │         │           │           │                               │    │
│  │         ▼           ▼           ▼                               │    │
│  │    ┌─────────┐ ┌─────────┐ ┌─────────┐                         │    │
│  │    │UserDao  │ │OrderDao │ │ProductDao│                         │    │
│  │    │_Impl    │ │_Impl    │ │_Impl     │                         │    │
│  │    └─────────┘ └─────────┘ └─────────┘                         │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              │ 底层                                      │
│                              ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    SupportSQLiteDatabase                         │    │
│  │                           │                                      │    │
│  │                           ▼                                      │    │
│  │                    FrameworkSQLiteDatabase                       │    │
│  │                           │                                      │    │
│  │                           ▼                                      │    │
│  │                    SQLiteDatabase (Android 原生)                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Room 编译时处理：
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  @Entity User.java ──────► 生成建表 SQL                                 │
│                                                                          │
│  @Dao UserDao.java ──────► 生成 UserDao_Impl.java                       │
│       │                         │                                        │
│       │ @Query                  │ 编译时 SQL 语法检查                    │
│       │ @Insert                 │ 生成对应 SQL 执行代码                  │
│       │ @Update                 │ 参数类型检查                           │
│       │ @Delete                 │ 返回类型适配                           │
│       │                         │                                        │
│       │ @Query + LiveData ────► 自动注册 InvalidationTracker            │
│       │ @Query + Flow ────────► 自动转换为 Flow                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```


### 2.7 Paging 分页库原理

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Paging 3 架构设计                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                         UI Layer                                 │    │
│  │                                                                  │    │
│  │  ┌─────────────────┐      ┌─────────────────────────────────┐   │    │
│  │  │ RecyclerView    │ ◄─── │ PagingDataAdapter               │   │    │
│  │  │                 │      │   - differ: AsyncPagingDataDiffer│  │    │
│  │  └─────────────────┘      └─────────────────────────────────┘   │    │
│  │                                      │                          │    │
│  │                                      │ submitData(pagingData)   │    │
│  │                                      ▼                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      ViewModel Layer                             │    │
│  │                                                                  │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │                    Pager                                 │    │    │
│  │  │                                                          │    │    │
│  │  │  - config: PagingConfig (pageSize, prefetchDistance等)  │    │    │
│  │  │  - pagingSourceFactory: () -> PagingSource              │    │    │
│  │  │                                                          │    │    │
│  │  │  + flow: Flow<PagingData<T>>                            │    │    │
│  │  │    └── cachedIn(viewModelScope)                         │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                       Data Layer                                 │    │
│  │                                                                  │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │              PagingSource<Key, Value>                    │    │    │
│  │  │                                                          │    │    │
│  │  │  + load(params: LoadParams): LoadResult                 │    │    │
│  │  │    - LoadParams.Refresh (初始加载)                      │    │    │
│  │  │    - LoadParams.Append (向后加载)                       │    │    │
│  │  │    - LoadParams.Prepend (向前加载)                      │    │    │
│  │  │                                                          │    │    │
│  │  │  + getRefreshKey(state): Key?                           │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  │                           │                                      │    │
│  │           ┌───────────────┼───────────────┐                     │    │
│  │           ▼               ▼               ▼                     │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐     │    │
│  │  │ 网络数据源  │  │ 本地数据库  │  │ RemoteMediator      │     │    │
│  │  │             │  │ (Room)      │  │ (网络+本地混合)     │     │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘     │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Paging 加载流程：
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. 初始加载 (Refresh)                                                  │
│     ┌──────────────────────────────────────────────────────────────┐    │
│     │ PagingDataAdapter.submitData() → PageFetcher 触发初始加载    │    │
│     │         │                                                     │    │
│     │         ▼                                                     │    │
│     │ PagingSource.load(LoadParams.Refresh)                        │    │
│     │         │                                                     │    │
│     │         ▼                                                     │    │
│     │ LoadResult.Page(data, prevKey, nextKey)                      │    │
│     └──────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  2. 预加载触发 (Prefetch)                                               │
│     ┌──────────────────────────────────────────────────────────────┐    │
│     │ 滚动到 prefetchDistance 范围内                               │    │
│     │         │                                                     │    │
│     │         ▼                                                     │    │
│     │ PageFetcher 检测到需要加载更多                               │    │
│     │         │                                                     │    │
│     │         ▼                                                     │    │
│     │ PagingSource.load(LoadParams.Append/Prepend)                 │    │
│     └──────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```


---

## 3. 关键源码解析

### 3.1 Lifecycle 源码解析

```java
/**
 * LifecycleRegistry - Lifecycle 的核心实现类
 * 
 * 源码位置: androidx.lifecycle:lifecycle-runtime
 * 版本: 2.7.0 (Android W 适配版本)
 */
public class LifecycleRegistry extends Lifecycle {
    
    /**
     * 观察者存储容器
     * 使用 FastSafeIterableMap 保证迭代时可以安全添加/删除
     * Key: LifecycleObserver (用户注册的观察者)
     * Value: ObserverWithState (包装后的观察者，包含状态信息)
     */
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();
    
    /**
     * 当前生命周期状态
     * 初始状态为 INITIALIZED
     */
    private State mState;
    
    /**
     * 弱引用持有 LifecycleOwner，避免内存泄漏
     * 当 Activity/Fragment 被回收时，不会因为 LifecycleRegistry 持有而泄漏
     */
    private final WeakReference<LifecycleOwner> mLifecycleOwner;

    /**
     * 添加观察者
     * 
     * 关键点：
     * 1. 新观察者会立即同步到当前状态
     * 2. 如果当前状态是 RESUMED，新观察者会依次收到 ON_CREATE、ON_START、ON_RESUME
     */
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        enforceMainThreadIfNeeded("addObserver");
        
        // 计算初始状态：如果当前是 DESTROYED，则初始状态也是 DESTROYED
        // 否则初始状态为 INITIALIZED
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        
        // 包装观察者，记录其状态
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        
        // 存入 Map，如果已存在则返回已有的
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
        if (previous != null) {
            return; // 已经添加过，直接返回
        }
        
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            return; // Owner 已被回收
        }

        // 关键：将新观察者同步到当前状态
        // 这就是为什么后注册的观察者也能收到之前的生命周期事件
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        
        // 循环分发事件，直到观察者状态追上当前状态
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            final Event event = Event.upFrom(statefulObserver.mState);
            if (event == null) {
                throw new IllegalStateException("no event up from " + statefulObserver.mState);
            }
            // 分发事件给观察者
            statefulObserver.dispatchEvent(lifecycleOwner, event);
            popParentState();
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            sync();
        }
        mAddingObserverCounter--;
    }

    /**
     * 处理生命周期事件
     * 由 Activity/Fragment 在生命周期方法中调用
     */
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        enforceMainThreadIfNeeded("handleLifecycleEvent");
        moveToState(event.getTargetState());
    }

    /**
     * 移动到目标状态
     */
    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            return;
        }
        mHandlingEvent = true;
        sync(); // 同步所有观察者到新状态
        mHandlingEvent = false;
    }

    /**
     * 同步所有观察者状态
     * 
     * 关键逻辑：
     * - 如果新状态 < 当前状态：倒序遍历，分发"向下"事件 (ON_PAUSE, ON_STOP, ON_DESTROY)
     * - 如果新状态 > 当前状态：正序遍历，分发"向上"事件 (ON_CREATE, ON_START, ON_RESUME)
     */
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner is garbage collected");
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // 状态降级：倒序遍历
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            // 状态升级：正序遍历
            Map.Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
}
```


### 3.2 ViewModel 源码解析

```java
/**
 * ViewModelProvider - ViewModel 的获取入口
 * 
 * 源码位置: androidx.lifecycle:lifecycle-viewmodel
 * 版本: 2.7.0
 */
public class ViewModelProvider {

    /**
     * Factory 接口 - 用于创建 ViewModel 实例
     */
    public interface Factory {
        @NonNull
        <T extends ViewModel> T create(@NonNull Class<T> modelClass);
    }

    private final ViewModelStore mViewModelStore;
    private final Factory mFactory;

    /**
     * 构造函数
     * 
     * @param owner ViewModelStoreOwner，通常是 Activity 或 Fragment
     */
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), 
             owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }

    /**
     * 获取 ViewModel 实例
     * 
     * 核心逻辑：
     * 1. 先从 ViewModelStore 中查找
     * 2. 找不到则通过 Factory 创建新实例
     * 3. 存入 ViewModelStore
     */
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        // 生成唯一 Key
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        // 1. 先从 ViewModelStore 中获取
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            // 已存在且类型匹配，直接返回
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        }

        // 2. 不存在，通过 Factory 创建
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
        } else {
            viewModel = mFactory.create(modelClass);
        }
        
        // 3. 存入 ViewModelStore
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
}

/**
 * ViewModelStore - ViewModel 的存储容器
 * 
 * 关键点：
 * 1. 内部使用 HashMap 存储 ViewModel
 * 2. 配置变更时通过 NonConfigurationInstances 保留
 * 3. Activity 真正销毁时调用 clear() 清理所有 ViewModel
 */
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            // 如果有旧的 ViewModel，调用其 onCleared()
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    /**
     * 清理所有 ViewModel
     * 在 Activity 真正销毁时调用（非配置变更）
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear(); // 内部会调用 onCleared()
        }
        mMap.clear();
    }
}

/**
 * ComponentActivity 中 ViewModel 保留机制
 * 
 * 源码位置: androidx.activity:activity
 */
public class ComponentActivity extends ... implements ViewModelStoreOwner {

    private ViewModelStore mViewModelStore;

    /**
     * 获取 ViewModelStore
     * 
     * 关键：优先从 NonConfigurationInstances 恢复
     */
    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (mViewModelStore == null) {
            // 尝试从配置变更前的实例恢复
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }

    /**
     * 配置变更时保留 ViewModelStore
     * 
     * 系统在配置变更时会调用此方法，返回值会传递给新 Activity
     */
    @Override
    @Nullable
    public final Object onRetainNonConfigurationInstance() {
        // 自定义数据
        Object custom = onRetainCustomNonConfigurationInstance();

        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }

        if (viewModelStore == null && custom == null) {
            return null;
        }

        // 包装并返回，系统会保留这个对象
        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;
        return nci;
    }

    /**
     * Activity 销毁时的处理
     */
    @Override
    protected void onDestroy() {
        super.onDestroy();
        
        // 关键判断：只有非配置变更导致的销毁才清理 ViewModel
        if (mViewModelStore != null && !isChangingConfigurations()) {
            mViewModelStore.clear();
        }
    }
}
```


### 3.3 LiveData 源码解析

```java
/**
 * LiveData - 生命周期感知的可观察数据持有者
 * 
 * 源码位置: androidx.lifecycle:lifecycle-livedata-core
 * 版本: 2.7.0
 */
public abstract class LiveData<T> {

    // 数据版本号，每次 setValue 时自增
    // 用于判断观察者是否已经收到最新数据
    static final int START_VERSION = -1;
    private int mVersion = START_VERSION;

    // 实际数据
    private volatile Object mData;
    
    // 观察者集合
    private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();

    // 活跃观察者数量
    private int mActiveCount = 0;

    /**
     * 添加生命周期感知的观察者
     * 
     * @param owner 生命周期所有者
     * @param observer 观察者
     */
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        
        // 如果 owner 已经 DESTROYED，直接忽略
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            return;
        }
        
        // 包装观察者，关联生命周期
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        
        // 存入 Map，检查是否已存在
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        
        // 注册生命周期观察
        owner.getLifecycle().addObserver(wrapper);
    }

    /**
     * 设置数据（主线程）
     * 
     * 关键流程：
     * 1. 版本号自增
     * 2. 存储数据
     * 3. 分发给所有活跃观察者
     */
    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;  // 版本号自增
        mData = value;
        dispatchingValue(null);  // 分发数据
    }

    /**
     * 设置数据（任意线程）
     * 
     * 通过 Handler post 到主线程执行
     * 注意：连续调用 postValue，只有最后一个值会被分发
     */
    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;  // 已有待处理的 post，直接返回
        }
        // post 到主线程
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }

    /**
     * 分发数据给观察者
     * 
     * @param initiator 如果非空，只分发给这个观察者；否则分发给所有观察者
     */
    void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                // 只分发给指定观察者
                considerNotify(initiator);
                initiator = null;
            } else {
                // 遍历所有观察者
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }

    /**
     * 考虑是否通知观察者
     * 
     * 这是 LiveData 的核心方法，包含了所有分发条件判断
     */
    private void considerNotify(ObserverWrapper observer) {
        // 条件1：观察者必须是活跃状态
        if (!observer.mActive) {
            return;
        }
        
        // 条件2：再次检查生命周期状态（双重检查）
        // shouldBeActive() 检查 owner 是否至少处于 STARTED 状态
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        
        // 条件3：版本号检查，避免重复分发
        // 【粘性事件的根源】：新观察者 mLastVersion = -1，而 mVersion 可能 > -1
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        
        // 更新观察者版本号
        observer.mLastVersion = mVersion;
        
        // 回调观察者
        observer.mObserver.onChanged((T) mData);
    }

    /**
     * 生命周期绑定的观察者包装类
     */
    class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        /**
         * 判断是否应该处于活跃状态
         * 
         * 关键：只有 STARTED 或 RESUMED 状态才是活跃的
         */
        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        /**
         * 生命周期事件回调
         * 
         * 当 owner 生命周期变化时：
         * 1. DESTROYED 时自动移除观察者
         * 2. 其他状态变化时检查是否需要分发数据
         */
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
            if (currentState == DESTROYED) {
                // 自动移除观察者，避免内存泄漏
                removeObserver(mObserver);
                return;
            }
            Lifecycle.State prevState = null;
            while (prevState != currentState) {
                prevState = currentState;
                // 状态变化时检查是否需要分发数据
                activeStateChanged(shouldBeActive());
                currentState = mOwner.getLifecycle().getCurrentState();
            }
        }
    }
}
```


### 3.4 SavedStateHandle 源码解析

```java
/**
 * SavedStateHandle - 进程死亡后数据恢复
 * 
 * 源码位置: androidx.lifecycle:lifecycle-viewmodel-savedstate
 * 版本: 2.7.0
 */
public final class SavedStateHandle {

    // 普通数据存储
    final Map<String, Object> mRegular;
    
    // SavedStateProvider 存储
    final Map<String, SavedStateProvider> mSavedStateProviders = new HashMap<>();
    
    // LiveData 缓存
    private final Map<String, SavingStateLiveData<?>> mLiveDatas = new HashMap<>();

    /**
     * 从 Bundle 恢复数据
     */
    static SavedStateHandle createHandle(@Nullable Bundle restoredState,
            @Nullable Bundle defaultState) {
        if (restoredState == null && defaultState == null) {
            return new SavedStateHandle();
        }

        Map<String, Object> state = new HashMap<>();
        
        // 先加载默认值
        if (defaultState != null) {
            for (String key : defaultState.keySet()) {
                state.put(key, defaultState.get(key));
            }
        }
        
        // 再用恢复的数据覆盖
        if (restoredState != null) {
            for (String key : restoredState.keySet()) {
                state.put(key, restoredState.get(key));
            }
        }
        
        return new SavedStateHandle(state);
    }

    /**
     * 获取数据
     */
    @Nullable
    public <T> T get(@NonNull String key) {
        return (T) mRegular.get(key);
    }

    /**
     * 设置数据
     * 
     * 同时更新 LiveData（如果存在）
     */
    @MainThread
    public <T> void set(@NonNull String key, @Nullable T value) {
        validateValue(value);
        
        // 更新 LiveData
        @SuppressWarnings("unchecked")
        MutableLiveData<T> mutableLiveData = (MutableLiveData<T>) mLiveDatas.get(key);
        if (mutableLiveData != null) {
            mutableLiveData.setValue(value);
        } else {
            mRegular.put(key, value);
        }
    }

    /**
     * 获取 LiveData
     * 
     * 如果不存在则创建，实现数据与 LiveData 的双向绑定
     */
    @NonNull
    @MainThread
    public <T> MutableLiveData<T> getLiveData(@NonNull String key) {
        return getLiveDataInternal(key, false, null);
    }

    @NonNull
    private <T> MutableLiveData<T> getLiveDataInternal(
            @NonNull String key,
            boolean hasInitialValue,
            @Nullable T initialValue) {
        
        MutableLiveData<T> liveData = (MutableLiveData<T>) mLiveDatas.get(key);
        if (liveData != null) {
            return liveData;
        }
        
        SavingStateLiveData<T> mutableLd;
        
        // 检查是否有已保存的值
        if (mRegular.containsKey(key)) {
            mutableLd = new SavingStateLiveData<>(this, key, (T) mRegular.get(key));
        } else if (hasInitialValue) {
            mutableLd = new SavingStateLiveData<>(this, key, initialValue);
        } else {
            mutableLd = new SavingStateLiveData<>(this, key);
        }
        
        mLiveDatas.put(key, mutableLd);
        return mutableLd;
    }

    /**
     * 获取 StateFlow (Kotlin 协程支持)
     */
    @NonNull
    public <T> StateFlow<T> getStateFlow(@NonNull String key, @NonNull T initialValue) {
        // 内部实现：将 LiveData 转换为 StateFlow
        return getLiveData(key, initialValue).asFlow()
                .stateIn(/* scope */, SharingStarted.Eagerly, initialValue);
    }

    /**
     * 保存所有数据到 Bundle
     */
    @NonNull
    Bundle savedStateProvider() {
        // 收集所有 SavedStateProvider 的数据
        Map<String, Object> map = new HashMap<>(mRegular);
        for (Map.Entry<String, SavedStateProvider> entry : mSavedStateProviders.entrySet()) {
            map.put(entry.getKey(), entry.getValue().saveState());
        }
        
        // 转换为 Bundle
        Bundle bundle = new Bundle();
        for (Map.Entry<String, Object> entry : map.entrySet()) {
            // 使用 BundleSavedStateWriter 写入
            setBundleValue(bundle, entry.getKey(), entry.getValue());
        }
        return bundle;
    }

    /**
     * 特殊的 LiveData，setValue 时同步更新 SavedStateHandle
     */
    static class SavingStateLiveData<T> extends MutableLiveData<T> {
        private final SavedStateHandle mHandle;
        private final String mKey;

        @Override
        public void setValue(T value) {
            // 同步更新到 SavedStateHandle
            mHandle.mRegular.put(mKey, value);
            super.setValue(value);
        }
    }
}

/**
 * SavedStateViewModelFactory - 支持 SavedStateHandle 的 ViewModel 工厂
 */
public final class SavedStateViewModelFactory extends ViewModelProvider.KeyedFactory {

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
        // 检查 ViewModel 构造函数是否需要 SavedStateHandle
        Constructor<T>[] constructors = (Constructor<T>[]) modelClass.getConstructors();
        
        for (Constructor<T> constructor : constructors) {
            Class<?>[] paramTypes = constructor.getParameterTypes();
            
            // 支持多种构造函数签名
            if (paramTypes.length == 1 && paramTypes[0] == SavedStateHandle.class) {
                // ViewModel(SavedStateHandle)
                SavedStateHandle handle = createSavedStateHandle(key);
                return constructor.newInstance(handle);
            }
            
            if (paramTypes.length == 2 
                    && paramTypes[0] == Application.class 
                    && paramTypes[1] == SavedStateHandle.class) {
                // AndroidViewModel(Application, SavedStateHandle)
                SavedStateHandle handle = createSavedStateHandle(key);
                return constructor.newInstance(mApplication, handle);
            }
        }
        
        // 回退到默认创建
        return mFactory.create(modelClass);
    }
}
```


### 3.5 Room 源码解析

```java
/**
 * Room 编译时生成的 DAO 实现示例
 * 
 * 原始 DAO 接口：
 * @Dao
 * interface UserDao {
 *     @Query("SELECT * FROM user WHERE id = :userId")
 *     fun getUserById(userId: Long): User?
 *     
 *     @Query("SELECT * FROM user")
 *     fun getAllUsers(): LiveData<List<User>>
 *     
 *     @Insert(onConflict = OnConflictStrategy.REPLACE)
 *     fun insertUser(user: User): Long
 * }
 */
public final class UserDao_Impl implements UserDao {

    private final RoomDatabase __db;
    
    // 预编译的 SQL 语句
    private final EntityInsertionAdapter<User> __insertionAdapterOfUser;

    public UserDao_Impl(RoomDatabase __db) {
        this.__db = __db;
        
        // 初始化插入适配器
        this.__insertionAdapterOfUser = new EntityInsertionAdapter<User>(__db) {
            @Override
            public String createQuery() {
                return "INSERT OR REPLACE INTO `user` (`id`,`name`,`email`) VALUES (?,?,?)";
            }

            @Override
            public void bind(SupportSQLiteStatement stmt, User value) {
                stmt.bindLong(1, value.getId());
                stmt.bindString(2, value.getName());
                stmt.bindString(3, value.getEmail());
            }
        };
    }

    /**
     * 查询单个用户
     * 
     * 编译时生成的实现：
     * 1. 获取数据库连接
     * 2. 执行 SQL 查询
     * 3. 解析 Cursor 到实体对象
     */
    @Override
    public User getUserById(final long userId) {
        final String _sql = "SELECT * FROM user WHERE id = ?";
        final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire(_sql, 1);
        int _argIndex = 1;
        _statement.bindLong(_argIndex, userId);
        
        __db.assertNotSuspendingTransaction();
        
        final Cursor _cursor = DBUtil.query(__db, _statement, false, null);
        try {
            // 获取列索引
            final int _cursorIndexOfId = CursorUtil.getColumnIndexOrThrow(_cursor, "id");
            final int _cursorIndexOfName = CursorUtil.getColumnIndexOrThrow(_cursor, "name");
            final int _cursorIndexOfEmail = CursorUtil.getColumnIndexOrThrow(_cursor, "email");
            
            final User _result;
            if (_cursor.moveToFirst()) {
                // 解析数据
                final long _tmpId = _cursor.getLong(_cursorIndexOfId);
                final String _tmpName = _cursor.getString(_cursorIndexOfName);
                final String _tmpEmail = _cursor.getString(_cursorIndexOfEmail);
                _result = new User(_tmpId, _tmpName, _tmpEmail);
            } else {
                _result = null;
            }
            return _result;
        } finally {
            _cursor.close();
            _statement.release();
        }
    }

    /**
     * 返回 LiveData 的查询
     * 
     * 关键：使用 InvalidationTracker 监听数据变化
     */
    @Override
    public LiveData<List<User>> getAllUsers() {
        final String _sql = "SELECT * FROM user";
        final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire(_sql, 0);
        
        return __db.getInvalidationTracker().createLiveData(
            new String[]{"user"},  // 监听的表
            false,  // 是否 inTransaction
            new Callable<List<User>>() {
                @Override
                public List<User> call() throws Exception {
                    final Cursor _cursor = DBUtil.query(__db, _statement, false, null);
                    try {
                        // ... 解析 Cursor 到 List<User>
                        return _result;
                    } finally {
                        _cursor.close();
                    }
                }
            }
        );
    }

    /**
     * 插入操作
     */
    @Override
    public long insertUser(final User user) {
        __db.assertNotSuspendingTransaction();
        __db.beginTransaction();
        try {
            long _result = __insertionAdapterOfUser.insertAndReturnId(user);
            __db.setTransactionSuccessful();
            return _result;
        } finally {
            __db.endTransaction();
        }
    }
}

/**
 * InvalidationTracker - Room 数据变化追踪器
 * 
 * 核心机制：
 * 1. 使用 SQLite 触发器监听表变化
 * 2. 表数据变化时通知所有相关的 LiveData/Flow
 */
public class InvalidationTracker {

    // 观察者集合
    private final SafeIterableMap<Observer, ObserverWrapper> mObserverMap = new SafeIterableMap<>();
    
    // 表名到 ID 的映射
    private final HashMap<String, Integer> mTableIdLookup;

    /**
     * 创建 LiveData，自动监听表变化
     */
    public <T> LiveData<T> createLiveData(
            String[] tableNames,
            boolean inTransaction,
            Callable<T> computeFunction) {
        return new RoomTrackingLiveData<>(
                mDatabase,
                this,
                inTransaction,
                computeFunction,
                tableNames
        );
    }

    /**
     * 添加观察者
     * 
     * 当指定的表发生变化时，会回调观察者
     */
    public void addObserver(@NonNull Observer observer) {
        // ... 注册观察者
        // 内部会创建 SQLite 触发器监听表变化
    }

    /**
     * 刷新版本号
     * 
     * 在事务提交后调用，检查哪些表发生了变化
     */
    public void refreshVersionsAsync() {
        // 异步检查表版本号变化
        // 如果有变化，通知相关观察者
    }
}
```


### 3.6 Paging 源码解析

```kotlin
/**
 * PagingSource - 分页数据源
 * 
 * 源码位置: androidx.paging:paging-common
 * 版本: 3.2.1
 */
abstract class PagingSource<Key : Any, Value : Any> {

    /**
     * 加载数据
     * 
     * @param params 加载参数，包含 key、loadSize 等
     * @return LoadResult，可能是 Page（成功）或 Error（失败）
     */
    abstract suspend fun load(params: LoadParams<Key>): LoadResult<Key, Value>

    /**
     * 获取刷新时的 key
     * 
     * 用于在数据源失效后重新加载时，确定从哪个位置开始
     */
    abstract fun getRefreshKey(state: PagingState<Key, Value>): Key?

    /**
     * 标记数据源失效
     * 
     * 调用后会触发新的 PagingSource 创建和数据重新加载
     */
    fun invalidate() {
        if (invalid.compareAndSet(false, true)) {
            onInvalidatedCallbacks.forEach { it.onInvalidated() }
        }
    }

    /**
     * 加载参数
     */
    sealed class LoadParams<Key : Any>(
        val loadSize: Int,
        val placeholdersEnabled: Boolean
    ) {
        // 初始加载
        class Refresh<Key : Any>(
            override val key: Key?,
            loadSize: Int,
            placeholdersEnabled: Boolean
        ) : LoadParams<Key>(loadSize, placeholdersEnabled)

        // 向后加载
        class Append<Key : Any>(
            override val key: Key,
            loadSize: Int,
            placeholdersEnabled: Boolean
        ) : LoadParams<Key>(loadSize, placeholdersEnabled)

        // 向前加载
        class Prepend<Key : Any>(
            override val key: Key,
            loadSize: Int,
            placeholdersEnabled: Boolean
        ) : LoadParams<Key>(loadSize, placeholdersEnabled)
    }

    /**
     * 加载结果
     */
    sealed class LoadResult<Key : Any, Value : Any> {
        // 成功
        data class Page<Key : Any, Value : Any>(
            val data: List<Value>,
            val prevKey: Key?,  // 上一页的 key，null 表示没有上一页
            val nextKey: Key?,  // 下一页的 key，null 表示没有下一页
            val itemsBefore: Int = COUNT_UNDEFINED,
            val itemsAfter: Int = COUNT_UNDEFINED
        ) : LoadResult<Key, Value>()

        // 失败
        data class Error<Key : Any, Value : Any>(
            val throwable: Throwable
        ) : LoadResult<Key, Value>()

        // 无效（需要重新创建 PagingSource）
        class Invalid<Key : Any, Value : Any> : LoadResult<Key, Value>()
    }
}

/**
 * Pager - 分页数据流的入口
 */
class Pager<Key : Any, Value : Any>(
    config: PagingConfig,
    initialKey: Key? = null,
    remoteMediator: RemoteMediator<Key, Value>? = null,
    pagingSourceFactory: () -> PagingSource<Key, Value>
) {
    /**
     * 分页数据流
     * 
     * 使用 cachedIn(viewModelScope) 缓存，避免配置变更时重新加载
     */
    val flow: Flow<PagingData<Value>> = PageFetcher(
        pagingSourceFactory = pagingSourceFactory,
        initialKey = initialKey,
        config = config,
        remoteMediator = remoteMediator
    ).flow
}

/**
 * PageFetcher - 分页数据获取器
 * 
 * 核心类，负责协调数据加载
 */
internal class PageFetcher<Key : Any, Value : Any>(
    private val pagingSourceFactory: () -> PagingSource<Key, Value>,
    private val initialKey: Key?,
    private val config: PagingConfig,
    private val remoteMediator: RemoteMediator<Key, Value>?
) {
    val flow: Flow<PagingData<Value>> = channelFlow {
        // 创建 PagingSource
        val pagingSource = pagingSourceFactory()
        
        // 创建 PageFetcherSnapshot 处理实际加载
        val snapshot = PageFetcherSnapshot(
            initialKey = initialKey,
            pagingSource = pagingSource,
            config = config,
            retryFlow = retryChannel.receiveAsFlow(),
            triggerRemoteRefresh = remoteMediator != null,
            remoteMediatorConnection = remoteMediatorConnection,
            invalidate = { refresh() }
        )
        
        // 收集加载结果并发送
        snapshot.pageEventFlow.collect { event ->
            send(PagingData(event, snapshot.uiReceiver))
        }
    }
}

/**
 * PagingDataAdapter - RecyclerView 适配器
 */
abstract class PagingDataAdapter<T : Any, VH : RecyclerView.ViewHolder>(
    diffCallback: DiffUtil.ItemCallback<T>,
    mainDispatcher: CoroutineDispatcher = Dispatchers.Main,
    workerDispatcher: CoroutineDispatcher = Dispatchers.Default
) : RecyclerView.Adapter<VH>() {

    private val differ = AsyncPagingDataDiffer(
        diffCallback = diffCallback,
        updateCallback = AdapterListUpdateCallback(this),
        mainDispatcher = mainDispatcher,
        workerDispatcher = workerDispatcher
    )

    /**
     * 提交分页数据
     */
    suspend fun submitData(pagingData: PagingData<T>) {
        differ.submitData(pagingData)
    }

    /**
     * 获取指定位置的数据
     * 
     * 关键：访问数据时会触发预加载检查
     */
    fun getItem(position: Int): T? {
        return differ.getItem(position)  // 内部会检查是否需要加载更多
    }

    override fun getItemCount(): Int = differ.itemCount
}

/**
 * AsyncPagingDataDiffer - 异步分页数据差异计算
 */
class AsyncPagingDataDiffer<T : Any>(
    private val diffCallback: DiffUtil.ItemCallback<T>,
    private val updateCallback: ListUpdateCallback,
    private val mainDispatcher: CoroutineDispatcher,
    private val workerDispatcher: CoroutineDispatcher
) {
    /**
     * 获取数据项
     * 
     * 关键逻辑：访问数据时检查是否需要预加载
     */
    fun getItem(index: Int): T? {
        // 通知 PagingSource 当前访问位置
        // 如果接近边界，会触发加载更多
        presenter.get(index)?.also {
            // 检查是否需要预加载
            hintReceiver?.accessHint(
                ViewportHint.Access(
                    pageOffset = /* 计算页偏移 */,
                    indexInPage = /* 页内索引 */,
                    presentedItemsBefore = index,
                    presentedItemsAfter = itemCount - index - 1,
                    originalPageOffsetFirst = /* ... */,
                    originalPageOffsetLast = /* ... */
                )
            )
        }
        return presenter.get(index)
    }
}
```


---

## 4. 实战应用

### 4.1 ViewModel 最佳实践

```kotlin
/**
 * ViewModel 最佳实践示例
 */
class UserViewModel(
    private val userRepository: UserRepository,
    private val savedStateHandle: SavedStateHandle  // 支持进程死亡恢复
) : ViewModel() {

    // 使用 StateFlow 替代 LiveData（推荐）
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    // 使用 SavedStateHandle 保存关键状态
    private val userId: Long = savedStateHandle.get<Long>(KEY_USER_ID) ?: 0L

    // 一次性事件使用 SharedFlow（避免粘性问题）
    private val _events = MutableSharedFlow<UserEvent>()
    val events: SharedFlow<UserEvent> = _events.asSharedFlow()

    init {
        loadUser()
    }

    private fun loadUser() {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            
            userRepository.getUser(userId)
                .catch { e ->
                    _uiState.value = UserUiState.Error(e.message ?: "Unknown error")
                }
                .collect { user ->
                    _uiState.value = UserUiState.Success(user)
                }
        }
    }

    fun updateUser(name: String) {
        viewModelScope.launch {
            try {
                userRepository.updateUser(userId, name)
                _events.emit(UserEvent.UpdateSuccess)
            } catch (e: Exception) {
                _events.emit(UserEvent.UpdateFailed(e.message))
            }
        }
    }

    // 保存状态到 SavedStateHandle
    fun saveState() {
        savedStateHandle[KEY_USER_ID] = userId
    }

    companion object {
        private const val KEY_USER_ID = "user_id"
    }
}

// UI 状态密封类
sealed class UserUiState {
    object Loading : UserUiState()
    data class Success(val user: User) : UserUiState()
    data class Error(val message: String) : UserUiState()
}

// 一次性事件
sealed class UserEvent {
    object UpdateSuccess : UserEvent()
    data class UpdateFailed(val message: String?) : UserEvent()
}
```

### 4.2 LiveData 粘性事件解决方案

```kotlin
/**
 * 方案1: Event 包装类（推荐）
 */
open class Event<out T>(private val content: T) {
    
    private var hasBeenHandled = false

    /**
     * 获取内容，只能获取一次
     */
    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) {
            null
        } else {
            hasBeenHandled = true
            content
        }
    }

    /**
     * 获取内容，可重复获取（用于需要重复消费的场景）
     */
    fun peekContent(): T = content
}

// 使用示例
class MyViewModel : ViewModel() {
    private val _navigateEvent = MutableLiveData<Event<String>>()
    val navigateEvent: LiveData<Event<String>> = _navigateEvent

    fun navigate(destination: String) {
        _navigateEvent.value = Event(destination)
    }
}

// 观察者
viewModel.navigateEvent.observe(this) { event ->
    event.getContentIfNotHandled()?.let { destination ->
        // 只会执行一次
        navigateTo(destination)
    }
}

/**
 * 方案2: 使用 SharedFlow（推荐，Kotlin 协程方案）
 */
class MyViewModel : ViewModel() {
    // replay = 0 表示不缓存，新订阅者不会收到旧数据
    private val _events = MutableSharedFlow<NavigationEvent>(replay = 0)
    val events: SharedFlow<NavigationEvent> = _events.asSharedFlow()

    fun navigate(destination: String) {
        viewModelScope.launch {
            _events.emit(NavigationEvent.Navigate(destination))
        }
    }
}

// 在 Fragment 中收集
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { event ->
            when (event) {
                is NavigationEvent.Navigate -> navigateTo(event.destination)
            }
        }
    }
}

/**
 * 方案3: SingleLiveEvent（Google 官方示例，仅支持单观察者）
 */
class SingleLiveEvent<T> : MutableLiveData<T>() {

    private val pending = AtomicBoolean(false)

    @MainThread
    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        super.observe(owner) { t ->
            if (pending.compareAndSet(true, false)) {
                observer.onChanged(t)
            }
        }
    }

    @MainThread
    override fun setValue(t: T?) {
        pending.set(true)
        super.setValue(t)
    }
}
```

### 4.3 Room + Paging 实战

```kotlin
/**
 * Room + Paging 3 完整示例
 */

// 1. Entity 定义
@Entity(tableName = "articles")
data class Article(
    @PrimaryKey val id: Long,
    val title: String,
    val content: String,
    val createdAt: Long
)

// 2. DAO 定义
@Dao
interface ArticleDao {
    
    // 返回 PagingSource，Room 自动生成实现
    @Query("SELECT * FROM articles ORDER BY createdAt DESC")
    fun getArticlesPagingSource(): PagingSource<Int, Article>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(articles: List<Article>)
    
    @Query("DELETE FROM articles")
    suspend fun clearAll()
}

// 3. RemoteMediator（网络 + 本地混合加载）
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val database: AppDatabase,
    private val apiService: ArticleApiService
) : RemoteMediator<Int, Article>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, Article>
    ): MediatorResult {
        return try {
            val page = when (loadType) {
                LoadType.REFRESH -> 1
                LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
                LoadType.APPEND -> {
                    val lastItem = state.lastItemOrNull()
                        ?: return MediatorResult.Success(endOfPaginationReached = true)
                    // 计算下一页
                    (lastItem.id / state.config.pageSize + 1).toInt()
                }
            }

            // 从网络加载
            val response = apiService.getArticles(page, state.config.pageSize)
            
            // 写入数据库
            database.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    database.articleDao().clearAll()
                }
                database.articleDao().insertAll(response.articles)
            }

            MediatorResult.Success(endOfPaginationReached = response.articles.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}

// 4. Repository
class ArticleRepository(
    private val database: AppDatabase,
    private val apiService: ArticleApiService
) {
    @OptIn(ExperimentalPagingApi::class)
    fun getArticles(): Flow<PagingData<Article>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                prefetchDistance = 5,
                enablePlaceholders = false
            ),
            remoteMediator = ArticleRemoteMediator(database, apiService),
            pagingSourceFactory = { database.articleDao().getArticlesPagingSource() }
        ).flow
    }
}

// 5. ViewModel
class ArticleViewModel(
    private val repository: ArticleRepository
) : ViewModel() {
    
    val articles: Flow<PagingData<Article>> = repository.getArticles()
        .cachedIn(viewModelScope)  // 缓存，配置变更时不重新加载
}

// 6. UI 层使用
class ArticleFragment : Fragment() {
    
    private val viewModel: ArticleViewModel by viewModels()
    private lateinit var adapter: ArticlePagingAdapter

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        adapter = ArticlePagingAdapter()
        binding.recyclerView.adapter = adapter
        
        // 收集分页数据
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.articles.collectLatest { pagingData ->
                adapter.submitData(pagingData)
            }
        }
        
        // 监听加载状态
        viewLifecycleOwner.lifecycleScope.launch {
            adapter.loadStateFlow.collectLatest { loadStates ->
                binding.progressBar.isVisible = loadStates.refresh is LoadState.Loading
                binding.errorView.isVisible = loadStates.refresh is LoadState.Error
            }
        }
    }
}
```


### 4.4 DataBinding 双向绑定实战

```kotlin
/**
 * DataBinding 双向绑定示例
 */

// 1. ViewModel
class LoginViewModel : ViewModel() {
    
    // 使用 ObservableField 实现双向绑定
    val username = ObservableField<String>("")
    val password = ObservableField<String>("")
    
    // 或使用 LiveData（需要配合 @={} 语法）
    val email = MutableLiveData<String>()
    
    // 计算属性
    val isLoginEnabled: ObservableBoolean = object : ObservableBoolean() {
        override fun get(): Boolean {
            return !username.get().isNullOrBlank() && 
                   !password.get().isNullOrBlank()
        }
    }

    init {
        // 监听变化，更新计算属性
        username.addOnPropertyChangedCallback(object : Observable.OnPropertyChangedCallback() {
            override fun onPropertyChanged(sender: Observable?, propertyId: Int) {
                isLoginEnabled.notifyChange()
            }
        })
    }
}

// 2. XML 布局
/*
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable
            name="viewModel"
            type="com.example.LoginViewModel" />
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <!-- 双向绑定：@={} -->
        <EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@={viewModel.username}" />

        <EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@={viewModel.password}"
            android:inputType="textPassword" />

        <!-- 单向绑定：@{} -->
        <Button
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Login"
            android:enabled="@{viewModel.isLoginEnabled}" />

    </LinearLayout>
</layout>
*/

// 3. 自定义 BindingAdapter
@BindingAdapter("imageUrl")
fun loadImage(imageView: ImageView, url: String?) {
    url?.let {
        Glide.with(imageView.context)
            .load(it)
            .into(imageView)
    }
}

// 双向绑定适配器
@BindingAdapter("selectedItem")
fun setSelectedItem(spinner: Spinner, item: String?) {
    val adapter = spinner.adapter as? ArrayAdapter<String> ?: return
    val position = adapter.getPosition(item)
    if (position >= 0) {
        spinner.setSelection(position)
    }
}

@InverseBindingAdapter(attribute = "selectedItem")
fun getSelectedItem(spinner: Spinner): String? {
    return spinner.selectedItem as? String
}

@BindingAdapter("selectedItemAttrChanged")
fun setSelectedItemListener(spinner: Spinner, listener: InverseBindingListener?) {
    spinner.onItemSelectedListener = object : AdapterView.OnItemSelectedListener {
        override fun onItemSelected(parent: AdapterView<*>?, view: View?, position: Int, id: Long) {
            listener?.onChange()
        }
        override fun onNothingSelected(parent: AdapterView<*>?) {}
    }
}
```

### 4.5 Lifecycle 感知组件封装

```kotlin
/**
 * 生命周期感知的定位管理器
 */
class LocationManager(
    private val context: Context,
    private val lifecycle: Lifecycle,
    private val callback: (Location) -> Unit
) : DefaultLifecycleObserver {

    private val fusedLocationClient = LocationServices.getFusedLocationProviderClient(context)
    private var locationCallback: LocationCallback? = null

    init {
        lifecycle.addObserver(this)
    }

    override fun onStart(owner: LifecycleOwner) {
        startLocationUpdates()
    }

    override fun onStop(owner: LifecycleOwner) {
        stopLocationUpdates()
    }

    override fun onDestroy(owner: LifecycleOwner) {
        lifecycle.removeObserver(this)
    }

    private fun startLocationUpdates() {
        val request = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 10000L)
            .setMinUpdateIntervalMillis(5000L)
            .build()

        locationCallback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { callback(it) }
            }
        }

        fusedLocationClient.requestLocationUpdates(
            request,
            locationCallback!!,
            Looper.getMainLooper()
        )
    }

    private fun stopLocationUpdates() {
        locationCallback?.let {
            fusedLocationClient.removeLocationUpdates(it)
        }
    }
}

// 使用
class MapFragment : Fragment() {
    
    private lateinit var locationManager: LocationManager

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        locationManager = LocationManager(
            context = requireContext(),
            lifecycle = viewLifecycleOwner.lifecycle,
            callback = { location ->
                updateMapLocation(location)
            }
        )
    }
}
```


---

## 5. 常见面试题

### 5.1 ViewModel 相关

#### Q1: ViewModel 是如何在配置变更时保留数据的？（字节、OPPO 高频）

**答案要点：**

```
核心机制：NonConfigurationInstances

1. 保存流程：
   - Activity 配置变更时，系统调用 onRetainNonConfigurationInstance()
   - ComponentActivity 重写此方法，返回包含 ViewModelStore 的 NonConfigurationInstances
   - 系统将返回值保存在 ActivityClientRecord.lastNonConfigurationInstances

2. 恢复流程：
   - 新 Activity 创建时，调用 getLastNonConfigurationInstance()
   - 从 ActivityClientRecord 获取之前保存的 NonConfigurationInstances
   - 从中恢复 ViewModelStore，所有 ViewModel 实例得以保留

3. 关键源码位置：
   - ComponentActivity.onRetainNonConfigurationInstance()
   - ComponentActivity.getViewModelStore()
   - ActivityThread.performDestroyActivity() 中保存
   - ActivityThread.performLaunchActivity() 中恢复

4. 为什么 onSaveInstanceState 不行？
   - onSaveInstanceState 通过 Bundle 序列化，有大小限制（约 1MB）
   - NonConfigurationInstances 直接保存对象引用，无大小限制
   - 但 NonConfigurationInstances 只在配置变更时有效，进程死亡后无法恢复
```

#### Q2: ViewModel 的 onCleared() 什么时候调用？（美团）

**答案要点：**

```
调用时机：Activity/Fragment 真正销毁时（非配置变更）

源码分析：
1. ComponentActivity.onDestroy() 中：
   if (mViewModelStore != null && !isChangingConfigurations()) {
       mViewModelStore.clear();  // 只有非配置变更才清理
   }

2. Fragment 中：
   - FragmentManagerImpl.moveToState() 中
   - 当 Fragment 状态变为 DESTROYED 且非配置变更时调用

判断条件：
- Activity: !isChangingConfigurations()
- Fragment: !getActivity().isChangingConfigurations()

常见误区：
- 屏幕旋转时 onCleared() 不会调用
- 按返回键退出时会调用
- 调用 finish() 时会调用
- 进程被杀死时不会调用（进程都没了）
```

#### Q3: 如何在 Fragment 之间共享 ViewModel？（快手）

**答案要点：**

```kotlin
// 方案1: 使用 Activity 作为 ViewModelStoreOwner
class FragmentA : Fragment() {
    // 使用 activityViewModels() 委托
    private val sharedViewModel: SharedViewModel by activityViewModels()
}

class FragmentB : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
}

// 方案2: 使用父 Fragment 作为 ViewModelStoreOwner
class ChildFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by viewModels(
        ownerProducer = { requireParentFragment() }
    )
}

// 方案3: 使用 Navigation 的 NavBackStackEntry
class FragmentA : Fragment() {
    private val sharedViewModel: SharedViewModel by navGraphViewModels(R.id.nav_graph)
}

// 原理：
// 不同 Fragment 使用同一个 ViewModelStoreOwner
// 从同一个 ViewModelStore 获取 ViewModel
// 因此获取到的是同一个实例
```

### 5.2 LiveData 相关

#### Q4: LiveData 的粘性事件是什么？如何解决？（字节、vivo 高频）

**答案要点：**

```
什么是粘性事件：
- 新订阅的观察者会立即收到 LiveData 中已有的数据
- 即使这个数据是在订阅之前设置的

产生原因（源码分析）：
1. LiveData 内部维护 mVersion（数据版本号）
2. 每次 setValue() 时 mVersion++
3. 新观察者的 mLastVersion 初始值为 -1
4. considerNotify() 中判断：
   if (observer.mLastVersion >= mVersion) return;
5. 新观察者 mLastVersion(-1) < mVersion，所以会收到数据

解决方案：
1. Event 包装类：标记数据是否已消费
2. SingleLiveEvent：使用 AtomicBoolean 标记
3. SharedFlow：设置 replay = 0
4. 反射修改 mLastVersion（不推荐）

推荐方案：
- 状态数据：使用 StateFlow（天然支持粘性）
- 事件数据：使用 SharedFlow(replay = 0)（无粘性）
```

#### Q5: LiveData 的 setValue 和 postValue 有什么区别？（OPPO）

**答案要点：**

```
setValue：
- 必须在主线程调用
- 同步执行，立即分发数据
- 调用后观察者立即收到回调

postValue：
- 可以在任意线程调用
- 异步执行，通过 Handler post 到主线程
- 连续调用只有最后一个值会被分发

源码分析：
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;  // 覆盖之前的值
    }
    if (!postTask) {
        return;  // 已有待处理的 post，直接返回
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

注意事项：
- 高频更新场景使用 postValue 可能丢失中间值
- 需要保证每个值都被处理时，应使用 setValue + 切换到主线程
```

### 5.3 Lifecycle 相关

#### Q6: Lifecycle 是如何感知 Activity/Fragment 生命周期的？（OPPO、vivo 重点）

**答案要点：**

```
Activity 生命周期感知：

1. ComponentActivity 实现 LifecycleOwner 接口
2. 内部持有 LifecycleRegistry 实例
3. 通过 ReportFragment 注入生命周期回调

ReportFragment 机制（API 29 以下）：
- ComponentActivity.onCreate() 中添加一个无 UI 的 Fragment
- ReportFragment 的生命周期方法中调用 dispatch()
- dispatch() 调用 LifecycleRegistry.handleLifecycleEvent()

API 29+ 机制：
- 直接使用 Activity.registerActivityLifecycleCallbacks()
- 更简洁，无需 Fragment

Fragment 生命周期感知：
- Fragment 本身实现 LifecycleOwner
- FragmentManagerImpl 在状态转换时调用 dispatch()
- 通过 LifecycleRegistry 分发事件

关键源码：
// ReportFragment.java
@Override
public void onStart() {
    super.onStart();
    dispatch(Lifecycle.Event.ON_START);
}

private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

#### Q7: 为什么 LiveData 只在 STARTED 及以上状态才分发数据？（美团）

**答案要点：**

```
设计原因：
1. STARTED 状态表示 Activity/Fragment 对用户可见
2. 只有可见时更新 UI 才有意义
3. 避免在不可见时进行无意义的 UI 更新
4. 防止在 onStop 后更新 UI 导致的问题

源码位置：
// LifecycleBoundObserver.shouldBeActive()
@Override
boolean shouldBeActive() {
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}

状态判断：
- STARTED: onStart() 之后，onStop() 之前
- RESUMED: onResume() 之后，onPause() 之前
- isAtLeast(STARTED) 包含 STARTED 和 RESUMED

实际效果：
- onStart() 后：开始接收数据
- onStop() 后：停止接收数据
- 再次 onStart()：如果有新数据，会立即收到
```


### 5.4 SavedStateHandle 相关

#### Q8: ViewModel 和 SavedStateHandle 的区别？分别在什么场景使用？（字节）

**答案要点：**

```
ViewModel：
- 作用：配置变更时保留数据
- 原理：通过 NonConfigurationInstances 保存对象引用
- 限制：进程死亡后数据丢失
- 适用：大对象、复杂对象、网络请求结果

SavedStateHandle：
- 作用：进程死亡后恢复数据
- 原理：通过 onSaveInstanceState/Bundle 序列化
- 限制：只能保存可序列化数据，有大小限制（约 1MB）
- 适用：用户输入、页面状态、滚动位置

使用建议：
┌─────────────────────────────────────────────────────────────┐
│ 场景                    │ 使用方案                          │
├─────────────────────────────────────────────────────────────┤
│ 网络请求结果            │ ViewModel（可重新请求）           │
│ 用户输入的文本          │ SavedStateHandle                  │
│ 列表滚动位置            │ SavedStateHandle                  │
│ 复杂对象（如 Bitmap）   │ ViewModel + 磁盘缓存              │
│ 页面导航参数            │ SavedStateHandle                  │
└─────────────────────────────────────────────────────────────┘

代码示例：
class MyViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // 网络数据：使用普通变量，进程死亡后重新请求
    private val _users = MutableStateFlow<List<User>>(emptyList())
    
    // 用户输入：使用 SavedStateHandle，进程死亡后恢复
    val searchQuery: StateFlow<String> = savedStateHandle.getStateFlow("query", "")
    
    fun setSearchQuery(query: String) {
        savedStateHandle["query"] = query
    }
}
```

### 5.5 Room 相关

#### Q9: Room 是如何实现 LiveData 自动更新的？（OPPO、vivo）

**答案要点：**

```
核心机制：InvalidationTracker

1. 编译时处理：
   - Room 编译器检测到返回 LiveData 的 @Query 方法
   - 生成的 DAO 实现中使用 InvalidationTracker.createLiveData()

2. InvalidationTracker 原理：
   - 为每个表创建 SQLite 触发器（INSERT/UPDATE/DELETE）
   - 触发器在数据变化时更新 invalidation 表
   - 后台线程定期检查 invalidation 表
   - 发现变化时通知相关的 LiveData

3. 源码流程：
   // 生成的 DAO 实现
   @Override
   public LiveData<List<User>> getAllUsers() {
       return __db.getInvalidationTracker().createLiveData(
           new String[]{"user"},  // 监听的表
           new Callable<List<User>>() {
               @Override
               public List<User> call() {
                   // 执行查询
               }
           }
       );
   }

4. RoomTrackingLiveData：
   - 继承自 LiveData
   - 注册为 InvalidationTracker 的观察者
   - 表数据变化时，重新执行查询并更新值

触发器 SQL 示例：
CREATE TEMP TRIGGER IF NOT EXISTS `room_table_modification_trigger_user_UPDATE`
AFTER UPDATE ON `user` BEGIN
    UPDATE room_table_modification_log SET invalidated = 1 WHERE table_id = 0;
END
```

#### Q10: Room 的 @Transaction 注解有什么作用？（美团）

**答案要点：**

```
作用：
1. 保证多个数据库操作的原子性
2. 要么全部成功，要么全部回滚
3. 避免数据不一致

使用场景：
1. 多表关联查询
2. 批量插入/更新/删除
3. 先删后插的替换操作

源码实现：
@Dao
interface UserDao {
    @Transaction
    @Query("SELECT * FROM user WHERE id = :userId")
    fun getUserWithOrders(userId: Long): UserWithOrders
    
    @Transaction
    suspend fun replaceUsers(users: List<User>) {
        deleteAllUsers()
        insertUsers(users)
    }
}

// 生成的实现
@Override
public UserWithOrders getUserWithOrders(long userId) {
    __db.beginTransaction();
    try {
        // 查询 User
        // 查询关联的 Orders
        __db.setTransactionSuccessful();
        return result;
    } finally {
        __db.endTransaction();
    }
}

注意事项：
- @Transaction 方法不能是 suspend 函数（Room 2.1 之前）
- Room 2.1+ 支持 suspend + @Transaction
- 嵌套事务会被合并为一个事务
```

### 5.6 Paging 相关

#### Q11: Paging 3 的加载状态有哪些？如何处理？（快手）

**答案要点：**

```kotlin
加载状态类型：
sealed class LoadState {
    // 未加载
    object NotLoading : LoadState()
    
    // 加载中
    object Loading : LoadState()
    
    // 加载错误
    data class Error(val error: Throwable) : LoadState()
}

三种加载类型：
- refresh: 初始加载或刷新
- prepend: 向前加载（列表顶部）
- append: 向后加载（列表底部）

监听加载状态：
adapter.loadStateFlow.collectLatest { loadStates ->
    // 刷新状态
    when (loadStates.refresh) {
        is LoadState.Loading -> showLoading()
        is LoadState.Error -> showError((loadStates.refresh as LoadState.Error).error)
        is LoadState.NotLoading -> hideLoading()
    }
    
    // 追加加载状态
    when (loadStates.append) {
        is LoadState.Loading -> showFooterLoading()
        is LoadState.Error -> showFooterError()
        is LoadState.NotLoading -> {
            if (loadStates.append.endOfPaginationReached) {
                showNoMoreData()
            }
        }
    }
}

使用 LoadStateAdapter 显示加载状态：
class LoadStateFooterAdapter(
    private val retry: () -> Unit
) : LoadStateAdapter<LoadStateViewHolder>() {
    
    override fun onBindViewHolder(holder: LoadStateViewHolder, loadState: LoadState) {
        holder.bind(loadState)
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, loadState: LoadState): LoadStateViewHolder {
        return LoadStateViewHolder.create(parent, retry)
    }
}

// 使用
recyclerView.adapter = adapter.withLoadStateFooter(
    footer = LoadStateFooterAdapter { adapter.retry() }
)
```

#### Q12: Paging 3 中 PagingSource 失效后如何处理？（字节）

**答案要点：**

```kotlin
失效场景：
1. 数据源发生变化（如数据库更新）
2. 手动调用 invalidate()
3. RemoteMediator 返回 MediatorResult.Success(endOfPaginationReached = false)

失效处理流程：
1. PagingSource.invalidate() 被调用
2. PageFetcher 检测到失效
3. 调用 pagingSourceFactory 创建新的 PagingSource
4. 使用 getRefreshKey() 确定刷新位置
5. 从刷新位置重新加载数据

getRefreshKey 实现：
override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
    // 返回当前可见区域中间位置的 key
    return state.anchorPosition?.let { anchorPosition ->
        state.closestPageToPosition(anchorPosition)?.prevKey?.plus(1)
            ?: state.closestPageToPosition(anchorPosition)?.nextKey?.minus(1)
    }
}

Room + Paging 自动失效：
- Room 的 PagingSource 实现会自动监听表变化
- 表数据变化时自动调用 invalidate()
- 无需手动处理

手动触发刷新：
// 方式1: 调用 adapter.refresh()
adapter.refresh()

// 方式2: 使 PagingSource 失效
pagingSource.invalidate()

// 方式3: 重新收集 Flow
viewModel.articles.collectLatest { pagingData ->
    adapter.submitData(pagingData)
}
```

### 5.7 综合问题

#### Q13: 如何设计一个完整的 MVVM 架构？（字节、美团 架构题）

**答案要点：**

```
分层架构：

┌─────────────────────────────────────────────────────────────┐
│                        UI Layer                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Activity / Fragment / Compose                        │    │
│  │ - 只负责 UI 渲染和用户交互                          │    │
│  │ - 观察 ViewModel 的状态                             │    │
│  │ - 调用 ViewModel 的方法处理用户操作                 │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     ViewModel Layer                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ ViewModel                                            │    │
│  │ - 持有 UI 状态（StateFlow/LiveData）                │    │
│  │ - 处理业务逻辑                                      │    │
│  │ - 调用 Repository 获取数据                          │    │
│  │ - 不持有 View 引用                                  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Repository Layer                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Repository                                           │    │
│  │ - 数据源的唯一入口                                  │    │
│  │ - 协调多个数据源（网络、本地）                      │    │
│  │ - 实现缓存策略                                      │    │
│  │ - 数据转换和映射                                    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────────────┐  ┌──────────────────────────────┐    │
│  │ Remote DataSource│  │ Local DataSource              │    │
│  │ (Retrofit)       │  │ (Room / DataStore)            │    │
│  └──────────────────┘  └──────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

关键设计原则：
1. 单向数据流：UI → ViewModel → Repository → DataSource
2. 状态驱动：UI 根据状态渲染，不直接操作 View
3. 关注点分离：每层只负责自己的职责
4. 依赖注入：使用 Hilt/Koin 管理依赖
5. 可测试性：每层都可以独立测试
```

#### Q14: LiveData 和 Flow 如何选择？（字节、快手）

**答案要点：**

```
LiveData 优势：
1. 生命周期感知，自动管理订阅
2. 简单易用，学习成本低
3. 与 DataBinding 完美配合
4. 主线程安全

LiveData 劣势：
1. 粘性事件问题
2. 操作符有限
3. 不支持背压
4. 只能在主线程观察

Flow 优势：
1. 丰富的操作符
2. 支持背压
3. 可配置粘性（SharedFlow replay）
4. 更好的协程集成
5. 冷流/热流灵活选择

Flow 劣势：
1. 需要手动管理生命周期
2. 学习成本较高

选择建议：
┌─────────────────────────────────────────────────────────────┐
│ 场景                        │ 推荐方案                      │
├─────────────────────────────────────────────────────────────┤
│ 简单 UI 状态                │ StateFlow                     │
│ 一次性事件（导航、Toast）   │ SharedFlow(replay=0)          │
│ 数据库查询                  │ Flow（Room 原生支持）         │
│ 网络请求                    │ Flow + suspend                │
│ 与 DataBinding 配合         │ LiveData                      │
│ 旧项目维护                  │ LiveData（保持一致性）        │
│ 新项目                      │ Flow（更现代化）              │
└─────────────────────────────────────────────────────────────┘

Flow 生命周期安全收集：
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            // 只在 STARTED 及以上状态收集
        }
    }
}
```

#### Q15: 如何处理 ViewModel 中的协程取消？（美团）

**答案要点：**

```kotlin
viewModelScope 特性：
1. 绑定 ViewModel 生命周期
2. ViewModel.onCleared() 时自动取消
3. 使用 Dispatchers.Main.immediate

源码分析：
// ViewModel.kt
val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }

// ViewModel.clear() 中
final void clear() {
    mCleared = true;
    // 关闭所有 Closeable，包括 viewModelScope
    if (mBagOfTags != null) {
        synchronized (mBagOfTags) {
            for (Object value : mBagOfTags.values()) {
                closeWithRuntimeException(value);
            }
        }
    }
    onCleared();
}

最佳实践：
class MyViewModel : ViewModel() {
    
    // 使用 viewModelScope 启动协程
    fun loadData() {
        viewModelScope.launch {
            try {
                val result = repository.fetchData()
                _uiState.value = UiState.Success(result)
            } catch (e: CancellationException) {
                // 协程取消，不处理
                throw e
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
    
    // 需要在特定时机取消的任务
    private var searchJob: Job? = null
    
    fun search(query: String) {
        searchJob?.cancel()  // 取消之前的搜索
        searchJob = viewModelScope.launch {
            delay(300)  // 防抖
            val results = repository.search(query)
            _searchResults.value = results
        }
    }
}
```

---

## 6. 总结

### 核心知识点回顾

| 组件 | 核心原理 | 面试重点 |
|------|----------|----------|
| **Lifecycle** | LifecycleRegistry + 观察者模式 | 状态与事件对应、ReportFragment |
| **ViewModel** | ViewModelStore + NonConfigurationInstances | 配置变更保留机制、onCleared 时机 |
| **LiveData** | 版本号机制 + 生命周期感知 | 粘性事件原因与解决方案 |
| **SavedStateHandle** | SavedStateRegistry + Bundle | 与 ViewModel 的区别和配合 |
| **DataBinding** | 编译时代码生成 + 脏标记 | 双向绑定原理、BindingAdapter |
| **Room** | 编译时生成 + InvalidationTracker | LiveData 自动更新原理 |
| **Paging** | PagingSource + PageFetcher | 加载状态处理、失效机制 |

### 面试准备建议

1. **源码阅读**：重点阅读 LifecycleRegistry、ViewModelProvider、LiveData 核心类
2. **动手实践**：实现一个完整的 MVVM 架构项目
3. **对比理解**：LiveData vs Flow、ViewModel vs SavedStateHandle
4. **问题排查**：了解常见问题（粘性事件、内存泄漏）的原因和解决方案
