# Kotlin 协程进阶

## 1. 概述

本文深入探讨 Kotlin 协程的高级特性，包括结构化并发、异常处理、Channel、Flow 背压等进阶内容。

### 协程 vs 线程

| 特性 | 线程 | 协程 |
|------|------|------|
| 调度 | 操作系统调度 | 协程调度器调度 |
| 切换开销 | 大（上下文切换） | 小（用户态切换） |
| 内存占用 | 约 1MB 栈空间 | 约几 KB |
| 数量限制 | 受系统限制 | 可创建大量 |
| 阻塞 | 阻塞线程 | 挂起，不阻塞线程 |
| 取消 | 需要手动处理 | 结构化取消 |

## 2. 核心原理

### 2.1 CoroutineScope 与生命周期

```kotlin
// CoroutineScope 定义
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}

// Android 中的 Scope
class MainActivity : AppCompatActivity() {
    
    // 1. lifecycleScope - 与 Activity 生命周期绑定
    fun useLifecycleScope() {
        lifecycleScope.launch {
            // Activity 销毁时自动取消
        }
        
        // 在特定生命周期状态执行
        lifecycleScope.launchWhenStarted {
            // STARTED 状态执行，STOPPED 时暂停
        }
        
        // 推荐：repeatOnLifecycle
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // STARTED 时执行，STOPPED 时取消
                // 重新进入 STARTED 时重新执行
            }
        }
    }
}

class MyViewModel : ViewModel() {
    
    // 2. viewModelScope - 与 ViewModel 生命周期绑定
    fun useViewModelScope() {
        viewModelScope.launch {
            // ViewModel 清除时自动取消
        }
    }
}

// 3. 自定义 Scope
class MyRepository {
    private val scope = CoroutineScope(
        SupervisorJob() + Dispatchers.IO + CoroutineName("MyRepository")
    )
    
    fun doWork() {
        scope.launch {
            // 执行任务
        }
    }
    
    fun cancel() {
        scope.cancel()
    }
}
```

### 2.2 Job 与 SupervisorJob

```kotlin
// Job 层级关系
/*
┌─────────────────────────────────────────────────────────────┐
│                      Job 层级结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                    ┌──────────┐                             │
│                    │ Parent   │                             │
│                    │   Job    │                             │
│                    └────┬─────┘                             │
│                         │                                   │
│           ┌─────────────┼─────────────┐                    │
│           │             │             │                    │
│           ▼             ▼             ▼                    │
│      ┌────────┐    ┌────────┐    ┌────────┐               │
│      │ Child1 │    │ Child2 │    │ Child3 │               │
│      │  Job   │    │  Job   │    │  Job   │               │
│      └────────┘    └────────┘    └────────┘               │
│                                                             │
│   Job: 子协程异常会取消父协程和所有兄弟协程                    │
│   SupervisorJob: 子协程异常不影响父协程和兄弟协程              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
*/

// Job 示例：一个失败，全部取消
fun jobExample() = runBlocking {
    val job = Job()
    val scope = CoroutineScope(job + Dispatchers.Default)
    
    scope.launch {
        delay(1000)
        println("Child 1 completed")
    }
    
    scope.launch {
        delay(500)
        throw RuntimeException("Child 2 failed")  // 会取消 Child 1
    }
    
    job.join()
}

// SupervisorJob 示例：一个失败，其他继续
fun supervisorJobExample() = runBlocking {
    val supervisor = SupervisorJob()
    val scope = CoroutineScope(supervisor + Dispatchers.Default)
    
    scope.launch {
        delay(1000)
        println("Child 1 completed")  // 会正常执行
    }
    
    scope.launch {
        delay(500)
        throw RuntimeException("Child 2 failed")  // 不影响 Child 1
    }
    
    delay(2000)
    supervisor.cancel()
}

// supervisorScope
suspend fun supervisorScopeExample() {
    supervisorScope {
        launch {
            delay(1000)
            println("Child 1 completed")
        }
        
        launch {
            delay(500)
            throw RuntimeException("Child 2 failed")
        }
    }
}
```

