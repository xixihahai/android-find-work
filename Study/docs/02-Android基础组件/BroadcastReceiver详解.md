# BroadcastReceiver 详解

## 1. 概述

BroadcastReceiver（广播接收器）是 Android 四大组件之一，用于接收系统或应用发出的广播消息。它是一种全局的监听器，可以在应用内或跨应用之间传递消息。

**广播的核心特点：**
- 基于发布-订阅模式
- 可以跨进程通信
- 支持有序广播和无序广播
- Android 8.0+ 对隐式广播有严格限制

**广播的分类：**
- **标准广播（Normal Broadcast）**：异步执行，所有接收器几乎同时收到
- **有序广播（Ordered Broadcast）**：按优先级顺序传递，可被拦截
- **本地广播（Local Broadcast）**：仅在应用内传递（已废弃）
- **粘性广播（Sticky Broadcast）**：已废弃

## 2. 核心原理

### 2.1 广播机制架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        发送广播                                  │
│  sendBroadcast() / sendOrderedBroadcast() / sendStickyBroadcast()│
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ActivityManagerService                         │
│  - 解析 Intent，匹配接收器                                        │
│  - 权限检查                                                       │
│  - 按优先级排序（有序广播）                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│    静态注册接收器         │     │    动态注册接收器         │
│  - Manifest 中声明       │     │  - 代码中注册            │
│  - 进程不存在时可唤醒     │     │  - 随组件生命周期        │
└─────────────────────────┘     └─────────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onReceive(context, intent)                    │
│  - 运行在主线程                                                   │
│  - 执行时间限制（约 10 秒）                                        │
│  - 不能执行耗时操作                                               │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 静态注册 vs 动态注册

| 特性 | 静态注册 | 动态注册 |
|-----|---------|---------|
| 注册方式 | AndroidManifest.xml | Context.registerReceiver() |
| 生命周期 | 常驻，进程不存在可唤醒 | 随注册组件生命周期 |
| 接收时机 | 应用安装后即可接收 | 注册后才能接收 |
| Android 8.0+ | 大部分隐式广播无法接收 | 不受限制 |
| 使用场景 | 系统广播（开机、安装等） | 应用内广播、动态事件 |

### 2.3 有序广播传递流程

```
┌─────────────────┐
│  发送有序广播    │
│ sendOrdered     │
│ Broadcast()     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Receiver A     │────►│  Receiver B     │────►│  Receiver C     │
│  priority=100   │     │  priority=50    │     │  priority=0     │
│                 │     │                 │     │                 │
│ 可以：           │     │ 可以：           │     │                 │
│ - 修改数据       │     │ - 修改数据       │     │                 │
│ - 终止传递       │     │ - 终止传递       │     │                 │
│   abortBroadcast│     │   abortBroadcast│     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## 3. 关键源码解析

### 3.1 动态注册广播接收器

```java
// frameworks/base/core/java/android/app/ContextImpl.java
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return registerReceiver(receiver, filter, null, null);
}

@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {
    return registerReceiverInternal(receiver, getUserId(),
            filter, broadcastPermission, scheduler, getOuterContext(), 0);
}

