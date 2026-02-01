# Handler 机制

## 1. 概述

Handler 机制是 Android 消息驱动模型的核心，用于线程间通信和异步消息处理。

### 核心组件

| 组件 | 职责 | 关键方法 |
|------|------|----------|
| Handler | 发送和处理消息 | sendMessage()、handleMessage() |
| Looper | 消息循环，从队列取消息 | prepare()、loop() |
| MessageQueue | 消息队列，按时间排序 | enqueueMessage()、next() |
| Message | 消息载体 | obtain()、recycle() |

### 工作流程

```
┌─────────────────────────────────────────────────────────────┐
│                      Handler 机制工作流程                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────┐    sendMessage()    ┌──────────────────┐    │
│   │ Handler  │ ──────────────────► │  MessageQueue    │    │
│   └──────────┘                     │  (按时间排序)      │    │
│        ▲                           └────────┬─────────┘    │
│        │                                    │              │
│        │ dispatchMessage()                  │ next()       │
│        │                                    ▼              │
│        │                           ┌──────────────────┐    │
│        └─────────────────────────  │     Looper       │    │
│                                    │   (消息循环)       │    │
│                                    └──────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 核心原理

### 2.1 Looper 原理

Looper 负责创建消息循环，不断从 MessageQueue 中取出消息并分发。

#### ThreadLocal 存储 Looper

```java
// Looper.java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static Looper sMainLooper;  // 主线程 Looper

// 每个线程只能有一个 Looper
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    // 一个线程只能调用一次 prepare
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    // 创建 Looper 并存入 ThreadLocal
    sThreadLocal.set(new Looper(quitAllowed));
}

// 主线程 Looper 初始化（系统调用）
public static void prepareMainLooper() {
    prepare(false);  // 主线程 Looper 不允许退出
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

// 获取当前线程的 Looper
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

#### Looper 构造函数

```java
private Looper(boolean quitAllowed) {
    // 创建 MessageQueue
    mQueue = new MessageQueue(quitAllowed);
    // 记录当前线程
    mThread = Thread.currentThread();
}
```

### 2.2 消息循环 loop()

```java
// Looper.java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    
    final MessageQueue queue = me.mQueue;
    
    // 确保线程身份是本地进程
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    
    // 无限循环
    for (;;) {
        // 从消息队列取消息，可能阻塞
        Message msg = queue.next();
        if (msg == null) {
            // 消息队列退出，返回
            return;
        }
        
        // 分发消息给 Handler 处理
        // msg.target 就是发送消息的 Handler
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            // ...
        }
        
        // 回收消息到消息池
        msg.recycleUnchecked();
    }
}
```

### 2.3 MessageQueue 原理

MessageQueue 是一个按消息执行时间排序的优先级队列，底层使用单链表实现。

#### 消息入队 enqueueMessage()

```java
// MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
    // Handler 不能为空
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    
    synchronized (this) {
        // 消息已被使用
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
        
        // 队列已退出
        if (mQuitting) {
            msg.recycle();
            return false;
        }
        
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;  // 队列头
        boolean needWake;
        
        // 插入队列头部的情况：
        // 1. 队列为空
        // 2. 新消息执行时间为0（立即执行）
        // 3. 新消息执行时间早于队列头
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;  // 如果队列阻塞，需要唤醒
        } else {
            // 按时间顺序插入队列中间
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
            }
            msg.next = p;
            prev.next = msg;
        }
        
        // 唤醒队列
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

#### 消息出队 next()

```java
// MessageQueue.java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;  // 队列已销毁
    }
    
    int pendingIdleHandlerCount = -1;
    int nextPollTimeoutMillis = 0;
    
    for (;;) {
        // 阻塞等待，-1 表示无限等待，0 表示不等待
        nativePollOnce(ptr, nextPollTimeoutMillis);
        
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            
            // 处理同步屏障：找到下一个异步消息
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            
            if (msg != null) {
                if (now < msg.when) {
                    // 消息还没到执行时间，计算等待时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 取出消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有消息，无限等待
                nextPollTimeoutMillis = -1;
            }
            
            // 队列退出
            if (mQuitting) {
                dispose();
                return null;
            }
            
            // 处理 IdleHandler
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                mBlocked = true;
                continue;
            }
            
            // 执行 IdleHandler
            // ...
        }
    }
}
```

### 2.4 Message 复用池

Message 使用享元模式，通过对象池复用消息对象，避免频繁创建销毁。

```java
// Message.java
public final class Message implements Parcelable {
    // 消息池（单链表）
    private static Message sPool;
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50;
    
