# Activity 详解

## 1. 概述

Activity 是 Android 四大组件之一，代表一个具有用户界面的单一屏幕。它是用户与应用交互的入口点，负责管理用户界面和处理用户输入。每个 Activity 都有自己的生命周期，由系统通过 ActivityManager 进行管理。

Activity 的核心职责：
- 提供用户界面（通过 setContentView 设置布局）
- 处理用户交互事件
- 管理 Fragment 和其他 UI 组件
- 与其他 Activity 或应用进行交互

## 2. 核心原理

### 2.1 生命周期详解

```
                    ┌─────────────────────────────────────────┐
                    │              Activity 启动               │
                    └─────────────────┬───────────────────────┘
                                      ▼
                    ┌─────────────────────────────────────────┐
                    │            onCreate()                    │
                    │   - 创建 Activity，初始化必要组件          │
                    │   - 调用 setContentView() 设置布局        │
                    │   - 恢复保存的状态（如果有）               │
                    └─────────────────┬───────────────────────┘
                                      ▼
                    ┌─────────────────────────────────────────┐
                    │            onStart()                     │
                    │   - Activity 变为可见                     │
                    │   - 注册广播接收器等                       │
                    └─────────────────┬───────────────────────┘
                                      ▼
                    ┌─────────────────────────────────────────┐
                    │            onResume()                    │
                    │   - Activity 获得焦点，可与用户交互        │
                    │   - 开始动画、获取独占资源（如相机）        │
                    └─────────────────┬───────────────────────┘
                                      ▼
                    ┌─────────────────────────────────────────┐
                    │         Activity Running                 │
                    │   - 用户正在与 Activity 交互              │
                    └─────────────────┬───────────────────────┘

                                      │
                    ┌─────────────────┴───────────────────────┐
                    │            onPause()                     │
                    │   - Activity 失去焦点（部分可见）          │
                    │   - 暂停动画、释放独占资源                 │
                    │   - 保存关键数据（快速执行）               │
                    └─────────────────┬───────────────────────┘
                                      ▼
                    ┌─────────────────────────────────────────┐
                    │            onStop()                      │
                    │   - Activity 完全不可见                   │
                    │   - 释放不需要的资源                       │
                    │   - 注销广播接收器                         │
                    └─────────────────┬───────────────────────┘
                                      ▼
              ┌───────────────────────┴───────────────────────┐
              │                                               │
              ▼                                               ▼
┌─────────────────────────┐                 ┌─────────────────────────┐
│      onRestart()        │                 │      onDestroy()        │
│  - Activity 重新启动     │                 │  - Activity 被销毁       │
│  - 从 onStop 恢复        │                 │  - 释放所有资源          │
└───────────┬─────────────┘                 └─────────────────────────┘
            │
            └──────────────► onStart()
```

#### 生命周期回调时机详解

| 回调方法 | 调用时机 | 典型操作 |
|---------|---------|---------|
| onCreate() | Activity 首次创建时 | 初始化 UI、绑定数据、恢复状态 |
| onStart() | Activity 变为可见时 | 注册广播、初始化 UI 更新 |
| onResume() | Activity 获得焦点时 | 开始动画、获取相机等独占资源 |
| onPause() | Activity 失去焦点时 | 暂停动画、保存关键数据 |
| onStop() | Activity 完全不可见时 | 释放资源、注销广播 |
| onDestroy() | Activity 被销毁时 | 释放所有资源 |
| onRestart() | 从 onStop 状态恢复时 | 重新初始化在 onStop 中释放的资源 |

#### 异常情况下的生命周期

**1. 配置变更（如屏幕旋转）**

```
onPause() → onStop() → onSaveInstanceState() → onDestroy()
    ↓
onCreate(savedInstanceState) → onStart() → onRestoreInstanceState() → onResume()
```

**2. 内存不足被系统回收**
```
onPause() → onStop() → onSaveInstanceState() → [系统回收]
    ↓ (用户返回)
onCreate(savedInstanceState) → onStart() → onRestoreInstanceState() → onResume()
```

**3. 透明 Activity 或 Dialog 覆盖**
```
onPause() → [Dialog 显示] → onResume()  // 不会调用 onStop
```

### 2.2 启动模式（Launch Mode）

#### 四种标准启动模式

| 启动模式 | 说明 | 使用场景 |
|---------|------|---------|
| standard | 默认模式，每次启动都创建新实例 | 普通 Activity |
| singleTop | 栈顶复用，调用 onNewIntent() | 通知跳转、搜索页面 |
| singleTask | 栈内复用，清除其上所有 Activity | 主页面、登录页 |
| singleInstance | 独占一个任务栈 | 来电界面、闹钟 |

