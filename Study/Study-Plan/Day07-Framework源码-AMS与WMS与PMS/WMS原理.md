# WMS 原理

## 速记总结

> **一句话理解 WMS**：WMS 就像一个**舞台导演**，管理着所有窗口的位置、层级、动画和显示。每个 App 的界面都是一个"演员"，WMS 决定谁站前排、谁站后排、谁该上场、谁该退场。

### 核心概念速记表

| 概念 | 一句话记忆 | 关键类 |
|------|-----------|--------|
| Window | 不是真正的窗口，是一个抽象容器，真正显示的是 View | `PhoneWindow` |
| WindowManager | App 侧的遥控器，通过它添加/删除/更新窗口 | `WindowManagerImpl` → `WindowManagerGlobal` |
| WMS | System Server 中的窗口管家，真正干活的人 | `WindowManagerService` |
| WindowState | WMS 中每个窗口的档案卡，记录窗口所有信息 | `WindowState` |
| WindowToken | 窗口的身份证，防止非法窗口添加 | `WindowToken` / `ActivityRecord` |
| ViewRootImpl | App 和 WMS 之间的桥梁，管理 View 树的绘制和事件 | `ViewRootImpl` |
| Surface | 窗口的画布，真正的像素绘制区域 | `Surface` / `SurfaceControl` |
| SurfaceFlinger | 合成所有 Surface 并送到屏幕显示的 Native 服务 | `SurfaceFlinger`（C++） |
| Session | App 进程与 WMS 的 Binder 连接通道 | `Session` |
| InputChannel | 输入事件的管道，WMS 通过它向窗口分发触摸/按键事件 | `InputChannel` |

---

## 1. 概述

WindowManagerService（WMS）是 Android 系统的窗口管理服务，负责窗口的创建、显示、层级管理和输入事件分发。

### WMS 核心职责

| 职责 | 说明 |
|------|------|
| 窗口管理 | 窗口的添加、删除、更新 |
| 层级管理 | Z-Order 管理 |
| Surface 管理 | 与 SurfaceFlinger 交互 |
| 输入事件 | 事件分发到正确的窗口 |
| 动画管理 | 窗口动画、过渡动画 |
| 屏幕管理 | 多屏幕、分屏支持 |

## 2. 核心原理

### 2.1 WMS 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      WMS 架构图                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   App 进程                          System 进程              │
│   ┌──────────┐                     ┌──────────────────┐    │
│   │  Window  │                     │WindowManagerService│   │
│   │ Manager  │ ◄──── Binder ────► │                  │    │
│   └────┬─────┘                     └────────┬─────────┘    │
│        │                                    │              │
│        ▼                                    ▼              │
│   ┌──────────┐                     ┌──────────────────┐    │
│   │ViewRoot  │                     │  WindowState     │    │
│   │  Impl    │                     │                  │    │
│   └────┬─────┘                     └────────┬─────────┘    │
│        │                                    │              │
│        ▼                                    ▼              │
│   ┌──────────┐                     ┌──────────────────┐    │
│   │ Surface  │ ◄──── 共享内存 ────► │ SurfaceFlinger   │    │
│   └──────────┘                     └──────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Window 体系架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Window 体系架构关系图                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Activity                                                              │
│     │                                                                   │
│     │  attach() 时创建                                                   │
│     ▼                                                                   │
│   Window (抽象类)                                                        │
│     │                                                                   │
│     │  唯一实现类                                                        │
│     ▼                                                                   │
│   PhoneWindow                                                           │
│     │                                                                   │
│     │  内部持有                                                          │
│     ▼                                                                   │
│   DecorView (FrameLayout 子类，View 树的根)                              │
│     │                                                                   │
│     ├── TitleBar / ActionBar                                            │
│     │                                                                   │
│     └── ContentView (android.R.id.content)                              │
│           │                                                             │
│           └── 你 setContentView() 传入的布局                              │
│                                                                         │
│   DecorView 添加到 WindowManager 时：                                    │
│     │                                                                   │
│     ▼                                                                   │
│   ViewRootImpl (View 树的管理者)                                         │
│     │                                                                   │
│     ├── 管理 View 的 measure / layout / draw                            │
│     ├── 持有 Surface（绘制画布）                                          │
│     ├── 通过 IWindowSession (Binder) 与 WMS 通信                        │
│     └── 通过 InputChannel 接收输入事件                                    │
│                                                                         │
│   注意：一个 ViewRootImpl 对应一个 Window                                 │
│         Dialog 也有独立的 PhoneWindow 和 ViewRootImpl                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**核心关系链**：
- `Activity` → 持有 `PhoneWindow` → 持有 `DecorView` → 通过 `ViewRootImpl` → 与 `WMS` 通信
- 每个 `ViewRootImpl` 在 WMS 中对应一个 `WindowState`
- 每个 `WindowState` 持有一个 `SurfaceControl` → 对应 SurfaceFlinger 中的一个 Layer

