# Context 体系

## 1. 概述

Context 是 Android 中最重要的抽象类之一，代表应用程序的运行环境。它提供了访问应用资源、启动组件、获取系统服务等核心功能。理解 Context 的继承体系和使用场景，对于避免内存泄漏和正确使用 Android API 至关重要。

**Context 的核心功能：**
- 访问应用资源（getString、getDrawable 等）
- 启动组件（startActivity、startService、sendBroadcast）
- 获取系统服务（getSystemService）
- 访问应用目录（getFilesDir、getCacheDir）
- 获取包信息（getPackageName、getPackageManager）

## 2. 核心原理

### 2.1 Context 继承关系

```
                        ┌─────────────────┐
                        │     Context     │  (抽象类)
                        │                 │
                        │ - 定义核心 API   │
                        └────────┬────────┘
                                 │
                                 │ extends
                                 ▼
                        ┌─────────────────┐
                        │  ContextWrapper │  (装饰器模式)
                        │                 │
                        │ - 持有 mBase    │
                        │ - 委托给 mBase  │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
     ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
     │   Application   │ │ContextThemeWrapper│ │    Service     │
     │                 │ │                 │ │                 │
     │ - 全局单例      │ │ - 包含主题信息   │ │ - 后台服务      │
     │ - 生命周期最长  │ │                 │ │                 │
     └─────────────────┘ └────────┬────────┘ └─────────────────┘
                                  │
                                  │ extends
                                  ▼
                         ┌─────────────────┐
                         │    Activity     │
                         │                 │
                         │ - 包含 UI 信息   │
                         │ - 可以显示 Dialog│
                         └─────────────────┘

另外还有：
┌─────────────────┐
│   ContextImpl   │  (Context 的真正实现类)
│                 │
│ - 实现所有方法   │
│ - 被 Wrapper 持有│
└─────────────────┘
```

### 2.2 Context 数量

一个应用中 Context 的数量 = Activity 数量 + Service 数量 + 1（Application）

```
Context 总数 = Activity 实例数 + Service 实例数 + 1 (Application)
```

### 2.3 不同 Context 的能力对比

| 功能 | Application | Activity | Service |
|-----|-------------|----------|---------|
| 启动 Activity | ✅ (需要 FLAG_NEW_TASK) | ✅ | ✅ (需要 FLAG_NEW_TASK) |
| 显示 Dialog | ❌ | ✅ | ❌ |
| 启动 Service | ✅ | ✅ | ✅ |
| 发送 Broadcast | ✅ | ✅ | ✅ |
| 注册 Receiver | ✅ | ✅ | ✅ |
| 加载资源 | ✅ | ✅ | ✅ |
| 获取系统服务 | ✅ | ✅ | ✅ |
| 获取 Assets | ✅ | ✅ | ✅ |
| 创建 View | ❌ (无主题) | ✅ | ❌ (无主题) |
| Layout Inflation | ⚠️ (无主题) | ✅ | ⚠️ (无主题) |

**说明：**
- ✅ 完全支持
- ❌ 不支持
- ⚠️ 可以使用但可能有问题（如无主题信息）

## 3. 关键源码解析

### 3.1 Context 抽象类

```java
// frameworks/base/core/java/android/content/Context.java
public abstract class Context {
    
    // 获取资源
    public abstract Resources getResources();
    
    // 获取包管理器
    public abstract PackageManager getPackageManager();
    
    // 获取 ContentResolver
    public abstract ContentResolver getContentResolver();
    
    // 获取主线程 Looper
    public abstract Looper getMainLooper();
    
    // 获取 Application Context
    public abstract Context getApplicationContext();
    
    // 获取系统服务
    public abstract Object getSystemService(String name);
    
    // 启动 Activity
    public abstract void startActivity(Intent intent);
    
    // 启动 Service
    public abstract ComponentName startService(Intent service);
    
    // 发送广播
    public abstract void sendBroadcast(Intent intent);
    
    // 注册广播接收器
    public abstract Intent registerReceiver(BroadcastReceiver receiver,
            IntentFilter filter);
    
    // 获取 SharedPreferences
    public abstract SharedPreferences getSharedPreferences(String name, int mode);
    
    // 获取文件目录
    public abstract File getFilesDir();
    
    // 获取缓存目录
    public abstract File getCacheDir();
    
    // ... 更多抽象方法
}
```

### 3.2 ContextWrapper 装饰器