#### Android 12+ 新增：singleInstancePerTask

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleInstancePerTask" />
```

- 每个任务栈中只能有一个该 Activity 实例
- 不同任务栈可以有不同实例
- 适用于多窗口场景

#### 启动模式详细行为

**standard 模式：**
```
任务栈: [A] → 启动 B → [A, B] → 启动 B → [A, B, B]
```

**singleTop 模式：**
```
任务栈: [A, B] → 启动 B(singleTop) → [A, B] (调用 B.onNewIntent())
任务栈: [A, B] → 启动 A(singleTop) → [A, B, A] (创建新实例)
```

**singleTask 模式：**
```
任务栈: [A, B, C] → 启动 A(singleTask) → [A] (B、C 被销毁，调用 A.onNewIntent())
```

**singleInstance 模式：**
```
任务栈1: [A, B] → 启动 C(singleInstance) 
任务栈1: [A, B]
任务栈2: [C]  // C 独占新任务栈
```

### 2.3 Intent Flags

常用的 Intent Flags：

| Flag | 作用 |
|------|------|
| FLAG_ACTIVITY_NEW_TASK | 在新任务栈中启动 Activity |
| FLAG_ACTIVITY_CLEAR_TOP | 清除目标 Activity 之上的所有 Activity |
| FLAG_ACTIVITY_SINGLE_TOP | 等同于 singleTop 启动模式 |
| FLAG_ACTIVITY_CLEAR_TASK | 清除任务栈中所有 Activity，需配合 NEW_TASK |
| FLAG_ACTIVITY_NO_HISTORY | Activity 不保留在任务栈中 |
| FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS | 不在最近任务列表中显示 |

```kotlin
// 常见组合：清除栈并启动新 Activity
val intent = Intent(this, MainActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
}
startActivity(intent)
```

## 3. 关键源码解析

### 3.1 Activity 启动流程源码分析

Activity 启动涉及多个进程和组件的协作：

```
App 进程                    system_server 进程
   │                              │
   │  startActivity()             │
   ├─────────────────────────────►│
   │                              │ ActivityTaskManagerService
   │                              │ (ATMS，Android 10+)
   │                              │
   │                              │ ActivityStarter.execute()
   │                              │      │
   │                              │      ▼
   │                              │ ActivityStack.startActivityLocked()
   │                              │      │
   │                              │      ▼
   │                              │ 创建 ActivityRecord
   │                              │      │
   │                              │      ▼
   │  scheduleLaunchActivity()    │ ApplicationThread.scheduleLaunchActivity()
   │◄─────────────────────────────┤
   │                              │
   │ ActivityThread               │
   │ handleLaunchActivity()       │
   │      │                       │
   │      ▼                       │
   │ performLaunchActivity()      │
   │      │                       │
   │      ▼                       │
   │ Activity.onCreate()          │
```

#### 关键源码 1：Activity.startActivity()

```java
// frameworks/base/core/java/android/app/Activity.java
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    // 第二个参数 -1 表示不需要返回结果
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}

public void startActivityForResult(Intent intent, int requestCode, Bundle options) {
    // mParent 是 ActivityGroup，已废弃
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        // 核心：通过 Instrumentation 启动 Activity
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this,                    // Context
                mMainThread.getApplicationThread(), // IApplicationThread Binder
                mToken,                  // Activity 的 IBinder 标识
                this,                    // 发起者 Activity
                intent,                  // 启动意图
                requestCode,             // 请求码
                options);                // 启动选项
        // 处理返回结果
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        // ...
    }
}
```

#### 关键源码 2：Instrumentation.execStartActivity()

```java
// frameworks/base/core/java/android/app/Instrumentation.java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    
    // IApplicationThread 是 App 进程的 Binder 服务端
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    
    // ... 省略 ActivityMonitor 相关代码
    
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        
        // 核心：通过 Binder 调用 ATMS 的 startActivity
        // ActivityTaskManager.getService() 返回 ATMS 的代理对象
        int result = ActivityTaskManager.getService()
            .startActivity(
                whoThread,           // 调用者的 ApplicationThread
                who.getBasePackageName(), // 包名
                intent,              // Intent
                intent.resolveTypeIfNeeded(who.getContentResolver()),
                token,               // 调用者 Activity 的 token
                target != null ? target.mEmbeddedID : null,
                requestCode,         // 请求码
                0,                   // startFlags
                null,                // profilerInfo
                options);            // ActivityOptions
        
        // 检查启动结果
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