### 2.3 窗口层级详解

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          窗口层级 (Z-Order) 详解                           │
│                          数值越大，层级越高，越靠近用户                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌──────────────────────────────────────────┐  最高层                      │
│  │  系统窗口 (System Window)                 │  type: 2000-2999            │
│  │  Layer 15-35                              │                             │
│  ├──────────────────────────────────────────┤                              │
│  │  • TYPE_STATUS_BAR (2000)     → 状态栏   │  Layer 15                   │
│  │  • TYPE_SEARCH_BAR (2001)     → 搜索栏   │  Layer 7                    │
│  │  • TYPE_SYSTEM_ALERT (2003)   → 系统弹窗 │  Layer 9                    │
│  │  • TYPE_TOAST (2005)          → Toast    │  Layer 8                    │
│  │  • TYPE_INPUT_METHOD (2011)   → 输入法   │  Layer 13                   │
│  │  • TYPE_NAVIGATION_BAR (2019) → 导航栏   │  Layer 17                   │
│  │  • TYPE_APPLICATION_OVERLAY   → 悬浮窗   │  Layer 12                   │
│  │    (2038, Android 8.0+)                  │                             │
│  └──────────────────────────────────────────┘                              │
│                         ▲                                                  │
│                         │  系统窗口覆盖应用窗口                              │
│                         │                                                  │
│  ┌──────────────────────────────────────────┐  中间层                      │
│  │  应用窗口 (Application Window)            │  type: 1-99                 │
│  │  Layer 2                                  │                             │
│  ├──────────────────────────────────────────┤                              │
│  │  • TYPE_BASE_APPLICATION (1)  → Activity │                             │
│  │  • TYPE_APPLICATION (2)       → 普通窗口 │                              │
│  │  • TYPE_APPLICATION_STARTING  → 启动窗口 │                              │
│  │    (3, 白屏/Splash Screen)               │                             │
│  └──────────────────────────────────────────┘                              │
│                         ▲                                                  │
│                         │  应用窗口覆盖子窗口？不对！                        │
│                         │  子窗口层级 = 父窗口层级 + 偏移                    │
│                         │  所以子窗口在父窗口之上                            │
│                         │                                                  │
│  ┌──────────────────────────────────────────┐  依附于父窗口                 │
│  │  子窗口 (Sub Window)                      │  type: 1000-1999            │
│  │  Layer = 父窗口 Layer + 偏移              │                              │
│  ├──────────────────────────────────────────┤                              │
│  │  • TYPE_APPLICATION_PANEL (1000) → 面板  │                              │
│  │  • TYPE_APPLICATION_MEDIA (1001) → 媒体  │                              │
│  │  • TYPE_APPLICATION_SUB_PANEL   → 子面板 │                              │
│  │    (1002)                                │                              │
│  │  • TYPE_APPLICATION_ATTACHED_DIALOG      │                              │
│  │    (1003) → 附着 Dialog                  │                              │
│  │                                          │                              │
│  │  ⚠️ 子窗口必须依附于父窗口（需要父窗口的 Token）│                         │
│  └──────────────────────────────────────────┘                              │
│                                                                            │
│  💡 记忆口诀：系统最高应用中，子窗口跟着父亲走                                │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Surface 与 SurfaceFlinger 关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│              Surface 与 SurfaceFlinger 关系图                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   App 进程                          System 进程                          │
│   ┌───────────────┐                                                     │
│   │  Canvas.bindW │                                                     │
│   │  (Skia/HWUI)  │ 绘制内容                                            │
│   └───────┬───────┘                                                     │
│           ▼                                                             │
│   ┌───────────────┐                                                     │
│   │   Surface      │ ◄─── 每个 ViewRootImpl 持有一个 Surface             │
│   │   (画布)       │      对应一块 GraphicBuffer                         │
│   └───────┬───────┘                                                     │
│           │ dequeueBuffer / queueBuffer                                 │
│           ▼                                                             │
│   ┌───────────────┐         ┌─────────────────┐                         │
│   │ BufferQueue   │ ◄─────► │  SurfaceControl  │ WMS 通过它控制          │
│   │ (生产者-消费者)│         │  (遥控器)        │ Surface 的位置/层级/     │
│   └───────┬───────┘         └─────────────────┘ 透明度/变换矩阵         │
│           │                                                             │
│           ▼  Native 进程                                                │
│   ┌───────────────────────────────────────┐                             │
│   │         SurfaceFlinger                 │                            │
│   │                                        │                            │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │                            │
│   │  │ Layer 1  │ │ Layer 2  │ │ Layer 3  │  │ 每个 Surface 对应一个 Layer│
│   │  │(状态栏)  │ │(Activity)│ │(导航栏)  │  │                           │
│   │  └────┬────┘ └────┬────┘ └────┬────┘  │                            │
│   │       │           │           │        │                            │
│   │       ▼           ▼           ▼        │                            │
│   │     ┌──────────────────────────────┐   │                            │
│   │     │      合成 (Composition)       │   │                            │
│   │     │  GPU 合成 / HWC 硬件合成      │   │                            │
│   │     └──────────────┬───────────────┘   │                            │
│   │                    │                   │                            │
│   └────────────────────┼───────────────────┘                            │
│                        ▼                                                │
│              ┌──────────────────┐                                       │
│              │   Display (屏幕)  │   VSYNC 信号驱动刷新                   │
│              └──────────────────┘                                       │
│                                                                         │
│  💡 关键流程：                                                           │
│  1. App 通过 Surface 的 Canvas 绘制内容到 GraphicBuffer                  │
│  2. BufferQueue 传递 Buffer 给 SurfaceFlinger                           │
│  3. SurfaceFlinger 将所有 Layer 合成为最终画面                            │
│  4. 送到 Display 显示                                                    │
│                                                                         │
│  💡 WMS 的角色：                                                         │
│  - WMS 不负责绘制，它负责告诉 SurfaceFlinger 每个窗口的位置和层级          │
│  - WMS 通过 SurfaceControl 调整窗口属性（位置、大小、透明度、Z-Order）     │
│  - 实际像素绘制由 App 进程完成，合成显示由 SurfaceFlinger 完成             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.5 Window 类型（源码定义）