### 2.3 协程取消与超时

```kotlin
// 协程取消
class CancellationExample {
    
    // 1. 取消协程
    fun cancelCoroutine() = runBlocking {
        val job = launch {
            repeat(1000) { i ->
                println("Job: $i")
                delay(500)
            }
        }
        
        delay(1300)
        println("Cancelling...")
        job.cancel()  // 取消协程
        job.join()    // 等待取消完成
        // 或者 job.cancelAndJoin()
        println("Done")
    }
    
    // 2. 检查取消状态
    fun checkCancellation() = runBlocking {
        val job = launch(Dispatchers.Default) {
            var i = 0
            // 方式1：检查 isActive
            while (isActive) {
                println("Job: ${i++}")
            }
            
            // 方式2：使用 ensureActive()
            while (true) {
                ensureActive()  // 如果已取消，抛出 CancellationException
                println("Job: ${i++}")
            }
            
            // 方式3：使用 yield()
            while (true) {
                yield()  // 让出执行权，检查取消
                println("Job: ${i++}")
            }
        }
        
        delay(1000)
        job.cancelAndJoin()
    }
    
    // 3. 不可取消的代码块
    fun nonCancellable() = runBlocking {
        val job = launch {
            try {
                repeat(1000) { i ->
                    println("Job: $i")
                    delay(500)
                }
            } finally {
                // 在 finally 中执行清理
                withContext(NonCancellable) {
                    // 即使协程被取消，这里的代码也会执行
                    delay(1000)
                    println("Cleanup done")
                }
            }
        }
        
        delay(1300)
        job.cancelAndJoin()
    }
    
    // 4. 超时
    fun timeoutExample() = runBlocking {
        // withTimeout - 超时抛出 TimeoutCancellationException
        try {
            withTimeout(1300) {
                repeat(1000) { i ->
                    println("Job: $i")
                    delay(500)
                }
            }
        } catch (e: TimeoutCancellationException) {
            println("Timeout!")
        }
        
        // withTimeoutOrNull - 超时返回 null
        val result = withTimeoutOrNull(1300) {
            repeat(1000) { i ->
                println("Job: $i")
                delay(500)
            }
            "Done"
        }
        println("Result: $result")  // null
    }
}
```

### 2.4 协程异常处理

```kotlin
class ExceptionHandlingExample {
    
    // 1. try-catch
    fun tryCatchExample() = runBlocking {
        val job = launch {
            try {
                throw RuntimeException("Error")
            } catch (e: Exception) {
                println("Caught: $e")
            }
        }
        job.join()
    }
    
    // 2. CoroutineExceptionHandler
    fun exceptionHandlerExample() = runBlocking {
        val handler = CoroutineExceptionHandler { _, exception ->
            println("Caught: $exception")
        }
        
        val scope = CoroutineScope(SupervisorJob() + handler)
        
        scope.launch {
            throw RuntimeException("Error")
        }
        
        delay(1000)
    }
    
    // 3. async 异常处理
    fun asyncExceptionExample() = runBlocking {
        // async 的异常在 await() 时抛出
        val deferred = async {
            throw RuntimeException("Error")
        }
        
        try {
            deferred.await()
        } catch (e: Exception) {
            println("Caught: $e")
        }
        
        // supervisorScope 中的 async
        supervisorScope {
            val deferred1 = async {
                delay(1000)
                "Result 1"
            }
            
            val deferred2 = async {
                throw RuntimeException("Error")
            }
            
            try {
                println(deferred2.await())
            } catch (e: Exception) {
                println("Caught: $e")
            }
            
            println(deferred1.await())  // 正常执行
        }
    }
    
    // 4. 异常聚合
    fun aggregatedExceptionExample() = runBlocking {
        val handler = CoroutineExceptionHandler { _, exception ->
            println("Caught: $exception")
            exception.suppressed.forEach {
                println("Suppressed: $it")
            }
        }
        
        val job = GlobalScope.launch(handler) {
            launch {
                try {
                    delay(Long.MAX_VALUE)
                } finally {
                    throw RuntimeException("Child 1 failed")
                }
            }
            
            launch {
                delay(100)
                throw RuntimeException("Child 2 failed")
            }
        }
        
        job.join()
    }
}
```

