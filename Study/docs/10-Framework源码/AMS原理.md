# AMS 原理

## 1. 概述

ActivityManagerService（AMS）是 Android 系统最核心的服务之一，负责四大组件的启动、切换、调度，以及进程管理和内存管理。

### AMS 核心职责

| 职责 | 说明 |
|------|------|
| Activity 管理 | 启动、切换、生命周期管理 |
| Task 管理 | 任务栈管理、Back Stack |
| 进程管理 | 进程创建、优先级、回收 |
| 内存管理 | LMK 配合、内存回收 |
| 广播管理 | 广播分发、有序广播 |
| Service 管理 | Service 启动、绑定 |

## 2. 核心原理

### 2.1 AMS 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      AMS 架构图                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                 ActivityManagerService               │   │
│   └───────────────────────┬─────────────────────────────┘   │
│                           │                                 │
│           ┌───────────────┼───────────────┐                │
│           │               │               │                │
│           ▼               ▼               ▼                │
│   ┌───────────────┐ ┌───────────┐ ┌───────────────┐       │
│   │ActivityTask   │ │ProcessList│ │BroadcastQueue │       │
│   │ManagerService│ │           │ │               │       │
│   └───────┬───────┘ └─────┬─────┘ └───────────────┘       │
│           │               │                                │
│           ▼               ▼                                │
│   ┌───────────────┐ ┌───────────────┐                     │
│   │ RootWindow    │ │ ProcessRecord │                     │
│   │ Container     │ │               │                     │
│   └───────┬───────┘ └───────────────┘                     │
│           │                                                │
│           ▼                                                │
│   ┌───────────────┐                                       │
│   │    Task       │                                       │
│   │ (Back Stack)  │                                       │
│   └───────┬───────┘                                       │
│           │                                                │
│           ▼                                                │
│   ┌───────────────┐                                       │
│   │ActivityRecord │                                       │
│   └───────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Activity 启动流程

```java
// Activity.startActivity() 调用链
Activity.startActivity()
    → Activity.startActivityForResult()
    → Instrumentation.execStartActivity()
    → ActivityTaskManager.getService().startActivity()  // Binder 调用
    → ActivityTaskManagerService.startActivity()
    → ActivityStarter.execute()
    → ActivityStarter.executeRequest()
    → ActivityStarter.startActivityUnchecked()
    → RootWindowContainer.resumeFocusedTasksTopActivities()
    → Task.resumeTopActivityUncheckedLocked()
    → TaskFragment.resumeTopActivity()
    → ActivityTaskSupervisor.startSpecificActivity()
```

### 2.3 Activity 启动源码分析

```java
// ActivityTaskManagerService.java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent,
            resolvedType, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, bOptions, UserHandle.getCallingUserId());
}

// ActivityStarter.java
int executeRequest(Request request) {
    // 1. 权限检查
    boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo,
            resultWho, requestCode, callingPid, callingUid, callingPackage,
            callingFeatureId, request.ignoreTargetSecurity, inTask != null,
            callerApp, resultRecord, resultRootTask);
    
    // 2. 创建 ActivityRecord
    final ActivityRecord r = new ActivityRecord.Builder(mService)
            .setCaller(callerApp)
            .setLaunchedFromPid(callingPid)
            .setLaunchedFromUid(callingUid)
            .setLaunchedFromPackage(callingPackage)
            .setLaunchedFromFeature(callingFeatureId)
            .setIntent(intent)
            .setResolvedType(resolvedType)
            .setActivityInfo(aInfo)
            .setConfiguration(mService.getGlobalConfiguration())
            .setResultTo(resultRecord)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setComponentSpecified(request.componentSpecified)
            .setRootVoiceInteraction(voiceSession != null)
            .setActivityOptions(checkedOptions)
            .setSourceRecord(sourceRecord)
            .build();
    
    // 3. 启动 Activity
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, true /* doResume */, checkedOptions,
            inTask, inTaskFragment, restrictedBgActivity, intentGrants);
    
    return mLastStartActivityResult;
}

// 启动 Activity（不检查）
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment, boolean restrictedBgActivity,
        NeededUriGrants intentGrants) {
    
    // 计算启动模式
    computeLaunchingTaskFlags();
    
    // 计算源 Task
    computeSourceRootTask();
    
    // 设置启动标志
    mIntent.setFlags(mLaunchFlags);
    
    // 获取或创建 Task
    final Task reusedTask = getReusableTask();
    
    // 将 Activity 添加到 Task
    if (mTargetRootTask == null) {
        mTargetRootTask = getOrCreateRootTask(mStartActivity, mLaunchFlags,
                targetTask, mOptions);
    }
    
    // Resume Activity
    if (doResume) {
        mRootWindowContainer.resumeFocusedTasksTopActivities(
                mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
    }
    
    return START_SUCCESS;
}
```

