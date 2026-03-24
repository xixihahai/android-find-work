# Handler 机制

## 速记总结

### 一句话理解 Handler 机制
> Handler 就像一个快递公司：**Handler 是快递员（发件/收件）**，**Message 是快递包裹**，**MessageQueue 是快递分拣中心（按时间排序）**，**Looper 是传送带（不停循环取件派送）**。

### 核心要点速记
| 组件 | 一句话记忆 | 类比 |
|------|-----------|------|
| Handler | 负责发送和处理消息 | 快递员（投递+签收） |
| Message | 携带数据的消息对象 | 快递包裹 |
| MessageQueue | 按时间排序的消息队列 | 分拣中心 |
| Looper | 无限循环取消息分发 | 传送带 |
| ThreadLocal | 每个线程一份 Looper | 每个网点一条传送带 |

### 面试答题万能公式
> **说流程** → **说组件** → **说关键细节** → **说应用场景**

---

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

### Handler 消息机制完整流程

```
Handler 消息机制完整流程（生活类比：快递物流系统）
┌──────────────────────────────────────────────────────────────────────┐
│                         子线程（寄件方）                              │
│                                                                      │
│  ┌────────────────────┐                                              │
│  │     Handler         │                                              │
│  │  (快递员：发件)      │                                              │
│  │                     │                                              │
│  │  sendMessage()      │───────────┐                                  │
│  │  sendMessageDelayed()│           │                                  │
│  │  post(Runnable)     │           │                                  │
│  └────────────────────┘           │                                  │
│                                    │  msg.target = handler            │
│                                    │  enqueueMessage()                │
│                                    ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │              MessageQueue（快递分拣中心）                    │      │
│  │                                                            │      │
│  │   按 when 时间排序的单链表：                                 │      │
│  │                                                            │      │
│  │   ┌───────┐    ┌───────┐    ┌───────┐    ┌───────┐        │      │
│  │   │ msg1  │───→│ msg2  │───→│ msg3  │───→│ null  │        │      │
│  │   │when:10│    │when:20│    │when:35│    │       │        │      │
│  │   │target:│    │target:│    │target:│    │       │        │      │
│  │   │  H1   │    │  H2   │    │  H1   │    │       │        │      │
│  │   └───────┘    └───────┘    └───────┘    └───────┘        │      │
│  │                                                            │      │
│  │   阻塞/唤醒机制：                                           │      │
│  │   nativePollOnce() ← epoll_wait（Linux IO 多路复用）        │      │
│  │   nativeWake()     ← eventfd 写入唤醒                      │      │
│  └──────────────────────────────┬─────────────────────────────┘      │
│                                 │                                     │
├─────────────────────────────────┼────────────────────────────────────┤
│                         主线程（收件方）                              │
│                                 │                                     │
│                                 │ queue.next()                        │
│                                 │ （取出到期消息，未到期则阻塞等待）    │
│                                 ▼                                     │
│  ┌──────────────────────────────────────────────────┐                │
│  │               Looper（传送带：不停转）              │                │
│  │                                                   │                │
│  │   Looper.prepare()  → 创建 Looper + MessageQueue  │                │
│  │   Looper.loop()     → 开启无限循环                 │                │
│  │                                                   │                │
│  │   for (;;) {                                      │                │
│  │       Message msg = queue.next();  // 可能阻塞     │                │
│  │       if (msg == null) return;     // 队列退出     │                │
│  │       msg.target.dispatchMessage(msg); // 分发     │                │
│  │       msg.recycleUnchecked();      // 回收复用     │                │
│  │   }                                               │                │
│  └───────────────────────┬───────────────────────────┘                │
│                          │                                            │
│                          │ msg.target.dispatchMessage(msg)            │
│                          ▼                                            │
│  ┌──────────────────────────────────────────────────┐                │
│  │          Handler（快递员：收件处理）                │                │
│  │                                                   │                │
│  │   消息分发优先级（三级）：                          │                │
│  │   ┌─────────────────────────────────────────┐     │                │
│  │   │ 1. msg.callback != null                 │     │                │
│  │   │    → handleCallback(msg)  [post方式]     │     │                │
│  │   ├─────────────────────────────────────────┤     │                │
│  │   │ 2. mCallback != null                    │     │                │
│  │   │    → mCallback.handleMessage(msg)       │     │                │
│  │   ├─────────────────────────────────────────┤     │                │
│  │   │ 3. handleMessage(msg)  [重写方式]        │     │                │
│  │   └─────────────────────────────────────────┘     │                │
│  └──────────────────────────────────────────────────┘                │
│                                                                      │
│   ThreadLocal 保证：每个线程只有一个 Looper + 一个 MessageQueue       │
│   （类比：每个快递网点只有一条传送带 + 一个分拣中心）                   │
└──────────────────────────────────────────────────────────────────────┘
```