### 2.5 Channel

```kotlin
// Channel 是协程间通信的管道
class ChannelExample {
    
    // 1. 基本使用
    fun basicChannel() = runBlocking {
        val channel = Channel<Int>()
        
        // 生产者
        launch {
            for (x in 1..5) {
                channel.send(x)
            }
            channel.close()
        }
        
        // 消费者
        for (y in channel) {
            println("Received: $y")
        }
    }
    
    // 2. Channel 类型
    fun channelTypes() = runBlocking {
        // 无缓冲 Channel（默认）- 发送和接收必须同时准备好
        val rendezvousChannel = Channel<Int>()
        
        // 缓冲 Channel - 有固定容量
        val bufferedChannel = Channel<Int>(10)
        
        // 无限容量 Channel
        val unlimitedChannel = Channel<Int>(Channel.UNLIMITED)
        
        // 合并 Channel - 只保留最新值
        val conflatedChannel = Channel<Int>(Channel.CONFLATED)
    }
    
    // 3. produce 构建器
    fun produceExample() = runBlocking {
        val numbers = produce {
            for (x in 1..5) {
                send(x)
            }
        }
        
        numbers.consumeEach { println(it) }
    }
    
    // 4. 扇出（Fan-out）- 多个消费者
    fun fanOutExample() = runBlocking {
        val producer = produce {
            repeat(10) {
                send(it)
                delay(100)
            }
        }
        
        // 多个消费者
        repeat(3) { consumerId ->
            launch {
                for (msg in producer) {
                    println("Consumer $consumerId received $msg")
                }
            }
        }
    }
    
    // 5. 扇入（Fan-in）- 多个生产者
    fun fanInExample() = runBlocking {
        val channel = Channel<String>()
        
        // 多个生产者
        launch { sendString(channel, "foo", 200) }
        launch { sendString(channel, "bar", 500) }
        
        repeat(6) {
            println(channel.receive())
        }
        
        coroutineContext.cancelChildren()
    }
    
    private suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
        while (true) {
            delay(time)
            channel.send(s)
        }
    }
    
    // 6. 管道（Pipeline）
    fun pipelineExample() = runBlocking {
        val numbers = produce {
            for (x in 1..10) send(x)
        }
        
        val squares = produce {
            for (x in numbers) send(x * x)
        }
        
        val filtered = produce {
            for (x in squares) {
                if (x > 25) send(x)
            }
        }
        
        filtered.consumeEach { println(it) }
    }
}
```

### 2.6 Select 表达式

```kotlin
class SelectExample {
    
    // 1. 选择多个 Channel
    fun selectFromChannels() = runBlocking {
        val channel1 = produce {
            repeat(5) {
                delay(100)
                send("Channel1: $it")
            }
        }
        
        val channel2 = produce {
            repeat(5) {
                delay(150)
                send("Channel2: $it")
            }
        }
        
        repeat(10) {
            select<Unit> {
                channel1.onReceive { value ->
                    println(value)
                }
                channel2.onReceive { value ->
                    println(value)
                }
            }
        }
    }
    
    // 2. 选择发送
    fun selectOnSend() = runBlocking {
        val channel1 = Channel<String>()
        val channel2 = Channel<String>()
        
        launch {
            for (s in channel1) println("Channel1: $s")
        }
        launch {
            for (s in channel2) println("Channel2: $s")
        }
        
        repeat(10) { i ->
            select<Unit> {
                channel1.onSend("Message $i") {}
                channel2.onSend("Message $i") {}
            }
        }
        
        channel1.close()
        channel2.close()
    }
    
    // 3. 选择 Deferred
    fun selectDeferred() = runBlocking {
        val deferred1 = async {
            delay(100)
            "Result 1"
        }
        
        val deferred2 = async {
            delay(200)
            "Result 2"
        }
        
        val result = select<String> {
            deferred1.onAwait { it }
            deferred2.onAwait { it }
        }
        
        println("First result: $result")
    }
}
```

