# View 绑定与绑定流程

## 1. 概述

View 是 Android UI 系统的核心组件，所有用户界面元素都是 View 或其子类。View 的绑定流程（也称为绘制流程）是 Android 渲染机制的基础，包括三个核心阶段：**measure（测量）**、**layout（布局）** 和 **draw（绘制）**。

理解 View 的绑定流程对于：
- 自定义 View/ViewGroup 开发
- UI 性能优化
- 解决布局问题
- 理解 Android 渲染机制

都至关重要。本文将深入分析 View 树结构、三大绑定流程、自定义 View 实现以及相关源码。

> **目标版本**：Android W (API 35)  
> **重点关注**：OPPO/vivo 面试特别重视 Framework 源码分析

## 2. 核心原理

### 2.1 View 树结构

Android 的 UI 界面是一个树形结构，从 DecorView 开始，层层嵌套：

```
                    ┌─────────────────────────────────────────┐
                    │              PhoneWindow                 │
                    │         (Window 的唯一实现类)             │
                    └─────────────────┬───────────────────────┘
                                      │
                    ┌─────────────────▼───────────────────────┐
                    │              DecorView                   │
                    │    (FrameLayout 子类，顶层容器)           │
                    │    - 包含状态栏、导航栏区域                │
                    │    - 包含 ContentParent                  │
                    └─────────────────┬───────────────────────┘
                                      │
                    ┌─────────────────▼───────────────────────┐
                    │           LinearLayout                   │
                    │      (系统默认布局容器)                    │
                    └─────────────────┬───────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
              ▼                       ▼                       ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │   ActionBar     │   │  ContentParent  │   │  NavigationBar  │
    │   (标题栏区域)   │   │ (android.R.id.  │   │   (导航栏区域)   │
    │                 │   │    content)     │   │                 │
    └─────────────────┘   └────────┬────────┘   └─────────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │      用户布局 (Your Layout)   │
                    │   setContentView() 设置的    │
                    │         XML 布局             │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
        ┌──────────┐        ┌──────────┐        ┌──────────┐
        │  View 1  │        │ViewGroup │        │  View 3  │
        └──────────┘        │    2     │        └──────────┘
                            └────┬─────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │  View A  │ │  View B  │ │  View C  │
              └──────────┘ └──────────┘ └──────────┘
```


#### View 树的关键类

| 类名 | 说明 |
|------|------|
| **ViewRootImpl** | View 树的管理者，连接 WindowManager 和 DecorView |
| **DecorView** | 顶层 View，FrameLayout 子类 |
| **PhoneWindow** | Window 的唯一实现类，管理 DecorView |
| **WindowManager** | 窗口管理器，负责添加、更新、删除 View |
| **Choreographer** | 编舞者，协调 VSYNC 信号和绘制 |

### 2.2 绑定流程触发时机

View 的绑定流程由 `ViewRootImpl.performTraversals()` 触发，主要时机：

```
┌─────────────────────────────────────────────────────────────────┐
│                    绑定流程触发时机                               │
├─────────────────────────────────────────────────────────────────┤
│  1. Activity 首次显示                                            │
│     ActivityThread.handleResumeActivity()                       │
│         → WindowManager.addView()                               │
│         → ViewRootImpl.setView()                                │
│         → ViewRootImpl.requestLayout()                          │
│                                                                 │
│  2. View 调用 requestLayout()                                   │
│     - View 大小或位置需要改变                                     │
│     - 触发 measure + layout + draw                              │
│                                                                 │
│  3. View 调用 invalidate()                                      │
│     - View 内容需要重绘                                          │
│     - 只触发 draw                                                │
│                                                                 │
│  4. VSYNC 信号到来                                               │
│     - Choreographer 收到 VSYNC                                  │
│     - 执行已注册的绘制回调                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 MeasureSpec 详解

MeasureSpec 是一个 32 位的 int 值，封装了父容器对子 View 的测量要求：

```
┌────────────────────────────────────────────────────────────────┐
│                    MeasureSpec 结构 (32位)                      │
├────────────────────────────────────────────────────────────────┤
│  高 2 位: SpecMode (测量模式)                                    │
│  低 30 位: SpecSize (测量大小)                                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────┬──────────────────────────────────────────────┐   │
│  │  Mode    │                    Size                      │   │
│  │  (2位)   │                   (30位)                     │   │
│  └──────────┴──────────────────────────────────────────────┘   │
│                                                                │
│  Mode 取值:                                                     │
│  - UNSPECIFIED (00): 父容器不限制，子 View 可以是任意大小         │
│  - EXACTLY     (01): 精确模式，对应 match_parent 或具体数值       │
│  - AT_MOST     (10): 最大模式，对应 wrap_content                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

#### 三种测量模式详解

| 模式 | 值 | 说明 | 对应 LayoutParams |
|------|---|------|------------------|
| **UNSPECIFIED** | 0 | 父容器不限制子 View 大小 | 系统内部使用（如 ScrollView） |
| **EXACTLY** | 1 | 精确大小，子 View 必须是这个大小 | match_parent 或具体 dp 值 |
| **AT_MOST** | 2 | 最大大小，子 View 不能超过这个值 | wrap_content |