### 2.4 Task 与 Back Stack

```java
// Task.java
class Task extends TaskFragment {
    // Task 中的 Activity 列表
    final ArrayList<ActivityRecord> mActivities = new ArrayList<>();
    
    // Task ID
    final int mTaskId;
    
    // 启动模式相关
    int mLaunchMode;
    String mAffinity;
    
    // 添加 Activity 到 Task
    void addActivityToTop(ActivityRecord r) {
        addActivityAtIndex(getChildCount(), r);
    }
    
    // 移除 Activity
    void removeActivity(ActivityRecord r) {
        removeChild(r);
    }
    
    // 获取栈顶 Activity
    ActivityRecord topRunningActivity() {
        return topRunningActivity(true /* focusableOnly */);
    }
}
```

### 2.5 进程管理

```java
// ProcessList.java
final class ProcessList {
    // 进程优先级
    static final int FOREGROUND_APP_ADJ = 0;      // 前台进程
    static final int VISIBLE_APP_ADJ = 100;       // 可见进程
    static final int PERCEPTIBLE_APP_ADJ = 200;   // 可感知进程
    static final int BACKUP_APP_ADJ = 300;        // 备份进程
    static final int HEAVY_WEIGHT_APP_ADJ = 400;  // 重量级进程
    static final int SERVICE_ADJ = 500;           // 服务进程
    static final int HOME_APP_ADJ = 600;          // Home 进程
    static final int PREVIOUS_APP_ADJ = 700;      // 上一个进程
    static final int SERVICE_B_ADJ = 800;         // B 类服务
    static final int CACHED_APP_MIN_ADJ = 900;    // 缓存进程最小值
    static final int CACHED_APP_MAX_ADJ = 999;    // 缓存进程最大值
    
    // 启动进程
    ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, HostingRecord hostingRecord,
            int zygotePolicyFlags, boolean allowWhileBooting,
            boolean isolated) {
        
        // 检查是否已存在
        ProcessRecord app = getProcessRecordLocked(processName, info.uid);
        
        if (app == null) {
            // 创建 ProcessRecord
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid,
                    isSdkSandbox, sdkSandboxUid, sdkSandboxClientAppPackage,
                    hostingRecord);
        }
        
        // 启动进程
        final boolean success = startProcessLocked(app, hostingRecord,
                zygotePolicyFlags, abiOverride);
        
        return success ? app : null;
    }
}
```

### 2.6 Activity 生命周期管理

```java
// ActivityRecord.java
final class ActivityRecord extends WindowToken {
    // 生命周期状态
    enum State {
        INITIALIZING,
        STARTED,
        RESUMED,
        PAUSING,
        PAUSED,
        STOPPING,
        STOPPED,
        FINISHING,
        DESTROYING,
        DESTROYED,
        RESTARTING_PROCESS
    }
    
    State mState = State.INITIALIZING;
    
    // 设置状态
    void setState(State state, String reason) {
        if (mState == state) {
            return;
        }
        
        mState = state;
        
        // 通知状态变化
        if (mTaskSupervisor != null) {
            mTaskSupervisor.onActivityStateChanged(this, state, reason);
        }
    }
}

// ClientLifecycleManager.java - 管理生命周期事务
class ClientLifecycleManager {
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
    }
}

// ClientTransaction.java
public class ClientTransaction implements Parcelable {
    // 生命周期回调列表
    private List<ClientTransactionItem> mActivityCallbacks;
    
    // 最终生命周期状态
    private ActivityLifecycleItem mLifecycleStateRequest;
    
    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }
}
```

## 3. 关键源码解析

### 3.1 启动模式处理