### 2.7 Mutex 与并发安全

```kotlin
class ConcurrencySafetyExample {
    
    // 1. Mutex - 互斥锁
    fun mutexExample() = runBlocking {
        val mutex = Mutex()
        var counter = 0
        
        val jobs = List(100) {
            launch(Dispatchers.Default) {
                repeat(1000) {
                    mutex.withLock {
                        counter++
                    }
                }
            }
        }
        
        jobs.forEach { it.join() }
        println("Counter: $counter")  // 100000
    }
    
    // 2. 线程安全的数据结构
    fun threadSafeCollections() = runBlocking {
        // 使用 ConcurrentHashMap
        val map = ConcurrentHashMap<String, Int>()
        
        // 使用 AtomicInteger
        val counter = AtomicInteger(0)
        
        val jobs = List(100) {
            launch(Dispatchers.Default) {
                repeat(1000) {
                    counter.incrementAndGet()
                }
            }
        }
        
        jobs.forEach { it.join() }
        println("Counter: ${counter.get()}")
    }
    
    // 3. 线程限制
    fun threadConfinement() = runBlocking {
        // 使用单线程调度器
        val counterContext = newSingleThreadContext("CounterContext")
        var counter = 0
        
        val jobs = List(100) {
            launch(Dispatchers.Default) {
                repeat(1000) {
                    withContext(counterContext) {
                        counter++
                    }
                }
            }
        }
        
        jobs.forEach { it.join() }
        println("Counter: $counter")
        counterContext.close()
    }
    
    // 4. Actor 模式
    sealed class CounterMsg
    object IncCounter : CounterMsg()
    class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg()
    
    fun actorExample() = runBlocking {
        val counter = actor<CounterMsg> {
            var count = 0
            for (msg in channel) {
                when (msg) {
                    is IncCounter -> count++
                    is GetCounter -> msg.response.complete(count)
                }
            }
        }
        
        val jobs = List(100) {
            launch(Dispatchers.Default) {
                repeat(1000) {
                    counter.send(IncCounter)
                }
            }
        }
        
        jobs.forEach { it.join() }
        
        val response = CompletableDeferred<Int>()
        counter.send(GetCounter(response))
        println("Counter: ${response.await()}")
        
        counter.close()
    }
}
```

### 2.8 Flow 背压处理

```kotlin
class FlowBackpressureExample {
    
    // 1. buffer - 缓冲
    fun bufferExample() = runBlocking {
        val time = measureTimeMillis {
            flow {
                for (i in 1..3) {
                    delay(100)  // 生产耗时
                    emit(i)
                }
            }
            .buffer()  // 缓冲，生产和消费并行
            .collect { value ->
                delay(300)  // 消费耗时
                println(value)
            }
        }
        println("Time: $time ms")  // ~1000ms（无 buffer 约 1200ms）
    }
    
    // 2. conflate - 合并，只处理最新值
    fun conflateExample() = runBlocking {
        val time = measureTimeMillis {
            flow {
                for (i in 1..3) {
                    delay(100)
                    emit(i)
                }
            }
            .conflate()  // 跳过中间值
            .collect { value ->
                delay(300)
                println(value)
            }
        }
        println("Time: $time ms")
    }
    
    // 3. collectLatest - 只处理最新值，取消之前的处理
    fun collectLatestExample() = runBlocking {
        val time = measureTimeMillis {
            flow {
                for (i in 1..3) {
                    delay(100)
                    emit(i)
                }
            }
            .collectLatest { value ->
                println("Collecting $value")
                delay(300)
                println("Done $value")
            }
        }
        println("Time: $time ms")
    }
    
    // 4. 自定义背压策略
    fun customBackpressure() = runBlocking {
        flow {
            for (i in 1..100) {
                emit(i)
            }
        }
        .buffer(capacity = 10, onBufferOverflow = BufferOverflow.DROP_OLDEST)
        .collect { value ->
            delay(100)
            println(value)
        }
    }
}
```