#### MeasureSpec 生成规则

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    子 View 的 MeasureSpec 生成规则                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  父 SpecMode          子 LayoutParams        子 SpecMode      子 SpecSize   │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  EXACTLY              具体数值 (如 100dp)     EXACTLY         childSize     │
│  (父精确大小)          match_parent           EXACTLY         parentSize    │
│                       wrap_content           AT_MOST         parentSize    │
│                                                                             │
│  AT_MOST              具体数值 (如 100dp)     EXACTLY         childSize     │
│  (父最大大小)          match_parent           AT_MOST         parentSize    │
│                       wrap_content           AT_MOST         parentSize    │
│                                                                             │
│  UNSPECIFIED          具体数值 (如 100dp)     EXACTLY         childSize     │
│  (父不限制)            match_parent           UNSPECIFIED     0             │
│                       wrap_content           UNSPECIFIED     0             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```


### 2.4 Measure 过程

Measure 过程确定 View 的测量宽高，是绑定流程的第一步：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Measure 流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ViewRootImpl.performMeasure()                                  │
│         │                                                       │
│         ▼                                                       │
│  DecorView.measure(widthMeasureSpec, heightMeasureSpec)         │
│         │                                                       │
│         ▼                                                       │
│  View.measure() ─────────────────────────────────────────────┐  │
│         │                                                    │  │
│         │  检查是否需要重新测量:                               │  │
│         │  - forceLayout 标志                                │  │
│         │  - MeasureSpec 是否改变                            │  │
│         │                                                    │  │
│         ▼                                                    │  │
│  View.onMeasure() ◄──────────────────────────────────────────┘  │
│         │                                                       │
│         │  默认实现: getDefaultSize()                           │
│         │  ViewGroup: 遍历测量所有子 View                        │
│         │                                                       │
│         ▼                                                       │
│  setMeasuredDimension(width, height)                            │
│         │                                                       │
│         │  保存测量结果到:                                       │
│         │  - mMeasuredWidth                                     │
│         │  - mMeasuredHeight                                    │
│         │                                                       │
│         ▼                                                       │
│  测量完成，可通过 getMeasuredWidth/Height() 获取                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### ViewGroup 的测量流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   ViewGroup Measure 流程                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ViewGroup.onMeasure()                                          │
│         │                                                       │
│         ▼                                                       │
│  for (每个子 View) {                                             │
│         │                                                       │
│         ▼                                                       │
│    getChildMeasureSpec()  ◄─── 根据父 MeasureSpec 和子           │
│         │                      LayoutParams 计算子 MeasureSpec   │
│         ▼                                                       │
│    child.measure(childWidthSpec, childHeightSpec)               │
│         │                                                       │
│         ▼                                                       │
│    累加子 View 尺寸，计算自身大小                                  │
│  }                                                              │
│         │                                                       │
│         ▼                                                       │
│  setMeasuredDimension(totalWidth, totalHeight)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 Layout 过程

Layout 过程确定 View 的最终位置（四个顶点坐标）：

```
┌─────────────────────────────────────────────────────────────────┐
│                       Layout 流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ViewRootImpl.performLayout()                                   │
│         │                                                       │
│         ▼                                                       │
│  DecorView.layout(left, top, right, bottom)                     │
│         │                                                       │
│         ▼                                                       │
│  View.layout() ──────────────────────────────────────────────┐  │
│         │                                                    │  │
│         │  1. setFrame() 设置四个顶点位置                      │  │
│         │     - mLeft, mTop, mRight, mBottom                 │  │
│         │                                                    │  │
│         │  2. 如果位置改变，触发 onSizeChanged()              │  │
│         │                                                    │  │
│         ▼                                                    │  │
│  View.onLayout() ◄───────────────────────────────────────────┘  │
│         │                                                       │
│         │  View: 空实现                                         │
│         │  ViewGroup: 必须重写，确定子 View 位置                  │
│         │                                                       │
│         ▼                                                       │
│  for (每个子 View) {                                             │
│      child.layout(l, t, r, b)  // 递归布局子 View               │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### View 的坐标系统

```
┌─────────────────────────────────────────────────────────────────┐
│                      View 坐标系统                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  父 View 坐标系 (相对于父容器)                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ (0,0)                                                    │   │
│  │   ┌─────────────────────────────────────────────────┐    │   │
│  │   │                                                 │    │   │
│  │   │  (mLeft, mTop)                                  │    │   │
│  │   │     ┌───────────────────────────────────┐       │    │   │
│  │   │     │                                   │       │    │   │
│  │   │     │         子 View                   │       │    │   │
│  │   │     │                                   │       │    │   │
│  │   │     │   width = mRight - mLeft          │       │    │   │
│  │   │     │   height = mBottom - mTop         │       │    │   │
│  │   │     │                                   │       │    │   │
│  │   │     └───────────────────────────────────┘       │    │   │
│  │   │                              (mRight, mBottom)  │    │   │
│  │   │                                                 │    │   │
│  │   └─────────────────────────────────────────────────┘    │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  常用方法:                                                       │
│  - getLeft()/getTop()/getRight()/getBottom(): 相对父容器位置     │
│  - getX()/getY(): left + translationX/Y                        │
│  - getWidth()/getHeight(): 实际宽高 (layout 后)                  │
│  - getMeasuredWidth()/getMeasuredHeight(): 测量宽高 (measure 后) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


### 2.6 Draw 过程

Draw 过程将 View 绘制到屏幕上，是绑定流程的最后一步：

```
┌─────────────────────────────────────────────────────────────────┐
│                        Draw 流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ViewRootImpl.performDraw()                                     │
│         │                                                       │
│         ├──────────────────────────────────────────────────┐    │
│         │                                                  │    │
│         ▼                                                  ▼    │
│  软件绘制 (Software)                              硬件加速绘制    │
│  drawSoftware()                                  (Hardware)     │
│         │                                        ThreadedRenderer│
│         ▼                                        .draw()        │
│  Canvas (Skia)                                        │         │
│         │                                             ▼         │
│         │                                     DisplayList       │
│         │                                     (RenderNode)      │
│         │                                             │         │
│         └──────────────────┬──────────────────────────┘         │
│                            │                                    │
│                            ▼                                    │
│                    View.draw(canvas)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Draw 的六个步骤

```
┌─────────────────────────────────────────────────────────────────┐
│                    View.draw() 六个步骤                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: drawBackground(canvas)                                 │
│          绘制背景                                                │
│          │                                                      │
│          ▼                                                      │
│  Step 2: 保存 Canvas 图层 (如果需要)                              │
│          用于 fading edge 效果                                   │
│          │                                                      │
│          ▼                                                      │
│  Step 3: onDraw(canvas)  ◄─── 绘制自身内容                       │
│          子类重写此方法                                          │
│          │                                                      │
│          ▼                                                      │
│  Step 4: dispatchDraw(canvas)  ◄─── 绘制子 View                  │
│          ViewGroup 重写此方法                                    │
│          │                                                      │
│          ▼                                                      │
│  Step 5: 绘制 fading edge (渐变边缘)                             │
│          恢复 Canvas 图层                                        │
│          │                                                      │
│          ▼                                                      │
│  Step 6: drawForeground(canvas)                                 │
│          绘制前景、滚动条等装饰                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 硬件加速 vs 软件绘制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      硬件加速 vs 软件绘制对比                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  特性              软件绘制                      硬件加速                     │
│  ─────────────────────────────────────────────────────────────────────────  │
│  渲染线程          主线程 (UI Thread)            RenderThread               │
│  绘制 API          Skia (CPU)                   OpenGL ES / Vulkan (GPU)   │
│  绘制对象          Bitmap                       DisplayList (RenderNode)   │
│  invalidate        重绘整个 View 树              只重绘脏区域                 │
│  内存占用          较低                          较高 (GPU 缓存)             │
│  动画性能          较差                          优秀                        │
│  兼容性            所有 API 都支持               部分 API 不支持             │
│                                                                             │
│  不支持硬件加速的操作:                                                        │
│  - Canvas.drawBitmapMesh()                                                  │
│  - Canvas.drawPicture()                                                     │
│  - Paint.setMaskFilter() (部分)                                             │
│  - 某些 Xfermode                                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.7 requestLayout vs invalidate

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   requestLayout vs invalidate 对比                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  方法              触发流程                      使用场景                     │
│  ─────────────────────────────────────────────────────────────────────────  │
│  requestLayout()   measure → layout → draw      View 大小或位置改变          │
│                                                 如: 文本内容改变导致大小变化  │
│                                                                             │
│  invalidate()      draw                         View 内容改变但大小不变       │
│                                                 如: 背景颜色改变              │
│                                                                             │
│  postInvalidate()  draw (可在子线程调用)         同 invalidate，线程安全      │
│                                                                             │
│  forceLayout()     标记需要重新 measure          强制下次 requestLayout       │
│                    不会立即触发                  时重新测量                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### requestLayout 流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   requestLayout 流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  View.requestLayout()                                           │
│         │                                                       │
│         │  1. 设置 PFLAG_FORCE_LAYOUT 标志                       │
│         │  2. 设置 PFLAG_INVALIDATED 标志                        │
│         │                                                       │
│         ▼                                                       │
│  mParent.requestLayout()  ◄─── 向上传递到父 View                 │
│         │                                                       │
│         │  递归向上，直到 ViewRootImpl                           │
│         │                                                       │
│         ▼                                                       │
│  ViewRootImpl.requestLayout()                                   │
│         │                                                       │
│         │  检查是否在主线程                                       │
│         │  checkThread()                                        │
│         │                                                       │
│         ▼                                                       │
│  scheduleTraversals()                                           │
│         │                                                       │
│         │  1. 设置同步屏障                                       │
│         │  2. 注册 VSYNC 回调                                    │
│         │                                                       │
│         ▼                                                       │
│  Choreographer.postCallback(CALLBACK_TRAVERSAL, mTraversalRunnable)│
│         │                                                       │
│         │  等待下一个 VSYNC 信号                                  │
│         │                                                       │
│         ▼                                                       │
│  doTraversal() → performTraversals()                            │
│         │                                                       │
│         ▼                                                       │
│  performMeasure() → performLayout() → performDraw()             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


### 2.8 View 生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                      View 生命周期                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  构造函数                                                        │
│  View(Context) / View(Context, AttributeSet)                    │
│         │                                                       │
│         ▼                                                       │
│  onFinishInflate()  ◄─── XML 解析完成后调用                      │
│         │                                                       │
│         ▼                                                       │
│  onAttachedToWindow()  ◄─── View 附加到 Window                   │
│         │                                                       │
│         │  此时可以:                                             │
│         │  - 注册监听器                                          │
│         │  - 开始动画                                            │
│         │  - 获取 ViewTreeObserver                              │
│         │                                                       │
│         ▼                                                       │
│  onMeasure() ◄─── 可能多次调用                                   │
│         │                                                       │
│         ▼                                                       │
│  onSizeChanged()  ◄─── 大小改变时调用                            │
│         │                                                       │
│         ▼                                                       │
│  onLayout() ◄─── 可能多次调用                                    │
│         │                                                       │
│         ▼                                                       │
│  onDraw() ◄─── 可能多次调用                                      │
│         │                                                       │
│         │  View 可见并可交互                                     │
│         │                                                       │
│         ▼                                                       │
│  onWindowFocusChanged(true)  ◄─── 获得焦点                       │
│         │                                                       │
│         │  ... View 正常使用 ...                                 │
│         │                                                       │
│         ▼                                                       │
│  onWindowFocusChanged(false)  ◄─── 失去焦点                      │
│         │                                                       │
│         ▼                                                       │
│  onDetachedFromWindow()  ◄─── View 从 Window 移除                │
│         │                                                       │
│         │  此时应该:                                             │
│         │  - 注销监听器                                          │
│         │  - 停止动画                                            │
│         │  - 释放资源                                            │
│         │                                                       │
│         ▼                                                       │
│  View 销毁                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 关键生命周期回调

| 回调方法 | 调用时机 | 典型用途 |
|---------|---------|---------|
| `onFinishInflate()` | XML 解析完成 | 获取子 View 引用 |
| `onAttachedToWindow()` | 附加到 Window | 注册监听、开始动画 |
| `onMeasure()` | 测量阶段 | 确定 View 大小 |
| `onSizeChanged()` | 大小改变 | 更新依赖大小的资源 |
| `onLayout()` | 布局阶段 | 确定子 View 位置 |
| `onDraw()` | 绘制阶段 | 绑定 View 内容 |
| `onWindowFocusChanged()` | 焦点改变 | 处理焦点相关逻辑 |
| `onDetachedFromWindow()` | 从 Window 移除 | 释放资源、停止动画 |

## 3. 关键源码解析

### 3.1 ViewRootImpl.performTraversals() 源码

这是 View 绑定流程的核心入口方法：

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
private void performTraversals() {
    // 1. 获取 DecorView
    final View host = mView;
    if (host == null || !mAdded) {
        return;
    }
    
    // 2. 标记正在遍历
    mIsInTraversal = true;
    mWillDrawSoon = true;
    
    boolean windowSizeMayChange = false;
    WindowManager.LayoutParams lp = mWindowAttributes;
    
    // 3. 计算 DecorView 的期望宽高
    int desiredWindowWidth;
    int desiredWindowHeight;
    
    // 4. 获取窗口大小
    final int viewVisibility = getHostVisibility();
    final boolean viewVisibilityChanged = !mFirst
            && (mViewVisibility != viewVisibility || mNewSurfaceNeeded);
    
    // ... 省略窗口大小计算逻辑 ...
    
    // 5. 判断是否需要重新测量
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        // 6. 计算根 View 的 MeasureSpec
        // 对于 DecorView，通常是 EXACTLY 模式，大小为窗口大小
        int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
        int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
        
        // 7. 执行测量
        // 这里会递归测量整个 View 树
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        
        // 8. 检查是否需要重新测量
        // 某些情况下需要二次测量（如 wrap_content 的 DecorView）
        int width = host.getMeasuredWidth();
        int height = host.getMeasuredHeight();
        boolean measureAgain = false;
        
        if (lp.horizontalWeight > 0.0f) {
            width += (int) ((mWidth - width) * lp.horizontalWeight);
            childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width, MeasureSpec.EXACTLY);
            measureAgain = true;
        }
        if (lp.verticalWeight > 0.0f) {
            height += (int) ((mHeight - height) * lp.verticalWeight);
            childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height, MeasureSpec.EXACTLY);
            measureAgain = true;
        }
        
        if (measureAgain) {
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
        
        layoutRequested = true;
    }
    
    // 9. 执行布局
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
        
        // 10. 处理透明区域（用于 SurfaceView 等）
        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            host.getLocationInWindow(mTmpLocation);
            mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                    mTmpLocation[0] + host.mRight - host.mLeft,
                    mTmpLocation[1] + host.mBottom - host.mTop);
            host.gatherTransparentRegion(mTransparentRegion);
            // ... 设置透明区域 ...
        }
    }
    
    // 11. 执行绘制
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    if (!cancelDraw) {
        // 12. 处理动画
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }
        
        // 13. 执行绘制
        performDraw();
    }
    
    // 14. 标记遍历结束
    mIsInTraversal = false;
}
```


### 3.2 View.measure() 源码

```java
// frameworks/base/core/java/android/view/View.java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // 1. 检查是否是光学边界布局
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int oWidth  = insets.left + insets.right;
        int oHeight = insets.top  + insets.bottom;
        widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
        heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
    }

    // 2. 计算测量规格的缓存 key
    // 用于判断是否需要重新测量
    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
    
    // 3. 初始化测量缓存
    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

    // 4. 判断是否需要强制布局
    // PFLAG_FORCE_LAYOUT 标志由 requestLayout() 设置
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

    // 5. 判断 MeasureSpec 是否改变
    final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
            || heightMeasureSpec != mOldHeightMeasureSpec;
    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
            && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
    final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
            && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
    
    // 6. 判断是否需要重新测量
    // 条件：强制布局 或 (规格改变 且 (不是精确模式 或 大小不匹配))
    final boolean needsLayout = specChanged
            && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

    if (forceLayout || needsLayout) {
        // 7. 清除测量缓存标志
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        // 8. 解析 RTL 属性
        resolveRtlPropertiesIfNeeded();

        // 9. 尝试从缓存获取测量结果
        int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
        
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // 10. 缓存未命中，执行实际测量
            // 这里调用 onMeasure()，子类重写此方法实现自定义测量
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            
            // 11. 检查是否调用了 setMeasuredDimension()
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            // 12. 缓存命中，直接使用缓存的测量结果
            long value = mMeasureCache.valueAt(cacheIndex);
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        // 13. 检查是否正确设置了测量尺寸
        // 如果 onMeasure() 没有调用 setMeasuredDimension()，抛出异常
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("View with id " + getId() + ": "
                    + getClass().getName() + "#onMeasure() did not set the"
                    + " measured dimension by calling setMeasuredDimension()");
        }

        // 14. 设置需要布局标志
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }

    // 15. 保存当前 MeasureSpec，用于下次比较
    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    // 16. 缓存测量结果
    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL);
}
```

### 3.3 View.onMeasure() 默认实现

```java
// frameworks/base/core/java/android/view/View.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 默认实现：使用 getDefaultSize() 计算大小
    // 子类应该重写此方法实现自定义测量逻辑
    setMeasuredDimension(
        getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec)
    );
}

/**
 * 获取默认大小
 * @param size 建议的最小大小（来自 background drawable 或 minWidth/minHeight）
 * @param measureSpec 父容器的测量规格
 */
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            // 父容器不限制，使用建议的最小大小
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            // 最大模式或精确模式，使用父容器指定的大小
            // 注意：这就是为什么直接继承 View 时 wrap_content 和 match_parent 效果相同
            result = specSize;
            break;
    }
    return result;
}

/**
 * 获取建议的最小宽度
 * 取 minWidth 属性和 background drawable 最小宽度的较大值
 */
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

### 3.4 ViewGroup.measureChildWithMargins() 源码

```java
// frameworks/base/core/java/android/view/ViewGroup.java
/**
 * 测量子 View，考虑 padding 和 margin
 * 这是 ViewGroup 测量子 View 的标准方法
 */
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    
    // 1. 获取子 View 的 LayoutParams
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    // 2. 计算子 View 的 MeasureSpec
    // 考虑父容器的 padding、已使用的空间、子 View 的 margin
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed,
            lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed,
            lp.height);

    // 3. 测量子 View
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