```java
// WindowManager.LayoutParams
public static class LayoutParams {
    // 应用窗口 (1-99)
    public static final int TYPE_BASE_APPLICATION = 1;
    public static final int TYPE_APPLICATION = 2;
    public static final int TYPE_APPLICATION_STARTING = 3;
    
    // 子窗口 (1000-1999)
    public static final int TYPE_APPLICATION_PANEL = 1000;
    public static final int TYPE_APPLICATION_MEDIA = 1001;
    public static final int TYPE_APPLICATION_SUB_PANEL = 1002;
    
    // 系统窗口 (2000-2999)
    public static final int TYPE_STATUS_BAR = 2000;
    public static final int TYPE_SEARCH_BAR = 2001;
    public static final int TYPE_PHONE = 2002;
    public static final int TYPE_SYSTEM_ALERT = 2003;
    public static final int TYPE_TOAST = 2005;
    public static final int TYPE_SYSTEM_OVERLAY = 2006;
    public static final int TYPE_INPUT_METHOD = 2011;
    public static final int TYPE_APPLICATION_OVERLAY = 2038;
}
```

### 2.6 Window 添加流程

```java
// WindowManagerImpl.java
@Override
public void addView(View view, ViewGroup.LayoutParams params) {
    mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
            mContext.getUserId());
}

// WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
    
    // 1. 创建 ViewRootImpl
    ViewRootImpl root = new ViewRootImpl(view.getContext(), display);
    
    // 2. 保存到列表
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);
    
    // 3. 设置 View
    root.setView(view, wparams, panelParentView, userId);
}

// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs,
        View panelParentView, int userId) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            
            // 请求布局
            requestLayout();
            
            // 通过 Binder 添加窗口到 WMS
            res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), userId,
                    mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                    mTempControls);
        }
    }
}
```