## 3. 关键源码解析

### 3.1 CoroutineScope 源码

```kotlin
// CoroutineScope.kt
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}

// 创建 CoroutineScope
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())

internal class ContextScope(context: CoroutineContext) : CoroutineScope {
    override val coroutineContext: CoroutineContext = context
    override fun toString(): String = "CoroutineScope(coroutineContext=$coroutineContext)"
}

// MainScope - 主线程 Scope
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```

### 3.2 Job 源码

```kotlin
// Job.kt
public interface Job : CoroutineContext.Element {
    public companion object Key : CoroutineContext.Key<Job>
    
    // 状态
    public val isActive: Boolean
    public val isCompleted: Boolean
    public val isCancelled: Boolean
    
    // 取消
    public fun cancel(cause: CancellationException? = null)
    
    // 等待
    public suspend fun join()
    
    // 子 Job
    public val children: Sequence<Job>
}

// SupervisorJob
public fun SupervisorJob(parent: Job? = null): CompletableJob = 
    SupervisorJobImpl(parent)

private class SupervisorJobImpl(parent: Job?) : JobImpl(parent) {
    // 子协程失败不影响父协程
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

### 3.3 Dispatchers 源码

```kotlin
// Dispatchers.kt
public actual object Dispatchers {
    // 主线程调度器
    @JvmStatic
    public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher
    
    // 默认调度器（CPU 密集型）
    @JvmStatic
    public actual val Default: CoroutineDispatcher = DefaultScheduler
    
    // IO 调度器
    @JvmStatic
    public val IO: CoroutineDispatcher = DefaultIoScheduler
    
    // 无限制调度器
    @JvmStatic
    public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined
}

// DefaultScheduler
internal object DefaultScheduler : SchedulerCoroutineDispatcher(
    CORE_POOL_SIZE, MAX_POOL_SIZE,
    IDLE_WORKER_KEEP_ALIVE_NS, DEFAULT_SCHEDULER_NAME
)

// DefaultIoScheduler
internal object DefaultIoScheduler : CoroutineDispatcher() {
    // 共享 DefaultScheduler 的线程池，但有独立的并行度限制
    private val default = UnlimitedIoScheduler.limitedParallelism(
        systemProp(IO_PARALLELISM_PROPERTY_NAME, 64.coerceAtLeast(AVAILABLE_PROCESSORS))
    )
    
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        default.dispatch(context, block)
    }
}
```

### 3.4 launch 源码

```kotlin
// Builders.common.kt
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    // 创建新的上下文
    val newContext = newCoroutineContext(context)
    // 创建协程
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    // 启动协程
    coroutine.start(start, coroutine, block)
    return coroutine
}

// StandaloneCoroutine
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = true, active = active) {
    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}
```

## 4. 实战应用

### 4.1 协程最佳实践

```kotlin
class CoroutineBestPractices {
    
    // 1. 使用 viewModelScope/lifecycleScope
    class MyViewModel : ViewModel() {
        private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
        val uiState: StateFlow<UiState> = _uiState.asStateFlow()
        
        fun loadData() {
            viewModelScope.launch {
                _uiState.value = UiState.Loading
                try {
                    val data = repository.fetchData()
                    _uiState.value = UiState.Success(data)
                } catch (e: Exception) {
                    _uiState.value = UiState.Error(e.message)
                }
            }
        }
    }
    
    // 2. 使用 repeatOnLifecycle 收集 Flow
    class MyActivity : AppCompatActivity() {
        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)
            