/**
 * 根据父容器的 MeasureSpec 和子 View 的 LayoutParams 计算子 View 的 MeasureSpec
 * 这是 MeasureSpec 生成规则的核心实现
 */
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    // 1. 解析父容器的测量模式和大小
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    // 2. 计算可用大小（父容器大小减去 padding）
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
        // 父容器是精确模式
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                // 子 View 指定了具体大小
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 子 View 想要填满父容器
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 子 View 想要自适应大小，但不能超过父容器
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // 父容器是最大模式
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // 子 View 指定了具体大小
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 子 View 想要填满父容器，但父容器大小不确定
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 子 View 想要自适应大小
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // 父容器不限制大小（如 ScrollView）
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // 子 View 指定了具体大小
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 子 View 想要填满父容器，但父容器大小不确定
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 子 View 想要自适应大小
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }
    
    // 3. 组合成 MeasureSpec 返回
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```


### 3.5 View.layout() 源码

```java
// frameworks/base/core/java/android/view/View.java
public void layout(int l, int t, int r, int b) {
    // 1. 检查是否需要在 layout 前重新测量
    // 这种情况发生在使用了测量缓存时
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    // 2. 保存旧的位置
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    // 3. 设置新的位置
    // setFrame() 或 setOpticalFrame() 会设置 mLeft, mTop, mRight, mBottom
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    // 4. 如果位置改变或需要布局，调用 onLayout
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        // 5. 调用 onLayout，子类重写此方法确定子 View 位置
        onLayout(changed, l, t, r, b);

        // 6. 通知 OnLayoutChangeListener
        if (shouldDrawRoundScrollbar()) {
            if(mRoundScrollbarRenderer == null) {
                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
            }
        } else {
            mRoundScrollbarRenderer = null;
        }

        // 7. 清除需要布局标志
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

        // 8. 通知布局改变监听器
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    // 9. 处理焦点变化
    final boolean wasLayoutValid = isLayoutValid();
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    
    // 10. 处理延迟的焦点请求
    if (!wasLayoutValid && isFocused()) {
        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
        if (canTakeFocus()) {
            clearParentsWantFocus();
        } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            clearParentsWantFocus();
        } else if (!hasParentWantsFocus()) {
            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
        }
    } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {
        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
        View focused = findFocus();
        if (focused != null) {
            if (!restoreDefaultFocus() && !hasParentWantsFocus()) {
                focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            }
        }
    }
}

