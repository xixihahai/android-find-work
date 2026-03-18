# AMS 原理

---

## 速记总结

> **一句话理解 AMS**：AMS 就像一个**交通指挥中心**，管理着所有 Activity 的创建、调度、生命周期和任务栈。App 想启动 Activity？必须先"报备"给 AMS，由它决定能不能启动、放到哪个任务栈、什么时候回调生命周期。

### 核心职责速记表

| 职责 | 一句话记忆 | 关键类 |
|------|-----------|--------|
| Activity 管理 | AMS 是 Activity 的"户籍管理局"，每个 Activity 都在它这里有一份 ActivityRecord | ActivityRecord, ActivityStarter |
| Task 管理 | Task 就是一摞扑克牌，AMS 决定新牌放哪一摞、旧牌怎么翻 | Task, RootWindowContainer |
| 进程管理 | AMS 是进程的"生杀大权"持有者，决定谁活谁死 | ProcessList, ProcessRecord |
| 内存管理 | AMS 给每个进程打分（ADJ），分低的先被 LMK 回收 | OomAdjuster, LowMemoryKiller |
| 广播管理 | AMS 是广播电台的总调度，决定谁先收到、谁后收到 | BroadcastQueue |
| Service 管理 | AMS 管理 Service 的启动/绑定/解绑全流程 | ActiveServices |
| ContentProvider 管理 | AMS 负责 Provider 的发布和权限校验 | ContentProviderHelper |

### AMS 核心架构全景图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          System Server 进程                                 │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    ActivityManagerService (AMS)                        │  │
│  │                                                                       │  │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │  │
│  │   │  OomAdjuster │  │ ProcessList  │  │ActiveServices│              │  │
│  │   │  (进程优先级) │  │  (进程管理)   │  │ (Service管理)│              │  │
│  │   └──────────────┘  └──────────────┘  └──────────────┘              │  │
│  │                                                                       │  │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │  │
│  │   │BroadcastQueue│  │ContentProvider│  │   UriGrants  │              │  │
│  │   │  (广播管理)   │  │   Helper     │  │  (权限管理)   │              │  │
│  │   └──────────────┘  └──────────────┘  └──────────────┘              │  │
│  └───────────────────────────┬───────────────────────────────────────────┘  │
│                              │                                              │
│  ┌───────────────────────────┴───────────────────────────────────────────┐  │
│  │              ActivityTaskManagerService (ATMS)                         │  │
│  │                                                                       │  │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │  │
│  │   │ActivityStarter│  │  Task        │  │ActivityRecord│              │  │
│  │   │ (启动调度器)  │  │ (任务栈)     │  │ (Activity档案)│              │  │
│  │   └──────────────┘  └──────┬───────┘  └──────────────┘              │  │
│  │                            │                                         │  │
│  │   ┌──────────────┐  ┌─────┴────────┐  ┌──────────────┐              │  │
│  │   │ActivityTask  │  │RootWindow    │  │ClientLifecycle│              │  │
│  │   │ Supervisor   │  │ Container    │  │  Manager      │              │  │
│  │   │ (Activity监管)│  │ (窗口根容器) │  │ (生命周期事务)│              │  │
│  │   └──────────────┘  └──────────────┘  └──────────────┘              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              │ Binder IPC                                   │
└──────────────────────────────┼──────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│   App 进程 A      │ │   App 进程 B      │ │  Zygote 进程     │
│                  │ │                  │ │                  │
│ ActivityThread   │ │ ActivityThread   │ │ fork 新进程       │
│   ├─ Activity1   │ │   ├─ Activity1   │ │ 预加载 Framework  │
│   ├─ Activity2   │ │   └─ Service1    │ │ 共享内存资源       │
│   └─ Service1    │ │                  │ │                  │
│                  │ │                  │ │                  │
│ ApplicationThread│ │ ApplicationThread│ │ ZygoteServer     │
│ (Binder Stub)   │ │ (Binder Stub)   │ │ (Socket 通信)    │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