private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context, int flags) {
    
    IIntentReceiver rd = null;
    if (receiver != null) {
        // 创建 ReceiverDispatcher，封装 BroadcastReceiver
        if (mPackageInfo != null && context != null) {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            // 获取或创建 IIntentReceiver（Binder 对象）
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        } else {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            rd = new LoadedApk.ReceiverDispatcher(
                    receiver, context, scheduler, null, true).getIIntentReceiver();
        }
    }
    
    try {
        // 通过 Binder 调用 AMS 注册
        final Intent intent = ActivityManager.getService().registerReceiver(
                mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                broadcastPermission, userId, flags);
        if (intent != null) {
            intent.setExtrasClassLoader(getClassLoader());
            intent.prepareToEnterProcess();
        }
        return intent;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

### 3.2 发送广播

```java
// frameworks/base/core/java/android/app/ContextImpl.java
@Override
public void sendBroadcast(Intent intent) {
    warnIfCallingFromSystemProcess();
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        intent.prepareToLeaveProcess(this);
        // 调用 AMS 的 broadcastIntent
        ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                getUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}

@Override
public void sendOrderedBroadcast(Intent intent, String receiverPermission) {
    warnIfCallingFromSystemProcess();
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        intent.prepareToLeaveProcess(this);
        ActivityManager.getService().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, receiverPermission, AppOpsManager.OP_NONE,
                null, true,  // ordered = true，有序广播
                false, getUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

### 3.3 AMS 处理广播

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle resultExtras,
        String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean serialized, boolean sticky, int userId) {
    
    synchronized(this) {
        // 验证 Intent
        intent = verifyBroadcastLocked(intent);
        
        // 获取调用者信息
        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        
        // 委托给 broadcastIntentLocked 处理
        int res = broadcastIntentLocked(callerApp,
                callerApp != null ? callerApp.info.packageName : null,
                intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                requiredPermissions, appOp, bOptions, serialized, sticky,
                callingPid, callingUid, callingUid, callingPid, userId);
        
        return res;
    }
}
```

### 3.4 广播分发

```java
// frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java
final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
    BroadcastRecord r;
    
    // 1. 先处理无序广播（并行）
    while (mParallelBroadcasts.size() > 0) {
        r = mParallelBroadcasts.remove(0);
        final int N = r.receivers.size();
        for (int i = 0; i < N; i++) {
            Object target = r.receivers.get(i);
            // 分发给每个接收器
            deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
        }
    }
    
    // 2. 处理有序广播（串行）
    if (mPendingBroadcast != null) {
        // 等待上一个接收器处理完成
        return;
    }
    
    do {
        if (mOrderedBroadcasts.size() == 0) {
            return;
        }
        r = mOrderedBroadcasts.get(0);
        
        // 检查是否超时
        int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
        if (r.dispatchTime > 0) {
            long now = SystemClock.uptimeMillis();
            if ((numReceivers > 0) && (now > r.dispatchTime + (2 * mConstants.TIMEOUT * numReceivers))) {
                // 广播超时，强制结束
                broadcastTimeoutLocked(false);
                forceReceive = true;
                r.state = BroadcastRecord.IDLE;
            }
        }
        
        // 获取下一个接收器
        int recIdx = r.nextReceiver++;
        r.receiverTime = SystemClock.uptimeMillis();
        
        // 分发广播
        final Object nextReceiver = r.receivers.get(recIdx);
        if (nextReceiver instanceof BroadcastFilter) {
            // 动态注册的接收器
            deliverToRegisteredReceiverLocked(r, (BroadcastFilter)nextReceiver, r.ordered, recIdx);
        } else {
            // 静态注册的接收器
            ResolveInfo info = (ResolveInfo)nextReceiver;
            performReceiveLocked(r.callerApp, r.resultTo,
                    new Intent(r.intent), r.resultCode, r.resultData,
                    r.resultExtras, r.ordered, r.initialSticky, r.userId);
        }
    } while (r == null);
}
```

## 4. 实战应用

### 4.1 静态注册广播接收器

```xml
<!-- AndroidManifest.xml -->
<receiver
    android:name=".BootReceiver"
    android:enabled="true"
    android:exported="true">
    <intent-filter>
        <!-- 开机广播是 Android 8.0+ 豁免的隐式广播 -->
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

```kotlin
class BootReceiver : BroadcastReceiver() {
    
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // 开机完成，启动服务或执行初始化
            Log.d("BootReceiver", "Device boot completed")
            
            // 使用 WorkManager 调度任务（推荐）
            val workRequest = OneTimeWorkRequestBuilder<InitWorker>().build()
            WorkManager.getInstance(context).enqueue(workRequest)
        }
    }
}
```

### 4.2 动态注册广播接收器

```kotlin
class MainActivity : AppCompatActivity() {
    
    private val networkReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            val connectivityManager = context.getSystemService(ConnectivityManager::class.java)
            val network = connectivityManager.activeNetwork
            val capabilities = connectivityManager.getNetworkCapabilities(network)
            
            val isConnected = capabilities?.hasCapability(
                NetworkCapabilities.NET_CAPABILITY_INTERNET
            ) == true
            
            updateNetworkStatus(isConnected)
        }
    }
    
    override fun onStart() {
        super.onStart()
        // 动态注册
        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        registerReceiver(networkReceiver, filter)
    }
    
    override fun onStop() {
        super.onStop()
        // 必须注销，否则会内存泄漏
        unregisterReceiver(networkReceiver)
    }
    
    private fun updateNetworkStatus(isConnected: Boolean) {
        // 更新 UI
    }
}
```

### 4.3 发送自定义广播

```kotlin
// 发送标准广播
fun sendCustomBroadcast(context: Context) {
    val intent = Intent("com.example.MY_CUSTOM_ACTION").apply {
        putExtra("message", "Hello from broadcast")
        // Android 8.0+ 显式指定包名
        setPackage(context.packageName)
    }
    context.sendBroadcast(intent)
}