            lifecycleScope.launch {
                repeatOnLifecycle(Lifecycle.State.STARTED) {
                    viewModel.uiState.collect { state ->
                        updateUI(state)
                    }
                }
            }
        }
    }
    
    // 3. 并行请求
    suspend fun fetchDataParallel(): CombinedData = coroutineScope {
        val user = async { userRepository.getUser() }
        val posts = async { postRepository.getPosts() }
        val comments = async { commentRepository.getComments() }
        
        CombinedData(
            user = user.await(),
            posts = posts.await(),
            comments = comments.await()
        )
    }
    
    // 4. 超时处理
    suspend fun fetchWithTimeout(): Result<Data> {
        return try {
            val data = withTimeout(5000) {
                repository.fetchData()
            }
            Result.success(data)
        } catch (e: TimeoutCancellationException) {
            Result.failure(e)
        }
    }
    
    // 5. 重试机制
    suspend fun <T> retry(
        times: Int = 3,
        initialDelay: Long = 100,
        maxDelay: Long = 1000,
        factor: Double = 2.0,
        block: suspend () -> T
    ): T {
        var currentDelay = initialDelay
        repeat(times - 1) {
            try {
                return block()
            } catch (e: Exception) {
                // 记录日志
            }
            delay(currentDelay)
            currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
        }
        return block()  // 最后一次尝试
    }
}
```

### 4.2 Flow 最佳实践

```kotlin
class FlowBestPractices {
    
    // 1. Repository 层返回 Flow
    class UserRepository(private val userDao: UserDao) {
        fun getUsers(): Flow<List<User>> = userDao.getAllUsers()
            .catch { e ->
                emit(emptyList())
            }
    }
    
    // 2. ViewModel 层转换为 StateFlow
    class UserViewModel(private val repository: UserRepository) : ViewModel() {
        val users: StateFlow<List<User>> = repository.getUsers()
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5000),
                initialValue = emptyList()
            )
    }
    
    // 3. 搜索防抖
    class SearchViewModel : ViewModel() {
        private val _searchQuery = MutableStateFlow("")
        
        val searchResults: StateFlow<List<SearchResult>> = _searchQuery
            .debounce(300)  // 防抖
            .filter { it.isNotBlank() }
            .distinctUntilChanged()
            .flatMapLatest { query ->
                repository.search(query)
            }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5000),
                initialValue = emptyList()
            )
        
        fun onSearchQueryChanged(query: String) {
            _searchQuery.value = query
        }
    }
    
    // 4. 合并多个 Flow
    class DashboardViewModel : ViewModel() {
        val dashboardState: StateFlow<DashboardState> = combine(
            userRepository.getUser(),
            orderRepository.getOrders(),
            notificationRepository.getNotifications()
        ) { user, orders, notifications ->
            DashboardState(user, orders, notifications)
        }.stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = DashboardState.Loading
        )
    }
    
    // 5. SharedFlow 用于事件
    class EventViewModel : ViewModel() {
        private val _events = MutableSharedFlow<UiEvent>()
        val events: SharedFlow<UiEvent> = _events.asSharedFlow()
        
        fun showToast(message: String) {
            viewModelScope.launch {
                _events.emit(UiEvent.ShowToast(message))
            }
        }
        
        fun navigate(route: String) {
            viewModelScope.launch {
                _events.emit(UiEvent.Navigate(route))
            }
        }
    }
}
```

### 4.3 协程测试

```kotlin
class CoroutineTestExample {
    
    @Test
    fun `test coroutine with runTest`() = runTest {
        val viewModel = MyViewModel(FakeRepository())
        
        viewModel.loadData()
        
        // advanceUntilIdle() 等待所有协程完成
        advanceUntilIdle()
        
        assertEquals(UiState.Success(expectedData), viewModel.uiState.value)
    }
    
    @Test
    fun `test flow with turbine`() = runTest {
        val viewModel = MyViewModel(FakeRepository())
        
        viewModel.uiState.test {
            assertEquals(UiState.Loading, awaitItem())
            
            viewModel.loadData()
            
            assertEquals(UiState.Success(expectedData), awaitItem())
            
            cancelAndIgnoreRemainingEvents()
        }
    }
    