**核心交互关系**：
- **App → AMS**：App 通过 Binder 代理（`ActivityTaskManager.getService()`）向 AMS 发起请求
- **AMS → App**：AMS 通过 `IApplicationThread`（App 端的 Binder Stub）回调 App
- **AMS → Zygote**：AMS 需要新进程时，通过 Socket 通知 Zygote fork 子进程
- **ATMS 与 AMS 的关系**：Android 10+ 将 Activity 相关逻辑从 AMS 拆分到 ATMS，AMS 仍负责进程、广播、Service 管理

---

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

### AMS 在系统启动中的位置

```
SystemServer.main()
    → SystemServer.run()
    → startBootstrapServices()
        → ActivityTaskManagerService.Lifecycle.startService()  // 先启动 ATMS
        → ActivityManagerService.Lifecycle.startService()      // 再启动 AMS
        → AMS.setSystemProcess()                                // 注册到 ServiceManager
    → startCoreServices()
    → startOtherServices()
        → AMS.systemReady()                                    // 系统就绪，启动 Launcher
```

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

#### 完整跨进程时序图

```
  App 进程 (调用方)              System Server 进程                    App 进程 (目标方)
  ─────────────────          ──────────────────────              ──────────────────
        │                            │                                  │
  [1] startActivity()               │                                  │
        │                            │                                  │
  [2] Instrumentation               │                                  │
      .execStartActivity()          │                                  │
        │                            │                                  │
        │──── Binder IPC ──────────>│                                  │
        │                     [3] ATMS.startActivity()                 │
        │                            │                                  │
        │                     [4] ActivityStarter                       │
        │                         .execute()                           │
        │                            │                                  │
        │                     [5] 权限检查 + Intent 解析               │
        │                            │                                  │
        │                     [6] 创建 ActivityRecord                  │
        │                            │                                  │
        │                     [7] 计算启动模式                         │
        │                         computeLaunchingTaskFlags()          │
        │                            │                                  │
        │                     [8] 查找/创建 Task                       │
        │                         getReusableTask()                    │
        │                            │                                  │
        │                     [9] startActivityUnchecked()             │
        │                            │                                  │
        │                     [10] 暂停当前 Activity                   │
        │<── schedulePauseActivity ──│                                  │
        │                            │                                  │
  [11] handlePauseActivity()         │                                  │
        │                            │                                  │
        │── activityPaused() ──────>│                                  │
        │                            │                                  │
        │                     [12] 目标进程是否存在？                   │
        │                            │                                  │
        │                     [12a] 不存在：                            │
        │                         AMS → Zygote (Socket)                │
        │                         → fork 新进程                        │
        │                         → ActivityThread.main()        ─────>│
        │                         → AMS.attachApplication()            │
        │                            │                                  │
        │                     [12b] 已存在：                            │
        │                         直接调度                              │
        │                            │                                  │
        │                     [13] ClientTransaction                   │
        │                         + LaunchActivityItem                 │
        │                         + ResumeActivityItem                 │
        │                            │                                  │
        │                            │── scheduleTransaction ────────>│
        │                            │                                  │
        │                            │                           [14] handleLaunchActivity()
        │                            │                                  │
        │                            │                           [15] Activity.onCreate()
        │                            │                                  │
        │                            │                           [16] Activity.onStart()
        │                            │                                  │
        │                            │                           [17] Activity.onResume()
        │                            │                                  │
        │                            │                           [18] Activity 可见可交互
```

**关键步骤解读**：
- **步骤 2→3**：跨进程 Binder 调用，从 App 进程进入 System Server
- **步骤 5**：检查调用方是否有权限启动目标 Activity（exported、permission 等）
- **步骤 7-8**：根据 launchMode 和 Intent Flags 决定是复用已有 Task 还是创建新 Task
- **步骤 10-11**：**先暂停当前 Activity**，等暂停完成后才启动新 Activity（这是面试重点！）
- **步骤 12**：如果目标进程不存在，需要先通过 Zygote fork 新进程，这也是冷启动慢的原因
- **步骤 13**：通过 `ClientTransaction` 机制将生命周期事件打包发送给目标进程

#### 调用链代码

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

#### 可视化图解

