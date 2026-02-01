# Service 详解

## 1. 概述

Service 是 Android 四大组件之一，用于在后台执行长时间运行的操作，不提供用户界面。Service 可以由其他组件启动，即使用户切换到其他应用，Service 仍可在后台继续运行。

**Service 的核心特点：**
- 运行在主线程，耗时操作需要开启子线程
- 没有用户界面
- 可以被其他组件绑定并与之交互
- 优先级高于后台 Activity

**Service 的两种形式：**
- **启动服务（Started Service）**：通过 startService() 启动，独立运行
- **绑定服务（Bound Service）**：通过 bindService() 绑定，提供客户端-服务器接口

## 2. 核心原理

### 2.1 Service 生命周期

```
                    ┌─────────────────────────────────────────┐
                    │              Service 启动               │
                    └─────────────────┬───────────────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                               │
              ▼                                               ▼
┌─────────────────────────┐                 ┌─────────────────────────┐
│    startService()       │                 │    bindService()        │
└───────────┬─────────────┘                 └───────────┬─────────────┘
            │                                           │
            ▼                                           ▼
┌─────────────────────────┐                 ┌─────────────────────────┐
│      onCreate()         │                 │      onCreate()         │
│  - 首次创建时调用        │                 │  - 首次创建时调用        │
└───────────┬─────────────┘                 └───────────┬─────────────┘
            │                                           │
            ▼                                           ▼
┌─────────────────────────┐                 ┌─────────────────────────┐
│   onStartCommand()      │                 │      onBind()           │
│  - 每次 startService    │                 │  - 返回 IBinder         │
│    都会调用             │                 │  - 只调用一次           │
└───────────┬─────────────┘                 └───────────┬─────────────┘
            │                                           │
            ▼                                           ▼
┌─────────────────────────┐                 ┌─────────────────────────┐
│    Service Running      │                 │    Service Running      │
└───────────┬─────────────┘                 └───────────┬─────────────┘
            │                                           │
            ▼                                           ▼
┌─────────────────────────┐                 ┌─────────────────────────┐
│    stopService() 或     │                 │    unbindService()      │
│    stopSelf()           │                 │    (所有客户端解绑)      │
└───────────┬─────────────┘                 └───────────┬─────────────┘
            │                                           │
            ▼                                           ▼
┌─────────────────────────┐                 ┌─────────────────────────┐
│     onDestroy()         │                 │     onUnbind()          │
│  - 释放资源             │                 │     onDestroy()         │
└─────────────────────────┘                 └─────────────────────────┘
```

### 2.2 startService vs bindService

| 特性 | startService | bindService |
|-----|--------------|-------------|
| 启动方式 | startService(intent) | bindService(intent, conn, flags) |
| 生命周期 | 独立于启动者 | 依赖于绑定者 |
| 通信方式 | Intent 传递数据 | IBinder 接口 |
| 停止方式 | stopService() 或 stopSelf() | 所有客户端 unbindService() |
| 回调方法 | onStartCommand() | onBind() |
| 使用场景 | 后台下载、音乐播放 | 需要交互的服务 |

### 2.3 onStartCommand 返回值

```java
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    // flags: 0, START_FLAG_REDELIVERY, START_FLAG_RETRY
    // startId: 唯一标识此次启动
    
    return START_STICKY; // 返回值决定系统如何重启服务
}
```

| 返回值 | 说明 |
|-------|------|
| START_NOT_STICKY | 系统不会重建服务，除非有待处理的 Intent |
| START_STICKY | 系统重建服务并调用 onStartCommand()，但 Intent 为 null |
| START_REDELIVER_INTENT | 系统重建服务并传递最后一个 Intent |
| START_STICKY_COMPATIBILITY | START_STICKY 的兼容版本，不保证重建 |

## 3. 关键源码解析

### 3.1 Service 启动流程