```
同步屏障与异步消息（VIP 快递优先通道）
┌──────────────────────────────────────────────────────────────────┐
│                       MessageQueue                               │
│                                                                  │
│   正常情况（FIFO，先进先出）：                                     │
│   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐                    │
│   │ sync │──→│ sync │──→│ sync │──→│ null │                    │
│   └──────┘   └──────┘   └──────┘   └──────┘                    │
│                                                                  │
│   插入同步屏障后（target == null 的消息）：                        │
│   ┌────────┐   ┌──────┐   ┌───────┐   ┌──────┐   ┌──────┐     │
│   │barrier │──→│ sync │──→│ ASYNC │──→│ sync │──→│ null │     │
│   │target= │   │(跳过) │   │(优先!) │   │(跳过) │   │      │     │
│   │ null   │   └──────┘   └───────┘   └──────┘   └──────┘     │
│   └────────┘       ×          ✓           ×                     │
│                                                                  │
│   应用场景：ViewRootImpl.scheduleTraversals()                    │
│   ① postSyncBarrier()          → 插入屏障                       │
│   ② Choreographer 发送异步消息  → VSYNC 渲染回调                 │
│   ③ removeSyncBarrier()        → 渲染完成，移除屏障              │
└──────────────────────────────────────────────────────────────────┘
```

```
Message 对象池（快递箱回收复用）
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   obtain()  获取消息                    recycle()  回收消息       │
│   ┌─────────────────────┐              ┌─────────────────────┐  │
│   │ 池中有消息？         │              │ 池满了？(MAX=50)     │  │
│   │  ┌─YES─→ 取池头返回  │              │  ┌─YES─→ 丢弃GC     │  │
│   │  └─NO──→ new Message │              │  └─NO──→ 头插法入池  │  │
│   └─────────────────────┘              └─────────────────────┘  │
│                                                                  │
│   对象池结构（单链表，头插头取）：                                 │
│   sPool ──→ [msg] ──→ [msg] ──→ [msg] ──→ null                 │
│              ↑ 取                  ↑ 存                          │
│                                                                  │
│   好处：减少对象创建 → 降低 GC 压力 → 减少卡顿                   │
└──────────────────────────────────────────────────────────────────┘
```

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
        // 标记为正在使用（在池中时保持此标记，防止被重复回收）
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

### 4.5 常用场景案例

#### 场景1：子线程更新 UI

**为什么用**：Android 只允许主线程（创建 View 的线程）更新 UI，在 `ViewRootImpl.checkThread()` 中强制检查。

**使用依据**：所有 View 操作必须在创建 View 的线程执行，子线程直接操作 UI 会抛出 `CalledFromWrongThreadException`。

```kotlin
// 典型场景：网络请求完成后更新 UI
class NetworkActivity : AppCompatActivity() {
    private val mainHandler = Handler(Looper.getMainLooper())

    fun fetchUserInfo() {
        Thread {
            // 子线程：执行网络请求
            val user = api.getUserInfo()

            // 切换到主线程更新 UI
            mainHandler.post {
                nameTextView.text = user.name
                avatarImageView.setImageBitmap(user.avatar)
            }
        }.start()
    }
}
```

> **记忆要点**：子线程不能碰 UI → Handler.post() 切到主线程 → ViewRootImpl.checkThread() 是根因。

#### 场景2：延迟执行任务（postDelayed）

**为什么用**：`postDelayed()` 将延迟时间计算为 `SystemClock.uptimeMillis() + delayMillis` 赋给 `Message.when`，插入队列对应位置，由 `nativePollOnce()` 精确等待。

**使用依据**：替代 `Timer`/`TimerTask`，更安全——Handler 绑定线程生命周期，可随时 `removeCallbacks` 取消；Timer 有线程泄漏风险。

```kotlin
// 典型场景：搜索防抖（输入停止 300ms 后才搜索）
class SearchActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    private var searchRunnable: Runnable? = null

    fun onSearchTextChanged(query: String) {
        // 取消上一次延迟搜索
        searchRunnable?.let { handler.removeCallbacks(it) }

        // 延迟 300ms 执行搜索
        searchRunnable = Runnable { performSearch(query) }
        handler.postDelayed(searchRunnable!!, 300L)
    }
}

// 典型场景：倒计时/轮询
class CountdownHandler(private val textView: TextView) {
    private val handler = Handler(Looper.getMainLooper())
    private var secondsLeft = 60

    private val tick = object : Runnable {
        override fun run() {
            textView.text = "${secondsLeft}s"
            if (secondsLeft > 0) {
                secondsLeft--
                handler.postDelayed(this, 1000L) // 每秒执行一次
            }
        }
    }

    fun start() = handler.post(tick)
    fun stop() = handler.removeCallbacks(tick)
}
```