```
                    RootWindowContainer
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Task #1      Task #2      Task #3
        (微信主Task)  (浏览器Task)  (设置Task)
              │            │            │
              │            │            │
    ┌─────────┴──────┐    │     ┌──────┴──────┐
    │   Back Stack   │    │     │  Back Stack  │
    │                │    │     │              │
    │ ┌────────────┐ │    │     │┌────────────┐│
    │ │朋友圈页面   │ │◄─栈顶  ││WiFi详情    ││◄─栈顶
    │ │ActivityRecord│ │    │     ││ActivityRecord││
    │ ├────────────┤ │    │     │├────────────┤│
    │ │聊天详情页   │ │    │     ││网络设置    ││
    │ │ActivityRecord│ │    │     ││ActivityRecord││
    │ ├────────────┤ │    │     │├────────────┤│
    │ │消息列表页   │ │    │     ││设置主页    ││
    │ │ActivityRecord│ │    │     ││ActivityRecord││
    │ └────────────┘ │    │     │└────────────┘│
    │   栈底 ──────>│    │     │  栈底 ──────>│
    └────────────────┘    │     └──────────────┘
                          │
                   ┌──────┴──────┐
                   │  Back Stack  │
                   │              │
                   │┌────────────┐│
                   ││百度搜索页  ││◄─栈顶
                   │├────────────┤│
                   ││浏览器主页  ││
                   │└────────────┘│
                   │  栈底 ──────>│
                   └──────────────┘

按返回键的行为：
  1. 弹出当前 Task 栈顶 Activity（onDestroy）
  2. 下一个 Activity 变为栈顶（onResume）
  3. 当 Task 中最后一个 Activity 也被弹出 → Task 销毁
  4. 显示上一个 Task 的栈顶 Activity（或回到 Launcher）
```

#### Task 与 Activity 的数据结构关系

```
ActivityRecord ─────────────────────────────────────────────
  │  - mActivityComponent  (ComponentName，如 com.tencent.mm/.ui.LauncherUI)
  │  - mState              (RESUMED / PAUSED / STOPPED / DESTROYED ...)
  │  - app                 (ProcessRecord，所属进程)
  │  - task                (Task，所属任务栈)
  │  - mIntent             (启动该 Activity 的 Intent)
  │  - icicle              (保存的状态 Bundle，用于恢复)
  │  - resultTo            (等待 result 的来源 Activity)
  │
Task (继承 TaskFragment) ───────────────────────────────────
  │  - mTaskId             (唯一 Task ID)
  │  - mAffinity           (亲和性，默认为包名)
  │  - mLaunchMode         (启动模式)
  │  - mActivities         (ArrayList<ActivityRecord>，即 Back Stack)
  │  - mRootActivity       (根 Activity)
  │
RootWindowContainer ────────────────────────────────────────
     - mChildren           (所有 Task 的集合)
     - 管理焦点 Task、Task 之间的切换
```

#### 源码

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

#### 进程优先级详解（ADJ 值越低越重要）

```
  ADJ 值    优先级         说明                              示例                    被杀概率
  ─────    ──────         ────                              ────                    ────────
    0      前台进程        正在与用户交互的 Activity           正在聊天的微信           几乎不会
   100     可见进程        可见但不在前台（被对话框部分遮挡）    微信弹出分享选择器时      极低
   200     可感知进程      用户能感知的（前台 Service、音乐播放）网易云音乐后台播放       低
   300     备份进程        正在执行备份操作                    系统备份中的 App          中低
   400     重量级进程      系统认为重要的后台进程               大型游戏进入后台          中
   500     服务进程        运行 startService() 的进程          后台上传/下载 Service     中
   600     Home 进程      Launcher 进程                      桌面 Launcher            中低（特殊保护）
   700     上一个进程      用户刚离开的 App                    刚按 Home 键的 App       中高
   800     B 类服务       长时间运行的旧 Service               运行超过 30 分钟的 Service 高
  900-999  缓存进程       完全在后台的进程                     昨天打开过的 App          很高
```

#### LMK（Low Memory Killer）与 AMS 的协作