```java
// frameworks/base/core/java/android/content/ContextWrapper.java
public class ContextWrapper extends Context {
    
    // 被装饰的 Context（通常是 ContextImpl）
    Context mBase;
    
    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    // 设置被装饰的 Context
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    
    // 获取被装饰的 Context
    public Context getBaseContext() {
        return mBase;
    }
    
    // 所有方法都委托给 mBase
    @Override
    public Resources getResources() {
        return mBase.getResources();
    }
    
    @Override
    public PackageManager getPackageManager() {
        return mBase.getPackageManager();
    }
    
    @Override
    public void startActivity(Intent intent) {
        mBase.startActivity(intent);
    }
    
    @Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
    
    @Override
    public Object getSystemService(String name) {
        return mBase.getSystemService(name);
    }
    
    // ... 其他方法同样委托
}
```

### 3.3 ContextImpl 真正实现

```java
// frameworks/base/core/java/android/app/ContextImpl.java
class ContextImpl extends Context {
    
    // ActivityThread 引用
    final ActivityThread mMainThread;
    
    // LoadedApk 引用（包含 ClassLoader、Resources 等）
    final LoadedApk mPackageInfo;
    
    // 资源管理器
    private Resources mResources;
    
    // 包名
    private String mBasePackageName;
    
    // ContentResolver
    private final ApplicationContentResolver mContentResolver;
    
    // 系统服务缓存
    final Object[] mServiceCache = SystemServiceRegistry.createServiceCache();
    
    @Override
    public Resources getResources() {
        return mResources;
    }
    
    @Override
    public Object getSystemService(String name) {
        // 从 SystemServiceRegistry 获取系统服务
        return SystemServiceRegistry.getSystemService(this, name);
    }
    
    @Override
    public void startActivity(Intent intent) {
        startActivity(intent, null);
    }
    
    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();
        
        // 非 Activity Context 启动 Activity 需要 FLAG_ACTIVITY_NEW_TASK
        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (options == null || !ActivityOptions.fromBundle(options).getLaunchTaskId() != -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
        
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
    
    @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, false, mUser);
    }
    
    @Override
    public File getFilesDir() {
        synchronized (mSync) {
            if (mFilesDir == null) {
                mFilesDir = new File(getDataDir(), "files");
            }
            return ensurePrivateDirExists(mFilesDir);
        }
    }
    
    // 创建 Activity 的 Context
    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken,
            int displayId, Configuration overrideConfiguration) {
        
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo,
                activityInfo.splitName, activityToken, null, 0, null, null);
        
        // 设置资源
        final ResourcesManager resourcesManager = ResourcesManager.getInstance();
        context.setResources(resourcesManager.createBaseActivityResources(
                activityToken, packageInfo.getResDir(), ...));
        
        return context;
    }
    
    // 创建 Application 的 Context
    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo,
                null, null, null, 0, null, null);
        context.setResources(packageInfo.getResources());
        return context;
    }
}
```

### 3.4 Application Context 创建

```java
// frameworks/base/core/java/android/app/LoadedApk.java
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    
    if (mApplication != null) {
        return mApplication;
    }
    
    Application app = null;
    
    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }
    
    try {
        java.lang.ClassLoader cl = getClassLoader();
        
        // 创建 Application 的 ContextImpl
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        
        // 通过反射创建 Application 实例
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        
        // 设置 Application 的 Context
        appContext.setOuterContext(app);
    } catch (Exception e) {
        // ...
    }
    
    mActivityThread.mAllApplications.add(app);
    mApplication = app;
    
    return app;
}
```

## 4. 实战应用

### 4.1 正确获取 Context

```kotlin
// Activity 中
class MyActivity : AppCompatActivity() {
    
    fun getContexts() {
        // Activity Context（包含主题信息）
        val activityContext: Context = this
        
        // Application Context（全局单例）
        val appContext: Context = applicationContext
        
        // Base Context（ContextImpl）
        val baseContext: Context = baseContext
    }
}

// Fragment 中
class MyFragment : Fragment() {
    
    fun getContexts() {
        // 可能为 null（Fragment 未 attach 时）
        val context: Context? = context
        
        // 非空，但可能抛异常
        val requireContext: Context = requireContext()
        
        // Activity 引用
        val activity: FragmentActivity? = activity
        
        // Application Context
        val appContext: Context? = context?.applicationContext
    }
}

// View 中
class MyView(context: Context) : View(context) {
    
    fun getContexts() {
        // 通常是 Activity Context
        val viewContext: Context = context
        
        // Application Context
        val appContext: Context = context.applicationContext
    }
}

// Service 中
class MyService : Service() {
    
    fun getContexts() {
        // Service Context
        val serviceContext: Context = this
        
        // Application Context
        val appContext: Context = applicationContext
    }
}
```