### 2.7 WMS 添加窗口

```java
// WindowManagerService.java
public int addWindow(Session session, IWindow client, LayoutParams attrs,
        int viewVisibility, int displayId, int requestUserId,
        InsetsVisibilities requestedVisibilities, InputChannel outInputChannel,
        InsetsState outInsetsState, InsetsSourceControl[] outActiveControls) {
    
    // 1. 权限检查
    int res = mPolicy.checkAddPermission(type, isRoundedCornerOverlay, attrs.packageName,
            appOp);
    if (res != ADD_OKAY) {
        return res;
    }
    
    // 2. 创建 WindowState
    final WindowState win = new WindowState(this, session, client, token, parentWindow,
            appOp[0], attrs, viewVisibility, session.mUid, userId,
            session.mCanAddInternalSystemWindow);
    
    // 3. 添加到窗口列表
    win.attach();
    mWindowMap.put(client.asBinder(), win);
    
    // 4. 创建 Surface
    win.createSurfaceControl(true);
    
    // 5. 创建输入通道
    if (openInputChannels) {
        win.openInputChannel(outInputChannel);
    }
    
    return res;
}
```

### 2.8 Surface 管理

```java
// WindowState.java
void createSurfaceControl(boolean force) {
    if (mSurfaceControl == null || force) {
        // 创建 SurfaceControl
        mSurfaceControl = mWmService.mSurfaceControlFactory.apply(
                makeSurface()
                        .setParent(mSurfaceAnimator.getLeash())
                        .setName(getName())
                        .setFormat(PixelFormat.TRANSLUCENT)
                        .setCallsite("WindowState.createSurfaceControl"));
    }
}

// SurfaceControl.java
public static class Builder {
    public SurfaceControl build() {
        // 通过 JNI 创建 Native 层 SurfaceControl
        return new SurfaceControl(mSession, mName, mWidth, mHeight, mFormat,
                mFlags, mParent, mMetadata, mLocalOwnerView, mCallsite);
    }
}
```

### 2.9 输入事件分发

```java
// InputDispatcher.cpp
void InputDispatcher::dispatchOnce() {
    // 1. 获取待分发的事件
    if (!haveCommandsLocked()) {
        dispatchOnceInnerLocked(&nextWakeupTime);
    }
    
    // 2. 执行命令
    runCommandsLockedInterruptable();
}

void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    // 获取事件
    mPendingEvent = mInboundQueue.front();
    mInboundQueue.pop_front();
    
    // 根据事件类型分发
    switch (mPendingEvent->type) {
        case EventEntry::Type::KEY: {
            dispatchKeyLocked(currentTime, keyEntry, &dropReason, nextWakeupTime);
            break;
        }
        case EventEntry::Type::MOTION: {
            dispatchMotionLocked(currentTime, motionEntry, &dropReason, nextWakeupTime);
            break;
        }
    }
}

// 查找目标窗口
std::vector<InputTarget> InputDispatcher::findTouchedWindowTargetsLocked(
        nsecs_t currentTime, const MotionEntry& entry, bool* outConflictingPointerActions,
        InputEventInjectionResult* outInjectionResult) {
    
    // 遍历窗口，找到触摸点所在的窗口
    for (const sp<WindowInfoHandle>& windowHandle : windowHandles) {
        const WindowInfo& info = *windowHandle->getInfo();
        
        // 检查触摸点是否在窗口范围内
        if (info.touchableRegionContainsPoint(x, y)) {
            // 找到目标窗口
            targets.push_back(InputTarget{windowHandle, ...});
            break;
        }
    }
    
    return targets;
}
```

## 3. 关键源码解析

### 3.1 窗口层级管理

```java
// WindowState.java
int getBaseLayer() {
    // 根据窗口类型计算基础层级
    return mPolicy.getWindowLayerFromTypeLw(mAttrs.type, mOwnerCanAddInternalSystemWindow);
}

// WindowManagerPolicy.java
default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow) {
    if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
        return APPLICATION_LAYER;
    }
    
    switch (type) {
        case TYPE_STATUS_BAR:
            return 15;
        case TYPE_NAVIGATION_BAR:
            return 17;
        case TYPE_INPUT_METHOD:
            return 13;
        case TYPE_TOAST:
            return 8;
        // ...
    }
}
```