```
                    ┌─────────────────────────────┐
                    │      AMS (OomAdjuster)       │
                    │                             │
                    │  1. 根据组件状态计算每个进程  │
                    │     的 ADJ 值                │
                    │  2. 通过 /proc/pid/oom_adj   │
                    │     写入内核                  │
                    └──────────────┬──────────────┘
                                   │ 写入 ADJ 值
                                   ▼
                    ┌─────────────────────────────┐
                    │    Linux 内核 (LMK/lmkd)     │
                    │                             │
                    │  3. 当内存不足时，根据 ADJ    │
                    │     值从高到低杀死进程        │
                    │  4. ADJ 值相同时，杀死内存    │
                    │     占用最大的进程            │
                    └─────────────────────────────┘
                    
  内存水位线：
  ┌────────────────────────────────────────────────────┐
  │████████████████████████████████████████│  充足      │ → 不回收
  │████████████████████████████│           │  正常      │ → 回收 cached 进程
  │████████████████████│                  │  偏低      │ → 回收 service 进程
  │████████████│                          │  临界      │ → 回收 home/上一个进程
  │████│                                  │  危险      │ → 回收 可感知进程
  │█│                                     │  极危险    │ → 回收可见进程（几乎不会到这步）
  └────────────────────────────────────────────────────┘
```

**ADJ 更新时机**：AMS 在以下时机重新计算进程的 ADJ 值：
- Activity 生命周期变化（onResume / onPause / onStop）
- Service 启动/停止/绑定/解绑
- BroadcastReceiver 开始/结束处理广播
- ContentProvider 被引用/释放

#### 进程管理源码

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

#### ActivityRecord、Task、ActivityStack 的关系图

```
┌──────────────────────────────────────────────────────────────────┐
│                    RootWindowContainer                            │
│                                                                  │
│   ┌──────────────────────┐    ┌──────────────────────┐          │
│   │      Task #1         │    │      Task #2         │          │
│   │  affinity="com.wx"   │    │  affinity="com.br"   │          │
│   │                      │    │                      │          │
│   │  ┌────────────────┐  │    │  ┌────────────────┐  │          │
│   │  │ ActivityRecord │  │    │  │ ActivityRecord │  │          │
│   │  │ state=RESUMED  │◄─┼─ 焦点 │ state=STOPPED  │  │          │
│   │  │ 朋友圈页面     │  │    │  │ 搜索页面       │  │          │
│   │  ├────────────────┤  │    │  ├────────────────┤  │          │
│   │  │ ActivityRecord │  │    │  │ ActivityRecord │  │          │
│   │  │ state=STOPPED  │  │    │  │ state=STOPPED  │  │          │
│   │  │ 聊天页面       │  │    │  │ 浏览器主页     │  │          │
│   │  ├────────────────┤  │    │  └────────────────┘  │          │
│   │  │ ActivityRecord │  │    └──────────────────────┘          │
│   │  │ state=STOPPED  │  │                                      │
│   │  │ 消息列表       │  │                                      │
│   │  └────────────────┘  │                                      │
│   └──────────────────────┘                                      │
└──────────────────────────────────────────────────────────────────┘
```

#### 生命周期事务机制（ClientTransaction）

AMS 不直接调用 Activity 的生命周期方法，而是通过 **ClientTransaction 事务机制** 将生命周期变更打包发送给 App 进程：

```
  System Server                                   App 进程
  ─────────────                                   ────────
       │                                              │
  ClientLifecycleManager                              │
       │                                              │
  scheduleTransaction(                                │
    ClientTransaction {                               │
      client: IApplicationThread                      │
      callbacks: [LaunchActivityItem]                 │
      lifecycleStateRequest: ResumeActivityItem       │
    }                                                 │
  )                                                   │
       │── Binder: scheduleTransaction() ───────────>│
       │                                     ActivityThread.H
       │                                              │
       │                                     EXECUTE_TRANSACTION
       │                                              │
       │                                     TransactionExecutor
       │                                       .execute()
       │                                              │
       │                                     1. executeCallbacks()
       │                                        → LaunchActivityItem
       │                                        → handleLaunchActivity()
       │                                        → Activity.onCreate()
       │                                        → Activity.onStart()
       │                                              │
       │                                     2. executeLifecycleState()
       │                                        → ResumeActivityItem
       │                                        → handleResumeActivity()
       │                                        → Activity.onResume()
```

#### 源码

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

## 5. 常用场景案例