```java
// frameworks/base/core/java/android/app/ContextImpl.java
@Override
public ComponentName startService(Intent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, false, mUser);
}

private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    try {
        validateServiceIntent(service);
        service.prepareToLeaveProcess(this);
        
        // 通过 Binder 调用 AMS
        ComponentName cn = ActivityManager.getService().startService(
            mMainThread.getApplicationThread(),  // IApplicationThread
            service,                              // Intent
            service.resolveTypeIfNeeded(getContentResolver()),
            requireForeground,                    // 是否需要前台服务
            getOpPackageName(),                   // 包名
            user.getIdentifier());                // 用户 ID
            
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException("Not allowed to start service " + service);
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException("Unable to start service " + service);
            } else if (cn.getPackageName().equals("?")) {
                throw new IllegalStateException("Not allowed to start service " + service);
            }
        }
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

### 3.2 ActivityManagerService 处理 startService

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
@Override
public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, boolean requireForeground, String callingPackage,
        int userId) throws TransactionTooLargeException {
    
    // 权限检查
    enforceNotIsolatedCaller("startService");
    
    synchronized(this) {
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        
        // 委托给 ActiveServices 处理
        ComponentName res = mServices.startServiceLocked(caller, service,
                resolvedType, callingPid, callingUid,
                requireForeground, callingPackage, userId);
        
        return res;
    }
}
```

### 3.3 ActiveServices.startServiceLocked

```java
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
ComponentName startServiceLocked(IApplicationThread caller, Intent service,
        String resolvedType, int callingPid, int callingUid,
        boolean fgRequired, String callingPackage, int userId) {
    
    // 1. 查找或创建 ServiceRecord
    ServiceLookupResult res = retrieveServiceLocked(service, resolvedType,
            callingPackage, callingPid, callingUid, userId, true, callerFg, false);
    
    ServiceRecord r = res.record;
    
    // 2. 检查后台启动限制（Android 8.0+）
    if (!r.startRequested && !fgRequired) {
        // 检查是否允许后台启动
        final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                r.appInfo.targetSdkVersion, callingPid, false, false);
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
            // 不允许后台启动
            return null;
        }
    }
    
    // 3. 启动服务
    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    
    return cmp;
}

ComponentName startServiceInnerLocked(ServiceMap smap, Intent service,
        ServiceRecord r, boolean callerFg, boolean addToStarting) {
    
    // 记录启动请求
    r.startRequested = true;
    r.callStart = false;
    r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
            service, neededGrants, callingUid));
    
    // 真正启动服务
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
    
    return r.name;
}
```

### 3.4 Service 创建过程

```java
// frameworks/base/core/java/android/app/ActivityThread.java
private void handleCreateService(CreateServiceData data) {
    // 1. 获取 LoadedApk
    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    
    Service service = null;
    try {
        // 2. 通过反射创建 Service 实例
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
    } catch (Exception e) {
        // ...
    }
    
    try {
        // 3. 创建 Context
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);
        
        // 4. 获取或创建 Application
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        
        // 5. 调用 Service.attach()
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        
        // 6. 调用 onCreate()
        service.onCreate();
        
        // 7. 保存到 mServices 映射表
        mServices.put(data.token, service);
        
        // 8. 通知 AMS 服务创建完成
        ActivityManager.getService().serviceDoneExecuting(
                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
    } catch (Exception e) {
        // ...
    }
}
```

## 4. 实战应用

### 4.1 前台服务（Foreground Service）

Android 8.0+ 对后台服务有严格限制，长时间运行的服务必须使用前台服务。

```kotlin
class MusicService : Service() {
    
    companion object {
        const val CHANNEL_ID = "music_channel"
        const val NOTIFICATION_ID = 1
    }
    
    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = createNotification()
        
        // Android 14+ 需要指定前台服务类型
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
            startForeground(NOTIFICATION_ID, notification, 
                ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK)
        } else {
            startForeground(NOTIFICATION_ID, notification)
        }
        
        // 执行后台任务
        playMusic()
        
        return START_STICKY
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "Music Playback",
                NotificationManager.IMPORTANCE_LOW
            )
            val manager = getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(channel)
        }
    }
    
    private fun createNotification(): Notification {
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("正在播放音乐")
            .setContentText("歌曲名称")
            .setSmallIcon(R.drawable.ic_music)
            .build()
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
}
```

