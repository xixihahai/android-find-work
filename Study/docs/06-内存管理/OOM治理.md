# OOM 治理

## 1. 概述

OOM（Out Of Memory）是 Android 应用开发中最严重的稳定性问题之一，直接导致应用崩溃。与普通 Crash 不同，OOM 往往是内存问题长期积累的结果，排查难度大、修复成本高。

**OOM 的本质：** 当应用申请内存时，系统无法满足其内存需求，抛出 `OutOfMemoryError` 或直接被系统杀死。

**OOM 的危害：**
- 应用直接崩溃，用户体验极差
- 可能导致数据丢失
- 难以复现和定位
- 影响应用商店评分和留存率

**OOM 分类概览：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           OOM 类型分类                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │  Java Heap OOM  │  │   Native OOM    │  │   线程数 OOM    │         │
│  │                 │  │                 │  │                 │         │
│  │ • Bitmap 过大   │  │ • Native 内存   │  │ • 线程创建过多  │         │
│  │ • 对象过多      │  │   泄漏          │  │ • 虚拟内存不足  │         │
│  │ • 内存泄漏      │  │ • so 库内存     │  │ • 栈空间不足    │         │
│  │ • 大数组分配    │  │ • 图形内存      │  │                 │         │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘         │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐                              │
│  │    FD OOM       │  │  虚拟内存 OOM   │                              │
│  │                 │  │                 │                              │
│  │ • 文件未关闭    │  │ • 32位进程限制  │                              │
│  │ • Socket 泄漏   │  │ • mmap 过多     │                              │
│  │ • Cursor 未关闭 │  │ • 地址空间碎片  │                              │
│  └─────────────────┘  └─────────────────┘                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```


## 2. 核心原理

### 2.1 OOM 类型详解

#### 2.1.1 Java Heap OOM

Java Heap OOM 是最常见的 OOM 类型，当 Java 堆内存使用超过限制时触发。

**触发条件：**
- 堆内存使用量达到 `Runtime.maxMemory()` 限制
- GC 后仍无法释放足够内存
- 单次分配请求超过可用空间

**堆内存限制（不同设备差异较大）：**

| 设备类型 | 典型堆限制 | 大堆限制（largeHeap） |
|---------|-----------|---------------------|
| 低端机   | 128MB     | 256MB               |
| 中端机   | 256MB     | 512MB               |
| 高端机   | 512MB     | 1GB+                |

**常见触发场景：**

```kotlin
// 场景1：Bitmap 内存过大
val bitmap = BitmapFactory.decodeResource(resources, R.drawable.huge_image)
// 一张 4000x3000 的 ARGB_8888 图片占用：4000 * 3000 * 4 = 48MB

// 场景2：大数组分配
val hugeArray = ByteArray(100 * 1024 * 1024)  // 100MB

// 场景3：内存泄漏累积
class LeakyActivity : Activity() {
    companion object {
        val leakedActivities = mutableListOf<Activity>()  // 静态持有
    }
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        leakedActivities.add(this)  // 泄漏！
    }
}
```

**Java Heap OOM 错误信息：**

```
java.lang.OutOfMemoryError: Failed to allocate a 52428816 byte allocation 
with 16777216 free bytes and 46MB until OOM, target footprint 268435456, 
growth limit 268435456
    at dalvik.system.VMRuntime.newNonMovableArray(Native Method)
    at android.graphics.Bitmap.nativeCreate(Native Method)
    at android.graphics.Bitmap.createBitmap(Bitmap.java:1046)
```

#### 2.1.2 Native OOM

Native OOM 发生在 Native 层（C/C++）内存分配失败时，不受 Java 堆限制约束。

**Native 内存组成：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      Native 内存分布                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Native Heap                          │   │
│  │  • malloc/new 分配的内存                                 │   │
│  │  • JNI 层分配的内存                                      │   │
│  │  • 第三方 so 库内存                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Graphics 内存                         │   │
│  │  • OpenGL 纹理                                          │   │
│  │  • Vulkan 资源                                          │   │
│  │  • Hardware Bitmap (Android 8.0+)                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    mmap 内存                            │   │
│  │  • 文件映射                                             │   │
│  │  • 匿名映射                                             │   │
│  │  • Ashmem 共享内存                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Native OOM 特点：**
- 不会抛出 Java 异常，通常直接 Crash 或被系统杀死
- 难以通过常规工具检测
- Android 8.0+ Bitmap 默认存储在 Native 内存


#### 2.1.3 线程数 OOM

当应用创建的线程数超过系统限制时，会触发线程数 OOM。

**线程数限制：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      线程数限制因素                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 系统级限制                                                  │
│     • /proc/sys/kernel/threads-max：系统最大线程数              │
│     • 通常为 32768 或更高                                       │
│                                                                 │
│  2. 进程级限制                                                  │
│     • /proc/[pid]/limits 中的 Max processes                    │
│     • 通常为 32768                                              │
│                                                                 │
│  3. 虚拟内存限制（关键因素）                                     │
│     • 32 位进程：约 4GB 虚拟地址空间                            │
│     • 每个线程栈默认 1MB                                        │
│     • 理论上限约 4000 个线程（实际更少）                         │
│                                                                 │
│  4. 64 位进程                                                   │
│     • 虚拟地址空间充足                                          │
│     • 主要受系统级限制约束                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**线程数 OOM 错误信息：**

```
java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: 
Try again
    at java.lang.Thread.nativeCreate(Native Method)
    at java.lang.Thread.start(Thread.java:733)
```

**常见原因：**

```kotlin
// 错误示例1：无限制创建线程
fun badThreadUsage() {
    for (i in 0..10000) {
        Thread {
            // 执行任务
            Thread.sleep(Long.MAX_VALUE)
        }.start()
    }
}

// 错误示例2：线程池配置不当
val badExecutor = Executors.newCachedThreadPool()  // 无上限
repeat(10000) {
    badExecutor.execute {
        Thread.sleep(Long.MAX_VALUE)
    }
}

// 错误示例3：RxJava 调度器滥用
Observable.range(1, 10000)
    .flatMap { 
        Observable.just(it)
            .subscribeOn(Schedulers.newThread())  // 每次创建新线程
    }
    .subscribe()
```

#### 2.1.4 FD（文件描述符）OOM

每个进程可打开的文件描述符数量有限，超过限制会导致 OOM。

**FD 限制：**
- 软限制（soft limit）：通常 1024
- 硬限制（hard limit）：通常 32768
- Android 进程默认：1024（可通过 `ulimit -n` 查看）

**FD 类型：**

```
┌─────────────────────────────────────────────────────────────────┐
│                        FD 类型分布                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  文件类型 FD                                                    │
│  ├── 普通文件（/data/data/xxx/files/...）                       │
│  ├── 数据库文件（*.db, *.db-journal, *.db-wal）                 │
│  └── SharedPreferences 文件                                     │
│                                                                 │
│  Socket 类型 FD                                                 │
│  ├── TCP/UDP Socket                                             │
│  ├── Unix Domain Socket                                         │
│  └── Binder 连接                                                │
│                                                                 │
│  管道类型 FD                                                    │
│  ├── pipe                                                       │
│  ├── eventfd                                                    │
│  └── epoll                                                      │
│                                                                 │
│  其他类型 FD                                                    │
│  ├── /dev/ashmem（匿名共享内存）                                │
│  ├── /dev/binder                                                │
│  └── anon_inode（匿名 inode）                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**FD OOM 错误信息：**

```
java.io.IOException: Cannot run program "xxx": error=24, 
Too many open files

android.database.sqlite.SQLiteCantOpenDatabaseException: 
unable to open database file (code 14): Could not open database
```


### 2.2 Android 内存管理机制

#### 2.2.1 进程内存限制