#### 关键源码 3：ActivityTaskManagerService.startActivity()

```java
// frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho,
        int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho,
        int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions, int userId) {
    // 检查调用者权限
    enforceNotIsolatedCaller("startActivityAsUser");
    
    // 获取 ActivityStarter 并执行启动
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setUserId(userId)
            .execute();  // 执行启动流程
}
```

#### 关键源码 4：ActivityThread.handleLaunchActivity()

```java
// frameworks/base/core/java/android/app/ActivityThread.java
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    // 初始化 WindowManagerGlobal
    WindowManagerGlobal.initialize();
    
    // 创建 Activity 实例并调用 onCreate
    final Activity a = performLaunchActivity(r, customIntent);
    
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        if (!r.activity.mFinished && pendingActions != null) {
            pendingActions.setOldState(r.state);
            pendingActions.setRestoreInstanceState(true);
            pendingActions.setCallOnPostCreate(true);
        }
    }
    
    return a;
}

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // 1. 获取 ActivityInfo
    ActivityInfo aInfo = r.activityInfo;
    
    // 2. 获取 LoadedApk（包含 ClassLoader）
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    
    // 3. 获取 ComponentName
    ComponentName component = r.intent.getComponent();
    
    // 4. 创建 Context
    ContextImpl appContext = createBaseContextForActivity(r);
    
    Activity activity = null;
    try {
        // 5. 通过反射创建 Activity 实例
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
    } catch (Exception e) {
        // ...
    }
    
    try {
        // 6. 创建或获取 Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        
        if (activity != null) {
            // 7. 创建 Configuration
            Configuration config = new Configuration(mCompatConfiguration);
            
            // 8. 调用 Activity.attach()，关联 Window、Context 等
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);
            
            // 9. 设置主题
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }
            
            // 10. 调用 onCreate
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
        }
    } catch (Exception e) {
        // ...
    }
    
    return activity;
}
```

### 3.2 状态保存与恢复

#### onSaveInstanceState 调用时机

```java
// frameworks/base/core/java/android/app/Activity.java
protected void onSaveInstanceState(@NonNull Bundle outState) {
    // 保存 View 层级状态
    outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
    
    // 保存 Fragment 状态
    outState.putParcelable(FRAGMENTS_TAG, mFragments.saveAllState());
    
    // 保存 ActivityLifecycleCallbacks 状态
    if (mAutoFillResetNeeded) {
        outState.putBoolean(AUTOFILL_RESET_NEEDED, true);
    }
    
    // 调用 ActivityLifecycleCallbacks
    dispatchActivitySaveInstanceState(outState);
}
```

**调用时机（Android 9+）：**
- 在 `onStop()` 之后调用
- 配置变更时
- 按 Home 键
- 启动其他 Activity
- 屏幕关闭

**注意：** Android 9 之前在 `onStop()` 之前调用。

#### 状态恢复

```kotlin
class MyActivity : AppCompatActivity() {
    
    private var counter = 0
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // 方式1：在 onCreate 中恢复
        savedInstanceState?.let {
            counter = it.getInt("counter", 0)
        }
    }
    
    override fun onRestoreInstanceState(savedInstanceState: Bundle) {
        super.onRestoreInstanceState(savedInstanceState)
        // 方式2：在 onRestoreInstanceState 中恢复
        // 此方法只在有保存状态时才会调用
        counter = savedInstanceState.getInt("counter", 0)
    }
    
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putInt("counter", counter)
    }
}
```

### 3.3 Activity Result API（推荐方式）

```kotlin
// 新的 Activity Result API（AndroidX Activity 1.2.0+）
class MyActivity : AppCompatActivity() {
    
    // 1. 注册 ActivityResultLauncher
    private val getContent = registerForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri: Uri? ->
        // 处理返回的 URI
        uri?.let { handleSelectedImage(it) }
    }
    
    // 2. 自定义 Contract
    private val customLauncher = registerForActivityResult(
        object : ActivityResultContract<String, Int>() {
            override fun createIntent(context: Context, input: String): Intent {
                return Intent(context, SecondActivity::class.java).apply {
                    putExtra("data", input)
                }
            }
            
            override fun parseResult(resultCode: Int, intent: Intent?): Int {
                return if (resultCode == RESULT_OK) {
                    intent?.getIntExtra("result", 0) ?: 0
                } else {
                    -1
                }
            }
        }
    ) { result ->
        // 处理结果
        Log.d("TAG", "Result: $result")
    }
    
    fun selectImage() {
        // 3. 启动
        getContent.launch("image/*")
    }
    
    fun startCustomActivity() {
        customLauncher.launch("input_data")
    }
}
```