**Android 14+ 前台服务类型声明：**

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<service
    android:name=".MusicService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false" />
```

**前台服务类型（Android 14+）：**
| 类型 | 说明 | 所需权限 |
|-----|------|---------|
| camera | 相机访问 | FOREGROUND_SERVICE_CAMERA |
| connectedDevice | 连接设备 | FOREGROUND_SERVICE_CONNECTED_DEVICE |
| dataSync | 数据同步 | FOREGROUND_SERVICE_DATA_SYNC |
| health | 健康数据 | FOREGROUND_SERVICE_HEALTH |
| location | 位置访问 | FOREGROUND_SERVICE_LOCATION |
| mediaPlayback | 媒体播放 | FOREGROUND_SERVICE_MEDIA_PLAYBACK |
| mediaProjection | 屏幕录制 | FOREGROUND_SERVICE_MEDIA_PROJECTION |
| microphone | 麦克风 | FOREGROUND_SERVICE_MICROPHONE |
| phoneCall | 电话 | FOREGROUND_SERVICE_PHONE_CALL |
| remoteMessaging | 远程消息 | FOREGROUND_SERVICE_REMOTE_MESSAGING |
| shortService | 短时服务 | 无需额外权限 |
| specialUse | 特殊用途 | FOREGROUND_SERVICE_SPECIAL_USE |
| systemExempted | 系统豁免 | 仅系统应用 |

### 4.2 绑定服务（Bound Service）

```kotlin
// 服务端
class CalculatorService : Service() {
    
    private val binder = CalculatorBinder()
    
    inner class CalculatorBinder : Binder() {
        fun getService(): CalculatorService = this@CalculatorService
    }
    
    override fun onBind(intent: Intent?): IBinder = binder
    
    fun add(a: Int, b: Int): Int = a + b
    
    fun subtract(a: Int, b: Int): Int = a - b
}

// 客户端
class MainActivity : AppCompatActivity() {
    
    private var calculatorService: CalculatorService? = null
    private var isBound = false
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            val binder = service as CalculatorService.CalculatorBinder
            calculatorService = binder.getService()
            isBound = true
        }
        
        override fun onServiceDisconnected(name: ComponentName?) {
            calculatorService = null
            isBound = false
        }
    }
    
    override fun onStart() {
        super.onStart()
        Intent(this, CalculatorService::class.java).also { intent ->
            bindService(intent, connection, Context.BIND_AUTO_CREATE)
        }
    }
    
    override fun onStop() {
        super.onStop()
        if (isBound) {
            unbindService(connection)
            isBound = false
        }
    }
    
    fun calculate() {
        calculatorService?.let {
            val result = it.add(10, 20)
            Log.d("TAG", "Result: $result")
        }
    }
}
```

### 4.3 JobScheduler

```kotlin
// JobService 实现
class MyJobService : JobService() {
    
    override fun onStartJob(params: JobParameters?): Boolean {
        // 在后台线程执行任务
        Thread {
            doWork()
            // 任务完成，通知系统
            jobFinished(params, false) // false 表示不需要重新调度
        }.start()
        
        return true // true 表示任务在后台线程执行
    }
    
    override fun onStopJob(params: JobParameters?): Boolean {
        // 系统要求停止任务（如条件不再满足）
        return true // true 表示需要重新调度
    }
    
    private fun doWork() {
        // 执行实际工作
    }
}

// 调度任务
class MainActivity : AppCompatActivity() {
    
    fun scheduleJob() {
        val componentName = ComponentName(this, MyJobService::class.java)
        
        val jobInfo = JobInfo.Builder(JOB_ID, componentName)
            .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED) // WiFi
            .setRequiresCharging(true)                               // 充电中
            .setRequiresDeviceIdle(false)                           // 不要求空闲
            .setPersisted(true)                                      // 重启后保留
            .setPeriodic(15 * 60 * 1000)                            // 周期执行（最小15分钟）
            .build()
        
        val scheduler = getSystemService(Context.JOB_SCHEDULER_SERVICE) as JobScheduler
        val result = scheduler.schedule(jobInfo)
        