```
┌─────────────────────────────────────────────────────────────────┐
│                    Android 进程内存架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   虚拟地址空间                           │   │
│  │  32 位进程：4GB（用户空间约 3GB）                        │   │
│  │  64 位进程：理论 256TB（实际受物理内存限制）              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   物理内存映射                           │   │
│  │  • 按需分配（Demand Paging）                            │   │
│  │  • 共享库内存共享                                        │   │
│  │  • Copy-on-Write 机制                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Low Memory Killer                     │   │
│  │  • 根据 oom_adj 值决定杀进程优先级                       │   │
│  │  • 内存不足时按优先级杀死进程                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 Java Heap 内存分配流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    new Object() 内存分配流程                     │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              1. 尝试在 TLAB 中分配（Thread Local）               │
│                 快速路径，无锁分配                               │
├─────────────────────────────────────────────────────────────────┤
│                    成功 → 返回对象地址                           │
│                    失败 ↓                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              2. 尝试在 Region 中分配                             │
│                 需要加锁                                        │
├─────────────────────────────────────────────────────────────────┤
│                    成功 → 返回对象地址                           │
│                    失败 ↓                                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              3. 触发 GC（Concurrent GC）                        │
│                 尝试回收内存                                     │
├─────────────────────────────────────────────────────────────────┤
│                    GC 后有足够空间 → 分配成功                    │
│                    仍然不足 ↓                                   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              4. 触发 Full GC（Stop-The-World）                  │
│                 清理软引用                                       │
├─────────────────────────────────────────────────────────────────┤
│                    GC 后有足够空间 → 分配成功                    │
│                    仍然不足 ↓                                   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              5. 尝试扩展堆空间                                   │
│                 向系统申请更多内存                               │
├─────────────────────────────────────────────────────────────────┤
│                    扩展成功 → 分配成功                           │
│                    达到 maxMemory 限制 ↓                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              6. 抛出 OutOfMemoryError                           │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 32 位与 64 位进程差异

```
┌─────────────────────────────────────────────────────────────────┐
│                  32 位 vs 64 位进程对比                          │
├─────────────────────────────┬───────────────────────────────────┤
│          32 位进程          │           64 位进程               │
├─────────────────────────────┼───────────────────────────────────┤
│  虚拟地址空间：4GB          │  虚拟地址空间：256TB              │
│  用户空间：约 3GB           │  用户空间：约 128TB               │
├─────────────────────────────┼───────────────────────────────────┤
│  指针大小：4 字节           │  指针大小：8 字节                 │
│  对象头：8 字节             │  对象头：12-16 字节               │
├─────────────────────────────┼───────────────────────────────────┤
│  线程栈：约 4000 个上限     │  线程栈：几乎无限制               │
│  mmap 空间受限              │  mmap 空间充足                    │
├─────────────────────────────┼───────────────────────────────────┤
│  容易出现虚拟内存 OOM       │  虚拟内存 OOM 罕见                │
│  地址空间碎片化问题         │  碎片化问题不明显                 │
├─────────────────────────────┼───────────────────────────────────┤
│  so 库：armeabi-v7a         │  so 库：arm64-v8a                 │
│  兼容性好                   │  性能更好                         │
└─────────────────────────────┴───────────────────────────────────┘
```


## 3. 关键源码解析

### 3.1 Java Heap OOM 触发源码

#### 3.1.1 ART 虚拟机内存分配

```cpp
// art/runtime/gc/heap.cc
/**
 * ART 虚拟机堆内存分配入口
 * 当 Java 层 new 对象时最终调用到这里
 */
mirror::Object* Heap::AllocObjectWithAllocator(Thread* self,
                                                ObjPtr<mirror::Class> klass,
                                                size_t byte_count,
                                                AllocatorType allocator,
                                                const PreFenceVisitor& pre_fence_visitor) {
    // 1. 检查是否需要 GC
    if (UNLIKELY(ShouldAllocLargeObject(klass, byte_count))) {
        return AllocLargeObject<kInstrumented, PreFenceVisitor>(
            self, &klass, byte_count, pre_fence_visitor);
    }
    
    mirror::Object* obj;
    
    // 2. 尝试快速分配（TLAB）
    if (allocator == kAllocatorTypeTLAB) {
        obj = self->AllocTlab(byte_count);
        if (LIKELY(obj != nullptr)) {
            return obj;
        }
    }
    
    // 3. 慢速路径分配
    obj = TryToAllocate<kInstrumented, false>(self, allocator, byte_count, 
                                               &bytes_allocated, &usable_size,
                                               &bytes_tl_bulk_allocated);
    
    if (UNLIKELY(obj == nullptr)) {
        // 4. 分配失败，尝试 GC 后重试
        obj = AllocateInternalWithGc(self, allocator, kInstrumented, 
                                      byte_count, &bytes_allocated,
                                      &usable_size, &bytes_tl_bulk_allocated, 
                                      &klass);
        
        if (obj == nullptr) {
            // 5. GC 后仍然失败，抛出 OOM
            ThrowOutOfMemoryError(self, byte_count, allocator);
            return nullptr;
        }
    }
    
    return obj;
}

/**
 * 带 GC 的内存分配
 */
mirror::Object* Heap::AllocateInternalWithGc(Thread* self,
                                              AllocatorType allocator,
                                              bool instrumented,
                                              size_t alloc_size,
                                              size_t* bytes_allocated,
                                              size_t* usable_size,
                                              size_t* bytes_tl_bulk_allocated,
                                              ObjPtr<mirror::Class>* klass) {
    mirror::Object* ptr = nullptr;
    
    // 1. 第一次尝试：Concurrent GC
    collector::GcType tried_type = collector::kGcTypeNone;
    const bool gc_ran = CollectGarbageInternal(
        collector::kGcTypeSticky, kGcCauseForAlloc, false) != collector::kGcTypeNone;
    
    if (gc_ran) {
        ptr = TryToAllocate<true, false>(self, allocator, alloc_size,
                                          bytes_allocated, usable_size,
                                          bytes_tl_bulk_allocated);
        if (ptr != nullptr) {
            return ptr;
        }
    }
    
    // 2. 第二次尝试：Partial GC
    tried_type = CollectGarbageInternal(collector::kGcTypePartial, 
                                         kGcCauseForAlloc, false);
    if (tried_type != collector::kGcTypeNone) {
        ptr = TryToAllocate<true, false>(self, allocator, alloc_size,
                                          bytes_allocated, usable_size,
                                          bytes_tl_bulk_allocated);
        if (ptr != nullptr) {
            return ptr;
        }
    }
    
    // 3. 第三次尝试：Full GC（清理软引用）
    CollectGarbageInternal(collector::kGcTypeFull, kGcCauseForAlloc, true);
    ptr = TryToAllocate<true, true>(self, allocator, alloc_size,
                                     bytes_allocated, usable_size,
                                     bytes_tl_bulk_allocated);
    
    // 4. 尝试扩展堆
    if (ptr == nullptr) {
        const size_t current_footprint = GetBytesAllocated();
        const size_t max_allowed = growth_limit_;  // 即 maxMemory()
        
        if (current_footprint < max_allowed) {
            // 还有扩展空间
            size_t grow_bytes = std::min(alloc_size, max_allowed - current_footprint);
            if (UNLIKELY(!TryGrowHeap(grow_bytes))) {
                // 扩展失败
                return nullptr;
            }
            ptr = TryToAllocate<true, true>(self, allocator, alloc_size,
                                             bytes_allocated, usable_size,
                                             bytes_tl_bulk_allocated);
        }
    }
    
    return ptr;  // 可能为 nullptr，调用方会抛出 OOM
}

/**
 * 抛出 OutOfMemoryError
 */
void Heap::ThrowOutOfMemoryError(Thread* self, size_t byte_count, 
                                  AllocatorType allocator_type) {
    std::ostringstream oss;
    
    // 构建详细的错误信息
    oss << "Failed to allocate a " << byte_count << " byte allocation with "
        << GetFreeMemory() << " free bytes and "
        << PrettySize(GetMaxMemory() - GetBytesAllocated()) << " until OOM, "
        << "target footprint " << PrettySize(target_footprint_.load()) << ", "
        << "growth limit " << PrettySize(growth_limit_);
    
    // 抛出 Java 异常
    self->ThrowOutOfMemoryError(oss.str().c_str());
}
```


#### 3.1.2 获取堆内存信息

```java
// libcore/ojluni/src/main/java/java/lang/Runtime.java
public class Runtime {
    
    /**
     * 获取 JVM 最大可用内存
     * 即 dalvik.vm.heapgrowthlimit 或 dalvik.vm.heapsize（largeHeap）
     */
    public native long maxMemory();
    
    /**
     * 获取 JVM 当前已分配的总内存
     * 包括已使用和空闲部分
     */
    public native long totalMemory();
    
    /**
     * 获取 JVM 当前空闲内存
     */
    public native long freeMemory();
}