    // 获取消息（优先从池中获取）
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0;  // 清除使用标记
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    // 回收消息到池中
    void recycleUnchecked() {
        // 清除所有状态
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;
        
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
}
```

### 2.5 同步屏障

同步屏障用于优先处理异步消息，常用于 UI 渲染等高优先级任务。

```java
// MessageQueue.java
// 发送同步屏障（target 为 null 的消息）
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        // 创建屏障消息，target 为 null
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        
        // 按时间插入队列
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) {
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}

// 移除同步屏障
public void removeSyncBarrier(int token) {
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        // 找到屏障消息
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        // 移除屏障
        if (prev != null) {
            prev.next = p.next;
        } else {
            mMessages = p.next;
        }
        p.recycleUnchecked();
        
        // 唤醒队列
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

#### 同步屏障在 UI 渲染中的应用

```java
// ViewRootImpl.java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 发送同步屏障，确保 UI 渲染优先执行
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 发送异步消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}

void unscheduleTraversals() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        mChoreographer.removeCallbacks(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}
```

### 2.6 IdleHandler

IdleHandler 在消息队列空闲时执行，适合执行低优先级任务。

```java
// MessageQueue.java
public static interface IdleHandler {
    // 返回 true 保持活跃，返回 false 执行后移除
    boolean queueIdle();
}

public void addIdleHandler(@NonNull IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}

public void removeIdleHandler(@NonNull IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.remove(handler);
    }
}
```

#### IdleHandler 使用场景

```kotlin
// 延迟初始化
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // 主线程空闲时执行初始化
        Looper.myQueue().addIdleHandler {
            // 执行非紧急初始化
            initAnalytics()
            initPush()
            false  // 只执行一次
        }
    }
}

// Activity 首帧绘制完成后执行
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        Looper.myQueue().addIdleHandler {
            // 首帧绘制完成，执行预加载
            preloadData()
            false
        }
    }
}
```

## 3. 关键源码解析

### 3.1 Handler 消息发送

```java
// Handler.java
public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    // 设置 target 为当前 Handler
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();
    
    // 如果是异步 Handler，设置消息为异步
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

### 3.2 Handler 消息处理

```java
// Handler.java
public void dispatchMessage(@NonNull Message msg) {
    // 优先级1：Message 的 callback（post(Runnable) 方式）
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        // 优先级2：Handler 的 mCallback
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 优先级3：Handler 的 handleMessage 方法
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}

// 子类重写此方法处理消息
public void handleMessage(@NonNull Message msg) {
}
```

### 3.3 Native 层实现

MessageQueue 的阻塞和唤醒依赖 Native 层的 epoll 机制。

```cpp
// android_os_MessageQueue.cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}

// Looper.cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        // 处理响应
        while (mResponseIndex < mResponses.size()) {
            // ...
        }
        if (result != 0) {
            return result;
        }
        // 调用 pollInner
        result = pollInner(timeoutMillis);
    }
}

int Looper::pollInner(int timeoutMillis) {
    // 使用 epoll_wait 等待事件
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    
    // 处理事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        
        if (fd == mWakeEventFd.get()) {
            if (epollEvents & EPOLLIN) {
                // 读取唤醒事件
                awoken();
            }
        }
    }
    return result;
}

// 唤醒
void Looper::wake() {
    uint64_t inc = 1;
    // 向 eventfd 写入数据，触发 epoll_wait 返回
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
}
```

## 4. 实战应用

### 4.1 HandlerThread 使用

```kotlin
class HandlerThreadExample {
    private lateinit var handlerThread: HandlerThread
    private lateinit var backgroundHandler: Handler
    
    fun start() {
        // 创建并启动 HandlerThread
        handlerThread = HandlerThread("BackgroundThread")
        handlerThread.start()
        
        // 获取 Handler
        backgroundHandler = Handler(handlerThread.looper) { msg ->
            when (msg.what) {
                MSG_DOWNLOAD -> {
                    // 在后台线程执行下载
                    downloadFile(msg.obj as String)
                    true
                }
                else -> false
            }
        }
    }
    