        if (result == JobScheduler.RESULT_SUCCESS) {
            Log.d("TAG", "Job scheduled successfully")
        }
    }
    
    companion object {
        const val JOB_ID = 1001
    }
}
```

### 4.4 WorkManager（推荐）

WorkManager 是 Android Jetpack 组件，用于可延迟的后台任务，保证任务执行。

```kotlin
// 定义 Worker
class UploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val imageUri = inputData.getString("image_uri") ?: return Result.failure()
            
            // 显示进度
            setProgress(workDataOf("progress" to 0))
            
            // 执行上传
            uploadImage(imageUri)
            
            setProgress(workDataOf("progress" to 100))
            
            // 返回结果
            Result.success(workDataOf("result" to "upload_success"))
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
    
    private suspend fun uploadImage(uri: String) {
        // 上传逻辑
    }
}

// 使用 WorkManager
class MainActivity : AppCompatActivity() {
    
    fun scheduleUpload(imageUri: String) {
        // 1. 创建约束条件
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
        
        // 2. 创建输入数据
        val inputData = workDataOf("image_uri" to imageUri)
        
        // 3. 创建工作请求
        val uploadRequest = OneTimeWorkRequestBuilder<UploadWorker>()
            .setConstraints(constraints)
            .setInputData(inputData)
            .setBackoffCriteria(
                BackoffPolicy.EXPONENTIAL,
                OneTimeWorkRequest.MIN_BACKOFF_MILLIS,
                TimeUnit.MILLISECONDS
            )
            .addTag("upload")
            .build()
        
        // 4. 提交任务
        WorkManager.getInstance(this)
            .enqueueUniqueWork(
                "upload_$imageUri",
                ExistingWorkPolicy.REPLACE,
                uploadRequest
            )
        
        // 5. 观察任务状态
        WorkManager.getInstance(this)
            .getWorkInfoByIdLiveData(uploadRequest.id)
            .observe(this) { workInfo ->
                when (workInfo?.state) {
                    WorkInfo.State.SUCCEEDED -> {
                        val result = workInfo.outputData.getString("result")
                        Log.d("TAG", "Upload succeeded: $result")
                    }
                    WorkInfo.State.FAILED -> {
                        Log.d("TAG", "Upload failed")
                    }
                    WorkInfo.State.RUNNING -> {
                        val progress = workInfo.progress.getInt("progress", 0)
                        Log.d("TAG", "Progress: $progress%")
                    }
                    else -> {}
                }
            }
    }
    
    // 链式任务
    fun chainedWork() {
        val compress = OneTimeWorkRequestBuilder<CompressWorker>().build()
        val upload = OneTimeWorkRequestBuilder<UploadWorker>().build()
        val cleanup = OneTimeWorkRequestBuilder<CleanupWorker>().build()
        
        WorkManager.getInstance(this)
            .beginWith(compress)
            .then(upload)
            .then(cleanup)
            .enqueue()
    }
    
    // 并行任务
    fun parallelWork() {
        val upload1 = OneTimeWorkRequestBuilder<UploadWorker>().build()
        val upload2 = OneTimeWorkRequestBuilder<UploadWorker>().build()
        val merge = OneTimeWorkRequestBuilder<MergeWorker>().build()
        
        WorkManager.getInstance(this)
            .beginWith(listOf(upload1, upload2))
            .then(merge)
            .enqueue()
    }
}
```

### 4.5 保活方案（了解即可）

> **注意：** 保活方案在高版本 Android 上效果有限，且可能影响用户体验和电池寿命。推荐使用 WorkManager 等官方方案。

```kotlin
// 1. 前台服务保活
class KeepAliveService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(1, createNotification())
        return START_STICKY
    }
}

// 2. 双进程守护（效果有限）
// 主进程和守护进程相互监听，一方被杀后重启另一方

// 3. JobScheduler 定时唤醒
// 定期检查服务状态并重启

// 4. 账号同步（SyncAdapter）
// 利用系统账号同步机制保活

