# Kotlin Flow

## 1. 概述

Kotlin Flow 是 Kotlin 协程库中用于处理异步数据流的 API。它是冷流（Cold Stream），只有在收集时才会执行。Flow 提供了丰富的操作符，可以方便地进行数据转换、过滤、组合等操作。本文将深入讲解 Flow 的原理、StateFlow/SharedFlow 的使用，以及与 LiveData、RxJava 的对比。

## 2. 核心原理

### 2.1 Flow 基础

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Flow 基础概念                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  什么是 Flow:                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 异步数据流，可以发射多个值                                    │   │
│  │  - 冷流 (Cold Stream)：只有在收集时才开始执行                   │   │
│  │  - 基于协程，支持挂起函数                                       │   │
│  │  - 提供丰富的操作符                                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  冷流 vs 热流:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  冷流 (Cold Stream):                                            │   │
│  │  - 每个收集者都会触发新的执行                                   │   │
│  │  - 没有收集者时不会执行                                         │   │
│  │  - 例如: Flow                                                   │   │
│  │                                                                 │   │
│  │  热流 (Hot Stream):                                             │   │
│  │  - 不管有没有收集者都会发射数据                                 │   │
│  │  - 多个收集者共享同一个数据源                                   │   │
│  │  - 例如: StateFlow, SharedFlow                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  创建 Flow:                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 1. flow 构建器                                              │   │
│  │  val flow = flow {                                              │   │
│  │      emit(1)                                                    │   │
│  │      emit(2)                                                    │   │
│  │      emit(3)                                                    │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 2. flowOf                                                   │   │
│  │  val flow = flowOf(1, 2, 3)                                     │   │
│  │                                                                 │   │
│  │  // 3. asFlow 扩展                                              │   │
│  │  val flow = listOf(1, 2, 3).asFlow()                            │   │
│  │  val flow = (1..3).asFlow()                                     │   │
│  │                                                                 │   │
│  │  // 4. channelFlow (可以在不同协程中发射)                       │   │
│  │  val flow = channelFlow {                                       │   │
│  │      send(1)                                                    │   │
│  │      launch { send(2) }                                         │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Flow 操作符

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Flow 操作符                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  转换操作符:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  map        → 转换每个元素                                      │   │
│  │  transform  → 自定义转换，可以发射多个值                        │   │
│  │  flatMapConcat → 串行展开                                       │   │
│  │  flatMapMerge  → 并行展开                                       │   │
│  │  flatMapLatest → 只处理最新的                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  过滤操作符:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  filter     → 过滤元素                                          │   │
│  │  filterNot  → 过滤不满足条件的                                  │   │
│  │  take       → 取前 n 个                                         │   │
│  │  drop       → 跳过前 n 个                                       │   │
│  │  distinctUntilChanged → 去除连续重复                            │   │
│  │  debounce   → 防抖                                              │   │
│  │  sample     → 采样                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  组合操作符:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  combine    → 组合多个 Flow，任一发射时触发                     │   │
│  │  zip        → 配对组合，等待两个都发射                          │   │
│  │  merge      → 合并多个 Flow                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  终端操作符:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  collect    → 收集所有值                                        │   │
│  │  first      → 获取第一个值                                      │   │
│  │  single     → 获取唯一值                                        │   │
│  │  toList     → 转换为 List                                       │   │
│  │  reduce     → 累积计算                                          │   │
│  │  fold       → 带初始值的累积计算                                │   │
│  │  launchIn   → 在指定作用域中收集                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  上下文操作符:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  flowOn     → 切换上游的执行上下文                              │   │
│  │  buffer     → 缓冲，提高并发性能                                │   │
│  │  conflate   → 合并，只处理最新值                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 StateFlow 与 SharedFlow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    StateFlow 与 SharedFlow                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  StateFlow:                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 热流，始终持有一个值                                         │   │
│  │  - 新收集者会立即收到当前值                                     │   │
│  │  - 只发射与当前值不同的值 (distinctUntilChanged)                │   │
│  │  - 类似 LiveData，但更强大                                      │   │
│  │                                                                 │   │
│  │  // 创建                                                        │   │
│  │  private val _state = MutableStateFlow(initialValue)            │   │
│  │  val state: StateFlow<T> = _state.asStateFlow()                 │   │
│  │                                                                 │   │
│  │  // 更新值                                                      │   │
│  │  _state.value = newValue                                        │   │
│  │  _state.update { current -> current + 1 }                       │   │
│  │                                                                 │   │
│  │  // 收集                                                        │   │
│  │  state.collect { value -> /* 处理 */ }                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  SharedFlow:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 热流，可以有多个收集者                                       │   │
│  │  - 可配置重放缓存 (replay)                                      │   │
│  │  - 可配置额外缓冲 (extraBufferCapacity)                         │   │
│  │  - 适合事件广播                                                 │   │
│  │                                                                 │   │
│  │  // 创建                                                        │   │
│  │  private val _events = MutableSharedFlow<Event>(                │   │
│  │      replay = 0,              // 重放给新收集者的数量           │   │
│  │      extraBufferCapacity = 1, // 额外缓冲                       │   │
│  │      onBufferOverflow = BufferOverflow.DROP_OLDEST              │   │
│  │  )                                                              │   │
│  │  val events: SharedFlow<Event> = _events.asSharedFlow()         │   │
│  │                                                                 │   │
│  │  // 发射                                                        │   │
│  │  _events.emit(event)      // 挂起直到发射成功                   │   │
│  │  _events.tryEmit(event)   // 非挂起，返回是否成功               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  StateFlow vs SharedFlow:                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  特性              │ StateFlow        │ SharedFlow              │   │
│  │  ─────────────────────────────────────────────────────────────  │   │
│  │  初始值            │ 必须有           │ 可选                    │   │
│  │  当前值            │ 有 (.value)      │ 无                      │   │
│  │  重复值            │ 不发射           │ 可发射                  │   │
│  │  replay            │ 固定为 1         │ 可配置                  │   │
│  │  用途              │ 状态管理         │ 事件广播                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Flow vs LiveData vs RxJava

```
┌─────────────────────────────────────────────────────────────────────────┐
│                Flow vs LiveData vs RxJava                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  对比表:                                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  特性          │ Flow        │ LiveData    │ RxJava            │   │
│  │  ─────────────────────────────────────────────────────────────  │   │
│  │  冷/热流       │ 冷流        │ 热流        │ 都有              │   │
│  │  生命周期感知  │ 需配合      │ 内置        │ 需配合            │   │
│  │  背压处理      │ 支持        │ 不支持      │ 支持              │   │
│  │  操作符        │ 丰富        │ 有限        │ 非常丰富          │   │
│  │  线程切换      │ flowOn      │ postValue   │ subscribeOn/observeOn│ │
│  │  取消          │ 协程取消    │ 自动        │ dispose           │   │
│  │  学习曲线      │ 中等        │ 简单        │ 陡峭              │   │
│  │  依赖大小      │ 小          │ 小          │ 大                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Flow 的优势:                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. 与协程无缝集成                                              │   │
│  │  2. 冷流特性，按需执行                                          │   │
│  │  3. 结构化并发，自动取消                                        │   │
│  │  4. 操作符丰富，类似 RxJava                                     │   │
│  │  5. 依赖小，是 Kotlin 标准库的一部分                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  LiveData 的优势:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. 生命周期感知，自动管理订阅                                  │   │
│  │  2. 简单易用，学习成本低                                        │   │
│  │  3. 与 ViewModel 配合良好                                       │   │
│  │  4. 适合简单的 UI 状态管理                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  推荐使用场景:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - UI 状态: StateFlow (替代 LiveData)                           │   │
│  │  - 一次性事件: SharedFlow (替代 SingleLiveEvent)                │   │
│  │  - 数据流处理: Flow                                             │   │
│  │  - 简单场景: LiveData 仍然可用                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. 关键源码解析

### 3.1 Flow 接口

```kotlin
/**
 * Flow 接口定义
 */
public interface Flow<out T> {
    /**
     * 收集 Flow 发射的值
     * 这是一个挂起函数，会挂起直到 Flow 完成
     */
    public suspend fun collect(collector: FlowCollector<T>)
}

/**
 * FlowCollector 接口
 */
public fun interface FlowCollector<in T> {
    /**
     * 发射一个值
     */
    public suspend fun emit(value: T)
}

/**
 * flow 构建器实现
 */
public fun <T> flow(
    @BuilderInference block: suspend FlowCollector<T>.() -> Unit
): Flow<T> = SafeFlow(block)

private class SafeFlow<T>(
    private val block: suspend FlowCollector<T>.() -> Unit
) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
```

### 3.2 Flow 操作符实现

```kotlin
/**
 * map 操作符实现
 */
public inline fun <T, R> Flow<T>.map(
    crossinline transform: suspend (value: T) -> R
): Flow<R> = transform { value ->
    // 对每个值应用转换函数，然后发射
    return@transform emit(transform(value))
}

/**
 * filter 操作符实现
 */
public inline fun <T> Flow<T>.filter(
    crossinline predicate: suspend (T) -> Boolean
): Flow<T> = transform { value ->
    // 只发射满足条件的值
    if (predicate(value)) return@transform emit(value)
}

/**
 * transform 操作符 - 所有转换操作符的基础
 */
public inline fun <T, R> Flow<T>.transform(
    @BuilderInference crossinline transform: suspend FlowCollector<R>.(value: T) -> Unit
): Flow<R> = flow {
    collect { value ->
        // 对每个值应用转换
        return@collect transform(value)
    }
}

/**
 * flowOn 操作符 - 切换上游执行上下文
 */
public fun <T> Flow<T>.flowOn(context: CoroutineContext): Flow<T> {
    return flow {
        // 在指定上下文中收集上游
        withContext(context) {
            collect { value ->
                // 发射到下游 (在原上下文中)
                emit(value)
            }
        }
    }
}

/**
 * collect 扩展函数
 */
public suspend inline fun <T> Flow<T>.collect(
    crossinline action: suspend (value: T) -> Unit
): Unit = collect(object : FlowCollector<T> {
    override suspend fun emit(value: T) = action(value)
})
```

### 3.3 StateFlow 实现

```kotlin
/**
 * StateFlow 接口
 */
public interface StateFlow<out T> : SharedFlow<T> {
    /**
     * 当前值
     */
    public val value: T
}

/**
 * MutableStateFlow 接口
 */
public interface MutableStateFlow<T> : StateFlow<T>, MutableSharedFlow<T> {
    /**
     * 可读写的当前值
     */
    public override var value: T
    
    /**
     * 比较并设置值
     */
    public fun compareAndSet(expect: T, update: T): Boolean
}

/**
 * MutableStateFlow 实现 (简化)
 */
internal class StateFlowImpl<T>(
    initialState: T
) : AbstractSharedFlow<StateFlowSlot>(), MutableStateFlow<T> {
    
    // 使用原子引用存储当前值
    private val _state = atomic(initialState)
    
    override var value: T
        get() = _state.value
        set(value) { updateState(value) }
    
    override fun compareAndSet(expect: T, update: T): Boolean {
        return _state.compareAndSet(expect, update)
    }
    
    private fun updateState(newState: T) {
        while (true) {
            val oldState = _state.value
            // 值相同则不更新 (distinctUntilChanged)
            if (oldState == newState) return
            if (_state.compareAndSet(oldState, newState)) {
                // 通知所有收集者
                notifyCollectors()
                return
            }
        }
    }
    
    override suspend fun collect(collector: FlowCollector<T>) {
        // 立即发射当前值
        collector.emit(value)
        // 然后等待新值
        while (true) {
            val newValue = awaitNewValue()
            collector.emit(newValue)
        }
    }
}
```

### 3.4 SharedFlow 实现

```kotlin
/**
 * SharedFlow 接口
 */
public interface SharedFlow<out T> : Flow<T> {
    /**
     * 重放缓存
     */
    public val replayCache: List<T>
}

/**
 * MutableSharedFlow 接口
 */
public interface MutableSharedFlow<T> : SharedFlow<T>, FlowCollector<T> {
    /**
     * 发射值 (挂起)
     */
    override suspend fun emit(value: T)
    
    /**
     * 尝试发射值 (非挂起)
     */
    public fun tryEmit(value: T): Boolean
    
    /**
     * 当前订阅者数量
     */
    public val subscriptionCount: StateFlow<Int>
}

/**
 * 创建 MutableSharedFlow
 */
public fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
): MutableSharedFlow<T> {
    require(replay >= 0) { "replay cannot be negative" }
    require(extraBufferCapacity >= 0) { "extraBufferCapacity cannot be negative" }
    
    val bufferCapacity = replay + extraBufferCapacity
    return SharedFlowImpl(replay, bufferCapacity, onBufferOverflow)
}
```

## 4. 实战应用

### 4.1 ViewModel 中使用 StateFlow

```kotlin
/**
 * 使用 StateFlow 管理 UI 状态
 */
class UserViewModel(
    private val userRepository: UserRepository
) : ViewModel() {
    
    // UI 状态
    sealed class UiState {
        object Loading : UiState()
        data class Success(val user: User) : UiState()
        data class Error(val message: String) : UiState()
    }
    
    // 使用 StateFlow 替代 LiveData
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    // 一次性事件使用 SharedFlow
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            
            try {
                val user = userRepository.getUser(userId)
                _uiState.value = UiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
                _events.emit(Event.ShowToast("加载失败"))
            }
        }
    }
}

