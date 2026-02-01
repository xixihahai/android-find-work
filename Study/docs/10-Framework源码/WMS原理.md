# WMS 原理

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

### 2.2 Window 类型

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

### 2.3 Window 添加流程

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

### 2.4 WMS 添加窗口

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

### 2.5 Surface 管理

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

### 2.6 输入事件分发

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

## 5. 常见面试题

### 问题1：Window、WindowManager、WMS 的关系是什么？

**答案要点**：
- **Window**：抽象概念，表示一个窗口
- **WindowManager**：应用端接口，用于添加、删除、更新窗口
- **WMS**：系统服务，真正管理所有窗口
- 关系：WindowManager 通过 Binder 与 WMS 通信

### 问题2：Window 有哪些类型？

**答案要点**：
- **应用窗口（1-99）**：Activity 窗口
- **子窗口（1000-1999）**：PopupWindow、Dialog
- **系统窗口（2000-2999）**：Toast、状态栏、导航栏、悬浮窗

### 问题3：Activity、Window、DecorView 的关系是什么？

**答案要点**：
- Activity 包含一个 PhoneWindow
- PhoneWindow 包含一个 DecorView
- DecorView 是 View 树的根节点
- Activity.setContentView() 将布局添加到 DecorView

### 问题4：WMS 如何管理窗口层级？

**答案要点**：
- 根据窗口类型计算基础层级
- 同类型窗口按添加顺序排列
- 系统窗口层级高于应用窗口
- 子窗口层级依赖父窗口

### 问题5：输入事件是如何分发到正确窗口的？

**答案要点**：
1. InputDispatcher 获取事件
2. 根据触摸坐标查找目标窗口
3. 通过 InputChannel 发送到目标窗口
4. ViewRootImpl 接收并分发给 View 树