// 5. 推送保活
// 利用厂商推送通道保持连接
```

## 5. 常见面试题

### 问题1：Service 的生命周期是怎样的？startService 和 bindService 有什么区别？

**答案要点：**
- **startService 生命周期**：onCreate() → onStartCommand() → onDestroy()
- **bindService 生命周期**：onCreate() → onBind() → onUnbind() → onDestroy()
- **区别**：
  - startService：服务独立运行，需要主动调用 stopService/stopSelf 停止
  - bindService：服务依赖绑定者，所有客户端解绑后自动销毁
  - startService 通过 Intent 传递数据，bindService 通过 IBinder 接口通信
  - 可以同时使用两种方式，此时需要同时 stopService 和 unbindService 才能销毁

### 问题2：onStartCommand 的返回值有什么作用？

**答案要点：**
- **START_NOT_STICKY**：服务被杀后不重建，除非有待处理的 Intent
- **START_STICKY**：服务被杀后重建，但 Intent 为 null
- **START_REDELIVER_INTENT**：服务被杀后重建，并重新传递最后一个 Intent
- 选择依据：
  - 音乐播放等需要持续运行：START_STICKY
  - 下载等需要恢复任务：START_REDELIVER_INTENT
  - 一次性任务：START_NOT_STICKY

### 问题3：Android 8.0 对后台服务有什么限制？如何解决？

**答案要点：**
- **限制**：应用在后台时，不能使用 startService() 启动后台服务
- **解决方案**：
  1. 使用前台服务：startForegroundService() + startForeground()
  2. 使用 JobScheduler 或 WorkManager
  3. 使用 BroadcastReceiver 的隐式广播豁免
- **代码示例**：
  ```kotlin
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
      context.startForegroundService(intent)
  } else {
      context.startService(intent)
  }
  ```

### 问题4：Android 14 对前台服务有什么新要求？

**答案要点：**
- 必须在 Manifest 中声明 foregroundServiceType
- 必须声明对应的权限（如 FOREGROUND_SERVICE_MEDIA_PLAYBACK）
- 调用 startForeground() 时必须指定类型
- 新增 shortService 类型，限时 3 分钟
- 部分类型有额外限制（如 dataSync 24小时内限制6小时）

### 问题5：Service 和 IntentService 的区别？IntentService 为什么被废弃？

**答案要点：**
- **Service**：运行在主线程，需要手动创建子线程
- **IntentService**：
  - 自动创建工作线程处理 Intent
  - 任务完成后自动停止
  - 串行处理多个请求
- **废弃原因**：
  - Android 8.0+ 后台限制使其难以正常工作
  - 推荐使用 WorkManager 替代
  - JobIntentService 也已废弃

### 问题6：WorkManager、JobScheduler、AlarmManager 的区别和使用场景？

**答案要点：**
| 组件 | 特点 | 使用场景 |
|-----|------|---------|
| WorkManager | 保证执行、支持约束、链式任务 | 可延迟的后台任务（推荐） |
| JobScheduler | 系统级调度、批量执行 | 需要特定条件的任务 |
| AlarmManager | 精确时间触发 | 闹钟、定时提醒 |

- WorkManager 内部会根据 API 级别选择 JobScheduler 或 AlarmManager
- 优先使用 WorkManager，它是官方推荐的统一方案

### 问题7：如何实现跨进程的 Service 通信？

**答案要点：**
1. **AIDL**：定义接口，生成 Proxy/Stub，适合复杂交互
2. **Messenger**：基于 Handler，适合简单消息传递
3. **ContentProvider**：适合数据共享
4. **BroadcastReceiver**：适合一对多通知
- 详细实现见 Intent与IPC机制.md

### 问题8：Service 运行在哪个线程？如何处理耗时操作？

**答案要点：**
- Service 默认运行在主线程（UI 线程）
- 耗时操作必须在子线程执行，否则会 ANR
- 处理方式：
  1. 手动创建线程或线程池
  2. 使用 CoroutineWorker（WorkManager）
  3. 使用 RxJava
  ```kotlin
  override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
      CoroutineScope(Dispatchers.IO).launch {
          doHeavyWork()
          stopSelf(startId)
      }
      return START_NOT_STICKY
  }
  ```