// 使用示例：获取内存使用情况
fun getMemoryInfo(): String {
    val runtime = Runtime.getRuntime()
    val maxMemory = runtime.maxMemory() / 1024 / 1024      // 最大可用
    val totalMemory = runtime.totalMemory() / 1024 / 1024  // 当前已分配
    val freeMemory = runtime.freeMemory() / 1024 / 1024    // 当前空闲
    val usedMemory = totalMemory - freeMemory              // 当前已使用
    
    return """
        Max Memory: ${maxMemory}MB
        Total Memory: ${totalMemory}MB
        Used Memory: ${usedMemory}MB
        Free Memory: ${freeMemory}MB
        Available: ${maxMemory - usedMemory}MB
    """.trimIndent()
}
```

### 3.2 线程数 OOM 源码分析

#### 3.2.1 线程创建流程

```cpp
// art/runtime/thread.cc
/**
 * 创建新线程
 */
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, 
                                 size_t stack_size, bool is_daemon) {
    Thread* self = static_cast<JNIEnvExt*>(env)->GetSelf();
    Runtime* runtime = Runtime::Current();
    
    // 1. 检查运行时状态
    if (runtime->IsShuttingDownLocked()) {
        // 运行时正在关闭，拒绝创建
        return;
    }
    
    Thread* child_thread = new Thread(is_daemon);
    
    // 2. 设置栈大小（默认 1MB）
    if (stack_size == 0) {
        stack_size = runtime->GetDefaultStackSize();
    }
    
    // 3. 创建 pthread
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    pthread_attr_setstacksize(&attr, stack_size);
    
    // 4. 调用 pthread_create
    int pthread_create_result = pthread_create(&new_pthread,
                                                &attr,
                                                Thread::CreateCallback,
                                                child_thread);
    
    pthread_attr_destroy(&attr);
    
    // 5. 检查创建结果
    if (pthread_create_result != 0) {
        // 创建失败，通常是资源不足
        // 错误码 EAGAIN (11): Resource temporarily unavailable
        child_thread->DecrementRef();
        
        // 抛出 OOM 异常
        std::string msg = StringPrintf(
            "pthread_create (%s stack) failed: %s",
            PrettySize(stack_size).c_str(),
            strerror(pthread_create_result));
        
        self->ThrowOutOfMemoryError(msg.c_str());
        return;
    }
}
```

#### 3.2.2 线程数监控

```kotlin
/**
 * 线程数监控工具
 */
object ThreadMonitor {
    
    private const val TAG = "ThreadMonitor"
    private const val WARNING_THRESHOLD = 500   // 警告阈值
    private const val CRITICAL_THRESHOLD = 800  // 危险阈值
    
    /**
     * 获取当前进程线程数
     */
    fun getThreadCount(): Int {
        return try {
            // 方法1：通过 /proc/self/status 读取
            File("/proc/self/status").readLines()
                .find { it.startsWith("Threads:") }
                ?.split("\\s+".toRegex())
                ?.getOrNull(1)
                ?.toIntOrNull() ?: -1
        } catch (e: Exception) {
            // 方法2：通过 Thread.getAllStackTraces()
            Thread.getAllStackTraces().size
        }
    }
    
    /**
     * 获取线程详细信息
     */
    fun getThreadDetails(): List<ThreadInfo> {
        return Thread.getAllStackTraces().map { (thread, stackTrace) ->
            ThreadInfo(
                id = thread.id,
                name = thread.name,
                state = thread.state,
                isDaemon = thread.isDaemon,
                priority = thread.priority,
                stackTrace = stackTrace.take(5).map { it.toString() }
            )
        }
    }
    
    /**
     * 按线程名前缀分组统计
     */
    fun getThreadCountByPrefix(): Map<String, Int> {
        return Thread.getAllStackTraces().keys
            .groupBy { thread ->
                // 提取线程名前缀
                val name = thread.name
                when {
                    name.startsWith("OkHttp") -> "OkHttp"
                    name.startsWith("RxCached") -> "RxJava-Cached"
                    name.startsWith("RxComputation") -> "RxJava-Computation"
                    name.startsWith("RxNewThread") -> "RxJava-NewThread"
                    name.startsWith("pool-") -> "ThreadPool"
                    name.startsWith("AsyncTask") -> "AsyncTask"
                    name.startsWith("Binder:") -> "Binder"
                    name.startsWith("FinalizerWatchdog") -> "Finalizer"
                    name.startsWith("HeapTaskDaemon") -> "GC"
                    else -> "Other"
                }
            }
            .mapValues { it.value.size }
            .toSortedMap()
    }
    
    /**
     * 检查线程数是否异常
     */
    fun checkThreadHealth(): ThreadHealthStatus {
        val count = getThreadCount()
        return when {
            count >= CRITICAL_THRESHOLD -> {
                Log.e(TAG, "Critical: Thread count $count exceeds $CRITICAL_THRESHOLD")
                dumpThreadInfo()
                ThreadHealthStatus.CRITICAL
            }
            count >= WARNING_THRESHOLD -> {
                Log.w(TAG, "Warning: Thread count $count exceeds $WARNING_THRESHOLD")
                ThreadHealthStatus.WARNING
            }
            else -> ThreadHealthStatus.NORMAL
        }
    }
    
    /**
     * 导出线程信息用于分析
     */
    private fun dumpThreadInfo() {
        val details = getThreadCountByPrefix()
        Log.e(TAG, "Thread distribution: $details")
        
        // 找出可疑的线程组
        details.filter { it.value > 50 }.forEach { (prefix, count) ->
            Log.e(TAG, "Suspicious thread group: $prefix = $count")
        }
    }
    
    enum class ThreadHealthStatus {
        NORMAL, WARNING, CRITICAL
    }
    
    data class ThreadInfo(
        val id: Long,
        val name: String,
        val state: Thread.State,
        val isDaemon: Boolean,
        val priority: Int,
        val stackTrace: List<String>
    )
}
```


### 3.3 FD 泄漏检测源码

#### 3.3.1 FD 监控实现

```kotlin
/**
 * FD（文件描述符）监控工具
 */
object FDMonitor {
    
    private const val TAG = "FDMonitor"
    private const val FD_WARNING_THRESHOLD = 800
    private const val FD_CRITICAL_THRESHOLD = 900
    
    /**
     * 获取当前进程 FD 数量
     */
    fun getFDCount(): Int {
        return try {
            File("/proc/self/fd").listFiles()?.size ?: -1
        } catch (e: Exception) {
            Log.e(TAG, "Failed to get FD count", e)
            -1
        }
    }
    
    /**
     * 获取 FD 限制
     */
    fun getFDLimit(): Pair<Int, Int> {
        return try {
            val limits = File("/proc/self/limits").readLines()
                .find { it.contains("Max open files") }
                ?.split("\\s+".toRegex())
            
            val softLimit = limits?.getOrNull(3)?.toIntOrNull() ?: -1
            val hardLimit = limits?.getOrNull(4)?.toIntOrNull() ?: -1
            Pair(softLimit, hardLimit)
        } catch (e: Exception) {
            Pair(-1, -1)
        }
    }
    
    /**
     * 获取 FD 详细信息
     */
    fun getFDDetails(): List<FDInfo> {
        val fdDir = File("/proc/self/fd")
        val fdList = mutableListOf<FDInfo>()
        
        fdDir.listFiles()?.forEach { fdFile ->
            try {
                val fdNum = fdFile.name.toIntOrNull() ?: return@forEach
                val target = fdFile.canonicalPath  // 读取符号链接目标
                
                val type = when {
                    target.startsWith("/dev/") -> FDType.DEVICE
                    target.startsWith("socket:") -> FDType.SOCKET
                    target.startsWith("pipe:") -> FDType.PIPE
                    target.startsWith("anon_inode:") -> FDType.ANON_INODE
                    target.contains(".db") -> FDType.DATABASE
                    target.startsWith("/data/") -> FDType.FILE
                    else -> FDType.OTHER
                }
                
                fdList.add(FDInfo(fdNum, target, type))
            } catch (e: Exception) {
                // 忽略无法读取的 FD
            }
        }
        
        return fdList.sortedBy { it.fd }
    }
    
    /**
     * 按类型统计 FD
     */
    fun getFDCountByType(): Map<FDType, Int> {
        return getFDDetails()
            .groupBy { it.type }
            .mapValues { it.value.size }
    }
    
    /**
     * 检查 FD 健康状态
     */
    fun checkFDHealth(): FDHealthStatus {
        val count = getFDCount()
        val (softLimit, _) = getFDLimit()
        
        return when {
            count >= FD_CRITICAL_THRESHOLD || count >= softLimit * 0.9 -> {
                Log.e(TAG, "Critical: FD count $count, limit $softLimit")
                dumpFDInfo()
                FDHealthStatus.CRITICAL
            }
            count >= FD_WARNING_THRESHOLD || count >= softLimit * 0.8 -> {
                Log.w(TAG, "Warning: FD count $count, limit $softLimit")
                FDHealthStatus.WARNING
            }
            else -> FDHealthStatus.NORMAL
        }
    }
    
    /**
     * 导出 FD 信息
     */
    private fun dumpFDInfo() {
        val byType = getFDCountByType()
        Log.e(TAG, "FD distribution: $byType")
        
        // 找出可疑的 FD 类型
        byType.filter { it.value > 100 }.forEach { (type, count) ->
            Log.e(TAG, "Suspicious FD type: $type = $count")
            
            // 输出该类型的详细信息
            getFDDetails()
                .filter { it.type == type }
                .take(20)
                .forEach { fd ->
                    Log.e(TAG, "  FD ${fd.fd}: ${fd.target}")
                }
        }
    }
    
    enum class FDType {
        FILE, SOCKET, PIPE, DEVICE, DATABASE, ANON_INODE, OTHER
    }
    
    enum class FDHealthStatus {
        NORMAL, WARNING, CRITICAL
    }
    
    data class FDInfo(
        val fd: Int,
        val target: String,
        val type: FDType
    )
}
```

#### 3.3.2 常见 FD 泄漏场景与修复

```kotlin
/**
 * FD 泄漏常见场景示例
 */