### 场景1：微信从聊天页跳转到朋友圈再返回

> **演示重点**：Task 和 Back Stack 的行为

**操作路径**：消息列表 → 聊天详情 → 发现 → 朋友圈 → 按返回键

```
步骤1：启动微信，进入消息列表
  Task #1: [ 消息列表(RESUMED) ]

步骤2：点击好友，进入聊天详情页
  Task #1: [ 消息列表(STOPPED) → 聊天详情(RESUMED) ]

步骤3：点击底部 Tab"发现"，进入发现页
  Task #1: [ 消息列表(STOPPED) → 聊天详情(STOPPED) → 发现页(RESUMED) ]

步骤4：点击"朋友圈"，进入朋友圈
  Task #1: [ 消息列表 → 聊天详情 → 发现页 → 朋友圈(RESUMED) ]

步骤5：按返回键（逐步弹栈）
  按1次: [ 消息列表 → 聊天详情 → 发现页(RESUMED) ]  ← 朋友圈 onDestroy
  按2次: [ 消息列表 → 聊天详情(RESUMED) ]             ← 发现页 onDestroy
  按3次: [ 消息列表(RESUMED) ]                        ← 聊天详情 onDestroy
  按4次: 回到 Launcher                                ← 消息列表 onDestroy，Task 销毁
```

**为什么这样**：standard 模式下，每次 startActivity 都创建新实例压入栈顶，返回键逐个弹出。这就是 Back Stack 的 LIFO（后进先出）行为。

**如果微信主页用 singleTask**（实际做法）：
```kotlin
// 微信 MainActivity 用 singleTask，点击底部 Tab 不会创建新实例
// 而是回到已有的 MainActivity，清除其上的 Activity
// 这就是为什么微信切 Tab 时聊天详情页会被关闭
```

---

### 场景2：通知栏点击跳转到 App 内页

> **演示重点**：FLAG_ACTIVITY_NEW_TASK 的使用

**问题**：从通知栏（非 Activity 上下文）启动 Activity，必须加 `FLAG_ACTIVITY_NEW_TASK`，否则会崩溃。

```kotlin
// 创建通知点击后的 PendingIntent
fun createNotificationIntent(context: Context, orderId: String): PendingIntent {
    // 构建完整的返回栈：主页 → 订单列表 → 订单详情
    val resultIntent = Intent(context, OrderDetailActivity::class.java).apply {
        putExtra("order_id", orderId)
    }
    
    // 方式1：TaskStackBuilder 自动处理返回栈（推荐）
    val stackBuilder = TaskStackBuilder.create(context).apply {
        addParentStack(OrderDetailActivity::class.java) // 添加父 Activity
        addNextIntent(resultIntent)
    }
    return stackBuilder.getPendingIntent(0, PendingIntent.FLAG_IMMUTABLE)
    
    // 方式2：手动设置 FLAG（不推荐，返回栈不完整）
    // resultIntent.flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
    // return PendingIntent.getActivity(context, 0, resultIntent, PendingIntent.FLAG_IMMUTABLE)
}
```

**AMS 内部处理流程**：
```
通知栏点击 → PendingIntent.send()
    → AMS 收到启动请求
    → 检查 FLAG_ACTIVITY_NEW_TASK
    → 情况A：App 的 Task 已存在
        → 将 Task 移到前台（moveTaskToFront）
        → 在 Task 顶部启动 OrderDetailActivity
    → 情况B：App 的 Task 不存在
        → 创建新 Task
        → 按 TaskStackBuilder 的配置依次创建 Activity
        → 用户按返回键时有正确的返回路径
```

**为什么需要 FLAG_ACTIVITY_NEW_TASK**：AMS 要求每个 Activity 必须属于一个 Task。从非 Activity 上下文（Service、BroadcastReceiver、通知）启动时，没有当前 Task 的上下文，所以必须用此 Flag 指定使用新 Task 或已有 Task。

---

### 场景3：App 被系统杀死后恢复

> **演示重点**：AMS 如何保存和恢复 Activity 状态