/**
 * 设置 View 的四个顶点位置
 * @return 如果位置改变返回 true
 */
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;

    // 1. 检查位置是否改变
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;

        // 2. 计算绘制标志
        int drawn = mPrivateFlags & PFLAG_DRAWN;
        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

        // 3. 使旧区域无效（需要重绘）
        invalidate(sizeChanged);

        // 4. 设置新位置
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        
        // 5. 更新 RenderNode 的位置
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

        mPrivateFlags |= PFLAG_HAS_BOUNDS;

        // 6. 如果大小改变，调用 onSizeChanged
        if (sizeChanged) {
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);
        }

        // ... 省略无障碍相关代码 ...
    }
    return changed;
}
```

### 3.6 View.draw() 源码

```java
// frameworks/base/core/java/android/view/View.java
@CallSuper
public void draw(Canvas canvas) {
    // 1. 获取私有标志
    final int privateFlags = mPrivateFlags;
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     *      7. If necessary, draw the default focus highlight
     */

    // Step 1: 绘制背景
    int saveCount;
    drawBackground(canvas);

    // 2. 检查是否需要绘制渐变边缘
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    
    // 3. 如果不需要渐变边缘，走快速路径
    if (!verticalEdges && !horizontalEdges) {
        // Step 3: 绘制内容
        onDraw(canvas);

        // Step 4: 绘制子 View
        dispatchDraw(canvas);

        // Step 6: 绘制前景和装饰
        drawAutofilledHighlight(canvas);
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }
        onDrawForeground(canvas);

        // Step 7: 绘制默认焦点高亮
        drawDefaultFocusHighlight(canvas);
        
        return;
    }

    // 4. 需要绘制渐变边缘的完整路径
    // ... 省略渐变边缘绘制代码 ...
    
    // Step 2: 保存图层
    boolean drawTop = false;
    boolean drawBottom = false;
    boolean drawLeft = false;
    boolean drawRight = false;

    float topFadeStrength = 0.0f;
    float bottomFadeStrength = 0.0f;
    float leftFadeStrength = 0.0f;
    float rightFadeStrength = 0.0f;

    // ... 计算渐变强度 ...

    saveCount = canvas.getSaveCount();
    int solidColor = getSolidColor();
    if (solidColor == 0) {
        canvas.saveUnclippedLayer(left, top, right, bottom);
    } else {
        canvas.saveLayer(left, top, right, bottom, null);
    }

    // Step 3: 绘制内容
    onDraw(canvas);

    // Step 4: 绘制子 View
    dispatchDraw(canvas);

    // Step 5: 绘制渐变边缘
    final Paint p = scrollabilityCache.paint;
    final Matrix matrix = scrollabilityCache.matrix;
    final Shader fade = scrollabilityCache.shader;

    // ... 绘制四个方向的渐变边缘 ...

    canvas.restoreToCount(saveCount);

    // Step 6: 绘制前景和装饰
    drawAutofilledHighlight(canvas);
    if (mOverlay != null && !mOverlay.isEmpty()) {
        mOverlay.getOverlayView().dispatchDraw(canvas);
    }
    onDrawForeground(canvas);

    // Step 7: 绘制默认焦点高亮
    drawDefaultFocusHighlight(canvas);
}
```

### 3.7 invalidate() 源码

```java
// frameworks/base/core/java/android/view/View.java
public void invalidate() {
    invalidate(true);
}