class FDLeakExamples {
    
    // ==================== 错误示例 ====================
    
    /**
     * 错误示例1：文件流未关闭
     */
    fun badFileRead(path: String): String {
        val fis = FileInputStream(path)
        val content = fis.bufferedReader().readText()
        // 忘记关闭！FD 泄漏
        return content
    }
    
    /**
     * 错误示例2：Cursor 未关闭
     */
    fun badCursorQuery(context: Context): List<String> {
        val cursor = context.contentResolver.query(
            ContactsContract.Contacts.CONTENT_URI,
            null, null, null, null
        )
        val names = mutableListOf<String>()
        while (cursor?.moveToNext() == true) {
            names.add(cursor.getString(0))
        }
        // 忘记关闭 Cursor！FD 泄漏
        return names
    }
    
    /**
     * 错误示例3：Socket 未关闭
     */
    fun badSocketConnect(host: String, port: Int) {
        val socket = Socket(host, port)
        val output = socket.getOutputStream()
        output.write("Hello".toByteArray())
        // 忘记关闭 Socket！FD 泄漏
    }
    
    /**
     * 错误示例4：ParcelFileDescriptor 未关闭
     */
    fun badPfdUsage(context: Context, uri: Uri) {
        val pfd = context.contentResolver.openFileDescriptor(uri, "r")
        val fd = pfd?.fileDescriptor
        // 使用 fd...
        // 忘记关闭 pfd！FD 泄漏
    }
    
    // ==================== 正确示例 ====================
    
    /**
     * 正确示例1：使用 use 扩展函数（推荐）
     */
    fun goodFileRead(path: String): String {
        return FileInputStream(path).use { fis ->
            fis.bufferedReader().use { reader ->
                reader.readText()
            }
        }
    }
    
    /**
     * 正确示例2：Cursor 正确关闭
     */
    fun goodCursorQuery(context: Context): List<String> {
        return context.contentResolver.query(
            ContactsContract.Contacts.CONTENT_URI,
            null, null, null, null
        )?.use { cursor ->
            val names = mutableListOf<String>()
            while (cursor.moveToNext()) {
                names.add(cursor.getString(0))
            }
            names
        } ?: emptyList()
    }
    
    /**
     * 正确示例3：Socket 正确关闭
     */
    fun goodSocketConnect(host: String, port: Int) {
        Socket(host, port).use { socket ->
            socket.getOutputStream().use { output ->
                output.write("Hello".toByteArray())
            }
        }
    }
    
    /**
     * 正确示例4：try-finally 方式（Java 兼容）
     */
    fun goodFileReadJavaStyle(path: String): String {
        var fis: FileInputStream? = null
        return try {
            fis = FileInputStream(path)
            fis.bufferedReader().readText()
        } finally {
            fis?.close()
        }
    }
}
```


### 3.4 64 位适配源码

#### 3.4.1 检测当前进程架构

```kotlin
/**
 * 64 位适配工具类
 */
object ArchitectureUtils {
    
    /**
     * 检测当前进程是否为 64 位
     */
    fun is64BitProcess(): Boolean {
        // 方法1：通过 Build.SUPPORTED_64_BIT_ABIS
        return Build.SUPPORTED_64_BIT_ABIS.isNotEmpty() && 
               Process.is64Bit()
    }
    
    /**
     * 检测设备是否支持 64 位
     */
    fun is64BitDevice(): Boolean {
        return Build.SUPPORTED_64_BIT_ABIS.isNotEmpty()
    }
    
    /**
     * 获取当前进程 ABI
     */
    fun getCurrentAbi(): String {
        return if (is64BitProcess()) {
            Build.SUPPORTED_64_BIT_ABIS.firstOrNull() ?: "unknown"
        } else {
            Build.SUPPORTED_32_BIT_ABIS.firstOrNull() ?: "unknown"
        }
    }
    
    /**
     * 获取虚拟内存信息
     */
    fun getVirtualMemoryInfo(): VirtualMemoryInfo {
        return try {
            val maps = File("/proc/self/maps").readLines()
            var totalVirtualSize = 0L
            var usedRegions = 0
            
            maps.forEach { line ->
                val parts = line.split("\\s+".toRegex())
                if (parts.isNotEmpty()) {
                    val addressRange = parts[0].split("-")
                    if (addressRange.size == 2) {
                        val start = addressRange[0].toLongOrNull(16) ?: 0
                        val end = addressRange[1].toLongOrNull(16) ?: 0
                        totalVirtualSize += (end - start)
                        usedRegions++
                    }
                }
            }
            
            VirtualMemoryInfo(
                totalVirtualSize = totalVirtualSize,
                usedRegions = usedRegions,
                is64Bit = is64BitProcess()
            )
        } catch (e: Exception) {
            VirtualMemoryInfo(0, 0, is64BitProcess())
        }
    }
    
    data class VirtualMemoryInfo(
        val totalVirtualSize: Long,
        val usedRegions: Int,
        val is64Bit: Boolean
    ) {
        fun getAvailableVirtualMemory(): Long {
            return if (is64Bit) {
                // 64 位进程虚拟内存几乎无限
                Long.MAX_VALUE
            } else {
                // 32 位进程约 3GB 用户空间
                (3L * 1024 * 1024 * 1024) - totalVirtualSize
            }
        }
    }
}
```

#### 3.4.2 AndroidManifest 64 位配置

```xml
<!-- AndroidManifest.xml -->
<application
    android:name=".MyApplication"
    android:extractNativeLibs="false"
    android:usesCleartextTraffic="false">
    
    <!-- 
    64 位适配关键配置：
    1. 确保 so 库包含 arm64-v8a
    2. 移除 android:extractNativeLibs="true"（Android 6.0+）
    3. 使用 App Bundle 按需分发
    -->
    
</application>
```

```groovy
// build.gradle (Module)
android {
    defaultConfig {
        // 配置支持的 ABI
        ndk {
            // 推荐：同时支持 32 位和 64 位
            abiFilters 'armeabi-v7a', 'arm64-v8a'
            
            // 或者：仅 64 位（减小包体积，但不兼容旧设备）
            // abiFilters 'arm64-v8a'
        }
    }
    
    // 按 ABI 分包（可选）
    splits {
        abi {
            enable true
            reset()
            include 'armeabi-v7a', 'arm64-v8a'
            universalApk false  // 不生成通用包
        }
    }
}
```

### 3.5 OOM 监控 Hook 实现

```kotlin
/**
 * OOM 监控 Hook
 * 通过 Hook 内存分配来监控大对象分配
 */
object OOMMonitorHook {
    
    private const val TAG = "OOMMonitor"
    private const val LARGE_ALLOCATION_THRESHOLD = 10 * 1024 * 1024  // 10MB
    
    /**
     * 初始化 OOM 监控
     */
    fun init(application: Application) {
        // 1. 监控 Java 堆内存
        monitorJavaHeap()
        
        // 2. 监控线程数
        monitorThreadCount()
        
        // 3. 监控 FD 数量
        monitorFDCount()
        
        // 4. 注册低内存回调
        registerLowMemoryCallback(application)
    }
    
    /**
     * 监控 Java 堆内存
     */
    private fun monitorJavaHeap() {
        val runtime = Runtime.getRuntime()
        val maxMemory = runtime.maxMemory()
        
        // 定期检查内存使用
        val handler = Handler(Looper.getMainLooper())
        val checkRunnable = object : Runnable {
            override fun run() {
                val usedMemory = runtime.totalMemory() - runtime.freeMemory()
                val usageRatio = usedMemory.toFloat() / maxMemory
                
                when {
                    usageRatio > 0.9f -> {
                        Log.e(TAG, "Critical: Java heap usage ${(usageRatio * 100).toInt()}%")
                        // 触发紧急 GC
                        System.gc()
                        // 上报告警
                        reportOOMRisk("java_heap", usageRatio)
                    }
                    usageRatio > 0.8f -> {
                        Log.w(TAG, "Warning: Java heap usage ${(usageRatio * 100).toInt()}%")
                    }
                }
                
                handler.postDelayed(this, 5000)  // 每 5 秒检查一次
            }
        }
        handler.post(checkRunnable)
    }
    
    /**
     * 监控线程数
     */
    private fun monitorThreadCount() {
        val handler = Handler(Looper.getMainLooper())
        val checkRunnable = object : Runnable {
            override fun run() {
                val threadCount = ThreadMonitor.getThreadCount()
                
                when {
                    threadCount > 800 -> {
                        Log.e(TAG, "Critical: Thread count $threadCount")
                        reportOOMRisk("thread_count", threadCount.toFloat())
                    }
                    threadCount > 500 -> {
                        Log.w(TAG, "Warning: Thread count $threadCount")
                    }
                }
                
                handler.postDelayed(this, 10000)  // 每 10 秒检查一次
            }
        }
        handler.post(checkRunnable)
    }
    