### 4.2 Context 使用场景选择

```kotlin
// ✅ 使用 Application Context 的场景
class MyRepository(private val context: Context) {
    
    // 单例持有 Context，使用 Application Context
    private val appContext = context.applicationContext
    
    fun getSharedPreferences(): SharedPreferences {
        return appContext.getSharedPreferences("prefs", Context.MODE_PRIVATE)
    }
    
    fun getDatabase(): MyDatabase {
        return Room.databaseBuilder(
            appContext,  // 使用 Application Context
            MyDatabase::class.java,
            "database"
        ).build()
    }
}

// ✅ 使用 Activity Context 的场景
class MyActivity : AppCompatActivity() {
    
    fun showDialog() {
        // Dialog 需要 Activity Context
        AlertDialog.Builder(this)  // 使用 Activity Context
            .setTitle("Title")
            .setMessage("Message")
            .show()
    }
    
    fun inflateView() {
        // 需要主题信息的 View 创建
        val view = LayoutInflater.from(this)  // 使用 Activity Context
            .inflate(R.layout.my_layout, null)
    }
    
    fun startActivity() {
        // Activity 启动 Activity，无需 FLAG_NEW_TASK
        startActivity(Intent(this, OtherActivity::class.java))
    }
}

// ⚠️ 需要注意的场景
class MyService : Service() {
    
    fun startActivityFromService() {
        // Service 启动 Activity 需要 FLAG_NEW_TASK
        val intent = Intent(this, MyActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        }
        startActivity(intent)
    }
    
    fun showToast() {
        // Toast 可以使用任何 Context
        Toast.makeText(this, "Message", Toast.LENGTH_SHORT).show()
    }
}
```

### 4.3 避免 Context 导致的内存泄漏

```kotlin
// ❌ 错误：单例持有 Activity Context
object BadSingleton {
    private lateinit var context: Context
    
    fun init(context: Context) {
        this.context = context  // 如果传入 Activity，会导致泄漏
    }
}

// ✅ 正确：单例持有 Application Context
object GoodSingleton {
    private lateinit var appContext: Context
    
    fun init(context: Context) {
        this.appContext = context.applicationContext
    }
}

// ❌ 错误：匿名内部类持有外部类引用
class LeakyActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Handler 的匿名 Runnable 持有 Activity 引用
        Handler(Looper.getMainLooper()).postDelayed({
            updateUI()  // 隐式持有 Activity 引用
        }, 60000)
    }
}

// ✅ 正确：使用弱引用或在 onDestroy 中清理
class SafeActivity : AppCompatActivity() {
    
    private val handler = Handler(Looper.getMainLooper())
    private val updateRunnable = Runnable { updateUI() }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handler.postDelayed(updateRunnable, 60000)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacks(updateRunnable)
    }
}

// ❌ 错误：静态变量持有 View
class BadView {
    companion object {
        var cachedView: View? = null  // View 持有 Context 引用
    }
}

// ✅ 正确：使用 WeakReference
class SafeCache {
    companion object {
        private var cachedViewRef: WeakReference<View>? = null
        
        fun cacheView(view: View) {
            cachedViewRef = WeakReference(view)
        }
        
        fun getCachedView(): View? = cachedViewRef?.get()
    }
}
```

### 4.4 自定义 Application

```kotlin
class MyApplication : Application() {
    
    companion object {
        // 不推荐：静态持有 Application 实例
        // lateinit var instance: MyApplication
        
        // 推荐：通过依赖注入或 ContentProvider 获取
    }
    
    override fun attachBaseContext(base: Context?) {
        super.attachBaseContext(base)
        // 最早的初始化时机
        // 此时 ContentProvider 还未创建
    }
    
    override fun onCreate() {
        super.onCreate()
        // 初始化第三方库
        // 此时 ContentProvider 已创建
    }
    
    override fun onConfigurationChanged(newConfig: Configuration) {
        super.onConfigurationChanged(newConfig)
        // 配置变更（如语言、屏幕方向）
    }
    
    override fun onLowMemory() {
        super.onLowMemory()
        // 内存不足，清理缓存
    }
    
    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        // 根据 level 清理不同级别的缓存
        when (level) {
            TRIM_MEMORY_UI_HIDDEN -> {
                // UI 不可见，可以释放 UI 相关资源
            }
            TRIM_MEMORY_RUNNING_LOW -> {
                // 内存较低
            }
            TRIM_MEMORY_COMPLETE -> {
                // 内存极低，尽可能释放资源
            }
        }
    }
}
```