### 3.2 窗口动画

```java
// WindowStateAnimator.java
void applyAnimationLocked(int transit, boolean isEntrance) {
    // 获取动画
    Animation a = mPolicy.selectAnimation(mWin, transit);
    
    if (a != null) {
        // 设置动画
        mAnimation = a;
        mAnimating = true;
        mAnimationIsEntrance = isEntrance;
    }
}

// 执行动画
boolean stepAnimationLocked(long currentTime) {
    if (mAnimation == null) {
        return false;
    }
    
    // 计算动画进度
    mAnimation.getTransformation(currentTime, mTransformation);
    
    // 应用变换
    mSurfaceController.setMatrix(mTransformation.getMatrix());
    mSurfaceController.setAlpha(mTransformation.getAlpha());
    
    return !mAnimation.hasEnded();
}
```

## 4. 实战应用

### 4.1 悬浮窗实现

```kotlin
class FloatingWindowService : Service() {
    private lateinit var windowManager: WindowManager
    private lateinit var floatingView: View
    
    override fun onCreate() {
        super.onCreate()
        windowManager = getSystemService(WINDOW_SERVICE) as WindowManager
        
        // 创建悬浮窗布局
        floatingView = LayoutInflater.from(this)
            .inflate(R.layout.floating_window, null)
        
        // 设置窗口参数
        val params = WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,  // Android 8.0+
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = 0
            y = 100
        }
        
        // 添加悬浮窗
        windowManager.addView(floatingView, params)
        
        // 设置拖动
        setupDrag(params)
    }
    
    private fun setupDrag(params: WindowManager.LayoutParams) {
        var initialX = 0
        var initialY = 0
        var initialTouchX = 0f
        var initialTouchY = 0f
        
        floatingView.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    initialX = params.x
                    initialY = params.y
                    initialTouchX = event.rawX
                    initialTouchY = event.rawY
                    true
                }
                MotionEvent.ACTION_MOVE -> {
                    params.x = initialX + (event.rawX - initialTouchX).toInt()
                    params.y = initialY + (event.rawY - initialTouchY).toInt()
                    windowManager.updateViewLayout(floatingView, params)
                    true
                }
                else -> false
            }
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        windowManager.removeView(floatingView)
    }
}
```

## 5. 场景案例深度解析

### 5.1 Dialog 为什么不能用 Application Context？

**现象**：使用 Application Context 创建 Dialog 会抛出 `BadTokenException`。

```kotlin
// ❌ 崩溃：Unable to add window -- token null is not valid
val dialog = AlertDialog.Builder(applicationContext)
    .setTitle("Test")
    .setMessage("Hello")
    .create()
dialog.show()  // 💥 BadTokenException

// ✅ 正确：使用 Activity Context
val dialog = AlertDialog.Builder(this@MainActivity)
    .setTitle("Test")
    .setMessage("Hello")
    .create()
dialog.show()
```