public void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}

void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    // 1. 检查是否跳过无效化
    // 如果 View 不可见且没有动画，跳过
    if (skipInvalidate()) {
        return;
    }

    // 2. 检查是否需要重绘
    // PFLAG_DRAWN: View 已绘制
    // PFLAG_HAS_BOUNDS: View 有边界
    // PFLAG_DRAWING_CACHE_VALID: 绘制缓存有效
    // PFLAG_INVALIDATED: 已标记无效
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        
        // 3. 处理完全无效化
        if (fullInvalidate) {
            mLastIsOpaque = isOpaque();
            mPrivateFlags &= ~PFLAG_DRAWN;
        }

        // 4. 设置脏标志
        mPrivateFlags |= PFLAG_DIRTY;

        // 5. 清除绘制缓存
        if (invalidateCache) {
            mPrivateFlags |= PFLAG_INVALIDATED;
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }

        // 6. 向上传递无效区域到父 View
        // 最终传递到 ViewRootImpl
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);
            // 调用父 View 的 invalidateChild
            p.invalidateChild(this, damage);
        }

        // 7. 通知投影无效（用于阴影等）
        if (mBackground != null && mBackground.isProjected()) {
            final View receiver = getProjectionReceiver();
            if (receiver != null) {
                receiver.damageInParent();
            }
        }
    }
}
```


## 4. 实战应用

### 4.1 自定义 View 实现

#### 基本模板

```kotlin
class CustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    // 画笔
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.BLUE
        style = Paint.Style.FILL
    }
    
    // 自定义属性
    private var customColor: Int = Color.BLUE
    private var customRadius: Float = 50f
    
    init {
        // 解析自定义属性
        context.obtainStyledAttributes(attrs, R.styleable.CustomView).apply {
            customColor = getColor(R.styleable.CustomView_customColor, Color.BLUE)
            customRadius = getDimension(R.styleable.CustomView_customRadius, 50f)
            recycle()
        }
        paint.color = customColor
    }

    /**
     * 测量 - 确定 View 大小
     * 重点：正确处理 wrap_content
     */
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        // 计算期望的宽高
        val desiredWidth = (customRadius * 2 + paddingLeft + paddingRight).toInt()
        val desiredHeight = (customRadius * 2 + paddingTop + paddingBottom).toInt()
        
        // 根据 MeasureSpec 确定最终大小
        val width = resolveSize(desiredWidth, widthMeasureSpec)
        val height = resolveSize(desiredHeight, heightMeasureSpec)
        
        setMeasuredDimension(width, height)
    }
    
    /**
     * 大小改变回调
     * 适合在这里初始化依赖大小的资源
     */
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        // 更新依赖大小的资源，如 Shader、Path 等
    }

    /**
     * 绑定 - 绘制 View 内容
     */
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 计算圆心位置（考虑 padding）
        val cx = (width - paddingLeft - paddingRight) / 2f + paddingLeft
        val cy = (height - paddingTop - paddingBottom) / 2f + paddingTop
        
        // 绘制圆形
        canvas.drawCircle(cx, cy, customRadius, paint)
    }
    
    /**
     * 附加到窗口
     */
    override fun onAttachedToWindow() {
        super.onAttachedToWindow()
        // 注册监听器、开始动画等
    }
    
    /**
     * 从窗口分离
     */
    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        // 注销监听器、停止动画、释放资源
    }
    
    /**
     * 更新颜色
     */
    fun setColor(color: Int) {
        if (customColor != color) {
            customColor = color
            paint.color = color
            invalidate() // 只需要重绘，不需要重新测量
        }
    }
    
    /**
     * 更新半径
     */
    fun setRadius(radius: Float) {
        if (customRadius != radius) {
            customRadius = radius
            requestLayout() // 大小改变，需要重新测量和布局
        }
    }
}
```

#### resolveSize() 方法解析

```kotlin
/**
 * 根据 MeasureSpec 解析最终大小
 * 这是处理 wrap_content 的标准方法
 */
fun resolveSize(desiredSize: Int, measureSpec: Int): Int {
    val specMode = MeasureSpec.getMode(measureSpec)
    val specSize = MeasureSpec.getSize(measureSpec)
    
    return when (specMode) {
        MeasureSpec.EXACTLY -> {
            // 精确模式：必须使用指定大小
            specSize
        }
        MeasureSpec.AT_MOST -> {
            // 最大模式：取期望大小和最大大小的较小值
            minOf(desiredSize, specSize)
        }
        else -> {
            // UNSPECIFIED：使用期望大小
            desiredSize
        }
    }
}
```

### 4.2 自定义 ViewGroup 实现

```kotlin
/**
 * 自定义流式布局 ViewGroup
 * 子 View 从左到右排列，超出宽度自动换行
 */
class FlowLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : ViewGroup(context, attrs, defStyleAttr) {

    // 水平和垂直间距
    private var horizontalSpacing = 8.dp
    private var verticalSpacing = 8.dp
    
    // 每行的高度列表，用于 layout
    private val lineHeights = mutableListOf<Int>()

    /**
     * 测量 - 确定 ViewGroup 和所有子 View 的大小
     */
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
        val widthSize = MeasureSpec.getSize(widthMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)
        val heightSize = MeasureSpec.getSize(heightMeasureSpec)
        
        // 可用宽度（减去 padding）
        val availableWidth = widthSize - paddingLeft - paddingRight
        
        var currentLineWidth = 0
        var currentLineHeight = 0
        var totalWidth = 0
        var totalHeight = 0
        
        lineHeights.clear()
        