**常用内置 Contract：**
- `StartActivityForResult` - 通用启动
- `RequestPermission` - 请求单个权限
- `RequestMultiplePermissions` - 请求多个权限
- `TakePicture` - 拍照
- `TakePicturePreview` - 拍照预览
- `GetContent` - 选择内容
- `CreateDocument` - 创建文档
- `OpenDocument` - 打开文档

### 3.4 多窗口与画中画

#### 多窗口模式

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:resizeableActivity="true"
    android:supportsPictureInPicture="true"
    android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation">
    
    <!-- 多窗口布局配置 -->
    <layout
        android:defaultHeight="500dp"
        android:defaultWidth="600dp"
        android:gravity="top|end"
        android:minHeight="450dp"
        android:minWidth="300dp" />
</activity>
```

```kotlin
// 检查多窗口模式
override fun onMultiWindowModeChanged(isInMultiWindowMode: Boolean, newConfig: Configuration) {
    super.onMultiWindowModeChanged(isInMultiWindowMode, newConfig)
    if (isInMultiWindowMode) {
        // 进入多窗口模式
    } else {
        // 退出多窗口模式
    }
}
```

#### 画中画模式（PiP）

```kotlin
class VideoActivity : AppCompatActivity() {
    
    override fun onUserLeaveHint() {
        super.onUserLeaveHint()
        // 用户按 Home 键时进入画中画
        enterPictureInPictureMode()
    }
    
    private fun enterPictureInPictureMode() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val params = PictureInPictureParams.Builder()
                .setAspectRatio(Rational(16, 9))
                .setActions(listOf(
                    RemoteAction(
                        Icon.createWithResource(this, R.drawable.ic_play),
                        "Play",
                        "Play video",
                        PendingIntent.getBroadcast(
                            this, 0,
                            Intent("PLAY_ACTION"),
                            PendingIntent.FLAG_IMMUTABLE
                        )
                    )
                ))
                .build()
            
            enterPictureInPictureMode(params)
        }
    }
    
    override fun onPictureInPictureModeChanged(
        isInPictureInPictureMode: Boolean,
        newConfig: Configuration
    ) {
        super.onPictureInPictureModeChanged(isInPictureInPictureMode, newConfig)
        if (isInPictureInPictureMode) {
            // 隐藏不需要的 UI 元素
            hideControls()
        } else {
            // 恢复 UI
            showControls()
        }
    }
}
```

## 4. 实战应用

### 4.1 最佳实践

**1. 生命周期管理**
```kotlin
class MyActivity : AppCompatActivity() {
    
    // 使用 Lifecycle 组件管理生命周期
    private val locationObserver = object : DefaultLifecycleObserver {
        override fun onStart(owner: LifecycleOwner) {
            startLocationUpdates()
        }
        
        override fun onStop(owner: LifecycleOwner) {
            stopLocationUpdates()
        }
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(locationObserver)
    }
}
```

**2. 配置变更处理**
```kotlin
// 方式1：ViewModel 保存数据（推荐）
class MyViewModel : ViewModel() {
    val data = MutableLiveData<String>()
}

// 方式2：声明处理的配置变更
// android:configChanges="orientation|screenSize|keyboardHidden"
override fun onConfigurationChanged(newConfig: Configuration) {
    super.onConfigurationChanged(newConfig)
    // 手动处理配置变更
}
```

### 4.2 常见坑点

**1. 内存泄漏**
```kotlin
// ❌ 错误：匿名内部类持有 Activity 引用
handler.postDelayed({
    updateUI()
}, 10000)

// ✅ 正确：使用弱引用或在 onDestroy 中移除
private val updateRunnable = Runnable { updateUI() }

override fun onDestroy() {
    super.onDestroy()
    handler.removeCallbacks(updateRunnable)
}
```

**2. Fragment 事务在 onSaveInstanceState 后执行**
```kotlin
// ❌ 错误：可能抛出 IllegalStateException
override fun onStop() {
    super.onStop()
    supportFragmentManager.beginTransaction()
        .replace(R.id.container, MyFragment())
        .commit()
}