## 5. 常见面试题

### 问题1：Context 的继承关系是怎样的？有哪些实现类？

**答案要点：**
- **继承关系**：
  ```
  Context (抽象类)
    └── ContextWrapper (装饰器)
          ├── Application
          ├── Service
          └── ContextThemeWrapper
                └── Activity
  ```
- **实现类**：
  - ContextImpl：真正的实现类，被 ContextWrapper 持有
  - Application、Activity、Service 都是 ContextWrapper 的子类
- **设计模式**：装饰器模式，ContextWrapper 持有 ContextImpl 并委托调用

### 问题2：Application Context 和 Activity Context 的区别？

**答案要点：**
| 特性 | Application Context | Activity Context |
|-----|---------------------|------------------|
| 生命周期 | 应用整个生命周期 | Activity 生命周期 |
| 主题信息 | 无 | 有 |
| 显示 Dialog | 不能 | 可以 |
| 启动 Activity | 需要 FLAG_NEW_TASK | 不需要 |
| 创建 View | 可以但无主题 | 可以且有主题 |
| 内存泄漏风险 | 低 | 高（被长生命周期对象持有时） |

### 问题3：什么情况下会导致 Context 内存泄漏？如何避免？

**答案要点：**
- **泄漏场景**：
  1. 单例持有 Activity Context
  2. 静态变量持有 View（View 持有 Context）
  3. 匿名内部类/非静态内部类持有外部 Activity 引用
  4. Handler 的 Message 持有 Activity 引用
  5. 异步任务持有 Activity 引用
- **避免方法**：
  1. 单例使用 Application Context
  2. 使用弱引用（WeakReference）
  3. 在 onDestroy 中清理引用
  4. 使用静态内部类 + 弱引用
  5. 使用 Lifecycle 感知组件

### 问题4：为什么不能用 Application Context 显示 Dialog？

**答案要点：**
- Dialog 需要依附于一个 **Window**
- Activity 有自己的 Window（PhoneWindow）
- Application Context 没有 Window
- 强行使用会抛出 `BadTokenException`
- 解决方案：
  1. 使用 Activity Context
  2. 使用 TYPE_APPLICATION_OVERLAY（需要权限）

### 问题5：getApplication() 和 getApplicationContext() 的区别？

**答案要点：**
- **getApplication()**：
  - 只有 Activity 和 Service 有此方法
  - 返回类型是 Application
  - 可以直接调用 Application 的方法
- **getApplicationContext()**：
  - Context 的方法，所有 Context 都有
  - 返回类型是 Context
  - 需要强转才能调用 Application 方法
- **实际返回**：通常返回同一个 Application 实例

### 问题6：一个应用有多少个 Context？

**答案要点：**
- **数量公式**：Context 数量 = Activity 数量 + Service 数量 + 1（Application）
- **注意**：
  - BroadcastReceiver 和 ContentProvider 不是 Context
  - 它们通过参数获取 Context
  - 每个 Activity/Service 实例都有独立的 Context

### 问题7：ContextWrapper 的作用是什么？为什么要用装饰器模式？

**答案要点：**
- **作用**：
  - 包装 ContextImpl，提供统一的 Context 接口
  - 允许子类重写部分方法
  - 实现 Context 的多态
- **装饰器模式优势**：
  1. 分离接口和实现
  2. 允许动态扩展功能
  3. 子类可以选择性重写方法
  4. ContextThemeWrapper 可以添加主题功能

### 问题8：如何在 ContentProvider 中获取 Context？

**答案要点：**
- ContentProvider 有 `getContext()` 方法
- 在 `onCreate()` 之后可以安全调用
- **注意**：
  - ContentProvider.onCreate() 在 Application.onCreate() 之前调用
  - 此时 Application 可能未完全初始化
  - 避免在 ContentProvider.onCreate() 中做复杂初始化