        // 遍历测量所有子 View
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.visibility == GONE) continue
            
            // 测量子 View
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0)
            
            val lp = child.layoutParams as MarginLayoutParams
            val childWidth = child.measuredWidth + lp.leftMargin + lp.rightMargin
            val childHeight = child.measuredHeight + lp.topMargin + lp.bottomMargin
            
            // 检查是否需要换行
            if (currentLineWidth + childWidth > availableWidth && currentLineWidth > 0) {
                // 换行
                totalWidth = maxOf(totalWidth, currentLineWidth)
                totalHeight += currentLineHeight + verticalSpacing
                lineHeights.add(currentLineHeight)
                
                currentLineWidth = childWidth
                currentLineHeight = childHeight
            } else {
                // 同一行
                currentLineWidth += childWidth + if (currentLineWidth > 0) horizontalSpacing else 0
                currentLineHeight = maxOf(currentLineHeight, childHeight)
            }
        }
        
        // 处理最后一行
        totalWidth = maxOf(totalWidth, currentLineWidth)
        totalHeight += currentLineHeight
        lineHeights.add(currentLineHeight)
        
        // 加上 padding
        totalWidth += paddingLeft + paddingRight
        totalHeight += paddingTop + paddingBottom
        
        // 根据 MeasureSpec 确定最终大小
        val measuredWidth = when (widthMode) {
            MeasureSpec.EXACTLY -> widthSize
            MeasureSpec.AT_MOST -> minOf(totalWidth, widthSize)
            else -> totalWidth
        }
        
        val measuredHeight = when (heightMode) {
            MeasureSpec.EXACTLY -> heightSize
            MeasureSpec.AT_MOST -> minOf(totalHeight, heightSize)
            else -> totalHeight
        }
        
        setMeasuredDimension(measuredWidth, measuredHeight)
    }

    /**
     * 布局 - 确定所有子 View 的位置
     */
    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        val availableWidth = width - paddingLeft - paddingRight
        
        var currentLeft = paddingLeft
        var currentTop = paddingTop
        var lineIndex = 0
        
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.visibility == GONE) continue
            
            val lp = child.layoutParams as MarginLayoutParams
            val childWidth = child.measuredWidth
            val childHeight = child.measuredHeight
            val totalChildWidth = childWidth + lp.leftMargin + lp.rightMargin
            
            // 检查是否需要换行
            if (currentLeft + totalChildWidth > paddingLeft + availableWidth && 
                currentLeft > paddingLeft) {
                // 换行
                currentLeft = paddingLeft
                currentTop += lineHeights[lineIndex] + verticalSpacing
                lineIndex++
            }
            
            // 计算子 View 位置
            val childLeft = currentLeft + lp.leftMargin
            val childTop = currentTop + lp.topMargin
            val childRight = childLeft + childWidth
            val childBottom = childTop + childHeight
            
            // 布局子 View
            child.layout(childLeft, childTop, childRight, childBottom)
            
            // 更新当前位置
            currentLeft += totalChildWidth + horizontalSpacing
        }
    }

    /**
     * 生成默认的 LayoutParams
     */
    override fun generateDefaultLayoutParams(): LayoutParams {
        return MarginLayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT)
    }

    override fun generateLayoutParams(attrs: AttributeSet?): LayoutParams {
        return MarginLayoutParams(context, attrs)
    }

    override fun generateLayoutParams(p: LayoutParams?): LayoutParams {
        return MarginLayoutParams(p)
    }

    override fun checkLayoutParams(p: LayoutParams?): Boolean {
        return p is MarginLayoutParams
    }
    
    private val Int.dp: Int
        get() = (this * resources.displayMetrics.density).toInt()
}
```


### 4.3 性能优化最佳实践

#### 1. 减少布局层级

```kotlin
// ❌ 不推荐：多层嵌套
<LinearLayout>
    <LinearLayout>
        <LinearLayout>
            <TextView />
            <ImageView />
        </LinearLayout>
    </LinearLayout>
</LinearLayout>

// ✅ 推荐：使用 ConstraintLayout 扁平化
<ConstraintLayout>
    <TextView />
    <ImageView />
</ConstraintLayout>
```

#### 2. 避免过度绘制

```kotlin
// ❌ 不推荐：重复设置背景
<FrameLayout android:background="@color/white">
    <LinearLayout android:background="@color/white">
        <TextView android:background="@color/white" />
    </LinearLayout>
</FrameLayout>

// ✅ 推荐：只在必要的地方设置背景
<FrameLayout android:background="@color/white">
    <LinearLayout>
        <TextView />
    </LinearLayout>
</FrameLayout>
```

#### 3. 使用 ViewStub 延迟加载

```kotlin
// 布局中定义 ViewStub
<ViewStub
    android:id="@+id/stub_error"
    android:layout="@layout/layout_error"
    android:inflatedId="@+id/error_view" />

// 需要时才加载
val errorView = findViewById<ViewStub>(R.id.stub_error)?.inflate()
// 或
val errorView = findViewById<View>(R.id.error_view) // inflate 后可直接获取
```

#### 4. 使用 merge 减少层级

```xml
<!-- layout_merge.xml -->
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    <TextView android:id="@+id/text1" />
    <TextView android:id="@+id/text2" />
</merge>

<!-- 使用 include -->
<LinearLayout>
    <include layout="@layout/layout_merge" />
</LinearLayout>
```

#### 5. 优化 onDraw()

```kotlin
class OptimizedView(context: Context) : View(context) {
    
    // ✅ 在构造函数或 onSizeChanged 中创建对象
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val rect = RectF()
    private val path = Path()
    
    override fun onDraw(canvas: Canvas) {
        // ❌ 不要在 onDraw 中创建对象
        // val paint = Paint() // 每次绘制都创建新对象
        
        // ✅ 复用已创建的对象
        rect.set(0f, 0f, width.toFloat(), height.toFloat())
        canvas.drawRect(rect, paint)
    }
}
```

#### 6. 合理使用 invalidate

```kotlin
class AnimatedView(context: Context) : View(context) {
    
    private var progress = 0f
    
    fun setProgress(value: Float) {
        if (progress != value) {
            progress = value
            // ✅ 只重绘变化的区域
            invalidate(dirtyRect)
            // 而不是 invalidate() 重绘整个 View
        }
    }
    
    // ❌ 避免频繁调用 requestLayout
    fun updateSize(newSize: Int) {
        // 如果只是视觉变化，使用 invalidate
        // 只有真正需要重新测量时才用 requestLayout
    }
}
```

### 4.4 常见问题与解决方案

#### 问题1：getMeasuredWidth() 返回 0

```kotlin
// ❌ 问题：在 onCreate 中获取宽高
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    
    val width = textView.measuredWidth // 返回 0
}

// ✅ 解决方案1：使用 ViewTreeObserver
textView.viewTreeObserver.addOnGlobalLayoutListener(object : OnGlobalLayoutListener {
    override fun onGlobalLayout() {
        textView.viewTreeObserver.removeOnGlobalLayoutListener(this)
        val width = textView.measuredWidth // 正确获取
    }
})

// ✅ 解决方案2：使用 post
textView.post {
    val width = textView.measuredWidth // 正确获取
}

// ✅ 解决方案3：手动测量
textView.measure(
    MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED),
    MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED)
)
val width = textView.measuredWidth
```

#### 问题2：wrap_content 不生效

```kotlin
// ❌ 问题：直接继承 View，wrap_content 和 match_parent 效果相同
class MyView(context: Context) : View(context) {
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec) // 默认实现不处理 wrap_content
    }
}

// ✅ 解决方案：正确处理 wrap_content
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    val desiredWidth = calculateDesiredWidth()
    val desiredHeight = calculateDesiredHeight()
    
    val width = resolveSize(desiredWidth, widthMeasureSpec)
    val height = resolveSize(desiredHeight, heightMeasureSpec)
    
    setMeasuredDimension(width, height)
}
```

#### 问题3：requestLayout 无效

```kotlin
// ❌ 问题：在 onLayout 中调用 requestLayout
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    // 某些条件下调用 requestLayout
    if (someCondition) {
        requestLayout() // 可能被忽略
    }
}