    /**
     * 监控 FD 数量
     */
    private fun monitorFDCount() {
        val handler = Handler(Looper.getMainLooper())
        val checkRunnable = object : Runnable {
            override fun run() {
                val fdCount = FDMonitor.getFDCount()
                
                when {
                    fdCount > 900 -> {
                        Log.e(TAG, "Critical: FD count $fdCount")
                        reportOOMRisk("fd_count", fdCount.toFloat())
                    }
                    fdCount > 700 -> {
                        Log.w(TAG, "Warning: FD count $fdCount")
                    }
                }
                
                handler.postDelayed(this, 10000)
            }
        }
        handler.post(checkRunnable)
    }
    
    /**
     * 注册低内存回调
     */
    private fun registerLowMemoryCallback(application: Application) {
        application.registerComponentCallbacks(object : ComponentCallbacks2 {
            override fun onConfigurationChanged(newConfig: Configuration) {}
            
            override fun onLowMemory() {
                Log.e(TAG, "System low memory!")
                // 清理缓存
                clearCaches()
            }
            
            override fun onTrimMemory(level: Int) {
                Log.w(TAG, "onTrimMemory level: $level")
                when (level) {
                    ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> {
                        // 前台进程，内存极低
                        clearCaches()
                    }
                    ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW -> {
                        // 前台进程，内存较低
                        clearNonEssentialCaches()
                    }
                    ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN -> {
                        // UI 不可见，可以释放 UI 相关资源
                        releaseUIResources()
                    }
                }
            }
        })
    }
    
    private fun clearCaches() {
        // 清理图片缓存
        // 清理数据缓存
        System.gc()
    }
    
    private fun clearNonEssentialCaches() {
        // 清理非必要缓存
    }
    
    private fun releaseUIResources() {
        // 释放 UI 资源
    }
    
    private fun reportOOMRisk(type: String, value: Float) {
        // 上报到监控平台
    }
}
```


## 4. 实战应用

### 4.1 OOM 分析方法与工具

#### 4.1.1 Memory Profiler 使用

```
┌─────────────────────────────────────────────────────────────────┐
│                  Memory Profiler 分析流程                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 连接设备并启动 Profiler                                      │
│     Android Studio → View → Tool Windows → Profiler             │
│                                                                 │
│  2. 选择进程并进入 Memory 视图                                   │
│     点击 MEMORY 区域进入详细视图                                 │
│                                                                 │
│  3. 触发 Heap Dump                                              │
│     点击 "Dump Java heap" 按钮                                  │
│                                                                 │
│  4. 分析 Heap Dump                                              │
│     • 按 Retained Size 排序找大对象                             │
│     • 查看 Instance View 分析对象引用链                          │
│     • 使用 "Arrange by package" 按包名分组                       │
│                                                                 │
│  5. 对比多次 Dump                                               │
│     • 操作前后各 Dump 一次                                       │
│     • 对比新增对象，找出泄漏                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 MAT（Memory Analyzer Tool）分析

```bash
# 1. 导出 hprof 文件
# 方法1：通过 Android Studio
# Memory Profiler → Export heap dump

# 方法2：通过 adb
adb shell am dumpheap <pid> /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof

# 2. 转换 hprof 格式（Android 格式 → 标准格式）
hprof-conv heap.hprof heap-converted.hprof

# 3. 使用 MAT 打开分析
```

**MAT 关键分析视图：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      MAT 分析视图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Histogram（直方图）                                            │
│  ├── 按类统计对象数量和大小                                      │
│  ├── 找出数量异常多的类                                          │
│  └── 右键 → List objects → with incoming references             │
│                                                                 │
│  Dominator Tree（支配树）                                        │
│  ├── 按 Retained Heap 排序                                      │
│  ├── 找出占用内存最大的对象                                      │
│  └── 展开查看被支配的对象                                        │
│                                                                 │
│  Leak Suspects Report（泄漏嫌疑报告）                            │
│  ├── 自动分析可能的泄漏点                                        │
│  └── 提供详细的问题描述和建议                                    │
│                                                                 │
│  OQL（对象查询语言）                                             │
│  ├── SELECT * FROM android.app.Activity                         │
│  ├── SELECT * FROM java.lang.Thread                             │
│  └── 自定义查询找特定对象                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.1.3 LeakCanary 集成

```kotlin
// build.gradle
dependencies {
    // 仅在 debug 版本集成
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
}

// LeakCanary 2.x 无需手动初始化，自动检测以下泄漏：
// - Activity 泄漏
// - Fragment 泄漏
// - ViewModel 泄漏
// - Service 泄漏
// - RootView 泄漏

/**
 * 自定义 LeakCanary 配置
 */
class MyApplication : Application() {
    
    override fun onCreate() {
        super.onCreate()
        
        // 自定义配置（可选）
        LeakCanary.config = LeakCanary.config.copy(
            // 保留的泄漏数量
            retainedVisibleThreshold = 3,
            // 是否显示通知
            dumpHeap = true,
            // 自定义对象观察
            objectInspectors = LeakCanary.config.objectInspectors + 
                listOf(MyObjectInspector())
        )
    }
}

/**
 * 自定义对象检查器
 */
class MyObjectInspector : ObjectInspector {
    override fun inspect(
        reporter: ObjectReporter
    ) {
        reporter.whenInstanceOf("com.example.MyClass") { instance ->
            // 自定义检查逻辑
            reportLeakIfNeeded(instance)
        }
    }
}
```

### 4.2 线程数优化实践

#### 4.2.1 统一线程池管理

```kotlin
/**
 * 全局线程池管理器
 * 统一管理应用内所有线程池，避免线程数失控
 */
object ThreadPoolManager {
    
    // CPU 核心数
    private val CPU_COUNT = Runtime.getRuntime().availableProcessors()
    
    // 核心线程数：CPU 核心数 + 1
    private val CORE_POOL_SIZE = CPU_COUNT + 1
    
    // 最大线程数：CPU 核心数 * 2 + 1
    private val MAX_POOL_SIZE = CPU_COUNT * 2 + 1
    
    // 线程存活时间
    private const val KEEP_ALIVE_SECONDS = 30L
    
    /**
     * CPU 密集型任务线程池
     * 适用于计算密集型任务
     */
    val cpuExecutor: ThreadPoolExecutor by lazy {
        ThreadPoolExecutor(
            CORE_POOL_SIZE,
            MAX_POOL_SIZE,
            KEEP_ALIVE_SECONDS,
            TimeUnit.SECONDS,
            LinkedBlockingQueue(128),
            NamedThreadFactory("CPU"),
            ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略：调用者执行
        ).apply {
            allowCoreThreadTimeOut(true)  // 允许核心线程超时
        }
    }
    
    /**
     * IO 密集型任务线程池
     * 适用于网络请求、文件读写等
     */
    val ioExecutor: ThreadPoolExecutor by lazy {
        ThreadPoolExecutor(
            CORE_POOL_SIZE,
            MAX_POOL_SIZE * 2,  // IO 任务可以更多线程
            KEEP_ALIVE_SECONDS,
            TimeUnit.SECONDS,
            LinkedBlockingQueue(256),
            NamedThreadFactory("IO"),
            ThreadPoolExecutor.CallerRunsPolicy()
        ).apply {
            allowCoreThreadTimeOut(true)
        }
    }
    
    /**
     * 单线程执行器
     * 适用于需要顺序执行的任务
     */
    val singleExecutor: ExecutorService by lazy {
        Executors.newSingleThreadExecutor(NamedThreadFactory("Single"))
    }
    
    /**
     * 调度线程池
     * 适用于定时任务
     */
    val scheduledExecutor: ScheduledExecutorService by lazy {
        Executors.newScheduledThreadPool(2, NamedThreadFactory("Scheduled"))
    }
    
    /**
     * 自定义线程工厂
     */
    private class NamedThreadFactory(
        private val prefix: String
    ) : ThreadFactory {
        private val counter = AtomicInteger(0)
        
        override fun newThread(r: Runnable): Thread {
            return Thread(r, "$prefix-${counter.incrementAndGet()}").apply {
                isDaemon = true  // 设为守护线程
                priority = Thread.NORM_PRIORITY
            }
        }
    }
    
    /**
     * 获取线程池状态
     */
    fun getPoolStatus(): Map<String, PoolStatus> {
        return mapOf(
            "CPU" to getStatus(cpuExecutor),
            "IO" to getStatus(ioExecutor)
        )
    }
    
    private fun getStatus(executor: ThreadPoolExecutor): PoolStatus {
        return PoolStatus(
            corePoolSize = executor.corePoolSize,
            maximumPoolSize = executor.maximumPoolSize,
            activeCount = executor.activeCount,
            poolSize = executor.poolSize,
            queueSize = executor.queue.size,
            completedTaskCount = executor.completedTaskCount
        )
    }
    
    data class PoolStatus(
        val corePoolSize: Int,
        val maximumPoolSize: Int,
        val activeCount: Int,
        val poolSize: Int,
        val queueSize: Int,
        val completedTaskCount: Long
    )
}

// 使用示例
fun example() {
    // CPU 密集型任务
    ThreadPoolManager.cpuExecutor.execute {
        // 计算任务
    }
    
    // IO 密集型任务
    ThreadPoolManager.ioExecutor.execute {
        // 网络请求或文件操作
    }
    
    // 定时任务
    ThreadPoolManager.scheduledExecutor.scheduleAtFixedRate({
        // 定时执行
    }, 0, 1, TimeUnit.MINUTES)
}
```