    @Test
    fun `test with TestDispatcher`() = runTest {
        val testDispatcher = StandardTestDispatcher(testScheduler)
        val viewModel = MyViewModel(FakeRepository(), testDispatcher)
        
        viewModel.loadData()
        
        // 手动推进时间
        testScheduler.advanceTimeBy(1000)
        
        assertEquals(UiState.Success(expectedData), viewModel.uiState.value)
    }
}
```

## 5. 常见面试题

### 问题1：协程和线程的区别是什么？

**答案要点**：
- **调度**：线程由操作系统调度，协程由协程调度器调度
- **切换开销**：线程切换需要上下文切换，开销大；协程切换在用户态，开销小
- **内存占用**：线程约 1MB 栈空间，协程约几 KB
- **数量**：线程受系统限制，协程可创建大量
- **阻塞**：线程阻塞会占用资源，协程挂起不阻塞线程
- **取消**：线程取消需要手动处理，协程支持结构化取消

### 问题2：Job 和 SupervisorJob 的区别是什么？

**答案要点**：
- **Job**：子协程异常会取消父协程和所有兄弟协程
- **SupervisorJob**：子协程异常不影响父协程和兄弟协程
- **使用场景**：
  - Job：需要保证所有任务要么全部成功，要么全部失败
  - SupervisorJob：任务之间相互独立，一个失败不影响其他

### 问题3：如何处理协程中的异常？

**答案要点**：
1. **try-catch**：在协程内部捕获异常
2. **CoroutineExceptionHandler**：全局异常处理器
3. **supervisorScope**：隔离子协程异常
4. **async + await**：异常在 await() 时抛出
5. **runCatching**：Kotlin 标准库的 Result 包装

### 问题4：lifecycleScope 和 viewModelScope 的区别是什么？

**答案要点**：
- **lifecycleScope**：
  - 与 Activity/Fragment 生命周期绑定
  - Activity/Fragment 销毁时取消
  - 适合 UI 相关操作
- **viewModelScope**：
  - 与 ViewModel 生命周期绑定
  - ViewModel 清除时取消
  - 适合数据加载、业务逻辑

### 问题5：Channel 和 Flow 的区别是什么？

**答案要点**：
- **Channel**：
  - 热流，创建后立即开始
  - 一对一或多对多通信
  - 适合协程间通信
- **Flow**：
  - 冷流，收集时才开始
  - 一对多广播
  - 适合数据流处理

### 问题6：如何实现协程的取消？

**答案要点**：
1. 调用 `job.cancel()` 或 `job.cancelAndJoin()`
2. 协程内部需要配合检查取消状态：
   - `isActive` 属性
   - `ensureActive()` 函数
   - `yield()` 函数
3. 挂起函数（如 delay）会自动检查取消
4. 使用 `NonCancellable` 执行不可取消的清理代码

### 问题7：StateFlow 和 SharedFlow 的区别是什么？

**答案要点**：
- **StateFlow**：
  - 有初始值
  - 只保留最新值
  - 新订阅者立即收到当前值
  - 适合状态管理
- **SharedFlow**：
  - 无初始值
  - 可配置 replay 缓存
  - 可配置缓冲策略
  - 适合事件广播

### 问题8：如何处理 Flow 的背压？

**答案要点**：
1. **buffer()**：缓冲，生产和消费并行
2. **conflate()**：合并，只处理最新值
3. **collectLatest()**：取消之前的处理，只处理最新值
4. **BufferOverflow**：配置缓冲溢出策略（SUSPEND、DROP_OLDEST、DROP_LATEST）

### 问题9：Dispatchers.IO 和 Dispatchers.Default 的区别是什么？

**答案要点**：
- **Dispatchers.Default**：
  - 用于 CPU 密集型任务
  - 线程数等于 CPU 核心数
  - 适合计算、排序等
- **Dispatchers.IO**：
  - 用于 IO 密集型任务
  - 线程数最多 64 个
  - 适合网络请求、文件读写等
- **共享线程池**：两者共享线程池，但有不同的并行度限制

### 问题10：如何在协程中实现并发安全？

**答案要点**：
1. **Mutex**：协程版互斥锁
2. **线程安全集合**：ConcurrentHashMap、AtomicInteger 等
3. **线程限制**：使用单线程调度器
4. **Actor 模式**：通过 Channel 串行处理
5. **不可变数据**：使用不可变数据结构