    fun download(url: String) {
        backgroundHandler.obtainMessage(MSG_DOWNLOAD, url).sendToTarget()
    }
    
    fun stop() {
        handlerThread.quitSafely()
    }
    
    companion object {
        private const val MSG_DOWNLOAD = 1
    }
}
```

### 4.2 Handler 内存泄漏解决

```kotlin
// 错误写法：非静态内部类持有外部类引用
class LeakyActivity : AppCompatActivity() {
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            // 隐式持有 Activity 引用
            updateUI()
        }
    }
}

// 正确写法：静态内部类 + 弱引用
class SafeActivity : AppCompatActivity() {
    
    private val handler = SafeHandler(this)
    
    override fun onDestroy() {
        super.onDestroy()
        // 移除所有消息和回调
        handler.removeCallbacksAndMessages(null)
    }
    
    private fun updateUI() {
        // 更新 UI
    }
    
    // 静态内部类
    private class SafeHandler(activity: SafeActivity) : Handler(Looper.getMainLooper()) {
        private val activityRef = WeakReference(activity)
        
        override fun handleMessage(msg: Message) {
            val activity = activityRef.get() ?: return
            when (msg.what) {
                MSG_UPDATE_UI -> activity.updateUI()
            }
        }
    }
    
    companion object {
        private const val MSG_UPDATE_UI = 1
    }
}