**原理分析（Window Token 机制）**：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Window Token 验证机制                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Activity 启动时：                                               │
│  1. AMS 创建 ActivityRecord（包含一个 Token）                     │
│  2. Token 传递给 WMS，WMS 记录 Token → WindowToken 映射          │
│  3. Token 也传递给 Activity 的 PhoneWindow                       │
│                                                                 │
│  Dialog.show() 时：                                              │
│  1. Dialog 使用宿主 Activity 的 WindowManager                    │
│  2. WindowManager 持有 parentWindow（Activity 的 PhoneWindow）   │
│  3. 添加窗口时携带 parentWindow 的 Token                         │
│  4. WMS.addWindow() 验证 Token 是否合法                          │
│                                                                 │
│  Application Context 的问题：                                    │
│  1. Application 没有关联的 Activity                              │
│  2. 没有有效的 Window Token                                     │
│  3. WMS 验证 Token 失败 → BadTokenException                     │
│                                                                 │
│  💡 本质：Dialog 是应用窗口，需要父窗口的 Token 验证               │
│     Application 不是窗口，没有 Token                              │
│                                                                 │
│  ⚠️ 解决方案（如果必须用 Application Context）：                  │
│  设置窗口类型为系统窗口（需要权限）                               │
│  dialog.window?.setType(                                        │
│      WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**面试记忆要点**：Dialog 需要 Activity 的 Token → Application 没有 Token → WMS 校验失败。

### 5.2 悬浮窗权限机制

**问题**：Android 6.0 以前悬浮窗用 `TYPE_SYSTEM_ALERT`，为什么 Android 8.0+ 改成 `TYPE_APPLICATION_OVERLAY`？

```
┌─────────────────────────────────────────────────────────────────┐
│               悬浮窗权限演进                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Android 6.0 以前：                                              │
│  • 使用 TYPE_SYSTEM_ALERT / TYPE_PHONE / TYPE_SYSTEM_OVERLAY    │
│  • 声明 SYSTEM_ALERT_WINDOW 权限即可                             │
│  • 安装时自动授予 → 安全隐患大（恶意悬浮窗钓鱼）                  │
│                                                                 │
│  Android 6.0 ~ 7.x：                                            │
│  • 仍用 TYPE_SYSTEM_ALERT                                       │
│  • SYSTEM_ALERT_WINDOW 变为特殊权限                              │
│  • 需要引导用户到设置页手动开启                                   │
│  • Settings.canDrawOverlays(context) 检查                       │
│                                                                 │
│  Android 8.0+：                                                  │
│  • 弃用 TYPE_SYSTEM_ALERT 等旧类型                               │
│  • 统一使用 TYPE_APPLICATION_OVERLAY (2038)                      │
│  • 层级低于系统关键窗口（状态栏、输入法之下）                     │
│  • 仍需 SYSTEM_ALERT_WINDOW 权限                                │
│  • 用户可随时关闭某个 App 的悬浮窗权限                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```kotlin
// 完整的悬浮窗权限检查和请求
fun checkOverlayPermission(activity: Activity) {
    if (!Settings.canDrawOverlays(activity)) {
        // 引导用户开启权限
        val intent = Intent(
            Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
            Uri.parse("package:${activity.packageName}")
        )
        activity.startActivityForResult(intent, OVERLAY_PERMISSION_CODE)
    }
}

// 创建悬浮窗时的版本适配
fun getWindowType(): Int {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY  // Android 8.0+
    } else {
        @Suppress("DEPRECATION")
        WindowManager.LayoutParams.TYPE_SYSTEM_ALERT  // Android 8.0 以下
    }
}
```

### 5.3 Toast 的显示原理

**Toast 为什么能在任何地方显示？它不需要 Activity 的 Token 吗？**

```
┌─────────────────────────────────────────────────────────────────┐
│                   Toast 显示流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  App 进程                           System 进程                  │
│                                                                 │
│  Toast.show()                                                   │
│     │                                                           │
│     ▼                                                           │
│  Toast.TN (Binder 对象)                                         │
│     │                                                           │
│     │ INotificationManager.enqueueToast(pkg, TN, duration)      │
│     ▼                                                           │
│  NotificationManagerService (NMS)                               │
│     │                                                           │
│     │ 1. 将 Toast 加入队列 mToastQueue                          │
│     │ 2. 检查队列（每个包最多 5 个待显示）                        │
│     │ 3. 调用 TN.show() 回调给 App                              │
│     ▼                                                           │
│  TN.show()  (回到 App 进程)                                     │
│     │                                                           │
│     │ 通过 Handler 切换到主线程                                  │
│     ▼                                                           │
│  WindowManager.addView(mView, params)                           │
│     │                                                           │
│     │ params.type = TYPE_TOAST (2005)                           │
│     │ params.token = windowToken (NMS 提供)                     │
│     ▼                                                           │
│  WMS.addWindow()                                                │
│     │                                                           │
│     │ Toast 是系统窗口类型，NMS 提供合法 Token                   │
│     │ 所以不需要 Activity 的 Token                               │
│     ▼                                                           │
│  显示 → 定时 → NMS 回调 TN.hide() → WindowManager.removeView() │
│                                                                 │
│  💡 关键点：                                                     │
│  • Toast 的 Token 由 NMS 提供，不依赖 Activity                  │
│  • Toast 是系统窗口（TYPE_TOAST = 2005），层级在应用窗口之上      │
│  • NMS 控制 Toast 的显示时序（排队机制）                          │
│  • Android 11+ 限制了后台 Toast 的自定义 View（安全考虑）        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 Activity 的窗口创建完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│           Activity 窗口创建流程                                   │
│           从 Activity 创建到窗口显示                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ActivityThread.performLaunchActivity()                       │
│     │                                                           │
│     ├── activity = Instrumentation.newActivity()  // 反射创建    │
│     │                                                           │
│     ├── activity.attach(context, window, ...)                   │
│     │     │                                                     │
│     │     ├── mWindow = new PhoneWindow(this, window, ...)      │
│     │     │   // 创建 PhoneWindow                                │
│     │     │                                                     │
│     │     ├── mWindow.setWindowManager(wm, mToken, ...)         │
│     │     │   // 设置 WindowManager，传入 AMS 给的 Token         │
│     │     │                                                     │
│     │     └── mWindowManager = mWindow.getWindowManager()       │
│     │         // 获取 WindowManagerImpl（持有 parentWindow）     │
│     │                                                           │
│     └── mInstrumentation.callActivityOnCreate(activity, ...)    │
│           │                                                     │
│           └── activity.onCreate()                               │
│                 │                                               │
│                 └── setContentView(R.layout.activity_main)      │
│                       │                                         │
│                       └── PhoneWindow.setContentView()          │
│                             │                                   │
│                             ├── installDecor()                  │
│                             │   ├── generateDecor()  // 创建 DecorView │
│                             │   └── generateLayout() // 加载布局模板   │
│                             │                                   │
│                             └── mLayoutInflater.inflate(         │
│                                     layoutResID, mContentParent)│
│                                 // 将用户布局添加到 ContentView │
│                                                                 │
│  2. ActivityThread.handleResumeActivity()                       │
│     │                                                           │
│     ├── activity.performResume()  // 回调 onResume              │
│     │                                                           │
│     └── wm.addView(decor, l)  // ⭐ 关键！将 DecorView 添加到 WM│
│           │                                                     │
│           └── WindowManagerGlobal.addView()                     │
│                 │                                               │
│                 ├── root = new ViewRootImpl()  // 创建桥梁      │
│                 │                                               │
│                 └── root.setView(decor, ...)                    │
│                       │                                         │
│                       ├── requestLayout()  // 触发首次绘制      │
│                       │   └── scheduleTraversals()              │
│                       │       └── doTraversal()                 │
│                       │           └── performTraversals()       │
│                       │               ├── measureHierarchy()    │
│                       │               ├── performLayout()       │
│                       │               └── performDraw()         │
│                       │                                         │
│                       └── mWindowSession.addToDisplayAsUser()   │
│                           // Binder 调用 WMS.addWindow()        │
│                           // 窗口正式注册到 WMS                  │
│                                                                 │
│  💡 关键时序：                                                   │
│  • onCreate → PhoneWindow + DecorView 创建，但还没添加到 WMS    │
│  • onResume → DecorView 添加到 WindowManager → 注册到 WMS      │
│  • 所以 onCreate 中 View 还不可见，onResume 后才开始绘制显示    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 常见面试题