// 发送有序广播
fun sendOrderedCustomBroadcast(context: Context) {
    val intent = Intent("com.example.MY_ORDERED_ACTION").apply {
        setPackage(context.packageName)
    }
    
    context.sendOrderedBroadcast(
        intent,
        null,  // receiverPermission
        object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                // 最终接收器，所有接收器处理完后调用
                val resultData = getResultData()
                Log.d("TAG", "Final result: $resultData")
            }
        },
        null,  // scheduler
        Activity.RESULT_OK,
        "initial data",
        null   // initialExtras
    )
}

// 接收有序广播
class OrderedReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 获取上一个接收器传递的数据
        val data = getResultData()
        
        // 修改数据传递给下一个接收器
        setResultData("$data -> processed by OrderedReceiver")
        
        // 可以终止广播传递
        // abortBroadcast()
    }
}
```

### 4.4 带权限的广播

```xml
<!-- 定义自定义权限 -->
<permission
    android:name="com.example.MY_BROADCAST_PERMISSION"
    android:protectionLevel="signature" />

<!-- 声明使用权限 -->
<uses-permission android:name="com.example.MY_BROADCAST_PERMISSION" />

<!-- 接收器需要权限 -->
<receiver
    android:name=".SecureReceiver"
    android:permission="com.example.MY_BROADCAST_PERMISSION"
    android:exported="true">
    <intent-filter>
        <action android:name="com.example.SECURE_ACTION" />
    </intent-filter>
</receiver>
```

```kotlin
// 发送带权限的广播
fun sendSecureBroadcast(context: Context) {
    val intent = Intent("com.example.SECURE_ACTION")
    context.sendBroadcast(intent, "com.example.MY_BROADCAST_PERMISSION")
}
```

### 4.5 Android 8.0+ 广播限制与解决方案

**受限的隐式广播：**
- 大部分隐式广播无法通过静态注册接收
- 只有少数系统广播被豁免

**豁免的隐式广播（部分）：**
- `ACTION_BOOT_COMPLETED` - 开机完成
- `ACTION_LOCALE_CHANGED` - 语言变更
- `ACTION_USB_ACCESSORY_ATTACHED` - USB 连接
- `ACTION_HEADSET_PLUG` - 耳机插拔
- `ACTION_CONNECTION_STATE_CHANGED` - 蓝牙连接状态

**解决方案：**

```kotlin
// 方案1：动态注册（推荐）
class MyActivity : AppCompatActivity() {
    private val receiver = MyReceiver()
    
    override fun onStart() {
        super.onStart()
        val filter = IntentFilter("com.example.MY_ACTION")
        registerReceiver(receiver, filter)
    }
    
    override fun onStop() {
        super.onStop()
        unregisterReceiver(receiver)
    }
}

// 方案2：显式广播
fun sendExplicitBroadcast(context: Context) {
    val intent = Intent("com.example.MY_ACTION").apply {
        // 显式指定接收器
        setClassName("com.example.app", "com.example.app.MyReceiver")
        // 或指定包名
        // setPackage("com.example.app")
    }
    context.sendBroadcast(intent)
}

// 方案3：使用 JobScheduler 替代
class MyJobService : JobService() {
    override fun onStartJob(params: JobParameters?): Boolean {
        // 处理原本在广播中执行的逻辑
        return false
    }
    
    override fun onStopJob(params: JobParameters?): Boolean = false
}
```

### 4.6 本地广播（已废弃）

> **注意：** LocalBroadcastManager 已在 AndroidX 1.1.0 中废弃，推荐使用 LiveData、Flow 或其他观察者模式替代。

```kotlin
// 废弃的方式
class OldWay {
    fun sendLocalBroadcast(context: Context) {
        val intent = Intent("com.example.LOCAL_ACTION")
        LocalBroadcastManager.getInstance(context).sendBroadcast(intent)
    }
    
    fun registerLocalReceiver(context: Context) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                // 处理广播
            }
        }
        LocalBroadcastManager.getInstance(context)
            .registerReceiver(receiver, IntentFilter("com.example.LOCAL_ACTION"))
    }
}

// 推荐替代方案：使用 SharedFlow
class EventBus {
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    suspend fun emit(event: Event) {
        _events.emit(event)
    }
    
    companion object {
        val instance = EventBus()
    }
}

// 发送事件
lifecycleScope.launch {
    EventBus.instance.emit(MyEvent("data"))
}

