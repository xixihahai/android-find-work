# Kotlin协程原理

## 1. 概述

Kotlin 协程是一种轻量级的并发解决方案，它允许以同步的方式编写异步代码，避免了回调地狱。协程不是线程，而是运行在线程上的可挂起计算，一个线程可以运行多个协程。本文将深入讲解协程的实现原理、调度机制和异常处理。

## 2. 核心原理

### 2.1 协程基础概念

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          协程基础概念                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  协程 vs 线程:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  特性              │  线程                │  协程                │   │
│  │  ─────────────────────────────────────────────────────────────  │   │
│  │  调度              │  操作系统            │  程序自身            │   │
│  │  切换开销          │  大 (上下文切换)     │  小 (状态机切换)     │   │
│  │  内存占用          │  ~1MB               │  ~几KB              │   │
│  │  数量限制          │  受系统限制          │  可创建大量          │   │
│  │  阻塞              │  阻塞线程            │  挂起，不阻塞线程    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  核心组件:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. suspend 函数   → 可挂起函数，协程的基本单元                  │   │
│  │  2. Continuation  → 续体，保存协程挂起时的状态                   │   │
│  │  3. CoroutineScope → 协程作用域，管理协程生命周期                │   │
│  │  4. CoroutineContext → 协程上下文，包含调度器、Job 等            │   │
│  │  5. Dispatcher    → 调度器，决定协程在哪个线程执行               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 协程构建器

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          协程构建器                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  launch - 启动协程，不返回结果:                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  val job = scope.launch {                                       │   │
│  │      // 协程代码                                                │   │
│  │  }                                                              │   │
│  │  job.join()  // 等待完成                                        │   │
│  │  job.cancel() // 取消协程                                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  async - 启动协程，返回 Deferred 结果:                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  val deferred = scope.async {                                   │   │
│  │      // 计算并返回结果                                          │   │
│  │      "result"                                                   │   │
│  │  }                                                              │   │
│  │  val result = deferred.await()  // 获取结果                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  runBlocking - 阻塞当前线程，等待协程完成:                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  fun main() = runBlocking {                                     │   │
│  │      // 阻塞 main 线程直到协程完成                              │   │
│  │      launch { delay(1000) }                                     │   │
│  │  }                                                              │   │
│  │  // 主要用于测试和 main 函数                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  withContext - 切换上下文执行:                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  suspend fun fetchData(): String {                              │   │
│  │      return withContext(Dispatchers.IO) {                       │   │
│  │          // 在 IO 线程执行                                      │   │
│  │          api.getData()                                          │   │
│  │      }                                                          │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Continuation 与状态机

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Continuation 与状态机原理                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  suspend 函数编译原理:                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 源代码                                                      │   │
│  │  suspend fun fetchUser(): User {                                │   │
│  │      val token = getToken()      // 挂起点 1                    │   │
│  │      val user = getUser(token)   // 挂起点 2                    │   │
│  │      return user                                                │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 编译后 (简化)                                               │   │
│  │  fun fetchUser(continuation: Continuation<User>): Any? {        │   │
│  │      // 状态机                                                  │   │
│  │      val sm = continuation as? FetchUserSM                      │   │
│  │          ?: FetchUserSM(continuation)                           │   │
│  │                                                                 │   │
│  │      when (sm.label) {                                          │   │
│  │          0 -> {                                                 │   │
│  │              sm.label = 1                                       │   │
│  │              val result = getToken(sm)                          │   │
│  │              if (result == COROUTINE_SUSPENDED)                 │   │
│  │                  return COROUTINE_SUSPENDED                     │   │
│  │              sm.token = result as String                        │   │
│  │          }                                                      │   │
│  │          1 -> {                                                 │   │
│  │              sm.label = 2                                       │   │
│  │              val result = getUser(sm.token, sm)                 │   │
│  │              if (result == COROUTINE_SUSPENDED)                 │   │
│  │                  return COROUTINE_SUSPENDED                     │   │
│  │              return result                                      │   │
│  │          }                                                      │   │
│  │      }                                                          │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  状态机执行流程:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用 fetchUser()                                               │   │
│  │        │                                                        │   │
│  │        ↓                                                        │   │
│  │  ┌──────────┐                                                   │   │
│  │  │ label=0  │ → 执行到 getToken()                               │   │
│  │  │          │ → 挂起，返回 COROUTINE_SUSPENDED                  │   │
│  │  └──────────┘                                                   │   │
│  │        │ (getToken 完成后回调)                                  │   │
│  │        ↓                                                        │   │
│  │  ┌──────────┐                                                   │   │
│  │  │ label=1  │ → 执行到 getUser()                                │   │
│  │  │          │ → 挂起，返回 COROUTINE_SUSPENDED                  │   │
│  │  └──────────┘                                                   │   │
│  │        │ (getUser 完成后回调)                                   │   │
│  │        ↓                                                        │   │
│  │  ┌──────────┐                                                   │   │
│  │  │ label=2  │ → 返回结果                                        │   │
│  │  └──────────┘                                                   │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 协程调度器

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          协程调度器 (Dispatchers)                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  内置调度器:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  调度器                │  线程                │  适用场景        │   │
│  │  ─────────────────────────────────────────────────────────────  │   │
│  │  Dispatchers.Main     │  主线程 (UI)         │  UI 操作         │   │
│  │  Dispatchers.IO       │  共享线程池 (64+)    │  IO 操作         │   │
│  │  Dispatchers.Default  │  共享线程池 (CPU核数)│  CPU 密集计算    │   │
│  │  Dispatchers.Unconfined│ 不切换线程          │  特殊场景        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  调度器工作原理:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  launch(Dispatchers.IO) {                                       │   │
│  │      // 1. 创建协程                                             │   │
│  │      // 2. 调用 dispatcher.dispatch(context, block)             │   │
│  │      // 3. dispatcher 将 block 提交到对应线程池                 │   │
│  │      // 4. 线程池中的线程执行 block                             │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  Dispatchers.IO 和 Dispatchers.Default 共享线程池               │   │
│  │  但 IO 可以创建更多线程 (默认 64 或 CPU 核数，取较大值)         │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Android 主线程调度器:                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Dispatchers.Main 在 Android 上使用 Handler 实现                │   │
│  │                                                                 │   │
│  │  internal class HandlerDispatcher(                              │   │
│  │      private val handler: Handler                               │   │
│  │  ) : CoroutineDispatcher() {                                    │   │
│  │      override fun dispatch(context: CoroutineContext, block: Runnable) {│
│  │          handler.post(block)  // 通过 Handler 发送到主线程      │   │
│  │      }                                                          │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.5 协程上下文与 Job

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    协程上下文 (CoroutineContext) 与 Job                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CoroutineContext 结构:                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  CoroutineContext 是一个 Element 的集合，类似 Map                │   │
│  │                                                                 │   │
│  │  常见 Element:                                                  │   │
│  │  - Job:              协程的生命周期管理                         │   │
│  │  - CoroutineDispatcher: 调度器                                  │   │
│  │  - CoroutineName:    协程名称 (调试用)                          │   │
│  │  - CoroutineExceptionHandler: 异常处理器                        │   │
│  │                                                                 │   │
│  │  // 组合上下文                                                  │   │
│  │  val context = Dispatchers.IO + CoroutineName("MyCoroutine")    │   │
│  │                                                                 │   │
│  │  // 获取元素                                                    │   │
│  │  val job = context[Job]                                         │   │
│  │  val dispatcher = context[CoroutineDispatcher]                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Job 生命周期:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │                    ┌─────────┐                                  │   │
│  │                    │  New    │ (创建)                           │   │
│  │                    └────┬────┘                                  │   │
│  │                         │ start()                               │   │
│  │                         ↓                                       │   │
│  │                    ┌─────────┐                                  │   │
│  │         ┌─────────→│ Active  │←─────────┐                       │   │
│  │         │          └────┬────┘          │                       │   │
│  │         │               │               │                       │   │
│  │    子协程完成      完成/取消        子协程完成                   │   │
│  │         │               │               │                       │   │
│  │         │               ↓               │                       │   │
│  │         │          ┌─────────┐          │                       │   │
│  │         └──────────│Completing│──────────┘                       │   │
│  │                    └────┬────┘                                  │   │
│  │                         │                                       │   │
│  │              ┌──────────┴──────────┐                            │   │
│  │              ↓                     ↓                            │   │
│  │         ┌─────────┐          ┌─────────┐                        │   │
│  │         │Completed│          │Cancelled│                        │   │
│  │         └─────────┘          └─────────┘                        │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Job 常用方法:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  job.start()      // 启动协程                                   │   │
│  │  job.cancel()     // 取消协程                                   │   │
│  │  job.join()       // 等待协程完成                               │   │
│  │  job.isActive     // 是否活跃                                   │   │
│  │  job.isCompleted  // 是否完成                                   │   │
│  │  job.isCancelled  // 是否取消                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.6 结构化并发

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          结构化并发                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  什么是结构化并发:                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 协程有明确的作用域和生命周期                                 │   │
│  │  - 父协程会等待所有子协程完成                                   │   │
│  │  - 父协程取消时，所有子协程也会被取消                           │   │
│  │  - 子协程异常会传播到父协程                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  父子协程关系:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  CoroutineScope (父)                                            │   │
│  │       │                                                         │   │
│  │       ├── launch (子1)                                          │   │
│  │       │      │                                                  │   │
│  │       │      ├── launch (孙1)                                   │   │
│  │       │      └── launch (孙2)                                   │   │
│  │       │                                                         │   │
│  │       └── async (子2)                                           │   │
│  │              │                                                  │   │
│  │              └── launch (孙3)                                   │   │
│  │                                                                 │   │
│  │  取消父协程 → 所有子协程和孙协程都会被取消                      │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  CoroutineScope:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 创建作用域                                                  │   │
│  │  val scope = CoroutineScope(Dispatchers.Main + Job())           │   │
│  │                                                                 │   │
│  │  // 在作用域中启动协程                                          │   │
│  │  scope.launch { /* ... */ }                                     │   │
│  │                                                                 │   │
│  │  // 取消作用域中的所有协程                                      │   │
│  │  scope.cancel()                                                 │   │
│  │                                                                 │   │
│  │  // Android 中使用 lifecycleScope 和 viewModelScope             │   │
│  │  lifecycleScope.launch { /* 生命周期感知 */ }                   │   │
│  │  viewModelScope.launch { /* ViewModel 生命周期 */ }             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.7 协程异常处理

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          协程异常处理                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  异常传播规则:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  launch:                                                        │   │
│  │  - 异常会立即传播到父协程                                       │   │
│  │  - 父协程会取消所有子协程                                       │   │
│  │  - 异常最终传递给 CoroutineExceptionHandler                     │   │
│  │                                                                 │   │
│  │  async:                                                         │   │
│  │  - 异常不会立即传播                                             │   │
│  │  - 异常在调用 await() 时抛出                                    │   │
│  │  - 可以用 try-catch 捕获                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  异常处理方式:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. try-catch (在协程内部)                                      │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  launch {                                                │   │   │
│  │  │      try {                                               │   │   │
│  │  │          riskyOperation()                                │   │   │
│  │  │      } catch (e: Exception) {                            │   │   │
│  │  │          // 处理异常                                     │   │   │
│  │  │      }                                                   │   │   │
│  │  │  }                                                       │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  2. CoroutineExceptionHandler (全局处理)                        │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  val handler = CoroutineExceptionHandler { _, e ->       │   │   │
│  │  │      Log.e("Coroutine", "Exception: $e")                 │   │   │
│  │  │  }                                                       │   │   │
│  │  │  scope.launch(handler) {                                 │   │   │
│  │  │      throw RuntimeException("Error")                     │   │   │
│  │  │  }                                                       │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  3. SupervisorJob (隔离子协程异常)                              │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │  val scope = CoroutineScope(SupervisorJob())             │   │   │
│  │  │  scope.launch { throw Exception() }  // 不影响其他子协程 │   │   │
│  │  │  scope.launch { /* 继续执行 */ }                         │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. 关键源码解析

### 3.1 Continuation 接口

```kotlin
/**
 * Continuation 接口 - 协程的核心
 * 代表协程挂起后的续体
 */
public interface Continuation<in T> {
    /**
     * 协程上下文
     */
    public val context: CoroutineContext
    
    /**
     * 恢复协程执行
     * @param result 挂起点的返回值
     */
    public fun resumeWith(result: Result<T>)
}

// 扩展函数，简化调用
public inline fun <T> Continuation<T>.resume(value: T) {
    resumeWith(Result.success(value))
}

public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable) {
    resumeWith(Result.failure(exception))
}
```

### 3.2 suspend 函数编译后的代码

```kotlin
/**
 * 原始 suspend 函数
 */
suspend fun fetchUserData(): String {
    val token = fetchToken()      // 挂起点 1
    val user = fetchUser(token)   // 挂起点 2
    return user.name
}

/**
 * 编译后的代码 (简化版)
 * 编译器生成状态机
 */
fun fetchUserData(continuation: Continuation<String>): Any? {
    // 状态机类
    class FetchUserDataSM(
        completion: Continuation<String>
    ) : ContinuationImpl(completion) {
        var result: Any? = null
        var label: Int = 0
        
        // 保存中间变量
        var token: String? = null
        
        override fun invokeSuspend(result: Result<Any?>): Any? {
            this.result = result
            this.label = this.label or Int.MIN_VALUE
            return fetchUserData(this)
        }
    }
    
    // 获取或创建状态机
    val sm = continuation as? FetchUserDataSM
        ?: FetchUserDataSM(continuation)
    
    // 状态机执行
    when (sm.label) {
        0 -> {
            // 初始状态
            sm.label = 1
            val result = fetchToken(sm)
            if (result == COROUTINE_SUSPENDED) {
                return COROUTINE_SUSPENDED
            }
            sm.token = result as String
        }
        1 -> {
            // fetchToken 完成后
            sm.token = sm.result as String
            sm.label = 2
            val result = fetchUser(sm.token!!, sm)
            if (result == COROUTINE_SUSPENDED) {
                return COROUTINE_SUSPENDED
            }
            return (result as User).name
        }
        2 -> {
            // fetchUser 完成后
            val user = sm.result as User
            return user.name
        }
        else -> throw IllegalStateException("Invalid state")
    }
    
    // 继续执行下一个状态
    return fetchUserData(sm)
}
```

### 3.3 launch 实现原理

```kotlin
/**
 * launch 函数源码 (简化)
 */
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    // 1. 创建新的上下文，合并父上下文
    val newContext = newCoroutineContext(context)
    
    // 2. 创建协程对象
    val coroutine = if (start.isLazy) {
        LazyStandaloneCoroutine(newContext, block)
    } else {
        StandaloneCoroutine(newContext, active = true)
    }
    
    // 3. 启动协程
    coroutine.start(start, coroutine, block)
    
    return coroutine
}

/**
 * StandaloneCoroutine 类
 */
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, active) {
    
    override fun handleJobException(exception: Throwable): Boolean {
        // 处理异常，传递给 CoroutineExceptionHandler
        handleCoroutineException(context, exception)
        return true
    }
}

/**
 * 协程启动
 */
internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(
    receiver: R,
    completion: Continuation<T>
) {
    // 创建 Continuation
    val cont = createCoroutineUnintercepted(receiver, completion)
    // 拦截并调度
    cont.intercepted().resumeCancellableWith(Result.success(Unit))
}
```

### 3.4 Dispatchers.IO 实现

```kotlin
/**
 * Dispatchers.IO 实现原理
 */
public actual val IO: CoroutineDispatcher = DefaultScheduler.IO

internal object DefaultScheduler : ExperimentalCoroutineDispatcher() {
    
    // IO 调度器，共享线程池但允许更多线程
    val IO: CoroutineDispatcher = LimitingDispatcher(
        this,
        systemProp(IO_PARALLELISM_PROPERTY_NAME, 64.coerceAtLeast(AVAILABLE_PROCESSORS)),
        "Dispatchers.IO",
        TASK_PROBABLY_BLOCKING
    )
}

/**
 * LimitingDispatcher - 限制并发数的调度器
 */
private class LimitingDispatcher(
    private val dispatcher: ExperimentalCoroutineDispatcher,
    private val parallelism: Int,
    private val name: String?,
    override val taskMode: Int
) : ExecutorCoroutineDispatcher(), TaskContext, Executor {
    
    // 等待队列
    private val queue = ConcurrentLinkedQueue<Runnable>()
    
    // 当前运行的任务数
    private val inFlightTasks = atomic(0)
    
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        // 将任务加入队列
        queue.add(block)
        // 尝试分发
        tryDispatch()
    }
    
    private fun tryDispatch() {
        while (true) {
            val current = inFlightTasks.value
            if (current >= parallelism) {
                // 达到并发限制，等待
                return
            }
            if (inFlightTasks.compareAndSet(current, current + 1)) {
                // 从队列取出任务执行
                val task = queue.poll() ?: run {
                    inFlightTasks.decrementAndGet()
                    return
                }
                dispatcher.dispatchWithContext(task, this, false)
                return
            }
        }
    }
}
```

## 4. 实战应用

### 4.1 Android 中使用协程

```kotlin
/**
 * ViewModel 中使用协程
 */
class UserViewModel : ViewModel() {
    
    private val _user = MutableLiveData<User>()
    val user: LiveData<User> = _user
    
    private val _loading = MutableLiveData<Boolean>()
    val loading: LiveData<Boolean> = _loading
    
    private val _error = MutableLiveData<String>()
    val error: LiveData<String> = _error
    
    /**
     * 使用 viewModelScope
     * ViewModel 销毁时自动取消协程
     */
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _loading.value = true
            try {
                // 切换到 IO 线程执行网络请求
                val user = withContext(Dispatchers.IO) {
                    userRepository.getUser(userId)
                }
                // 自动切回主线程更新 UI
                _user.value = user
            } catch (e: Exception) {
                _error.value = e.message
            } finally {
                _loading.value = false
            }
        }
    }
    
    /**
     * 并行请求
     */
    fun loadUserAndPosts(userId: String) {
        viewModelScope.launch {
            try {
                // 并行执行两个请求
                val userDeferred = async(Dispatchers.IO) { 
                    userRepository.getUser(userId) 
                }
                val postsDeferred = async(Dispatchers.IO) { 
                    postRepository.getPosts(userId) 
                }
                
                // 等待两个请求都完成
                val user = userDeferred.await()
                val posts = postsDeferred.await()
                
                // 更新 UI
                _user.value = user.copy(posts = posts)
            } catch (e: Exception) {
                _error.value = e.message
            }
        }
    }
}
```

### 4.2 协程与 Retrofit

```kotlin
/**
 * Retrofit 接口定义
 */
interface ApiService {
    // suspend 函数，Retrofit 2.6+ 支持
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
    
    @GET("users/{id}/posts")
    suspend fun getUserPosts(@Path("id") id: String): List<Post>
}

/**
 * Repository 层
 */
class UserRepository(private val api: ApiService) {
    
    /**
     * 简单请求
     */
    suspend fun getUser(id: String): User {
        return api.getUser(id)
    }
    
    /**
     * 带重试的请求
     */
    suspend fun getUserWithRetry(id: String, maxRetries: Int = 3): User {
        var lastException: Exception? = null
        repeat(maxRetries) { attempt ->
            try {
                return api.getUser(id)
            } catch (e: Exception) {
                lastException = e
                // 指数退避
                delay((1000L * (attempt + 1)))
            }
        }
        throw lastException ?: Exception("Unknown error")
    }
    
    /**
     * 带超时的请求
     */
    suspend fun getUserWithTimeout(id: String): User {
        return withTimeout(5000L) {
            api.getUser(id)
        }
    }
}
```

### 4.3 协程取消与超时

```kotlin
/**
 * 协程取消示例
 */
class DownloadManager {
    
    private var downloadJob: Job? = null
    
    /**
     * 开始下载
     */
    fun startDownload(url: String, onProgress: (Int) -> Unit) {
        downloadJob = CoroutineScope(Dispatchers.IO).launch {
            try {
                for (progress in 0..100) {
                    // 检查是否被取消
                    ensureActive()
                    
                    // 模拟下载
                    delay(100)
                    
                    // 切换到主线程更新进度
                    withContext(Dispatchers.Main) {
                        onProgress(progress)
                    }
                }
            } catch (e: CancellationException) {
                // 协程被取消
                Log.d("Download", "Download cancelled")
            }
        }
    }
    
    /**
     * 取消下载
     */
    fun cancelDownload() {
        downloadJob?.cancel()
    }
}

/**
 * 超时处理
 */
suspend fun fetchDataWithTimeout(): Result<Data> {
    return try {
        // 5 秒超时
        val data = withTimeout(5000L) {
            api.fetchData()
        }
        Result.success(data)
    } catch (e: TimeoutCancellationException) {
        Result.failure(Exception("Request timeout"))
    }
}

/**
 * 可取消的超时 (超时后返回 null)
 */
suspend fun fetchDataOrNull(): Data? {
    return withTimeoutOrNull(5000L) {
        api.fetchData()
    }
}
```

### 4.4 常见坑点

1. **忘记切换线程**：在主线程执行耗时操作会导致 ANR
2. **协程泄漏**：没有正确取消协程，导致内存泄漏
3. **异常处理不当**：launch 中的异常会传播到父协程
4. **在 suspend 函数中使用 GlobalScope**：破坏结构化并发
5. **阻塞调用**：在协程中使用 Thread.sleep() 而不是 delay()

## 5. 常见面试题

### 问题1：协程的原理是什么？

**答案要点**：
- 协程是基于 **Continuation** 和 **状态机** 实现的
- suspend 函数编译后会生成状态机代码
- 每个挂起点对应一个状态
- 挂起时保存状态，恢复时从保存的状态继续执行
- 协程不是线程，是运行在线程上的可挂起计算

### 问题2：launch 和 async 的区别？

**答案要点**：
| 特性 | launch | async |
|------|--------|-------|
| 返回值 | Job | Deferred<T> |
| 获取结果 | 不能 | await() |
| 异常传播 | 立即传播 | await() 时抛出 |
| 用途 | 不需要返回值 | 需要返回值 |

### 问题3：协程的调度器有哪些？

**答案要点**：
- **Dispatchers.Main**：主线程，用于 UI 操作
- **Dispatchers.IO**：IO 线程池，用于网络、文件操作
- **Dispatchers.Default**：CPU 密集型计算
- **Dispatchers.Unconfined**：不切换线程，在当前线程执行

### 问题4：什么是结构化并发？

**答案要点**：
- 协程有明确的作用域和生命周期
- 父协程会等待所有子协程完成
- 父协程取消时，所有子协程也会被取消
- 子协程异常会传播到父协程
- 通过 CoroutineScope 管理协程生命周期

### 问题5：协程如何处理异常？

**答案要点**：
1. **try-catch**：在协程内部捕获异常
2. **CoroutineExceptionHandler**：全局异常处理器
3. **SupervisorJob**：隔离子协程异常，一个子协程失败不影响其他
4. **async + await**：异常在 await() 时抛出，可以 try-catch

### 问题6：withContext 和 async 的区别？

**答案要点**：
| 特性 | withContext | async |
|------|-------------|-------|
| 执行方式 | 串行，挂起当前协程 | 并行，不挂起 |
| 返回值 | 直接返回结果 | 返回 Deferred |
| 用途 | 切换上下文执行单个任务 | 并行执行多个任务 |
| 异常 | 直接抛出 | await() 时抛出 |

### 问题7：如何避免协程泄漏？

**答案要点**：
1. 使用 **lifecycleScope** 或 **viewModelScope**
2. 在 onDestroy/onCleared 中取消协程
3. 使用结构化并发，不要使用 GlobalScope
4. 正确处理协程取消（检查 isActive，使用 ensureActive()）

### 问题8：suspend 函数可以在哪里调用？

**答案要点**：
- 在另一个 suspend 函数中
- 在协程构建器中（launch、async、runBlocking）
- 在 Continuation.resumeWith() 中
- 不能在普通函数中直接调用