### 问题1：Window、WindowManager、WMS 的关系是什么？

**答案要点**：
- **Window**：抽象概念，表示一个窗口，唯一实现是 `PhoneWindow`
- **WindowManager**：应用端接口，用于添加、删除、更新窗口，实现类是 `WindowManagerImpl`
- **WMS**：系统服务，真正管理所有窗口的创建、层级、动画
- 关系：`WindowManagerImpl` → `WindowManagerGlobal` → `ViewRootImpl` → 通过 `IWindowSession`(Binder) → `WMS`

> **记忆要点**：Window 是抽象，WindowManager 是遥控器，WMS 是真正的管家。App 按遥控器，管家干活。

### 问题2：Window 有哪些类型？各自的特点？

**答案要点**：
- **应用窗口（1-99）**：Activity 窗口，需要 Activity Token
- **子窗口（1000-1999）**：PopupWindow 等，必须依附于父窗口，需要父窗口 Token
- **系统窗口（2000-2999）**：Toast、状态栏、导航栏、悬浮窗，需要系统权限，不需要 Activity Token

> **记忆要点**：数字越大层级越高。应用窗口靠 Token，子窗口靠父亲，系统窗口靠权限。Dialog 是应用窗口类型，需要 Activity Token。

### 问题3：Activity、Window、DecorView、ViewRootImpl 的关系是什么？