```java
// ActivityStarter.java
private void computeLaunchingTaskFlags() {
    // standard 模式
    if (mLaunchMode == LAUNCH_SINGLE_INSTANCE) {
        mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
    } else if (mLaunchMode == LAUNCH_SINGLE_TASK) {
        mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
    }
    
    // 处理 FLAG_ACTIVITY_NEW_TASK
    if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        if ((mLaunchFlags & FLAG_ACTIVITY_MULTIPLE_TASK) == 0) {
            // 查找已存在的 Task
            mInTask = mRootWindowContainer.findTask(mStartActivity, mPreferredTaskDisplayArea);
        }
    }
}

// 查找可复用的 Task
private Task getReusableTask() {
    // singleTask 或 singleInstance
    if (mLaunchMode == LAUNCH_SINGLE_TASK || mLaunchMode == LAUNCH_SINGLE_INSTANCE) {
        // 查找具有相同 affinity 的 Task
        Task task = mRootWindowContainer.findTask(mStartActivity, mPreferredTaskDisplayArea);
        if (task != null) {
            // 找到了，复用这个 Task
            return task;
        }
    }
    
    // singleTop
    if ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
            || mLaunchMode == LAUNCH_SINGLE_TOP) {
        // 检查栈顶是否是同一个 Activity
        final ActivityRecord top = mInTask != null
                ? mInTask.getTopNonFinishingActivity()
                : mTargetRootTask.getTopNonFinishingActivity();
        if (top != null && top.mActivityComponent.equals(mStartActivity.mActivityComponent)) {
            // 栈顶是同一个 Activity，调用 onNewIntent
            return mInTask;
        }
    }
    
    return null;
}
```

### 3.2 onNewIntent 调用

```java
// ActivityRecord.java
void deliverNewIntentLocked(int callingUid, Intent intent, String referrer) {
    // 创建 NewIntentItem
    final NewIntentItem item = NewIntentItem.obtain(
            Collections.singletonList(new ReferrerIntent(intent, referrer)),
            mState == RESUMED);
    
    // 添加到事务
    final ClientTransaction transaction = ClientTransaction.obtain(
            app.getThread(), token);
    transaction.addCallback(item);
    
    // 执行事务
    mAtmService.getLifecycleManager().scheduleTransaction(transaction);
}

// ActivityThread.java
public void handleNewIntent(IBinder token, List<ReferrerIntent> intents) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        for (ReferrerIntent intent : intents) {
            // 调用 Activity.onNewIntent()
            r.activity.performNewIntent(intent);
        }
    }
}
```

## 4. 实战应用

### 4.1 启动模式选择

```kotlin
// 1. standard - 默认模式，每次都创建新实例
// 适用：普通页面

// 2. singleTop - 栈顶复用
// 适用：通知跳转、搜索页面
class SearchActivity : AppCompatActivity() {
    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        // 处理新的搜索请求
        handleSearch(intent?.getStringExtra("query"))
    }
}

// 3. singleTask - 栈内复用
// 适用：主页面、登录页面
// AndroidManifest.xml
// <activity android:name=".MainActivity" android:launchMode="singleTask"/>

// 4. singleInstance - 独占 Task
// 适用：来电界面、闹钟界面
```

### 4.2 Task 管理

```kotlin
// 清除 Task 中的其他 Activity
val intent = Intent(this, MainActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_SINGLE_TOP
}
startActivity(intent)

// 创建新 Task
val intent = Intent(this, NewTaskActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK
}
startActivity(intent)

// 清除整个 Task
finishAffinity()
```

## 5. 常见面试题

### 问题1：Activity 的启动流程是怎样的？

**答案要点**：
1. Activity.startActivity() 调用 Instrumentation.execStartActivity()
2. 通过 Binder 调用 ATMS.startActivity()
3. ActivityStarter 处理启动请求
4. 计算启动模式、查找或创建 Task
5. 创建 ActivityRecord
6. 通过 ClientTransaction 调度生命周期
7. ActivityThread 执行生命周期回调

### 问题2：四种启动模式的区别是什么？

**答案要点**：
- **standard**：每次都创建新实例
- **singleTop**：栈顶复用，调用 onNewIntent
- **singleTask**：栈内复用，清除其上的 Activity
- **singleInstance**：独占 Task，全局唯一

### 问题3：AMS 如何管理进程优先级？

**答案要点**：
- 前台进程（ADJ=0）：正在交互的 Activity
- 可见进程（ADJ=100）：可见但不在前台
- 服务进程（ADJ=500）：运行 Service
- 缓存进程（ADJ=900-999）：后台进程
- LMK 根据 ADJ 值决定回收顺序

### 问题4：onNewIntent 什么时候调用？

**答案要点**：
- singleTop 模式，Activity 在栈顶时
- singleTask 模式，Activity 已存在时
- singleInstance 模式，Activity 已存在时
- 使用 FLAG_ACTIVITY_SINGLE_TOP 标志时

### 问题5：Task 和 Back Stack 的关系是什么？

**答案要点**：
- Task 是一组 Activity 的集合
- Back Stack 是 Task 中 Activity 的栈结构
- 按下返回键，弹出栈顶 Activity
- Task 可以移到前台或后台