```
用户操作：打开 App → 编辑文本 → 按 Home 键 → 打开多个大型 App → 系统内存不足

  步骤1：App 在后台，AMS 标记为缓存进程（ADJ=900+）
  
  步骤2：LMK 杀死 App 进程
         但 AMS 中的 TaskRecord 和 ActivityRecord 信息仍然保留！
         
  步骤3：用户从最近任务切回 App
         AMS 发现进程已死，但 Task 信息还在
         → 通知 Zygote fork 新进程
         → 创建新的 ActivityThread
         → 恢复 Task 中的 Activity（只恢复栈顶 Activity）
         → 传入之前保存的 Bundle 数据
```

**状态保存与恢复的代码**：

```kotlin
class EditActivity : AppCompatActivity() {
    
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        // AMS 在 Activity stop 时通知 App 保存状态
        // 保存的数据存储在 AMS 的 ActivityRecord.icicle 字段中
        outState.putString("draft_text", editText.text.toString())
        outState.putInt("cursor_position", editText.selectionStart)
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_edit)
        
        // 恢复状态
        savedInstanceState?.let { bundle ->
            editText.setText(bundle.getString("draft_text", ""))
            editText.setSelection(bundle.getInt("cursor_position", 0))
        }
    }
}
```

**AMS 内部的保存机制**：
```java
// ActivityRecord 中保存状态
class ActivityRecord {
    Bundle icicle;  // 保存的 Activity 状态（onSaveInstanceState 的数据）
    PersistableBundle persistentState;  // 持久化状态
    
    // 当 Activity stop 时，AMS 通过 Binder 收到 App 端回传的 Bundle
    // 保存在 icicle 字段中，即使进程被杀，这些数据仍在 system_server 中
    
    // 恢复时，AMS 将 icicle 通过 LaunchActivityItem 传回给新的 App 进程
}
```

> **面试关键点**：进程被杀 ≠ Task 被销毁。AMS 在 system_server 进程中，App 进程死亡不影响 AMS 中的记录。这就是为什么"被杀死的 App 还能恢复到之前的页面"。

---

### 场景4：多进程 App（如微信的 :push 进程）

> **演示重点**：AMS 的进程管理

**微信的进程架构**：
```xml
<!-- AndroidManifest.xml -->
<!-- 主进程：UI 相关 -->
<activity android:name=".ui.LauncherUI" />

<!-- :push 进程：推送 Service -->
<service android:name=".app.WXPushService" 
         android:process=":push" />

<!-- :tools 进程：小程序等 -->
<activity android:name=".plugin.appbrand.ui.AppBrandUI" 
          android:process=":tools" />

<!-- :sandbox 进程：隔离执行 -->
<service android:name=".sandbox.SandboxService" 
         android:process=":sandbox" />
```

**AMS 管理多进程的流程**：
```
AMS 的 ProcessList 中：
  ┌─────────────────────────────────────────────────────────────┐
  │  ProcessRecord: com.tencent.mm          (主进程, ADJ=0)     │
  │    - ActivityThread 管理所有 Activity                        │
  │    - uid=10086, pid=12345                                   │
  ├─────────────────────────────────────────────────────────────┤
  │  ProcessRecord: com.tencent.mm:push     (推送进程, ADJ=200) │
  │    - 运行 WXPushService（前台 Service）                     │
  │    - uid=10086, pid=12346                                   │
  │    - 与主进程共享 uid，但独立的进程空间                       │
  ├─────────────────────────────────────────────────────────────┤
  │  ProcessRecord: com.tencent.mm:tools    (工具进程, ADJ=500) │
  │    - 小程序运行环境                                         │
  │    - uid=10086, pid=12347                                   │
  ├─────────────────────────────────────────────────────────────┤
  │  ProcessRecord: com.tencent.mm:sandbox  (沙箱进程, ADJ=800) │
  │    - 隔离执行不信任的代码                                    │
  │    - uid=10086, pid=12348                                   │
  └─────────────────────────────────────────────────────────────┘
```

**多进程的注意事项**：