**答案要点**：
- Activity 在 `attach()` 中创建 PhoneWindow
- PhoneWindow 在 `setContentView()` 中创建 DecorView
- DecorView 是 View 树的根节点（FrameLayout 子类）
- `handleResumeActivity()` 时创建 ViewRootImpl，将 DecorView 添加到 WindowManager
- ViewRootImpl 是 View 树和 WMS 之间的桥梁，管理绘制和事件分发

> **记忆要点**：Activity 包 Window，Window 包 DecorView，ViewRootImpl 管绘制和通信。onCreate 创建，onResume 才显示。

### 问题4：WMS 如何管理窗口层级？

**答案要点**：
- 根据窗口类型通过 `WindowManagerPolicy.getWindowLayerFromTypeLw()` 计算基础层级
- 同类型窗口按添加顺序排列（后添加的在上面）
- 系统窗口层级（如状态栏 Layer 15）高于应用窗口（Layer 2）
- 子窗口层级 = 父窗口层级 + 偏移量
- 最终层级信息传递给 SurfaceFlinger 的 Layer 进行合成

> **记忆要点**：类型决定基础层级，同类型看顺序，子窗口跟父走。

### 问题5：输入事件是如何分发到正确窗口的？

**答案要点**：
1. 硬件产生事件 → InputReader 读取 → 放入 InputDispatcher 队列
2. InputDispatcher 根据触摸坐标遍历窗口列表（`findTouchedWindowTargetsLocked`）
3. 通过 `touchableRegionContainsPoint()` 找到触摸点所在的窗口
4. 通过 `InputChannel`（socketpair）发送事件到目标窗口的 `ViewRootImpl`
5. `ViewRootImpl` 接收后通过 `DecorView.dispatchTouchEvent()` 分发给 View 树

> **记忆要点**：InputReader 读 → InputDispatcher 找窗口 → InputChannel 传 → ViewRootImpl 收 → View 树分发。

### 问题6：Dialog 为什么不能用 Application Context 创建？

**答案要点**：
- Dialog 需要一个有效的 Window Token 来通过 WMS 的验证
- Activity Context 持有 Activity 的 Token（由 AMS 创建并注册到 WMS）
- Application Context 没有关联的 Activity，因此没有有效的 Token
- WMS.addWindow() 校验 Token 失败，抛出 `BadTokenException`
- 解决：使用 Activity Context，或设置系统窗口类型（需要 `SYSTEM_ALERT_WINDOW` 权限）

> **记忆要点**：Dialog 需要 Token → Token 来自 Activity → Application 没有 Activity → 没 Token → 崩溃。

### 问题7：Toast 能在任何地方显示的原理是什么？

**答案要点**：
- Toast 是系统窗口类型（TYPE_TOAST = 2005），不需要 Activity Token
- Toast 显示流程：`Toast.show()` → `NMS.enqueueToast()` → NMS 管理队列 → 回调 `TN.show()` → `WindowManager.addView()`
- Token 由 NotificationManagerService 提供，WMS 认可 NMS 的 Token
- NMS 负责 Toast 的排队、计时、回调隐藏

> **记忆要点**：Toast 的 Token 由 NMS 提供（不是 Activity），所以 Application Context 也能用。NMS 是 Toast 的导演。

### 问题8：Surface、SurfaceFlinger、WMS 三者什么关系？

**答案要点**：
- **Surface**：窗口的画布，App 通过 Canvas/OpenGL 在上面绘制内容，底层是 GraphicBuffer
- **SurfaceFlinger**：Native 层合成服务，收集所有 Surface 的内容（Layer），合成最终画面送到屏幕
- **WMS**：不负责绘制，负责管理窗口元数据（位置、大小、层级、可见性），通过 SurfaceControl 告诉 SurfaceFlinger 每个 Layer 的属性
- 流程：App 绘制 → Surface(BufferQueue) → SurfaceFlinger 合成 → Display 显示
- WMS 通过 SurfaceControl 调整 SurfaceFlinger 中 Layer 的 Z-Order / 位置 / 透明度

> **记忆要点**：App 画画（Surface），WMS 排座位（SurfaceControl），SurfaceFlinger 合成拍照送屏幕。三者分工明确。