// 接收事件
lifecycleScope.launch {
    EventBus.instance.events.collect { event ->
        // 处理事件
    }
}
```

## 5. 常见面试题

### 问题1：静态注册和动态注册广播接收器有什么区别？

**答案要点：**
| 特性 | 静态注册 | 动态注册 |
|-----|---------|---------|
| 注册位置 | AndroidManifest.xml | 代码中 registerReceiver() |
| 生命周期 | 常驻，应用未启动也能接收 | 随注册组件生命周期 |
| Android 8.0+ | 大部分隐式广播无法接收 | 不受限制 |
| 注销 | 无需手动注销 | 必须调用 unregisterReceiver() |
| 使用场景 | 开机广播等系统事件 | 应用内事件、网络变化等 |

### 问题2：有序广播和无序广播的区别？

**答案要点：**
- **无序广播（标准广播）**：
  - 使用 sendBroadcast() 发送
  - 所有接收器几乎同时收到
  - 接收器之间无法通信
  - 无法被拦截
- **有序广播**：
  - 使用 sendOrderedBroadcast() 发送
  - 按 priority 优先级顺序传递
  - 可以通过 setResultData() 传递数据
  - 可以通过 abortBroadcast() 终止传递
  - 可以设置最终接收器

### 问题3：Android 8.0 对广播有什么限制？如何解决？

**答案要点：**
- **限制**：静态注册的接收器无法接收大部分隐式广播
- **豁免广播**：开机完成、语言变更、USB 连接等少数系统广播
- **解决方案**：
  1. 使用动态注册
  2. 发送显式广播（指定包名或组件名）
  3. 使用 JobScheduler/WorkManager 替代
  4. 使用 LiveData/Flow 替代应用内广播

### 问题4：onReceive 方法有什么限制？如何执行耗时操作？

**答案要点：**
- **限制**：
  - 运行在主线程
  - 执行时间约 10 秒，超时会 ANR
  - 不能执行耗时操作
  - 不能显示 Dialog（没有 Activity Context）
- **执行耗时操作**：
  1. 启动 Service（推荐前台服务）
  2. 使用 goAsync() 延长生命周期
  ```kotlin
  override fun onReceive(context: Context, intent: Intent) {
      val pendingResult = goAsync()
      CoroutineScope(Dispatchers.IO).launch {
          try {
              doHeavyWork()
          } finally {
              pendingResult.finish()
          }
      }
  }
  ```
  3. 使用 WorkManager 调度任务

### 问题5：LocalBroadcastManager 为什么被废弃？有什么替代方案？

**答案要点：**
- **废弃原因**：
  - 设计上与全局广播耦合
  - 不支持 Lifecycle 感知
  - 有更好的替代方案
- **替代方案**：
  1. **LiveData**：生命周期感知，适合 UI 更新
  2. **SharedFlow/StateFlow**：Kotlin 协程，更灵活
  3. **EventBus**：第三方库，功能丰富
  4. **接口回调**：简单场景

### 问题6：粘性广播是什么？为什么被废弃？

**答案要点：**
- **粘性广播**：
  - 使用 sendStickyBroadcast() 发送
  - 广播会保留，后注册的接收器也能收到
  - 典型应用：电池状态广播
- **废弃原因**：
  - 安全问题：任何应用都能访问
  - 无法保证数据时效性
  - 容易造成内存泄漏
- **替代方案**：
  - 使用 LiveData/StateFlow 保存状态
  - 使用 SharedPreferences 持久化
  - 主动查询状态而非被动接收

### 问题7：广播的发送和接收流程是怎样的？

**答案要点：**
1. **发送端**：
   - 调用 sendBroadcast() 或 sendOrderedBroadcast()
   - 通过 Binder 调用 AMS 的 broadcastIntent()
2. **AMS 处理**：
   - 解析 Intent，匹配接收器
   - 权限检查
   - 将广播加入队列（并行队列或有序队列）
3. **分发**：
   - BroadcastQueue 处理队列
   - 无序广播并行分发
   - 有序广播串行分发
4. **接收端**：
   - 通过 ApplicationThread 回调
   - 在主线程执行 onReceive()

### 问题8：如何保证广播的安全性？

**答案要点：**
1. **发送端保护**：
   - 使用 setPackage() 指定接收包名
   - 使用权限限制接收者
   ```kotlin
   sendBroadcast(intent, "com.example.MY_PERMISSION")
   ```
2. **接收端保护**：
   - 设置 android:exported="false"
   - 声明接收权限
   ```xml
   <receiver android:permission="com.example.MY_PERMISSION">
   ```
3. **使用签名级权限**：
   ```xml
   <permission android:protectionLevel="signature" />
   ```
4. **使用本地通信替代**：LiveData、Flow、EventBus