> **记忆要点**：postDelayed 靠 Message.when 排序 + epoll_wait 精确等待，比 Timer 更安全可控。

#### 场景3：HandlerThread 处理串行耗时任务

**为什么用**：`HandlerThread` 是自带 Looper 的线程，可以串行处理一系列后台任务，无需反复创建/销毁线程。

**使用依据**：`IntentService` 内部就是用 `HandlerThread` 实现的；适合数据库写入、文件 IO、日志写入等需要串行保序的场景。

```kotlin
// 典型场景：日志写入（必须串行，避免文件并发写入）
class LogWriter {
    private val handlerThread = HandlerThread("LogWriter").apply { start() }
    private val handler = Handler(handlerThread.looper)

    fun write(log: String) {
        handler.post {
            // 在后台线程串行写入文件，不阻塞主线程
            File("app.log").appendText("${System.currentTimeMillis()}: $log\n")
        }
    }

    fun shutdown() {
        handlerThread.quitSafely()
    }
}

// 典型场景：串行图片压缩
class ImageCompressor {
    private val handlerThread = HandlerThread("ImageCompressor").apply { start() }
    private val handler = Handler(handlerThread.looper)
    private val mainHandler = Handler(Looper.getMainLooper())

    fun compress(bitmap: Bitmap, callback: (File) -> Unit) {
        handler.post {
            val file = doCompress(bitmap) // 耗时压缩操作
            mainHandler.post { callback(file) } // 回主线程回调
        }
    }
}
```

> **记忆要点**：HandlerThread = 自带 Looper 的线程，适合串行后台任务，IntentService 就用它。

#### 场景4：IdleHandler 延迟初始化（启动优化）

**为什么用**：`IdleHandler` 在主线程消息队列空闲时执行，不占用启动关键路径时间，不影响首帧渲染速度。

**使用依据**：启动优化常用手段——将非关键初始化延迟到空闲时执行，比固定 `postDelayed` 更智能（不需要猜延迟时间）。

```kotlin
// 典型场景：Application 启动优化
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 关键初始化（必须立即执行）
        initCrashReporter()
        initNetwork()

        // 非关键初始化 → 延迟到主线程空闲时
        Looper.myQueue().addIdleHandler {
            initAnalytics()     // 数据统计
            initPush()          // 推送服务
            initSkinEngine()    // 换肤引擎
            false // 返回 false 表示只执行一次
        }
    }
}

// 典型场景：首帧绘制完成后预加载
class MainActivity : AppCompatActivity() {
    override fun onResume() {
        super.onResume()
        Looper.myQueue().addIdleHandler {
            // 首帧渲染完成，主线程空闲，开始预加载
            preloadNextPageData()
            preloadWebView()
            false
        }
    }
}
```

> **记忆要点**：IdleHandler = 主线程空闲时执行，比 postDelayed 更智能，启动优化必备。

#### 场景5：同步屏障 + 异步消息（VSYNC 渲染优先）

**为什么用**：发送同步屏障后，`MessageQueue.next()` 会跳过所有同步消息，优先取出异步消息。保证 UI 渲染回调（VSYNC 信号）不被其他消息阻塞。

**使用依据**：`ViewRootImpl.scheduleTraversals()` 使用此机制——先插入同步屏障，再通过 `Choreographer` 发送异步消息处理 measure/layout/draw，渲染完成后移除屏障。这是 Android 保证 60fps 流畅渲染的核心机制。

```java
// 系统源码：ViewRootImpl.java（开发者一般不直接使用）
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // ① 插入同步屏障 → 阻塞所有同步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // ② 通过 Choreographer 发送异步消息 → 收到 VSYNC 后执行渲染
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}

void unscheduleTraversals() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // ③ 渲染完成 → 移除同步屏障 → 恢复处理同步消息
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
    }
}

// 时序：
// scheduleTraversals() → postSyncBarrier() → VSYNC 到来
//     → 异步消息被优先取出 → doTraversal() → measure/layout/draw
//     → removeSyncBarrier() → 继续处理普通消息
```

> **记忆要点**：同步屏障 = VIP 通道，让渲染消息插队执行，保证 UI 不卡顿。

## 5. 常见面试题

### 问题1：Handler 机制的工作原理是什么？

**答案要点**：
- Handler 机制由 Handler、Looper、MessageQueue、Message 四个组件组成
- Looper.prepare() 创建 Looper 和 MessageQueue，存入 ThreadLocal
- Looper.loop() 开启无限循环，从 MessageQueue 取消息
- Handler.sendMessage() 将消息加入 MessageQueue
- MessageQueue.next() 取出消息，可能阻塞（epoll 机制）
- Looper 调用 Handler.dispatchMessage() 处理消息

> **记忆要点**：Handler 发消息 → MessageQueue 排队 → Looper 循环取出 → Handler 处理，四大组件形成闭环。