#### 4.2.2 协程替代线程

```kotlin
/**
 * 使用协程替代传统线程
 * 协程更轻量，不会创建真正的系统线程
 */
class CoroutineExample {
    
    // 协程作用域，绑定生命周期
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    
    /**
     * 错误示例：每个任务创建新线程
     */
    fun badExample() {
        repeat(1000) { i ->
            Thread {
                // 执行任务
                Thread.sleep(1000)
            }.start()
        }
        // 可能导致线程数 OOM！
    }
    
    /**
     * 正确示例：使用协程
     */
    fun goodExample() {
        repeat(1000) { i ->
            scope.launch(Dispatchers.IO) {
                // 执行任务
                delay(1000)
            }
        }
        // 协程复用线程池，不会创建 1000 个线程
    }
    
    /**
     * 并发控制：限制并发数
     */
    fun limitedConcurrency() {
        val semaphore = Semaphore(10)  // 最多 10 个并发
        
        repeat(1000) { i ->
            scope.launch(Dispatchers.IO) {
                semaphore.withPermit {
                    // 执行任务
                    delay(1000)
                }
            }
        }
    }
    
    /**
     * 使用 Flow 处理大量数据
     */
    fun flowExample() {
        scope.launch {
            (1..1000).asFlow()
                .buffer(10)  // 缓冲区大小
                .map { processItem(it) }
                .collect { result ->
                    // 处理结果
                }
        }
    }
    
    private suspend fun processItem(item: Int): String {
        delay(100)
        return "Processed $item"
    }
    
    fun onDestroy() {
        scope.cancel()  // 取消所有协程
    }
}
```

### 4.3 FD 泄漏排查实践

#### 4.3.1 FD 泄漏排查流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    FD 泄漏排查流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 确认 FD 泄漏                                                │
│     adb shell ls -la /proc/<pid>/fd | wc -l                     │
│     观察 FD 数量是否持续增长                                     │
│                                                                 │
│  2. 分析 FD 类型分布                                            │
│     adb shell ls -la /proc/<pid>/fd                             │
│     统计各类型 FD 数量                                          │
│                                                                 │
│  3. 定位泄漏源                                                  │
│     • Socket 类型：检查网络连接代码                              │
│     • 文件类型：检查文件操作代码                                 │
│     • 数据库类型：检查 Cursor 和 DB 连接                         │
│                                                                 │
│  4. 代码审查                                                    │
│     • 搜索 FileInputStream/FileOutputStream                     │
│     • 搜索 Socket/ServerSocket                                  │
│     • 搜索 Cursor 使用                                          │
│     • 检查是否正确关闭                                          │
│                                                                 │
│  5. 修复验证                                                    │
│     • 修复后重新测试                                            │
│     • 观察 FD 数量是否稳定                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 FD 泄漏自动检测

```kotlin
/**
 * FD 泄漏自动检测工具
 * 通过 Hook 文件操作来检测未关闭的资源
 */
object FDLeakDetector {
    
    private val openedResources = ConcurrentHashMap<Int, ResourceInfo>()
    private const val TAG = "FDLeakDetector"
    
    /**
     * 记录资源打开
     */
    fun onResourceOpened(fd: Int, type: String, stackTrace: Array<StackTraceElement>) {
        openedResources[fd] = ResourceInfo(
            fd = fd,
            type = type,
            openTime = System.currentTimeMillis(),
            stackTrace = stackTrace.take(10).map { it.toString() }
        )
    }
    
    /**
     * 记录资源关闭
     */
    fun onResourceClosed(fd: Int) {
        openedResources.remove(fd)
    }
    
    /**
     * 检查长时间未关闭的资源
     */
    fun checkLeakedResources(thresholdMs: Long = 60_000): List<ResourceInfo> {
        val now = System.currentTimeMillis()
        return openedResources.values.filter { 
            now - it.openTime > thresholdMs 
        }
    }
    
    /**
     * 打印泄漏报告
     */
    fun printLeakReport() {
        val leaked = checkLeakedResources()
        if (leaked.isEmpty()) {
            Log.i(TAG, "No FD leaks detected")
            return
        }
        
        Log.e(TAG, "=== FD Leak Report ===")
        Log.e(TAG, "Total leaked: ${leaked.size}")
        
        leaked.groupBy { it.type }.forEach { (type, resources) ->
            Log.e(TAG, "\n$type: ${resources.size} leaks")
            resources.take(5).forEach { resource ->
                Log.e(TAG, "  FD ${resource.fd}, opened ${resource.getAgeSeconds()}s ago")
                resource.stackTrace.take(5).forEach { frame ->
                    Log.e(TAG, "    at $frame")
                }
            }
        }
    }
    
    data class ResourceInfo(
        val fd: Int,
        val type: String,
        val openTime: Long,
        val stackTrace: List<String>
    ) {
        fun getAgeSeconds(): Long = (System.currentTimeMillis() - openTime) / 1000
    }
}

/**
 * 包装 Closeable 以自动检测泄漏
 */
inline fun <T : Closeable, R> T.useWithLeakDetection(
    type: String,
    block: (T) -> R
): R {
    val fd = hashCode()  // 简化处理，实际应获取真实 FD
    val stackTrace = Thread.currentThread().stackTrace
    
    FDLeakDetector.onResourceOpened(fd, type, stackTrace)
    
    return try {
        block(this)
    } finally {
        close()
        FDLeakDetector.onResourceClosed(fd)
    }
}
```

### 4.4 大图加载优化

```kotlin
/**
 * 大图加载优化
 * 避免 Bitmap 导致的 OOM
 */
object BitmapOptimizer {
    
    /**
     * 计算采样率
     */
    fun calculateInSampleSize(
        options: BitmapFactory.Options,
        reqWidth: Int,
        reqHeight: Int
    ): Int {
        val (height, width) = options.outHeight to options.outWidth
        var inSampleSize = 1
        
        if (height > reqHeight || width > reqWidth) {
            val halfHeight = height / 2
            val halfWidth = width / 2
            
            while (halfHeight / inSampleSize >= reqHeight &&
                   halfWidth / inSampleSize >= reqWidth) {
                inSampleSize *= 2
            }
        }
        
        return inSampleSize
    }
    
    /**
     * 安全加载 Bitmap
     */
    fun decodeSampledBitmap(
        path: String,
        reqWidth: Int,
        reqHeight: Int
    ): Bitmap? {
        return try {
            // 1. 先获取图片尺寸（不加载到内存）
            val options = BitmapFactory.Options().apply {
                inJustDecodeBounds = true
            }
            BitmapFactory.decodeFile(path, options)
            
            // 2. 计算采样率
            options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight)
            
            // 3. 加载采样后的图片
            options.inJustDecodeBounds = false
            options.inPreferredConfig = Bitmap.Config.RGB_565  // 减少内存占用
            
            BitmapFactory.decodeFile(path, options)
        } catch (e: OutOfMemoryError) {
            Log.e("BitmapOptimizer", "OOM when decoding bitmap", e)
            System.gc()
            null
        }
    }
    
    /**
     * 使用 BitmapRegionDecoder 加载超大图局部
     */
    fun decodeRegion(
        path: String,
        rect: Rect
    ): Bitmap? {
        return try {
            val decoder = BitmapRegionDecoder.newInstance(path, false)
            val options = BitmapFactory.Options().apply {
                inPreferredConfig = Bitmap.Config.RGB_565
            }
            decoder?.decodeRegion(rect, options)
        } catch (e: Exception) {
            Log.e("BitmapOptimizer", "Failed to decode region", e)
            null
        }
    }
    
    /**
     * Bitmap 复用池
     */
    object BitmapPool {
        private val pool = LinkedList<Bitmap>()
        private const val MAX_SIZE = 10
        
        @Synchronized
        fun get(width: Int, height: Int, config: Bitmap.Config): Bitmap? {
            val iterator = pool.iterator()
            while (iterator.hasNext()) {
                val bitmap = iterator.next()
                if (bitmap.width == width && 
                    bitmap.height == height && 
                    bitmap.config == config) {
                    iterator.remove()
                    return bitmap
                }
            }
            return null
        }
        
        @Synchronized
        fun put(bitmap: Bitmap) {
            if (pool.size >= MAX_SIZE) {
                pool.removeFirst().recycle()
            }
            pool.add(bitmap)
        }
        
        @Synchronized
        fun clear() {
            pool.forEach { it.recycle() }
            pool.clear()
        }
    }
}
```