// ✅ 解决方案：使用 post 延迟调用
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    if (someCondition) {
        post { requestLayout() }
    }
}
```


## 5. 常见面试题

### 问题1：View 的绑定流程是怎样的？measure、layout、draw 各自的作用是什么？

**答案要点：**

View 的绑定流程由 `ViewRootImpl.performTraversals()` 触发，包含三个核心阶段：

1. **Measure（测量）**
   - 作用：确定 View 的测量宽高（measuredWidth/measuredHeight）
   - 入口：`View.measure()` → `View.onMeasure()`
   - 核心：根据父容器的 MeasureSpec 和自身的 LayoutParams 计算大小
   - 结果：通过 `setMeasuredDimension()` 保存测量结果

2. **Layout（布局）**
   - 作用：确定 View 的最终位置（left、top、right、bottom）
   - 入口：`View.layout()` → `View.onLayout()`
   - 核心：ViewGroup 需要重写 `onLayout()` 确定子 View 位置
   - 结果：通过 `setFrame()` 设置四个顶点坐标

3. **Draw（绘制）**
   - 作用：将 View 绘制到屏幕上
   - 入口：`View.draw()` → `View.onDraw()`
   - 六个步骤：背景 → 保存图层 → 内容 → 子 View → 渐变边缘 → 前景装饰
   - 硬件加速：使用 DisplayList 和 RenderThread 提升性能

**流程图：**
```
performTraversals()
    ├── performMeasure() → measure() → onMeasure()
    ├── performLayout() → layout() → onLayout()
    └── performDraw() → draw() → onDraw()
```

---

### 问题2：MeasureSpec 是什么？三种模式分别代表什么含义？

**答案要点：**

MeasureSpec 是一个 32 位的 int 值，封装了父容器对子 View 的测量要求：
- **高 2 位**：SpecMode（测量模式）
- **低 30 位**：SpecSize（测量大小）

**三种模式：**

| 模式 | 值 | 含义 | 对应 LayoutParams |
|------|---|------|------------------|
| **UNSPECIFIED** | 0 | 父容器不限制子 View 大小，子 View 可以是任意大小 | 系统内部使用（ScrollView 测量子 View） |
| **EXACTLY** | 1 | 精确模式，子 View 必须是指定大小 | match_parent 或具体 dp 值 |
| **AT_MOST** | 2 | 最大模式，子 View 不能超过指定大小 | wrap_content |

**MeasureSpec 生成规则（重点）：**
- 子 View 的 MeasureSpec 由父容器的 MeasureSpec 和子 View 的 LayoutParams 共同决定
- 核心方法：`ViewGroup.getChildMeasureSpec()`
- 例如：父 EXACTLY + 子 wrap_content = 子 AT_MOST

---

### 问题3：requestLayout() 和 invalidate() 的区别是什么？分别在什么场景使用？

**答案要点：**

| 方法 | 触发流程 | 使用场景 |
|------|---------|---------|
| `requestLayout()` | measure → layout → draw | View 的大小或位置需要改变 |
| `invalidate()` | draw | View 的内容需要重绘，但大小不变 |
| `postInvalidate()` | draw（可在子线程调用） | 同 invalidate，线程安全版本 |

**requestLayout() 流程：**
1. 设置 `PFLAG_FORCE_LAYOUT` 标志
2. 向上传递到父 View，直到 ViewRootImpl
3. ViewRootImpl 调用 `scheduleTraversals()`
4. 等待 VSYNC 信号后执行 `performTraversals()`

**invalidate() 流程：**
1. 设置 `PFLAG_DIRTY` 标志
2. 向上传递脏区域到 ViewRootImpl
3. 只触发 draw 流程，不重新测量和布局

**使用建议：**
- 改变背景颜色、文字颜色等 → `invalidate()`
- 改变文字内容导致大小变化 → `requestLayout()`
- 动画中频繁更新 → 尽量使用 `invalidate()` 避免重新测量

---

### 问题4：自定义 View 时如何正确处理 wrap_content？

**答案要点：**

**问题原因：**
View 的默认 `onMeasure()` 实现使用 `getDefaultSize()`，对于 AT_MOST 和 EXACTLY 模式都返回 specSize，导致 wrap_content 和 match_parent 效果相同。

**解决方案：**
```kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    // 1. 计算期望的宽高（根据内容）
    val desiredWidth = calculateContentWidth() + paddingLeft + paddingRight
    val desiredHeight = calculateContentHeight() + paddingTop + paddingBottom
    
    // 2. 使用 resolveSize 处理 MeasureSpec
    val width = resolveSize(desiredWidth, widthMeasureSpec)
    val height = resolveSize(desiredHeight, heightMeasureSpec)
    
    // 3. 设置测量结果
    setMeasuredDimension(width, height)
}
```

**resolveSize() 逻辑：**
- EXACTLY：返回 specSize
- AT_MOST：返回 min(desiredSize, specSize)
- UNSPECIFIED：返回 desiredSize

---

### 问题5：View 的 getWidth() 和 getMeasuredWidth() 有什么区别？

**答案要点：**

| 方法 | 计算方式 | 获取时机 | 含义 |
|------|---------|---------|------|
| `getMeasuredWidth()` | `mMeasuredWidth` | measure 后 | 测量宽度 |
| `getWidth()` | `mRight - mLeft` | layout 后 | 实际宽度 |

**关键区别：**
1. **时机不同**：getMeasuredWidth 在 measure 后有效，getWidth 在 layout 后有效
2. **值可能不同**：通常相等，但在某些情况下可能不同
   - 例如：在 `onLayout()` 中手动设置了不同的位置
   - 例如：使用了 `setFrame()` 设置了不同的边界

**常见问题：**
```kotlin
// ❌ 在 onCreate 中获取，返回 0
val width = view.width

// ✅ 使用 ViewTreeObserver 或 post
view.post {
    val width = view.width // 正确获取
}
```

---

### 问题6：View 的绘制流程中，硬件加速和软件绘制有什么区别？

**答案要点：**

| 特性 | 软件绘制 | 硬件加速 |
|------|---------|---------|
| 渲染线程 | 主线程 (UI Thread) | RenderThread |
| 绘制 API | Skia (CPU) | OpenGL ES / Vulkan (GPU) |
| 绘制对象 | Bitmap | DisplayList (RenderNode) |
| invalidate | 重绘整个 View 树 | 只重绘脏区域 |
| 内存占用 | 较低 | 较高（GPU 缓存） |
| 动画性能 | 较差 | 优秀 |

**硬件加速优势：**
1. 使用 GPU 渲染，性能更好
2. RenderThread 独立于主线程，不阻塞 UI
3. DisplayList 可以缓存绘制命令，避免重复绘制
4. 支持更高效的动画和变换

**不支持硬件加速的操作：**
- `Canvas.drawBitmapMesh()`
- `Canvas.drawPicture()`
- 部分 `Paint.setMaskFilter()`

**禁用硬件加速：**
```kotlin
// View 级别
view.setLayerType(View.LAYER_TYPE_SOFTWARE, null)