/**
 * Activity/Fragment 中收集
 */
class UserActivity : AppCompatActivity() {
    
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 收集 UI 状态
        lifecycleScope.launch {
            // repeatOnLifecycle 确保只在 STARTED 状态时收集
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UiState.Loading -> showLoading()
                        is UiState.Success -> showUser(state.user)
                        is UiState.Error -> showError(state.message)
                    }
                }
            }
        }
        
        // 收集一次性事件
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.events.collect { event ->
                    when (event) {
                        is Event.ShowToast -> showToast(event.message)
                        is Event.Navigate -> navigate(event.destination)
                    }
                }
            }
        }
    }
}
```

### 4.2 搜索防抖

```kotlin
/**
 * 使用 Flow 实现搜索防抖
 */
class SearchViewModel : ViewModel() {
    
    // 搜索关键词
    private val _searchQuery = MutableStateFlow("")
    
    // 搜索结果
    val searchResults: StateFlow<List<SearchResult>> = _searchQuery
        .debounce(300)  // 防抖 300ms
        .filter { it.isNotBlank() }  // 过滤空白
        .distinctUntilChanged()  // 去重
        .flatMapLatest { query ->
            // 只处理最新的搜索
            flow {
                emit(emptyList())  // 先清空
                val results = searchRepository.search(query)
                emit(results)
            }
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

/**
 * 使用 callbackFlow 将回调转换为 Flow
 */
fun EditText.textChanges(): Flow<String> = callbackFlow {
    val watcher = object : TextWatcher {
        override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {}
        override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {}
        override fun afterTextChanged(s: Editable?) {
            trySend(s?.toString() ?: "")
        }
    }
    addTextChangedListener(watcher)
    
    // 当 Flow 被取消时移除监听
    awaitClose { removeTextChangedListener(watcher) }
}
```

### 4.3 组合多个数据源

```kotlin
/**
 * 使用 combine 组合多个 Flow
 */
class DashboardViewModel(
    private val userRepository: UserRepository,
    private val orderRepository: OrderRepository,
    private val notificationRepository: NotificationRepository
) : ViewModel() {
    
    data class DashboardState(
        val user: User?,
        val orders: List<Order>,
        val notifications: List<Notification>,
        val isLoading: Boolean
    )
    
    val dashboardState: StateFlow<DashboardState> = combine(
        userRepository.observeUser(),
        orderRepository.observeOrders(),
        notificationRepository.observeNotifications()
    ) { user, orders, notifications ->
        DashboardState(
            user = user,
            orders = orders,
            notifications = notifications,
            isLoading = false
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = DashboardState(null, emptyList(), emptyList(), true)
    )
}

/**
 * 使用 zip 配对组合
 */
fun loadUserWithProfile(userId: String): Flow<Pair<User, Profile>> {
    return userRepository.getUser(userId)
        .zip(profileRepository.getProfile(userId)) { user, profile ->
            user to profile
        }
}
```

### 4.4 错误处理

```kotlin
/**
 * Flow 错误处理
 */
class DataRepository {
    
    fun getData(): Flow<Data> = flow {
        val data = api.fetchData()
        emit(data)
    }
    .retry(3) { cause ->
        // 重试条件
        cause is IOException
    }
    .catch { e ->
        // 捕获异常，发射默认值
        emit(Data.empty())
    }
    .onStart {
        // 开始时
        Log.d("Flow", "Started")
    }
    .onCompletion { cause ->
        // 完成时 (正常或异常)
        if (cause != null) {
            Log.e("Flow", "Completed with error", cause)
        } else {
            Log.d("Flow", "Completed successfully")
        }
    }
    .flowOn(Dispatchers.IO)
}

/**
 * 在 ViewModel 中处理错误
 */
class MyViewModel : ViewModel() {
    
    private val _error = MutableSharedFlow<String>()
    val error: SharedFlow<String> = _error.asSharedFlow()
    
    fun loadData() {
        viewModelScope.launch {
            dataRepository.getData()
                .catch { e ->
                    _error.emit(e.message ?: "Unknown error")
                }
                .collect { data ->
                    // 处理数据
                }
        }
    }
}
```

### 4.5 常见坑点

1. **在 UI 层直接使用 collect**：应该使用 repeatOnLifecycle 避免后台收集
2. **StateFlow 不发射相同值**：如果需要发射相同值，使用 SharedFlow
3. **忘记使用 flowOn**：耗时操作应该在 IO 线程执行
4. **SharedFlow 的 replay 设置不当**：事件通常设置 replay = 0
5. **Flow 在 ViewModel 外部创建**：应该在 ViewModel 中创建并暴露

## 5. 常见面试题

### 问题1：Flow 和 LiveData 的区别？

**答案要点**：
| 特性 | Flow | LiveData |
|------|------|----------|
| 类型 | 冷流 | 热流 |
| 生命周期 | 需配合 repeatOnLifecycle | 内置感知 |
| 操作符 | 丰富 | 有限 |
| 线程切换 | flowOn | postValue |
| 背压 | 支持 | 不支持 |

### 问题2：StateFlow 和 SharedFlow 的区别？

**答案要点**：
- **StateFlow**：必须有初始值，有 .value 属性，不发射重复值，replay = 1
- **SharedFlow**：可选初始值，无 .value，可发射重复值，replay 可配置
- **使用场景**：StateFlow 用于状态，SharedFlow 用于事件

### 问题3：什么是冷流和热流？

**答案要点**：
- **冷流**：每个收集者触发新的执行，没有收集者时不执行（Flow）
- **热流**：不管有没有收集者都发射数据，多个收集者共享数据源（StateFlow、SharedFlow）

### 问题4：flowOn 和 withContext 的区别？

**答案要点**：
- **flowOn**：切换上游的执行上下文，不影响下游
- **withContext**：切换整个代码块的上下文
- **注意**：flowOn 只影响它之前的操作符

### 问题5：如何在 UI 层安全地收集 Flow？

**答案要点**：
```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            // 只在 STARTED 状态时收集
        }
    }
}
```
- 使用 repeatOnLifecycle 确保只在活跃状态收集
- 避免后台收集导致的资源浪费和崩溃

### 问题6：stateIn 和 shareIn 的区别？

**答案要点**：
- **stateIn**：将 Flow 转换为 StateFlow，需要初始值
- **shareIn**：将 Flow 转换为 SharedFlow，可配置 replay
- **SharingStarted**：控制何时开始共享（Eagerly、Lazily、WhileSubscribed）

### 问题7：如何处理 Flow 中的异常？

**答案要点**：
1. **catch**：捕获上游异常，可以发射默认值
2. **retry**：重试失败的操作
3. **onCompletion**：无论成功失败都会调用
4. **try-catch**：在 collect 中捕获

### 问题8：combine 和 zip 的区别？

**答案要点**：
- **combine**：任一 Flow 发射时触发，使用各 Flow 的最新值
- **zip**：等待所有 Flow 都发射后才触发，一一配对
- **使用场景**：combine 用于组合状态，zip 用于配对数据