// ✅ 正确：使用 commitAllowingStateLoss 或检查状态
if (!isFinishing && !isDestroyed) {
    supportFragmentManager.beginTransaction()
        .replace(R.id.container, MyFragment())
        .commitAllowingStateLoss()
}
```

**3. startActivityForResult 已废弃**
```kotlin
// ❌ 废弃方式
startActivityForResult(intent, REQUEST_CODE)

// ✅ 推荐：使用 Activity Result API
val launcher = registerForActivityResult(StartActivityForResult()) { result ->
    // 处理结果
}
launcher.launch(intent)
```

## 5. 常见面试题

### 问题1：Activity 的生命周期有哪些？各个生命周期的调用时机是什么？

**答案要点：**
- 7 个生命周期回调：onCreate、onStart、onResume、onPause、onStop、onDestroy、onRestart
- onCreate：Activity 首次创建，初始化 UI 和数据
- onStart：Activity 变为可见，但不可交互
- onResume：Activity 获得焦点，可与用户交互
- onPause：Activity 失去焦点，部分可见（如 Dialog 覆盖）
- onStop：Activity 完全不可见
- onDestroy：Activity 被销毁
- onRestart：从 onStop 状态恢复时调用

### 问题2：Activity A 启动 Activity B，两者的生命周期调用顺序是什么？

**答案要点：**
```
A.onPause() → B.onCreate() → B.onStart() → B.onResume() → A.onStop()
```
- A 先 onPause，确保 A 的关键数据已保存
- B 完成启动后，A 才 onStop
- 如果 B 是透明 Activity，A 不会调用 onStop

### 问题3：四种启动模式的区别和使用场景？

**答案要点：**
- **standard**：默认模式，每次创建新实例，适用于普通页面
- **singleTop**：栈顶复用，调用 onNewIntent()，适用于通知跳转、搜索页面
- **singleTask**：栈内复用，清除其上 Activity，适用于主页面
- **singleInstance**：独占任务栈，适用于来电界面、闹钟
- Android 12+ 新增 singleInstancePerTask，每个任务栈一个实例

### 问题4：onSaveInstanceState 和 onRestoreInstanceState 的调用时机？

**答案要点：**
- onSaveInstanceState：
  - Android 9+：在 onStop() 之后调用
  - Android 9 之前：在 onStop() 之前调用
  - 调用场景：配置变更、按 Home 键、启动其他 Activity、屏幕关闭
- onRestoreInstanceState：
  - 在 onStart() 之后、onResume() 之前调用
  - 只有在有保存状态时才会调用
- 也可以在 onCreate 中通过 savedInstanceState 参数恢复

### 问题5：Activity 启动流程涉及哪些组件？简述流程。

**答案要点：**
1. **App 进程**：Activity.startActivity() → Instrumentation.execStartActivity()
2. **Binder IPC**：通过 AIDL 调用 system_server 进程
3. **system_server 进程**：
   - ActivityTaskManagerService (ATMS) 接收请求
   - ActivityStarter 处理启动逻辑
   - 创建 ActivityRecord
4. **回到 App 进程**：
   - ApplicationThread.scheduleLaunchActivity()
   - ActivityThread.handleLaunchActivity()
   - performLaunchActivity() 创建 Activity 实例
   - 调用 Activity.attach() 和 onCreate()

### 问题6：如何处理 Activity 的配置变更（如屏幕旋转）？

**答案要点：**
1. **默认行为**：Activity 销毁重建，通过 onSaveInstanceState 保存状态
2. **ViewModel**：推荐方式，数据在配置变更时保留
3. **configChanges 属性**：声明自己处理的配置变更，不重建 Activity
   ```xml
   android:configChanges="orientation|screenSize|keyboardHidden"
   ```
4. **onConfigurationChanged**：手动处理配置变更

### 问题7：Activity Result API 相比 startActivityForResult 有什么优势？

**答案要点：**
1. **类型安全**：通过 Contract 定义输入输出类型
2. **解耦**：可以在任何地方注册，不限于 Activity
3. **可测试**：Contract 可以单独测试
4. **简化代码**：不需要 requestCode，避免 onActivityResult 中的 switch-case
5. **内置 Contract**：提供常用操作如权限请求、拍照、选择文件等

### 问题8：什么情况下 Activity 的 onDestroy 不会被调用？

**答案要点：**
1. **进程被杀死**：系统内存不足直接杀死进程
2. **调用 System.exit()**：强制退出进程
3. **ANR 后被杀死**：应用无响应被系统杀死
4. **Crash**：应用崩溃

因此不应该在 onDestroy 中做关键数据保存，应该在 onPause 或 onStop 中保存。