// 更简洁的写法：使用 Lifecycle 感知
class LifecycleAwareHandler(
    private val lifecycleOwner: LifecycleOwner,
    looper: Looper,
    private val callback: (Message) -> Unit
) : Handler(looper), DefaultLifecycleObserver {
    
    init {
        lifecycleOwner.lifecycle.addObserver(this)
    }
    
    override fun handleMessage(msg: Message) {
        if (lifecycleOwner.lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
            callback(msg)
        }
    }
    
    override fun onDestroy(owner: LifecycleOwner) {
        removeCallbacksAndMessages(null)
        lifecycleOwner.lifecycle.removeObserver(this)
    }
}
```

### 4.3 主线程 Looper 为什么不会 ANR

```java
// ActivityThread.java
public static void main(String[] args) {
    // 准备主线程 Looper
    Looper.prepareMainLooper();
    
    // 创建 ActivityThread
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
    
    // 获取主线程 Handler
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    
    // 开始消息循环
    Looper.loop();
    
    // 正常情况不会执行到这里
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

**为什么不会 ANR？**

1. **ANR 的本质**：ANR 是指应用在特定时间内没有响应特定事件（如 Input 事件、Service 启动等），而不是主线程阻塞。

2. **Looper.loop() 的阻塞**：
   - `MessageQueue.next()` 在没有消息时会调用 `nativePollOnce()` 阻塞
   - 这是一种高效的等待机制（epoll），不消耗 CPU
   - 当有新消息时会被唤醒

3. **ANR 触发条件**：
   - Input 事件 5 秒内没有处理完
   - Service 前台 20 秒/后台 200 秒内没有处理完
   - BroadcastReceiver 前台 10 秒/后台 60 秒内没有处理完

4. **关键区别**：
   - Looper 阻塞是在等待消息，此时没有需要处理的事件
   - ANR 是有事件需要处理，但处理时间过长

### 4.4 子线程更新 UI 的方式

```kotlin
class UpdateUIExample : AppCompatActivity() {
    
    private lateinit var textView: TextView
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        textView = findViewById(R.id.textView)
        
        // 方式1：Handler.post()
        val handler = Handler(Looper.getMainLooper())
        Thread {
            val result = fetchData()
            handler.post {
                textView.text = result
            }
        }.start()
        
        // 方式2：Activity.runOnUiThread()
        Thread {
            val result = fetchData()
            runOnUiThread {
                textView.text = result
            }
        }.start()
        
        // 方式3：View.post()
        Thread {
            val result = fetchData()
            textView.post {
                textView.text = result
            }
        }.start()
        
        // 方式4：协程（推荐）
        lifecycleScope.launch {
            val result = withContext(Dispatchers.IO) {
                fetchData()
            }
            textView.text = result
        }
    }
    
    private fun fetchData(): String {
        Thread.sleep(1000)
        return "Data loaded"
    }
}
```

## 5. 常见面试题

### 问题1：Handler 机制的工作原理是什么？

**答案要点**：
- Handler 机制由 Handler、Looper、MessageQueue、Message 四个组件组成
- Looper.prepare() 创建 Looper 和 MessageQueue，存入 ThreadLocal
- Looper.loop() 开启无限循环，从 MessageQueue 取消息
- Handler.sendMessage() 将消息加入 MessageQueue
- MessageQueue.next() 取出消息，可能阻塞（epoll 机制）
- Looper 调用 Handler.dispatchMessage() 处理消息

### 问题2：一个线程可以有几个 Handler？几个 Looper？几个 MessageQueue？

**答案要点**：
- 一个线程可以有多个 Handler
- 一个线程只能有一个 Looper（ThreadLocal 保证）
- 一个 Looper 只有一个 MessageQueue
- 多个 Handler 共享同一个 MessageQueue

### 问题3：Handler 如何导致内存泄漏？如何解决？

**答案要点**：
- **泄漏原因**：非静态内部类 Handler 持有外部 Activity 引用，Message 持有 Handler 引用，MessageQueue 持有 Message 引用
- **引用链**：MessageQueue → Message → Handler → Activity
- **解决方案**：
  1. 使用静态内部类 + 弱引用
  2. 在 onDestroy() 中调用 `handler.removeCallbacksAndMessages(null)`
  3. 使用 Lifecycle 感知的 Handler

### 问题4：主线程的 Looper.loop() 为什么不会导致 ANR？

**答案要点**：
- ANR 是指应用在规定时间内没有响应特定事件，不是主线程阻塞
- Looper.loop() 阻塞是在等待消息，使用 epoll 机制，不消耗 CPU
- 当有消息时会被唤醒处理
- ANR 触发条件是有事件需要处理但处理时间过长

### 问题5：什么是同步屏障？有什么作用？

**答案要点**：
- 同步屏障是 target 为 null 的特殊 Message
- 作用是优先处理异步消息，跳过同步消息
- 应用场景：UI 渲染（ViewRootImpl.scheduleTraversals）
- 发送：`MessageQueue.postSyncBarrier()`
- 移除：`MessageQueue.removeSyncBarrier(token)`
- 注意：同步屏障是隐藏 API，普通应用无法直接使用

### 问题6：IdleHandler 是什么？有什么使用场景？

**答案要点**：
- IdleHandler 在消息队列空闲时执行
- 使用场景：
  1. 延迟初始化（启动优化）
  2. 首帧绘制完成后执行任务
  3. 低优先级任务
- 返回 true 保持活跃，返回 false 执行后移除
- 注意：不要执行耗时操作，会影响下一个消息的处理

### 问题7：Handler.post() 和 View.post() 有什么区别？

**答案要点**：
- Handler.post() 直接将 Runnable 封装成 Message 发送到 MessageQueue
- View.post() 的行为取决于 View 是否 attach 到 Window：
  - 已 attach：通过 ViewRootImpl 的 Handler 发送
  - 未 attach：存入 RunQueue，在 dispatchAttachedToWindow 时执行
- View.post() 可以保证在 View 测量完成后执行，获取正确的宽高

### 问题8：Message 的复用机制是怎样的？

**答案要点**：
- Message 使用享元模式，维护一个最大 50 个的对象池
- `Message.obtain()` 优先从池中获取
- `Message.recycle()` 回收到池中
- 池使用单链表实现，头插法存取
- 好处：减少对象创建，降低 GC 压力

### 问题9：子线程如何创建 Handler？

**答案要点**：
```kotlin
// 方式1：手动创建
Thread {
    Looper.prepare()
    val handler = Handler(Looper.myLooper()!!)
    Looper.loop()
}.start()

// 方式2：使用 HandlerThread（推荐）
val handlerThread = HandlerThread("MyThread")
handlerThread.start()
val handler = Handler(handlerThread.looper)
```

### 问题10：Handler 的 postDelayed() 是如何实现延迟的？

**答案要点**：
- postDelayed() 计算消息执行时间：`SystemClock.uptimeMillis() + delayMillis`
- MessageQueue 按执行时间排序（单链表）
- next() 方法中，如果消息未到执行时间，计算等待时间
- 调用 `nativePollOnce(ptr, nextPollTimeoutMillis)` 阻塞等待
- 底层使用 epoll_wait 实现精确等待
- 注意：使用 uptimeMillis（系统启动时间），不受系统时间修改影响