```kotlin
// 问题1：每个进程有独立的 Application 实例
// MyApp.onCreate() 会执行多次！
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        val processName = getProcessName()
        if (processName == packageName) {
            // 主进程初始化
            initMainProcess()
        } else if (processName.endsWith(":push")) {
            // push 进程只初始化推送
            initPushOnly()
        }
        // 不要在 :push 进程中初始化 UI 相关的 SDK！
    }
}

// 问题2：多进程间不共享内存（单例、static 变量各自独立）
// 需要使用 ContentProvider / AIDL / Messenger 进行跨进程通信

// 问题3：SharedPreferences 在多进程下不安全
// 应使用 ContentProvider 封装 或 MMKV（支持多进程）
```

**AMS 如何决定杀哪个进程**：
- 主进程（用户正在交互）→ ADJ=0，几乎不会被杀
- :push 进程（前台 Service）→ ADJ=200，低内存时较安全
- :tools 进程（后台 Service）→ ADJ=500，内存紧张时可能被杀
- :sandbox 进程（无活跃组件）→ ADJ=800-999，优先被回收

## 6. 常见面试题

### 问题1：Activity 的启动流程是怎样的？

**答案要点**：
1. Activity.startActivity() → Instrumentation.execStartActivity()
2. 通过 Binder IPC 调用到 system_server 的 ATMS.startActivity()
3. ActivityStarter 处理启动请求：权限检查、Intent 解析
4. 计算启动模式（computeLaunchingTaskFlags）、查找或创建 Task
5. 创建 ActivityRecord 作为 Activity 在 AMS 中的"档案"
6. **先暂停当前 Activity**（schedulePauseActivity），等 paused 回调后再继续
7. 如果目标进程不存在，通过 Zygote fork 新进程
8. 通过 ClientTransaction 将 LaunchActivityItem + ResumeActivityItem 发送到目标进程
9. 目标进程的 ActivityThread 收到事务后，依次回调 onCreate → onStart → onResume

> **记忆要点**：两次跨进程（App→AMS→App），先暂停旧的再启动新的，ClientTransaction 是传递生命周期事件的载体。

### 问题2：四种启动模式的区别是什么？

**答案要点**：
- **standard**：每次都创建新实例，放入调用方的 Task
- **singleTop**：栈顶复用，非栈顶仍创建新实例，复用时调用 onNewIntent
- **singleTask**：栈内复用，整个 Task 中只有一个实例，复用时清除其上的 Activity 并调用 onNewIntent
- **singleInstance**：独占 Task，整个系统中只有一个实例，该 Task 只能有这一个 Activity

| 模式 | 新实例？ | 在哪个 Task？ | 典型场景 |
|------|---------|--------------|---------|
| standard | 每次新建 | 调用方 Task | 普通页面 |
| singleTop | 栈顶时不新建 | 调用方 Task | 搜索页、通知跳转 |
| singleTask | 栈内有就不新建 | 自己的 Task（按 affinity） | 主页面、登录页 |
| singleInstance | 全局唯一 | 独占一个 Task | 来电、闹钟、地图导航 |

> **记忆要点**：standard 无脑新建，singleTop 看栈顶，singleTask 看栈内（带清除），singleInstance 独占 Task。

### 问题3：AMS 如何管理进程优先级？

**答案要点**：
1. AMS 中的 OomAdjuster 根据进程中运行的组件状态计算 ADJ 值
2. ADJ 值通过 `/proc/<pid>/oom_score_adj` 写入 Linux 内核
3. LMK（lmkd）在内存不足时，按 ADJ 从高到低杀死进程
4. 五级优先级：前台(0) > 可见(100) > 可感知(200) > 服务(500) > 缓存(900-999)
5. ADJ 值是动态更新的，每次组件状态变化都会重新计算

> **记忆要点**：AMS 打分（ADJ），内核执行（LMK），分高先死。前台进程打 0 分（最安全），后台缓存打 999 分（最危险）。

### 问题4：onNewIntent 什么时候调用？

**答案要点**：
- singleTop 模式，目标 Activity **已经在栈顶**时 → 不新建，调 onNewIntent
- singleTask 模式，目标 Activity **在栈内**时 → 清除其上 Activity，调 onNewIntent
- singleInstance 模式，目标 Activity **已存在**时 → 直接复用，调 onNewIntent
- 使用 `FLAG_ACTIVITY_SINGLE_TOP` + `FLAG_ACTIVITY_CLEAR_TOP` 组合 Flag 时