### 4.5 OOM 防护与兜底

```kotlin
/**
 * OOM 防护策略
 */
object OOMProtection {
    
    /**
     * 安全的内存分配
     * 在分配大内存前检查可用空间
     */
    inline fun <T> safeAllocate(
        requiredBytes: Long,
        crossinline allocator: () -> T,
        crossinline fallback: () -> T
    ): T {
        val runtime = Runtime.getRuntime()
        val availableMemory = runtime.maxMemory() - 
            (runtime.totalMemory() - runtime.freeMemory())
        
        return if (availableMemory > requiredBytes * 1.2) {  // 留 20% 余量
            try {
                allocator()
            } catch (e: OutOfMemoryError) {
                Log.e("OOMProtection", "OOM during allocation", e)
                System.gc()
                fallback()
            }
        } else {
            Log.w("OOMProtection", "Insufficient memory, using fallback")
            fallback()
        }
    }
    
    /**
     * 带重试的内存分配
     */
    inline fun <T> allocateWithRetry(
        maxRetries: Int = 3,
        crossinline allocator: () -> T
    ): T? {
        repeat(maxRetries) { attempt ->
            try {
                return allocator()
            } catch (e: OutOfMemoryError) {
                Log.w("OOMProtection", "OOM on attempt ${attempt + 1}, retrying...")
                System.gc()
                Thread.sleep(100L * (attempt + 1))
            }
        }
        return null
    }
    
    /**
     * 内存紧张时的降级策略
     */
    fun applyDegradation(level: DegradationLevel) {
        when (level) {
            DegradationLevel.LIGHT -> {
                // 轻度降级：降低图片质量
                ImageLoader.setQuality(Quality.MEDIUM)
            }
            DegradationLevel.MODERATE -> {
                // 中度降级：禁用动画、减少缓存
                AnimationManager.disable()
                CacheManager.reduceSize(0.5f)
            }
            DegradationLevel.SEVERE -> {
                // 重度降级：清空缓存、禁用非核心功能
                CacheManager.clear()
                FeatureManager.disableNonEssential()
            }
        }
    }
    
    enum class DegradationLevel {
        LIGHT, MODERATE, SEVERE
    }
}

/**
 * 全局 OOM 兜底处理
 */
class OOMCatcher : Thread.UncaughtExceptionHandler {
    
    private val defaultHandler = Thread.getDefaultUncaughtExceptionHandler()
    
    override fun uncaughtException(t: Thread, e: Throwable) {
        if (e is OutOfMemoryError) {
            // 1. 记录 OOM 信息
            logOOMInfo(e)
            
            // 2. 尝试释放内存
            emergencyMemoryRelease()
            
            // 3. 上报 OOM
            reportOOM(e)
            
            // 4. 可选：尝试恢复（谨慎使用）
            // attemptRecovery()
        }
        
        // 交给默认处理器
        defaultHandler?.uncaughtException(t, e)
    }
    
    private fun logOOMInfo(e: OutOfMemoryError) {
        val runtime = Runtime.getRuntime()
        Log.e("OOMCatcher", """
            OOM Caught!
            Message: ${e.message}
            Max Memory: ${runtime.maxMemory() / 1024 / 1024}MB
            Total Memory: ${runtime.totalMemory() / 1024 / 1024}MB
            Free Memory: ${runtime.freeMemory() / 1024 / 1024}MB
            Thread Count: ${Thread.getAllStackTraces().size}
        """.trimIndent())
    }
    
    private fun emergencyMemoryRelease() {
        try {
            // 清理所有缓存
            // 注意：此时内存可能已经非常紧张
        } catch (e: Exception) {
            // 忽略
        }
    }
    
    private fun reportOOM(e: OutOfMemoryError) {
        // 上报到监控平台
    }
}
```

### 4.6 OOM 监控体系设计

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OOM 监控体系架构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        数据采集层                                │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  • Java Heap 使用率（定时采集）                                  │   │
│  │  • Native 内存使用（通过 Debug.getNativeHeapSize）               │   │
│  │  • 线程数监控（/proc/self/status）                               │   │
│  │  • FD 数量监控（/proc/self/fd）                                  │   │
│  │  • PSS/RSS 监控（ActivityManager.getProcessMemoryInfo）          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        预警分析层                                │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  • 阈值告警（超过 80%/90% 触发）                                 │   │
│  │  • 趋势分析（内存持续增长检测）                                  │   │
│  │  • 异常检测（突增/突降检测）                                     │   │
│  │  • 关联分析（页面/操作与内存关联）                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        响应处理层                                │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  • 自动 GC 触发                                                  │   │
│  │  • 缓存清理                                                      │   │
│  │  • 功能降级                                                      │   │
│  │  • Heap Dump 采集                                                │   │
│  │  • 告警通知                                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        上报存储层                                │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  • 本地存储（SQLite/文件）                                       │   │
│  │  • 网络上报（压缩、采样、批量）                                  │   │
│  │  • 后端存储（时序数据库）                                        │   │
│  │  • 可视化展示（Grafana/自研平台）                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```


## 5. 常见面试题

### 5.1 OOM 类型与原理

**问题1：Android 中有哪些类型的 OOM？各自的触发条件是什么？**

**答案要点：**

1. **Java Heap OOM**
   - 触发条件：Java 堆内存使用超过 `Runtime.maxMemory()` 限制
   - 错误信息：`java.lang.OutOfMemoryError: Failed to allocate xxx bytes`
   - 常见原因：Bitmap 过大、内存泄漏、大数组分配

2. **Native OOM**
   - 触发条件：Native 层内存分配失败
   - 特点：不抛 Java 异常，通常直接 Crash 或被 LMK 杀死
   - 常见原因：Native 内存泄漏、图形内存过大、so 库内存问题

3. **线程数 OOM**
   - 触发条件：线程数超过系统限制或虚拟内存不足
   - 错误信息：`pthread_create (1040KB stack) failed: Try again`
   - 常见原因：无限制创建线程、线程池配置不当

4. **FD OOM**
   - 触发条件：文件描述符数量超过限制（通常 1024）
   - 错误信息：`Too many open files`
   - 常见原因：文件/Socket/Cursor 未关闭

5. **虚拟内存 OOM（32 位进程）**
   - 触发条件：虚拟地址空间耗尽（32 位约 3GB）
   - 常见原因：mmap 过多、地址空间碎片化

---

**问题2：Java Heap OOM 和 Native OOM 有什么区别？如何分别排查？**

**答案要点：**

| 对比项 | Java Heap OOM | Native OOM |
|-------|--------------|------------|
| 内存区域 | Java 堆（受 maxMemory 限制） | Native 堆（不受 Java 堆限制） |
| 异常类型 | 抛出 OutOfMemoryError | 通常直接 Crash 或被杀死 |
| 检测工具 | Memory Profiler、MAT、LeakCanary | malloc_debug、ASan、Perfetto |
| 常见原因 | Bitmap、内存泄漏、大对象 | Native 泄漏、图形内存、so 库 |
| Android 8.0+ | Bitmap 存储在 Native | Hardware Bitmap 默认 Native |

**排查方法：**

```kotlin
// Java Heap 排查
fun checkJavaHeap() {
    val runtime = Runtime.getRuntime()
    val used = runtime.totalMemory() - runtime.freeMemory()
    val max = runtime.maxMemory()
    Log.d("Memory", "Java Heap: ${used/1024/1024}MB / ${max/1024/1024}MB")
}

// Native 内存排查
fun checkNativeMemory() {
    val nativeHeap = Debug.getNativeHeapAllocatedSize()
    Log.d("Memory", "Native Heap: ${nativeHeap/1024/1024}MB")
}
```

---

**问题3：为什么 Android 8.0 之后 Bitmap 不容易导致 Java Heap OOM 了？**

**答案要点：**

1. **Android 8.0 之前**：
   - Bitmap 像素数据存储在 Java Heap
   - 受 `dalvik.vm.heapgrowthlimit` 限制
   - 容易触发 Java Heap OOM

2. **Android 8.0 之后**：
   - 引入 Hardware Bitmap，像素数据存储在 Native 内存
   - 不占用 Java Heap 空间
   - 但仍可能导致 Native OOM 或被 LMK 杀死

3. **源码变化**：
```java
// Android 8.0+ Bitmap 创建
// 默认使用 HARDWARE 配置，像素存储在 GPU 内存
Bitmap.Config.HARDWARE  // 新增配置
```

4. **注意事项**：
   - Hardware Bitmap 不能直接获取像素
   - 某些操作需要 copy 到 Software Bitmap
   - Native 内存仍需监控

---

### 5.2 线程数与 FD 优化

**问题4：如何排查和解决线程数 OOM？**

**答案要点：**

1. **排查方法**：
```bash
# 查看线程数
adb shell cat /proc/<pid>/status | grep Threads