### 问题2：一个线程可以有几个 Handler？几个 Looper？几个 MessageQueue？

**答案要点**：
- 一个线程可以有多个 Handler
- 一个线程只能有一个 Looper（ThreadLocal 保证）
- 一个 Looper 只有一个 MessageQueue
- 多个 Handler 共享同一个 MessageQueue

> **记忆要点**：多个 Handler、一个 Looper、一个 MessageQueue——多个快递员共用一条传送带和一个分拣中心。

### 问题3：Handler 如何导致内存泄漏？如何解决？

**答案要点**：
- **泄漏原因**：非静态内部类 Handler 持有外部 Activity 引用，Message 持有 Handler 引用，MessageQueue 持有 Message 引用
- **引用链**：MessageQueue → Message → Handler → Activity
- **解决方案**：
  1. 使用静态内部类 + 弱引用
  2. 在 onDestroy() 中调用 `handler.removeCallbacksAndMessages(null)`
  3. 使用 Lifecycle 感知的 Handler

> **记忆要点**：引用链 MQ→Msg→Handler→Activity 导致泄漏，解法是静态内部类+弱引用+onDestroy 清消息。

### 问题4：主线程的 Looper.loop() 为什么不会导致 ANR？

**答案要点**：
- ANR 是指应用在规定时间内没有响应特定事件，不是主线程阻塞
- Looper.loop() 阻塞是在等待消息，使用 epoll 机制，不消耗 CPU
- 当有消息时会被唤醒处理
- ANR 触发条件是有事件需要处理但处理时间过长

> **记忆要点**：Looper 阻塞是"没事等着"，ANR 是"有事干不完"——等快递不是问题，送不完才是问题。

### 问题5：什么是同步屏障？有什么作用？

**答案要点**：
- 同步屏障是 target 为 null 的特殊 Message
- 作用是优先处理异步消息，跳过同步消息
- 应用场景：UI 渲染（ViewRootImpl.scheduleTraversals）
- 发送：`MessageQueue.postSyncBarrier()`
- 移除：`MessageQueue.removeSyncBarrier(token)`
- 注意：同步屏障是隐藏 API，普通应用无法直接使用

> **记忆要点**：同步屏障 = target 为 null 的 Message，让异步消息插队，用于 UI 渲染优先保证流畅。

### 问题6：IdleHandler 是什么？有什么使用场景？

**答案要点**：
- IdleHandler 在消息队列空闲时执行
- 使用场景：
  1. 延迟初始化（启动优化）
  2. 首帧绘制完成后执行任务
  3. 低优先级任务
- 返回 true 保持活跃，返回 false 执行后移除
- 注意：不要执行耗时操作，会影响下一个消息的处理

> **记忆要点**：IdleHandler = 队列空闲时的回调，返回 false 一次性、true 持续，适合启动优化延迟初始化。

### 问题7：Handler.post() 和 View.post() 有什么区别？

**答案要点**：
- Handler.post() 直接将 Runnable 封装成 Message 发送到 MessageQueue
- View.post() 的行为取决于 View 是否 attach 到 Window：
  - 已 attach：通过 ViewRootImpl 的 Handler 发送
  - 未 attach：存入 RunQueue，在 dispatchAttachedToWindow 时执行
- View.post() 可以保证在 View 测量完成后执行，获取正确的宽高

> **记忆要点**：View.post() 能拿到正确宽高因为它等 attach 后才执行，Handler.post() 则立即入队。

### 问题8：Message 的复用机制是怎样的？

**答案要点**：
- Message 使用享元模式，维护一个最大 50 个的对象池
- `Message.obtain()` 优先从池中获取
- `Message.recycle()` 回收到池中
- 池使用单链表实现，头插法存取
- 好处：减少对象创建，降低 GC 压力

> **记忆要点**：Message 用享元模式池化复用（上限50），obtain() 取、recycle() 还，单链表头插头取。

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

> **记忆要点**：子线程必须先 prepare() 再 loop()，推荐直接用 HandlerThread 省去手动管理。

### 问题10：Handler 的 postDelayed() 是如何实现延迟的？

**答案要点**：
- postDelayed() 计算消息执行时间：`SystemClock.uptimeMillis() + delayMillis`
- MessageQueue 按执行时间排序（单链表）
- next() 方法中，如果消息未到执行时间，计算等待时间
- 调用 `nativePollOnce(ptr, nextPollTimeoutMillis)` 阻塞等待
- 底层使用 epoll_wait 实现精确等待
- 注意：使用 uptimeMillis（系统启动时间），不受系统时间修改影响

> **记忆要点**：延迟 = 算出 when 时间插入链表对应位置 + epoll_wait 精确阻塞等待，用 uptimeMillis 不受调时间影响。