**注意**：onNewIntent 调用时，需要手动调用 `setIntent(intent)` 更新 Intent，否则后续 `getIntent()` 返回的是旧的 Intent。

> **记忆要点**：只有"复用已有实例"的场景才会调 onNewIntent。新建实例走 onCreate，复用实例走 onNewIntent。

### 问题5：Task 和 Back Stack 的关系是什么？

**答案要点**：
- Task 是一个逻辑概念，代表一组相关 Activity 的集合（如"查看订单"这一系列操作）
- Back Stack 是 Task 内部 Activity 的栈结构，遵循 LIFO（后进先出）
- 按下返回键 → 弹出栈顶 Activity（onDestroy）→ 下一个 Activity 变为栈顶（onResume）
- Task 可以整体移到前台或后台（按 Home 键 = Task 移到后台，从最近任务恢复 = Task 移到前台）
- 每个 Task 有一个 affinity（亲和性），默认等于 App 包名

> **记忆要点**：Task 是一摞牌，Back Stack 决定牌的顺序。返回键弹一张牌，Home 键把整摞牌放到旁边。

### 问题6：AMS 和 ATMS 的关系是什么？为什么要拆分？

**答案要点**：
1. Android 10（API 29）之前，所有逻辑都在 AMS 中，AMS 代码量超过 2 万行，维护困难
2. Android 10 将 Activity/Task 相关逻辑拆分到 ATMS（ActivityTaskManagerService）
3. 拆分后的职责划分：
   - **ATMS**：Activity 启动、Task 管理、窗口管理、生命周期调度
   - **AMS**：进程管理、广播分发、Service 管理、ContentProvider 管理
4. ATMS 是 AMS 的"内部服务"，由 AMS 持有引用，对 App 端透明
5. App 调用 `ActivityTaskManager.getService()` 获取 ATMS 的 Binder 代理

> **记忆要点**：ATMS 管 Activity 和 Task，AMS 管进程和其他三大组件。拆分是为了降低单个类的复杂度。

### 问题7：App 在后台被杀死后，为什么还能恢复到之前的页面？

**答案要点**：
1. AMS 运行在 system_server 进程，App 被杀不影响 AMS
2. AMS 中保留了 App 的 TaskRecord 和 ActivityRecord 信息
3. ActivityRecord 的 `icicle` 字段保存了 `onSaveInstanceState()` 的 Bundle 数据
4. 用户重新打开 App 时，AMS 通知 Zygote fork 新进程
5. AMS 将保存的状态通过 `LaunchActivityItem` 传给新的 App 进程
6. App 端在 `onCreate(savedInstanceState)` 中收到之前保存的 Bundle，恢复 UI 状态

> **记忆要点**：进程死了 ≠ Task 死了。AMS（system_server 中）帮你保管着 Task 信息和 Bundle 状态。

### 问题8：从 Service 或 BroadcastReceiver 中启动 Activity 为什么需要 FLAG_ACTIVITY_NEW_TASK？

**答案要点**：
1. AMS 要求每个 Activity 必须运行在一个 Task 中
2. 从 Activity A 启动 Activity B 时，B 默认进入 A 所在的 Task（有上下文）
3. 但 Service 和 BroadcastReceiver 不属于任何 Task，没有"当前 Task"的概念
4. `FLAG_ACTIVITY_NEW_TASK` 告诉 AMS：创建新 Task 或复用已有的匹配 Task
5. 如果不加这个 Flag，AMS 在 `ActivityStarter.executeRequest()` 中会抛出异常
6. Android 9+ 还要求后台启动 Activity 的额外限制（防止流氓 App 弹窗）

```java
// ActivityStarter.java 中的检查
if (sourceRecord == null) {
    // 没有源 Activity（来自 Service/BroadcastReceiver）
    if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0) {
        // 没有 NEW_TASK flag，报错！
        Slog.w(TAG, "startActivity called from non-Activity context; "
                + "forcing Intent.FLAG_ACTIVITY_NEW_TASK");
        mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;  // Android 高版本会自动补上
    }
}
```

> **记忆要点**：Activity 有 Task 上下文，Service/BR 没有。没有上下文就必须用 FLAG 指明 Task 归属。