# 查看线程详情
adb shell ls -la /proc/<pid>/task

# 代码中获取
Thread.getAllStackTraces().size
```

2. **常见原因**：
   - 无限制创建线程（`new Thread().start()`）
   - 线程池配置不当（`newCachedThreadPool` 无上限）
   - RxJava `Schedulers.newThread()` 滥用
   - 第三方 SDK 线程泄漏

3. **解决方案**：
   - 统一线程池管理
   - 使用协程替代线程
   - 限制最大线程数
   - 监控线程数量

4. **最佳实践**：
```kotlin
// 统一线程池
val executor = ThreadPoolExecutor(
    corePoolSize,
    maxPoolSize,  // 设置上限
    keepAliveTime,
    TimeUnit.SECONDS,
    LinkedBlockingQueue(capacity),  // 有界队列
    rejectedHandler  // 拒绝策略
)
```

---

**问题5：FD 泄漏如何排查？有哪些常见的泄漏场景？**

**答案要点：**

1. **排查方法**：
```bash
# 查看 FD 数量
adb shell ls /proc/<pid>/fd | wc -l

# 查看 FD 详情
adb shell ls -la /proc/<pid>/fd

# 查看 FD 限制
adb shell cat /proc/<pid>/limits | grep "open files"
```

2. **常见泄漏场景**：
   - `FileInputStream`/`FileOutputStream` 未关闭
   - `Cursor` 未关闭
   - `Socket` 未关闭
   - `ParcelFileDescriptor` 未关闭
   - 数据库连接未关闭

3. **解决方案**：
```kotlin
// 使用 use 扩展函数
FileInputStream(path).use { fis ->
    // 自动关闭
}

// Cursor 正确关闭
context.contentResolver.query(uri, null, null, null, null)?.use { cursor ->
    // 使用 cursor
}
```

4. **监控建议**：
   - 定期检查 FD 数量
   - 设置告警阈值（如 800）
   - 按类型统计 FD 分布

---

### 5.3 64 位适配与内存优化

**问题6：32 位和 64 位进程在内存方面有什么区别？为什么要做 64 位适配？**

**答案要点：**

1. **内存差异**：

| 对比项 | 32 位进程 | 64 位进程 |
|-------|----------|----------|
| 虚拟地址空间 | 4GB（用户空间约 3GB） | 256TB |
| 指针大小 | 4 字节 | 8 字节 |
| 对象头 | 8 字节 | 12-16 字节 |
| 线程栈上限 | 约 4000 个 | 几乎无限 |
| mmap 空间 | 受限 | 充足 |

2. **64 位适配原因**：
   - Google Play 强制要求（2019 年 8 月起）
   - 解决虚拟内存 OOM 问题
   - 更好的性能（更多寄存器、更大缓存）
   - 支持更大内存

3. **适配注意事项**：
   - 确保所有 so 库有 arm64-v8a 版本
   - 检查 JNI 代码中的指针操作
   - 测试内存占用变化（64 位略高）

---

**问题7：如何设计一个 OOM 监控系统？**

**答案要点：**

1. **监控指标**：
   - Java Heap 使用率
   - Native 内存使用
   - 线程数
   - FD 数量
   - PSS/RSS

2. **采集策略**：
```kotlin
// 定时采集
val handler = Handler(Looper.getMainLooper())
handler.postDelayed(object : Runnable {
    override fun run() {
        collectMemoryMetrics()
        handler.postDelayed(this, 5000)  // 5 秒采集一次
    }
}, 5000)
```

3. **告警阈值**：
   - Java Heap > 80%：警告
   - Java Heap > 90%：严重
   - 线程数 > 500：警告
   - FD 数 > 800：警告

4. **响应策略**：
   - 自动触发 GC
   - 清理缓存
   - 功能降级
   - 采集 Heap Dump
   - 上报告警

5. **数据分析**：
   - 趋势分析（内存是否持续增长）
   - 关联分析（哪个页面/操作导致内存增长）
   - 版本对比（新版本内存是否恶化）

---

### 5.4 OOM 实战排查

**问题8：线上出现 OOM，如何快速定位和解决？（字节/美团高频题）**

**答案要点：**

1. **信息收集**：
   - OOM 类型（Java Heap/Native/线程/FD）
   - 错误堆栈
   - 设备信息（内存大小、Android 版本）
   - 用户操作路径

2. **分析步骤**：
```
┌─────────────────────────────────────────────────────────────────┐
│                    OOM 排查流程                                  │
├─────────────────────────────────────────────────────────────────┤
│  1. 确定 OOM 类型                                               │
│     • 看错误信息判断是哪种 OOM                                   │
│                                                                 │
│  2. 分析堆栈                                                    │
│     • 找到触发 OOM 的代码位置                                    │
│     • 分析是分配问题还是泄漏问题                                 │
│                                                                 │
│  3. 复现问题                                                    │
│     • 根据用户路径复现                                          │
│     • 使用低内存设备测试                                        │
│                                                                 │
│  4. 采集 Heap Dump                                              │
│     • 使用 Memory Profiler 或 MAT 分析                          │
│     • 找出大对象和泄漏对象                                       │
│                                                                 │
│  5. 修复验证                                                    │
│     • 修复后压测验证                                            │
│     • 监控线上 OOM 率                                           │
└─────────────────────────────────────────────────────────────────┘
```

3. **常见解决方案**：
   - Bitmap 优化：采样、复用、及时回收
   - 内存泄漏：修复泄漏点
   - 线程优化：统一线程池
   - FD 优化：确保资源关闭

---

**问题9：如何优化图片加载避免 OOM？（快手/OPPO 高频题）**

**答案要点：**

1. **采样加载**：
```kotlin
fun decodeSampledBitmap(path: String, reqWidth: Int, reqHeight: Int): Bitmap? {
    val options = BitmapFactory.Options()
    options.inJustDecodeBounds = true
    BitmapFactory.decodeFile(path, options)
    
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight)
    options.inJustDecodeBounds = false
    options.inPreferredConfig = Bitmap.Config.RGB_565  // 减少内存
    
    return BitmapFactory.decodeFile(path, options)
}
```

2. **Bitmap 复用**：
```kotlin
options.inBitmap = reusableBitmap  // 复用已有 Bitmap
options.inMutable = true
```

3. **大图分块加载**：
```kotlin
val decoder = BitmapRegionDecoder.newInstance(path, false)
val region = decoder.decodeRegion(visibleRect, options)
```

4. **使用图片加载框架**：
   - Glide：自动管理内存、支持缓存
   - Coil：Kotlin 优先、协程支持

5. **内存缓存策略**：
   - LruCache 限制缓存大小
   - 根据可用内存动态调整

---

**问题10：OPPO/vivo 面试：从 Framework 层分析 OOM 的触发流程？**

**答案要点：**

1. **Java Heap OOM 触发流程**：
```
new Object()
    ↓
Heap::AllocObjectWithAllocator()  // ART 虚拟机
    ↓
TryToAllocate()  // 尝试分配
    ↓ 失败
CollectGarbageInternal()  // 触发 GC
    ↓ 仍然失败
AllocateInternalWithGc()  // 带 GC 的分配
    ↓ 多次 GC 后仍失败
ThrowOutOfMemoryError()  // 抛出 OOM
```

2. **关键源码位置**：
   - `art/runtime/gc/heap.cc`：堆内存管理
   - `art/runtime/thread.cc`：线程创建
   - `bionic/libc/bionic/pthread_create.cpp`：pthread 创建

3. **内存限制来源**：
```
dalvik.vm.heapgrowthlimit  // 普通应用堆限制
dalvik.vm.heapsize         // largeHeap 应用堆限制
```

4. **LMK 机制**：
   - 当系统内存不足时，LMK 根据 oom_adj 值杀死进程
   - 即使没有触发 OOM 异常，也可能被系统杀死

---

## 6. 总结

### 6.1 OOM 治理核心要点

1. **预防为主**：
   - 合理使用内存，避免大对象
   - 及时释放资源
   - 使用对象池和缓存

2. **监控预警**：
   - 建立完善的监控体系
   - 设置合理的告警阈值
   - 及时响应内存异常

3. **快速定位**：
   - 掌握各类 OOM 的特征
   - 熟练使用分析工具
   - 建立排查流程

4. **持续优化**：
   - 定期进行内存分析
   - 关注内存指标变化
   - 持续优化内存使用

### 6.2 工具清单

| 工具 | 用途 | 适用场景 |
|-----|------|---------|
| Memory Profiler | Java Heap 分析 | 开发调试 |
| MAT | Heap Dump 深度分析 | 泄漏排查 |
| LeakCanary | 自动检测泄漏 | 开发测试 |
| Perfetto | 系统级内存分析 | 性能优化 |
| ASan | Native 内存问题 | Native 开发 |

---

*文档版本: v1.0*  
*更新时间: 2024-01*  
*适用版本: Android W*