// Activity 级别
<activity android:hardwareAccelerated="false" />
```

---

### 问题7：自定义 ViewGroup 时需要注意什么？onMeasure 和 onLayout 分别要做什么？

**答案要点：**

**onMeasure() 职责：**
1. 遍历测量所有子 View（调用 `measureChild()` 或 `measureChildWithMargins()`）
2. 根据子 View 的测量结果计算自身大小
3. 调用 `setMeasuredDimension()` 设置测量结果

```kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    var totalWidth = 0
    var totalHeight = 0
    
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        if (child.visibility != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0)
            totalWidth += child.measuredWidth
            totalHeight = maxOf(totalHeight, child.measuredHeight)
        }
    }
    
    setMeasuredDimension(
        resolveSize(totalWidth + paddingLeft + paddingRight, widthMeasureSpec),
        resolveSize(totalHeight + paddingTop + paddingBottom, heightMeasureSpec)
    )
}
```

**onLayout() 职责：**
1. 根据测量结果确定每个子 View 的位置
2. 调用 `child.layout(l, t, r, b)` 布局每个子 View

```kotlin
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    var currentLeft = paddingLeft
    
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        if (child.visibility != GONE) {
            val lp = child.layoutParams as MarginLayoutParams
            child.layout(
                currentLeft + lp.leftMargin,
                paddingTop + lp.topMargin,
                currentLeft + lp.leftMargin + child.measuredWidth,
                paddingTop + lp.topMargin + child.measuredHeight
            )
            currentLeft += child.measuredWidth + lp.leftMargin + lp.rightMargin
        }
    }
}
```

**注意事项：**
1. 必须重写 `generateLayoutParams()` 系列方法
2. 正确处理 padding 和 margin
3. 处理子 View 的 GONE 状态
4. 考虑 RTL 布局方向

---

### 问题8：View 的生命周期有哪些回调？各自的调用时机是什么？

**答案要点：**

| 回调方法 | 调用时机 | 典型用途 |
|---------|---------|---------|
| 构造函数 | View 创建时 | 初始化属性、解析自定义属性 |
| `onFinishInflate()` | XML 解析完成后 | 获取子 View 引用 |
| `onAttachedToWindow()` | View 附加到 Window | 注册监听器、开始动画 |
| `onMeasure()` | 测量阶段（可能多次） | 确定 View 大小 |
| `onSizeChanged()` | 大小改变时 | 更新依赖大小的资源 |
| `onLayout()` | 布局阶段（可能多次） | 确定子 View 位置 |
| `onDraw()` | 绘制阶段（可能多次） | 绘制 View 内容 |
| `onWindowFocusChanged()` | 窗口焦点改变 | 处理焦点相关逻辑 |
| `onDetachedFromWindow()` | View 从 Window 移除 | 释放资源、停止动画 |

**重要时机：**
- `onAttachedToWindow()`：此时 View 已有 ViewRootImpl，可以安全地使用 Handler
- `onSizeChanged()`：适合初始化依赖大小的资源（如 Shader、Path）
- `onDetachedFromWindow()`：必须释放资源，避免内存泄漏

---

### 问题9：为什么 View.post() 可以获取到 View 的宽高？

**答案要点：**

**原因分析：**
1. `View.post()` 将 Runnable 添加到消息队列
2. 在 `performTraversals()` 执行完成后，Runnable 才会执行
3. 此时 measure 和 layout 已完成，宽高已确定

**源码分析：**
```java
// View.java
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        // View 已附加到 Window，直接 post 到 Handler
        return attachInfo.mHandler.post(action);
    }
    // View 未附加，添加到待执行队列
    getRunQueue().post(action);
    return true;
}
```

**执行时机：**
```
ActivityThread.handleResumeActivity()
    → WindowManager.addView()
    → ViewRootImpl.setView()
    → ViewRootImpl.requestLayout()
    → scheduleTraversals()
    → [VSYNC 信号]
    → performTraversals() (measure + layout + draw)
    → [消息队列继续执行]
    → post 的 Runnable 执行 (此时宽高已确定)
```

---

### 问题10：DecorView 是什么？View 树的结构是怎样的？（OPPO/vivo 重点）

**答案要点：**

**DecorView：**
- 是 FrameLayout 的子类
- 是 Window 的顶层 View
- 包含标题栏（ActionBar）和内容区域（ContentParent）
- `setContentView()` 的布局被添加到 ContentParent（id 为 `android.R.id.content`）

**View 树结构：**
```
PhoneWindow
    └── DecorView (FrameLayout)
            └── LinearLayout
                    ├── ActionBar (标题栏)
                    └── ContentParent (FrameLayout, id=android.R.id.content)
                            └── 用户布局 (setContentView 设置的)
                                    ├── ViewGroup
                                    │       ├── View
                                    │       └── View
                                    └── View
```

**关键类关系：**
- **Activity** 持有 **PhoneWindow**
- **PhoneWindow** 持有 **DecorView**
- **ViewRootImpl** 管理 **DecorView**，连接 WindowManager
- **WindowManager** 负责添加、更新、删除 View

**源码路径（OPPO/vivo 面试重点）：**
- DecorView：`frameworks/base/core/java/com/android/internal/policy/DecorView.java`
- PhoneWindow：`frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java`
- ViewRootImpl：`frameworks/base/core/java/android/view/ViewRootImpl.java`

---

### 问题11：Choreographer 的作用是什么？VSYNC 信号如何触发绘制？（字节/快手重点）

**答案要点：**

**Choreographer（编舞者）：**
- 协调动画、输入和绘制的时机
- 接收 VSYNC 信号，在合适的时机触发绘制
- 保证绘制与屏幕刷新同步，避免撕裂

**VSYNC 触发绘制流程：**
```
1. View.requestLayout() / invalidate()
       ↓
2. ViewRootImpl.scheduleTraversals()
       ↓
3. 设置同步屏障 (MessageQueue.postSyncBarrier)
       ↓
4. Choreographer.postCallback(CALLBACK_TRAVERSAL, mTraversalRunnable)
       ↓
5. 等待 VSYNC 信号 (通过 native 层 DisplayEventReceiver)
       ↓
6. VSYNC 到来，执行 doFrame()
       ↓
7. 按顺序执行回调：INPUT → ANIMATION → TRAVERSAL → COMMIT
       ↓
8. TRAVERSAL 回调执行 doTraversal()
       ↓
9. 移除同步屏障，执行 performTraversals()
```

**同步屏障的作用：**
- 阻止同步消息执行，优先处理异步消息
- 确保 VSYNC 回调（异步消息）优先执行
- 保证绘制的及时性

---

### 问题12：如何优化自定义 View 的性能？（美团/字节重点）

**答案要点：**

**1. onDraw() 优化：**
```kotlin
// ❌ 避免在 onDraw 中创建对象
override fun onDraw(canvas: Canvas) {
    val paint = Paint() // 每次绘制都创建
}

// ✅ 在构造函数或 onSizeChanged 中创建
private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
```

**2. 减少 invalidate 范围：**
```kotlin
// ✅ 只重绘脏区域
invalidate(dirtyLeft, dirtyTop, dirtyRight, dirtyBottom)
```

**3. 使用硬件加速：**
```kotlin
// 对于复杂绘制，使用硬件层缓存
setLayerType(LAYER_TYPE_HARDWARE, null)
```

**4. 避免过度测量：**
```kotlin
// ✅ 使用测量缓存
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    if (measureSpecChanged) {
        // 只在必要时重新计算
    }
}
```

**5. 使用 ViewStub 延迟加载：**
```xml
<ViewStub android:id="@+id/stub" android:layout="@layout/heavy_layout" />
```

**6. 减少布局层级：**
- 使用 ConstraintLayout 扁平化布局
- 使用 merge 标签减少层级
- 避免不必要的嵌套

---

*文档版本: v1.0*  
*更新时间: 2025-01*  
*目标版本: Android W (API 35)*

