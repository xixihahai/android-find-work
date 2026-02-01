# Jetpack Compose 深度解析

## 1. 概述

### 1.1 什么是 Jetpack Compose

Jetpack Compose 是 Google 推出的**现代化声明式 UI 框架**，用于构建 Android 原生界面。它彻底改变了 Android UI 开发的方式，从传统的命令式 XML 布局转向声明式 Kotlin 代码。

```
┌─────────────────────────────────────────────────────────────┐
│                    Compose 架构层次                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Material Design 组件                    │   │
│  │         (Button, Card, TextField, etc.)             │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Foundation 基础组件                     │   │
│  │         (Layout, Text, Image, Scroll, etc.)         │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Compose UI                          │   │
│  │    (Modifier, Layout, Drawing, Input, etc.)         │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Compose Runtime                       │   │
│  │   (Composition, State, Recomposition, Effects)      │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Compose Compiler                       │   │
│  │        (Kotlin Compiler Plugin, Code Gen)           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 声明式 vs 命令式

| 特性 | 命令式 (View) | 声明式 (Compose) |
|------|--------------|------------------|
| UI 描述 | 描述如何构建 UI | 描述 UI 应该是什么样子 |
| 状态管理 | 手动同步状态与 UI | 状态变化自动触发 UI 更新 |
| 代码量 | XML + Java/Kotlin | 纯 Kotlin |
| 复用性 | 自定义 View 复杂 | 函数组合简单 |
| 预览 | 需要运行 App | IDE 实时预览 |


### 1.3 Compose 核心优势

1. **更少的代码**：消除 XML 布局文件，减少样板代码
2. **直观**：声明式语法，代码即 UI
3. **加速开发**：实时预览、热重载
4. **强大**：直接访问 Android 平台 API
5. **互操作性**：与现有 View 系统无缝集成

---

## 2. 核心原理

### 2.1 声明式 UI 原理

声明式 UI 的核心思想是：**UI = f(State)**，即 UI 是状态的函数。

```
┌─────────────────────────────────────────────────────────────┐
│                   声明式 UI 数据流                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│    │  State   │───▶│ Compose  │───▶│    UI    │            │
│    │  状态    │    │  函数    │    │   界面   │            │
│    └──────────┘    └──────────┘    └──────────┘            │
│         ▲                                │                  │
│         │                                │                  │
│         │         ┌──────────┐          │                  │
│         └─────────│  Event   │◀─────────┘                  │
│                   │  事件    │                              │
│                   └──────────┘                              │
│                                                             │
│    单向数据流：State → UI → Event → State → UI ...          │
└─────────────────────────────────────────────────────────────┘
```

**工作流程**：
1. 状态（State）作为输入传递给 Composable 函数
2. Composable 函数根据状态生成 UI 描述
3. Compose 运行时将 UI 描述渲染到屏幕
4. 用户交互产生事件（Event）
5. 事件处理器更新状态
6. 状态变化触发重组（Recomposition），UI 自动更新

### 2.2 Composable 函数

#### 2.2.1 基本概念

`@Composable` 注解标记的函数是 Compose UI 的基本构建块：

```kotlin
@Composable
fun Greeting(name: String) {
    // 这是一个 Composable 函数
    // 它描述了 UI 应该是什么样子
    Text(text = "Hello, $name!")
}
```

#### 2.2.2 Composable 函数的特性

```kotlin
// 1. 可以调用其他 Composable 函数
@Composable
fun UserCard(user: User) {
    Column {
        Avatar(user.avatarUrl)  // 调用其他 Composable
        Text(user.name)
        Text(user.email)
    }
}

// 2. 可以有参数和默认值
@Composable
fun CustomButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,  // 默认参数
    enabled: Boolean = true
) {
    Button(
        onClick = onClick,
        modifier = modifier,
        enabled = enabled
    ) {
        Text(text)
    }
}

// 3. 可以有条件逻辑
@Composable
fun ConditionalContent(showDetails: Boolean) {
    Column {
        Text("Title")
        if (showDetails) {  // 条件渲染
            Text("Details...")
        }
    }
}

// 4. 可以有循环
@Composable
fun ItemList(items: List<String>) {
    Column {
        items.forEach { item ->  // 循环渲染
            Text(item)
        }
    }
}
```


### 2.3 Recomposition 机制（智能重组）

#### 2.3.1 什么是 Recomposition

Recomposition 是 Compose 的核心机制，当状态变化时，Compose 会**智能地**只重新执行受影响的 Composable 函数。

```
┌─────────────────────────────────────────────────────────────┐
│                  Recomposition 流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Initial Composition (首次组合)                            │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  @Composable fun Screen() {                         │  │
│   │      var count by remember { mutableStateOf(0) }    │  │
│   │      Column {                                       │  │
│   │          Text("Count: $count")  ◀── 读取 count      │  │
│   │          Button(onClick = { count++ }) {            │  │
│   │              Text("Increment")                      │  │
│   │          }                                          │  │
│   │      }                                              │  │
│   │  }                                                  │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   State Change (状态变化): count = 0 → count = 1           │
│                          │                                  │
│                          ▼                                  │
│   Recomposition (重组)                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  只有读取了 count 的 Text 会被重组                    │  │
│   │  Button 和其内部的 Text("Increment") 不会重组        │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.3.2 Recomposition 的智能跳过

Compose 编译器会在编译时分析代码，为 Composable 函数生成比较逻辑：

```kotlin
// 编译前
@Composable
fun UserInfo(name: String, age: Int) {
    Text("$name, $age years old")
}

// 编译后（简化示意）
@Composable
fun UserInfo(name: String, age: Int, $composer: Composer, $changed: Int) {
    // 检查参数是否变化
    if ($changed and 0b0011 == 0 && $composer.skipping) {
        // 参数未变化，跳过重组
        $composer.skipToGroupEnd()
        return
    }
    // 参数变化，执行重组
    Text("$name, $age years old")
}
```

#### 2.3.3 稳定性（Stability）

Compose 通过**稳定性**判断是否可以跳过重组：

```kotlin
// 稳定类型 - 可以安全跳过重组
// 1. 基本类型：Int, String, Float, Boolean 等
// 2. 不可变类型：使用 val 声明的属性
// 3. 标记 @Stable 或 @Immutable 的类

@Immutable  // 标记为不可变
data class User(
    val id: String,
    val name: String,
    val age: Int
)

@Stable  // 标记为稳定
class Counter {
    var value by mutableStateOf(0)  // 虽然可变，但通过 State 追踪
}

// 不稳定类型 - 每次都会重组
data class UnstableUser(
    val id: String,
    var name: String,  // var 导致不稳定
    val tags: List<String>  // List 默认不稳定
)
```

#### 2.3.4 Composition 树结构

```
┌─────────────────────────────────────────────────────────────┐
│                    Composition 树                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                      ┌─────────┐                            │
│                      │  Root   │                            │
│                      └────┬────┘                            │
│                           │                                 │
│              ┌────────────┼────────────┐                    │
│              │            │            │                    │
│         ┌────▼────┐  ┌────▼────┐  ┌────▼────┐              │
│         │ Header  │  │ Content │  │ Footer  │              │
│         └────┬────┘  └────┬────┘  └─────────┘              │
│              │            │                                 │
│         ┌────▼────┐  ┌────┴────┐                           │
│         │  Title  │  │         │                           │
│         └─────────┘  │    ┌────▼────┐                      │
│                      │    │  List   │                      │
│                      │    └────┬────┘                      │
│                      │         │                           │
│                      │    ┌────┴────┬────────┐             │
│                      │    │         │        │             │
│                      │ ┌──▼──┐  ┌──▼──┐  ┌──▼──┐          │
│                      │ │Item1│  │Item2│  │Item3│          │
│                      │ └─────┘  └─────┘  └─────┘          │
│                      │                                     │
│   Slot Table: 存储 Composition 树的数据结构                 │
│   - Group: 每个 Composable 对应一个 Group                   │
│   - 存储：参数、状态、子节点信息                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```


### 2.4 State 管理

#### 2.4.1 remember 与 mutableStateOf

```kotlin
// mutableStateOf: 创建可观察的状态
// remember: 在重组时保持状态

@Composable
fun Counter() {
    // 方式1：分开使用
    val count = remember { mutableStateOf(0) }
    
    // 方式2：使用委托属性（推荐）
    var count by remember { mutableStateOf(0) }
    
    // 方式3：解构
    val (count, setCount) = remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

#### 2.4.2 State 的工作原理

```
┌─────────────────────────────────────────────────────────────┐
│                   State 订阅机制                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  var count by remember { mutableStateOf(0) }        │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              MutableState<Int>                      │  │
│   │  ┌─────────────────────────────────────────────┐   │  │
│   │  │  value: Int = 0                             │   │  │
│   │  │  readers: Set<RecomposeScope>  ◀── 订阅者   │   │  │
│   │  └─────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   读取 State 时：                                           │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  Text("Count: $count")                              │  │
│   │       │                                             │  │
│   │       ▼                                             │  │
│   │  当前 RecomposeScope 被添加到 readers 集合           │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   写入 State 时：                                           │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  count++  // value = 1                              │  │
│   │       │                                             │  │
│   │       ▼                                             │  │
│   │  通知所有 readers 进行重组                           │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.4.3 不同类型的 State

```kotlin
// 1. mutableStateOf - 单个值
var name by remember { mutableStateOf("") }

// 2. mutableStateListOf - 可观察列表
val items = remember { mutableStateListOf<String>() }
items.add("New Item")  // 自动触发重组

// 3. mutableStateMapOf - 可观察 Map
val map = remember { mutableStateMapOf<String, Int>() }
map["key"] = 1  // 自动触发重组

// 4. derivedStateOf - 派生状态（性能优化）
val items = remember { mutableStateListOf<Item>() }
val selectedCount by remember {
    derivedStateOf { items.count { it.selected } }
}
// 只有当 selectedCount 实际变化时才触发重组

// 5. snapshotFlow - 将 State 转换为 Flow
val count by remember { mutableStateOf(0) }
LaunchedEffect(Unit) {
    snapshotFlow { count }
        .collect { value ->
            // 响应 count 变化
        }
}
```

#### 2.4.4 状态提升（State Hoisting）

```kotlin
// 状态提升模式：将状态移到调用者
// 使 Composable 变成无状态，更易测试和复用

// ❌ 有状态的 Composable
@Composable
fun StatefulCounter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// ✅ 无状态的 Composable（推荐）
@Composable
fun StatelessCounter(
    count: Int,           // 状态下传
    onIncrement: () -> Unit  // 事件上传
) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}

// 使用
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    StatelessCounter(
        count = count,
        onIncrement = { count++ }
    )
}
```


### 2.5 副作用 API（Side Effects）

副作用是指在 Composable 函数作用域之外发生的操作，如网络请求、数据库操作、日志记录等。

#### 2.5.1 副作用 API 概览

```
┌─────────────────────────────────────────────────────────────┐
│                    副作用 API 选择指南                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  需要执行挂起函数？                                          │
│       │                                                     │
│       ├── 是 ──▶ LaunchedEffect / rememberCoroutineScope   │
│       │                                                     │
│       └── 否 ──▶ 需要清理资源？                             │
│                      │                                      │
│                      ├── 是 ──▶ DisposableEffect           │
│                      │                                      │
│                      └── 否 ──▶ 需要在每次重组后执行？       │
│                                     │                       │
│                                     ├── 是 ──▶ SideEffect  │
│                                     │                       │
│                                     └── 否 ──▶ 普通代码     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.5.2 LaunchedEffect

用于在 Composable 中启动协程，当 key 变化时会取消并重新启动。

```kotlin
@Composable
fun UserProfile(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }
    
    // 当 userId 变化时，取消之前的协程，启动新的
    LaunchedEffect(userId) {
        // 这里可以安全地调用挂起函数
        user = repository.getUser(userId)
    }
    
    // 使用 Unit 作为 key，只在首次组合时执行
    LaunchedEffect(Unit) {
        // 只执行一次
        analytics.logScreenView("UserProfile")
    }
    
    user?.let { UserCard(it) }
}

// LaunchedEffect 源码简化
@Composable
fun LaunchedEffect(
    key1: Any?,
    block: suspend CoroutineScope.() -> Unit
) {
    val currentBlock by rememberUpdatedState(block)
    
    DisposableEffect(key1) {
        val job = scope.launch {
            currentBlock()
        }
        onDispose {
            job.cancel()  // 清理时取消协程
        }
    }
}
```

#### 2.5.3 DisposableEffect

用于需要清理的副作用，如注册/注销监听器。

```kotlin
@Composable
fun LifecycleObserver(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current,
    onStart: () -> Unit,
    onStop: () -> Unit
) {
    // 当 lifecycleOwner 变化时，会先调用 onDispose，再重新执行
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_START -> onStart()
                Lifecycle.Event.ON_STOP -> onStop()
                else -> {}
            }
        }
        
        // 注册监听器
        lifecycleOwner.lifecycle.addObserver(observer)
        
        // 返回清理逻辑
        onDispose {
            // 注销监听器
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}

// 监听系统返回键
@Composable
fun BackHandler(enabled: Boolean = true, onBack: () -> Unit) {
    val backCallback = remember {
        object : OnBackPressedCallback(enabled) {
            override fun handleOnBackPressed() {
                onBack()
            }
        }
    }
    
    val backDispatcher = LocalOnBackPressedDispatcherOwner.current
        ?.onBackPressedDispatcher
    
    DisposableEffect(backDispatcher) {
        backDispatcher?.addCallback(backCallback)
        onDispose {
            backCallback.remove()
        }
    }
}
```

#### 2.5.4 SideEffect

在每次成功重组后执行，用于将 Compose 状态同步到非 Compose 代码。

```kotlin
@Composable
fun AnalyticsScreen(screenName: String) {
    // 每次重组成功后都会执行
    SideEffect {
        // 将 Compose 状态同步到分析系统
        analytics.setCurrentScreen(screenName)
    }
    
    // UI 内容
    Text("Screen: $screenName")
}

// 注意：SideEffect 不能执行挂起函数
// 如果需要挂起函数，使用 LaunchedEffect
```

#### 2.5.5 rememberUpdatedState

用于在长期运行的副作用中引用最新的值。

```kotlin
@Composable
fun TimerEffect(onTick: () -> Unit) {
    // 保持对最新 onTick 的引用
    val currentOnTick by rememberUpdatedState(onTick)
    
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            // 使用最新的 onTick，而不是首次组合时的
            currentOnTick()
        }
    }
}
```

#### 2.5.6 produceState

将非 Compose 状态转换为 Compose State。

```kotlin
@Composable
fun NetworkImage(url: String): State<Result<Bitmap>> {
    return produceState<Result<Bitmap>>(
        initialValue = Result.Loading,
        key1 = url
    ) {
        value = Result.Loading
        value = try {
            val bitmap = imageLoader.load(url)
            Result.Success(bitmap)
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
}

// 使用
@Composable
fun ImageScreen(url: String) {
    val imageState by NetworkImage(url)
    
    when (val result = imageState) {
        is Result.Loading -> CircularProgressIndicator()
        is Result.Success -> Image(bitmap = result.data)
        is Result.Error -> Text("Error: ${result.exception.message}")
    }
}
```

#### 2.5.7 rememberCoroutineScope

获取与当前 Composition 绑定的 CoroutineScope，用于在事件回调中启动协程。

```kotlin
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    // 获取与 Composition 绑定的 scope
    val scope = rememberCoroutineScope()
    
    Button(onClick = {
        // 在点击事件中启动协程
        scope.launch {
            listState.animateScrollToItem(0)
        }
    }) {
        Text("Scroll to Top")
    }
}
```


---

## 3. 关键源码解析

### 3.1 Compose 编译器原理

Compose 编译器是一个 Kotlin 编译器插件，它在编译时对 `@Composable` 函数进行转换。

#### 3.1.1 编译器转换过程

```
┌─────────────────────────────────────────────────────────────┐
│                 Compose 编译器工作流程                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   源代码                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  @Composable                                        │  │
│   │  fun Greeting(name: String) {                       │  │
│   │      Text("Hello, $name!")                          │  │
│   │  }                                                  │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   Compose Compiler Plugin                                   │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  1. 添加 Composer 参数                              │  │
│   │  2. 添加 $changed 参数（用于跳过检查）              │  │
│   │  3. 插入 Group 开始/结束调用                        │  │
│   │  4. 生成稳定性检查代码                              │  │
│   │  5. 处理 remember、State 等调用                     │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   转换后代码                                                │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  fun Greeting(                                      │  │
│   │      name: String,                                  │  │
│   │      $composer: Composer,                           │  │
│   │      $changed: Int                                  │  │
│   │  ) {                                                │  │
│   │      $composer.startRestartGroup(...)               │  │
│   │      // 跳过检查逻辑                                │  │
│   │      if (...) {                                     │  │
│   │          $composer.skipToGroupEnd()                 │  │
│   │      } else {                                       │  │
│   │          Text("Hello, $name!", $composer, ...)      │  │
│   │      }                                              │  │
│   │      $composer.endRestartGroup()                    │  │
│   │  }                                                  │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.1.2 $changed 参数位运算

```kotlin
/**
 * $changed 参数使用位运算来追踪参数变化状态
 * 每个参数占用 3 位：
 * - 00: 未知（需要比较）
 * - 01: 相同（可以跳过）
 * - 10: 不同（需要重组）
 * - 11: 静态（编译时常量）
 */

// 示例：两个参数的函数
@Composable
fun Example(a: Int, b: String) { ... }

// 转换后
fun Example(
    a: Int,
    b: String,
    $composer: Composer,
    $changed: Int  // 位布局: [unused][b状态][a状态]
) {
    // 检查 a 是否变化
    val aChanged = $changed and 0b0111  // 取低3位
    // 检查 b 是否变化  
    val bChanged = ($changed shr 3) and 0b0111  // 取次3位
    
    // 如果都未变化，跳过重组
    if (aChanged == 0b001 && bChanged == 0b001) {
        $composer.skipToGroupEnd()
        return
    }
    // 执行重组...
}
```

### 3.2 Slot Table 数据结构

Slot Table 是 Compose 存储 Composition 数据的核心数据结构。

```kotlin
/**
 * SlotTable 源码解析（简化版）
 * 
 * SlotTable 使用两个数组存储数据：
 * - groups: 存储 Group 的元数据（Int 数组）
 * - slots: 存储实际数据（Any? 数组）
 */
internal class SlotTable {
    // Group 数组，每个 Group 占用 5 个 Int
    // [key, groupInfo, parentAnchor, size, dataAnchor]
    var groups = IntArray(...)
    
    // Slot 数组，存储实际数据
    var slots = Array<Any?>(...)
    
    /**
     * Group 结构：
     * - key: 用于识别 Group 的唯一标识
     * - groupInfo: 包含节点类型、是否有数据等信息
     * - parentAnchor: 父 Group 的位置
     * - size: 子 Group 数量
     * - dataAnchor: 数据在 slots 数组中的起始位置
     */
}

/**
 * Composer 使用 SlotTable 进行组合
 */
internal class ComposerImpl(
    private val slotTable: SlotTable
) : Composer {
    
    // 开始一个 RestartGroup（可重组的 Group）
    override fun startRestartGroup(key: Int): Composer {
        // 1. 在 SlotTable 中创建/查找 Group
        // 2. 记录当前位置用于后续重组
        start(key, null, GroupKind.RestartGroup, null)
        return this
    }
    
    // 结束 RestartGroup
    override fun endRestartGroup(): ScopeUpdateScope? {
        // 1. 结束当前 Group
        // 2. 返回 ScopeUpdateScope 用于注册重组回调
        return endGroup()?.let { ... }
    }
    
    // remember 的实现
    override fun <T> remember(calculation: () -> T): T {
        // 1. 检查是否已有缓存值
        // 2. 如果有且未失效，返回缓存值
        // 3. 否则执行 calculation 并缓存结果
        return rememberedValue().let {
            if (it === Composer.Empty) {
                val value = calculation()
                updateRememberedValue(value)
                value
            } else {
                @Suppress("UNCHECKED_CAST")
                it as T
            }
        }
    }
}
```

### 3.3 State 源码解析

```kotlin
/**
 * MutableState 接口定义
 */
interface MutableState<T> : State<T> {
    override var value: T
    
    // 解构操作符支持
    operator fun component1(): T
    operator fun component2(): (T) -> Unit
}

/**
 * SnapshotMutableStateImpl 实现（简化版）
 * 这是 mutableStateOf 返回的实际类型
 */
internal class SnapshotMutableStateImpl<T>(
    value: T,
    val policy: SnapshotMutationPolicy<T>
) : StateObject, MutableState<T> {
    
    // 使用 StateRecord 存储值，支持快照系统
    private var next: StateRecord = StateStateRecord(value)
    
    override var value: T
        get() {
            // 读取时：
            // 1. 获取当前快照对应的 StateRecord
            // 2. 记录读取操作（用于追踪依赖）
            val snapshot = Snapshot.current
            snapshot.recordReadOf(this)  // 关键：记录读取
            return readable(this, snapshot).value
        }
        set(value) {
            // 写入时：
            // 1. 获取可写的 StateRecord
            // 2. 检查值是否真的变化（使用 policy）
            // 3. 如果变化，通知所有观察者
            val snapshot = Snapshot.current
            val record = writable(this, snapshot)
            if (!policy.equivalent(record.value, value)) {
                record.value = value
                snapshot.recordWriteOf(this)  // 关键：记录写入
            }
        }
}

/**
 * Snapshot 系统 - 实现状态隔离和变更追踪
 */
abstract class Snapshot {
    companion object {
        // 当前线程的快照
        val current: Snapshot
            get() = currentSnapshot()
    }
    
    // 记录状态读取
    abstract fun recordReadOf(state: StateObject)
    
    // 记录状态写入
    abstract fun recordWriteOf(state: StateObject)
}

/**
 * 重组调度器 - 当状态变化时触发重组
 */
internal class RecomposerImpl : Recomposer {
    
    // 状态变化时的回调
    override fun onStateChanged(state: StateObject) {
        // 1. 找到所有读取了该状态的 RecomposeScope
        // 2. 将这些 scope 标记为需要重组
        // 3. 调度重组任务
        val scopes = state.readers
        scopes.forEach { scope ->
            scope.invalidate()  // 标记为无效，需要重组
        }
        scheduleRecomposition()  // 调度重组
    }
}
```


### 3.4 Modifier 链式调用原理

```kotlin
/**
 * Modifier 接口定义
 */
interface Modifier {
    // 折叠操作：将 Modifier 链转换为单个结果
    fun <R> foldIn(initial: R, operation: (R, Element) -> R): R
    fun <R> foldOut(initial: R, operation: (Element, R) -> R): R
    
    // 组合操作符
    infix fun then(other: Modifier): Modifier =
        if (other === Modifier) this
        else CombinedModifier(this, other)
    
    // 单个 Modifier 元素
    interface Element : Modifier {
        override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R =
            operation(initial, this)
        override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R =
            operation(this, initial)
    }
    
    // 伴生对象作为空 Modifier
    companion object : Modifier {
        override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R = initial
        override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R = initial
        override infix fun then(other: Modifier): Modifier = other
    }
}

/**
 * CombinedModifier - 组合多个 Modifier
 */
class CombinedModifier(
    private val outer: Modifier,
    private val inner: Modifier
) : Modifier {
    // 从外到内折叠
    override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R =
        inner.foldIn(outer.foldIn(initial, operation), operation)
    
    // 从内到外折叠
    override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R =
        outer.foldOut(inner.foldOut(initial, operation), operation)
}

/**
 * Modifier 链式调用示例
 */
// 使用
Box(
    modifier = Modifier
        .fillMaxSize()      // LayoutModifier
        .background(Color.Red)  // DrawModifier
        .padding(16.dp)     // LayoutModifier
        .clickable { }      // PointerInputModifier
)

// 内部结构
// CombinedModifier(
//     outer = CombinedModifier(
//         outer = CombinedModifier(
//             outer = FillMaxSizeModifier,
//             inner = BackgroundModifier
//         ),
//         inner = PaddingModifier
//     ),
//     inner = ClickableModifier
// )
```

### 3.5 Layout 测量与布局

```kotlin
/**
 * Layout Composable 源码解析
 */
@Composable
inline fun Layout(
    content: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    // 创建 LayoutNode
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    
    ReusableComposeNode<LayoutNode, Applier<Any>>(
        factory = { LayoutNode() },
        update = {
            set(measurePolicy) { this.measurePolicy = it }
            set(density) { this.density = it }
            set(layoutDirection) { this.layoutDirection = it }
        },
        content = content
    )
}

/**
 * MeasurePolicy - 定义测量和布局逻辑
 */
fun interface MeasurePolicy {
    fun MeasureScope.measure(
        measurables: List<Measurable>,
        constraints: Constraints
    ): MeasureResult
}

/**
 * 自定义 Layout 示例
 */
@Composable
fun CustomColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        // 1. 测量所有子元素
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints)
        }
        
        // 2. 计算总高度
        val totalHeight = placeables.sumOf { it.height }
        val maxWidth = placeables.maxOfOrNull { it.width } ?: 0
        
        // 3. 返回布局结果
        layout(maxWidth, totalHeight) {
            var yPosition = 0
            placeables.forEach { placeable ->
                // 4. 放置子元素
                placeable.placeRelative(x = 0, y = yPosition)
                yPosition += placeable.height
            }
        }
    }
}

/**
 * Constraints 约束系统
 */
data class Constraints(
    val minWidth: Int = 0,
    val maxWidth: Int = Infinity,
    val minHeight: Int = 0,
    val maxHeight: Int = Infinity
) {
    companion object {
        // 固定大小约束
        fun fixed(width: Int, height: Int) = Constraints(
            minWidth = width, maxWidth = width,
            minHeight = height, maxHeight = height
        )
        
        // 无限制约束
        val Unspecified = Constraints()
    }
}
```

---

## 4. 实战应用

### 4.1 Compose 与 View 互操作

#### 4.1.1 在 Compose 中使用 View

```kotlin
// AndroidView - 在 Compose 中嵌入传统 View
@Composable
fun MapViewComposable(
    modifier: Modifier = Modifier,
    onMapReady: (GoogleMap) -> Unit
) {
    var mapView: MapView? = null
    
    AndroidView(
        modifier = modifier,
        factory = { context ->
            // 创建 View
            MapView(context).apply {
                mapView = this
                onCreate(null)
                getMapAsync { map ->
                    onMapReady(map)
                }
            }
        },
        update = { view ->
            // View 更新时调用
        }
    )
    
    // 处理生命周期
    val lifecycle = LocalLifecycleOwner.current.lifecycle
    DisposableEffect(lifecycle) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> mapView?.onResume()
                Lifecycle.Event.ON_PAUSE -> mapView?.onPause()
                Lifecycle.Event.ON_DESTROY -> mapView?.onDestroy()
                else -> {}
            }
        }
        lifecycle.addObserver(observer)
        onDispose {
            lifecycle.removeObserver(observer)
        }
    }
}

// AndroidViewBinding - 使用 ViewBinding
@Composable
fun LegacyLayoutComposable() {
    AndroidViewBinding(LegacyLayoutBinding::inflate) { binding ->
        binding.textView.text = "Hello from Compose"
        binding.button.setOnClickListener {
            // 处理点击
        }
    }
}
```

#### 4.1.2 在 View 中使用 Compose

```kotlin
// 方式1：ComposeView
class MyFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            // 设置 ViewCompositionStrategy
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                MaterialTheme {
                    MyComposableContent()
                }
            }
        }
    }
}

// 方式2：在 XML 中使用 ComposeView
// layout.xml
// <androidx.compose.ui.platform.ComposeView
//     android:id="@+id/compose_view"
//     android:layout_width="match_parent"
//     android:layout_height="wrap_content" />

class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.layout)
        
        findViewById<ComposeView>(R.id.compose_view).setContent {
            MyComposableContent()
        }
    }
}

// ViewCompositionStrategy 选项
enum class ViewCompositionStrategy {
    // 当 View 从窗口分离时释放 Composition
    DisposeOnDetachedFromWindow,
    
    // 当 ViewTreeLifecycleOwner 销毁时释放
    DisposeOnViewTreeLifecycleDestroyed,
    
    // 当 Lifecycle 销毁时释放
    DisposeOnLifecycleDestroyed(lifecycle)
}
```


### 4.2 Compose 性能优化

#### 4.2.1 避免不必要的重组

```kotlin
// ❌ 错误：Lambda 每次都创建新实例，导致子组件重组
@Composable
fun BadExample(viewModel: MyViewModel) {
    val items by viewModel.items.collectAsState()
    
    LazyColumn {
        items(items) { item ->
            ItemRow(
                item = item,
                // 每次重组都创建新的 lambda
                onClick = { viewModel.onItemClick(item.id) }
            )
        }
    }
}

// ✅ 正确：使用 remember 缓存 lambda
@Composable
fun GoodExample(viewModel: MyViewModel) {
    val items by viewModel.items.collectAsState()
    
    LazyColumn {
        items(
            items = items,
            key = { it.id }  // 使用稳定的 key
        ) { item ->
            val onClick = remember(item.id) {
                { viewModel.onItemClick(item.id) }
            }
            ItemRow(item = item, onClick = onClick)
        }
    }
}

// ✅ 更好：使用方法引用
@Composable
fun BetterExample(viewModel: MyViewModel) {
    val items by viewModel.items.collectAsState()
    
    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemRow(
                item = item,
                onClick = viewModel::onItemClick,
                itemId = item.id
            )
        }
    }
}
```

#### 4.2.2 使用 derivedStateOf 减少重组

```kotlin
// ❌ 错误：每次 items 变化都会重组
@Composable
fun BadFilterExample(items: List<Item>) {
    val filteredItems = items.filter { it.isActive }  // 每次都计算
    
    LazyColumn {
        items(filteredItems) { item ->
            ItemRow(item)
        }
    }
}

// ✅ 正确：只有过滤结果变化时才重组
@Composable
fun GoodFilterExample(items: List<Item>) {
    val filteredItems by remember(items) {
        derivedStateOf { items.filter { it.isActive } }
    }
    
    LazyColumn {
        items(filteredItems) { item ->
            ItemRow(item)
        }
    }
}

// 滚动位置示例
@Composable
fun ScrollExample() {
    val listState = rememberLazyListState()
    
    // 只有当 showButton 的值真正变化时才重组
    val showButton by remember {
        derivedStateOf {
            listState.firstVisibleItemIndex > 0
        }
    }
    
    Box {
        LazyColumn(state = listState) { ... }
        
        if (showButton) {
            ScrollToTopButton()
        }
    }
}
```

#### 4.2.3 延迟读取 State

```kotlin
// ❌ 错误：在 Composition 阶段读取，整个函数都会重组
@Composable
fun BadScrollOffset(scrollState: ScrollState) {
    val offset = scrollState.value  // 在 Composition 阶段读取
    
    Box(
        modifier = Modifier
            .offset(y = (-offset).dp)  // offset 变化导致整个函数重组
    ) {
        Content()
    }
}

// ✅ 正确：延迟到 Layout/Draw 阶段读取
@Composable
fun GoodScrollOffset(scrollState: ScrollState) {
    Box(
        modifier = Modifier
            .offset {
                // 在 Layout 阶段读取，只影响布局，不触发重组
                IntOffset(0, -scrollState.value)
            }
    ) {
        Content()
    }
}

// 使用 graphicsLayer 进行动画（在 Draw 阶段）
@Composable
fun AnimatedContent(animatedProgress: Animatable<Float, *>) {
    Box(
        modifier = Modifier
            .graphicsLayer {
                // 在 Draw 阶段读取，性能最优
                alpha = animatedProgress.value
                scaleX = animatedProgress.value
                scaleY = animatedProgress.value
            }
    ) {
        Content()
    }
}
```

#### 4.2.4 使用 key 优化列表

```kotlin
@Composable
fun OptimizedList(items: List<Item>) {
    LazyColumn {
        items(
            items = items,
            // 使用稳定的 key，帮助 Compose 识别项目
            key = { item -> item.id }
        ) { item ->
            // 使用 contentType 优化不同类型的项目
            ItemRow(item)
        }
    }
}

// 混合类型列表
@Composable
fun MixedList(items: List<ListItem>) {
    LazyColumn {
        items(
            items = items,
            key = { it.id },
            contentType = { it.type }  // 相同类型可以复用
        ) { item ->
            when (item) {
                is ListItem.Header -> HeaderRow(item)
                is ListItem.Content -> ContentRow(item)
                is ListItem.Footer -> FooterRow(item)
            }
        }
    }
}
```

#### 4.2.5 稳定性优化

```kotlin
// 使用 @Stable 或 @Immutable 标记类
@Immutable
data class User(
    val id: String,
    val name: String,
    val avatar: String
)

@Stable
class UserState {
    var isLoggedIn by mutableStateOf(false)
    var user by mutableStateOf<User?>(null)
}

// 使用 ImmutableList（来自 kotlinx.collections.immutable）
@Composable
fun StableListExample(items: ImmutableList<Item>) {
    // ImmutableList 是稳定的，可以正确跳过重组
    LazyColumn {
        items(items) { item ->
            ItemRow(item)
        }
    }
}

// 配置 Compose 编译器报告
// build.gradle.kts
// composeCompiler {
//     reportsDestination = layout.buildDirectory.dir("compose_reports")
//     metricsDestination = layout.buildDirectory.dir("compose_metrics")
// }
```


### 4.3 CompositionLocal

```kotlin
/**
 * CompositionLocal - 隐式传递数据
 * 类似于 React 的 Context
 */

// 定义 CompositionLocal
val LocalUserTheme = compositionLocalOf<UserTheme> {
    error("No UserTheme provided")  // 未提供时抛出错误
}

// 或使用 staticCompositionLocalOf（值很少变化时）
val LocalAnalytics = staticCompositionLocalOf<Analytics> {
    error("No Analytics provided")
}

// 提供值
@Composable
fun App() {
    val theme = remember { UserTheme.Dark }
    
    CompositionLocalProvider(
        LocalUserTheme provides theme,
        LocalAnalytics provides AnalyticsImpl()
    ) {
        // 子组件可以访问这些值
        MainContent()
    }
}

// 使用值
@Composable
fun ThemedButton() {
    val theme = LocalUserTheme.current  // 获取当前值
    
    Button(
        colors = ButtonDefaults.buttonColors(
            containerColor = theme.primaryColor
        )
    ) {
        Text("Themed Button")
    }
}

// 系统提供的 CompositionLocal
@Composable
fun SystemLocalsExample() {
    val context = LocalContext.current
    val density = LocalDensity.current
    val configuration = LocalConfiguration.current
    val lifecycleOwner = LocalLifecycleOwner.current
    val view = LocalView.current
}
```

### 4.4 Navigation Compose

```kotlin
// 设置导航
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToDetail = { id ->
                    navController.navigate("detail/$id")
                }
            )
        }
        
        composable(
            route = "detail/{itemId}",
            arguments = listOf(
                navArgument("itemId") { type = NavType.StringType }
            )
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId")
            DetailScreen(itemId = itemId)
        }
        
        // 带动画的导航
        composable(
            route = "settings",
            enterTransition = { slideInHorizontally { it } },
            exitTransition = { slideOutHorizontally { -it } }
        ) {
            SettingsScreen()
        }
    }
}

// 类型安全的导航（Compose Navigation 2.8+）
@Serializable
data class DetailRoute(val itemId: String)

@Composable
fun TypeSafeNavigation() {
    val navController = rememberNavController()
    
    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            HomeScreen(
                onNavigateToDetail = { id ->
                    navController.navigate(DetailRoute(id))
                }
            )
        }
        
        composable<DetailRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<DetailRoute>()
            DetailScreen(itemId = route.itemId)
        }
    }
}
```

---

## 5. 常见面试题

### 5.1 基础概念题

#### Q1: 什么是声明式 UI？Compose 相比传统 View 有什么优势？

**答案要点**：

**声明式 UI 定义**：
- 声明式 UI 描述"UI 应该是什么样子"，而不是"如何构建 UI"
- 核心公式：`UI = f(State)`，UI 是状态的函数
- 状态变化时，框架自动计算 UI 差异并更新

**Compose 优势**：
1. **更少的代码**：消除 XML，减少 findViewById、ViewBinding 等样板代码
2. **单一语言**：纯 Kotlin 开发，类型安全，IDE 支持更好
3. **组合优于继承**：通过函数组合构建 UI，比自定义 View 更灵活
4. **实时预览**：@Preview 注解支持 IDE 实时预览
5. **状态管理简化**：自动追踪状态变化，无需手动同步 UI
6. **更好的性能**：智能重组，只更新变化的部分

```kotlin
// 传统 View 方式
class CounterView : LinearLayout {
    private var count = 0
    private lateinit var textView: TextView
    
    init {
        // 创建 UI
        textView = TextView(context)
        val button = Button(context)
        button.setOnClickListener {
            count++
            textView.text = "Count: $count"  // 手动更新 UI
        }
        addView(textView)
        addView(button)
    }
}

// Compose 方式
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Column {
        Text("Count: $count")  // 自动更新
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

---

#### Q2: 解释 Compose 的 Recomposition 机制，什么情况下会触发重组？如何避免不必要的重组？

**答案要点**：

**Recomposition 机制**：
1. 当 Composable 函数读取的 State 发生变化时触发
2. Compose 运行时会智能地只重新执行受影响的 Composable
3. 使用 Slot Table 存储 Composition 数据，支持增量更新

**触发重组的情况**：
1. `mutableStateOf` 创建的状态值变化
2. Composable 函数的参数变化
3. 读取的 `CompositionLocal` 值变化

**避免不必要重组的方法**：

```kotlin
// 1. 使用稳定的参数类型
@Immutable
data class User(val id: String, val name: String)

// 2. 使用 remember 缓存计算结果
val expensiveResult = remember(input) { 
    expensiveCalculation(input) 
}

// 3. 使用 derivedStateOf 减少重组频率
val showButton by remember {
    derivedStateOf { scrollState.value > 100 }
}

// 4. 延迟读取 State 到 Layout/Draw 阶段
Modifier.offset { IntOffset(0, -scrollState.value) }

// 5. 使用 key 帮助 Compose 识别列表项
LazyColumn {
    items(items, key = { it.id }) { item -> ... }
}

// 6. 拆分 Composable，缩小重组范围
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }
    ExpensiveChild()  // 不读取 count，不会重组
    CountDisplay(count)  // 只有这个会重组
}
```

---

#### Q3: remember 和 rememberSaveable 的区别是什么？它们的实现原理是什么？

**答案要点**：

| 特性 | remember | rememberSaveable |
|------|----------|------------------|
| 配置变更 | 数据丢失 | 数据保留 |
| 进程死亡 | 数据丢失 | 数据保留 |
| 存储位置 | Slot Table | SavedStateHandle |
| 适用场景 | 临时 UI 状态 | 需要持久化的状态 |

**remember 原理**：
```kotlin
// remember 在 Slot Table 中存储值
@Composable
inline fun <T> remember(calculation: () -> T): T {
    return currentComposer.cache(false, calculation)
}

// Composer.cache 实现
fun <T> cache(invalid: Boolean, block: () -> T): T {
    // 从 Slot Table 读取缓存值
    val cached = rememberedValue()
    return if (cached === Composer.Empty || invalid) {
        // 无缓存或失效，执行计算并存储
        val value = block()
        updateRememberedValue(value)
        value
    } else {
        cached as T
    }
}
```

**rememberSaveable 原理**：
```kotlin
// rememberSaveable 使用 SaveableStateRegistry
@Composable
fun <T : Any> rememberSaveable(
    vararg inputs: Any?,
    saver: Saver<T, out Any> = autoSaver(),
    key: String? = null,
    init: () -> T
): T {
    val registry = LocalSaveableStateRegistry.current
    // 从 SavedStateHandle 恢复或初始化
    val value = remember(*inputs) {
        registry?.consumeRestored(key)?.let { 
            saver.restore(it) 
        } ?: init()
    }
    // 注册保存回调
    DisposableEffect(value) {
        registry?.registerProvider(key) {
            saver.save(value)
        }
        onDispose { }
    }
    return value
}
```

---

### 5.2 原理深入题

#### Q4: 详细解释 Compose 编译器的工作原理，@Composable 注解做了什么？

**答案要点**：

**Compose 编译器是 Kotlin 编译器插件**，在编译时对 `@Composable` 函数进行转换：

**1. 添加隐藏参数**：
```kotlin
// 编译前
@Composable
fun Greeting(name: String) {
    Text("Hello, $name")
}

// 编译后
fun Greeting(
    name: String,
    $composer: Composer,  // 组合器
    $changed: Int         // 参数变化标记
) {
    // ...
}
```

**2. 插入 Group 调用**：
```kotlin
fun Greeting(name: String, $composer: Composer, $changed: Int) {
    $composer.startRestartGroup(123)  // 开始可重启组
    // ... 函数体
    $composer.endRestartGroup()?.updateScope { 
        // 重组时的回调
        Greeting(name, $composer, $changed or 0b1)
    }
}
```

**3. 生成跳过逻辑**：
```kotlin
fun Greeting(name: String, $composer: Composer, $changed: Int) {
    $composer.startRestartGroup(123)
    
    // 检查参数是否变化
    val $dirty = $changed
    if ($changed and 0b0110 == 0) {
        $dirty = $dirty or if ($composer.changed(name)) 0b0100 else 0b0010
    }
    
    // 如果参数未变化且可以跳过
    if ($dirty and 0b0011 == 0b0010 && $composer.skipping) {
        $composer.skipToGroupEnd()
    } else {
        Text("Hello, $name", $composer, 0)
    }
    
    $composer.endRestartGroup()
}
```

**4. $changed 位运算**：
- 每个参数占 3 位
- `00`: 未知，需要比较
- `01`: 相同，可跳过
- `10`: 不同，需重组
- `11`: 静态常量

---

#### Q5: Compose 的 State 是如何实现响应式更新的？Snapshot 系统是什么？

**答案要点**：

**State 响应式原理**：

```
┌─────────────────────────────────────────────────────────────┐
│                    State 响应式流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 创建 State                                              │
│     var count by mutableStateOf(0)                          │
│           │                                                 │
│           ▼                                                 │
│  2. 读取时订阅                                               │
│     Text("$count")  ──▶  当前 RecomposeScope 订阅 count     │
│           │                                                 │
│           ▼                                                 │
│  3. 写入时通知                                               │
│     count++  ──▶  通知所有订阅者（RecomposeScope）           │
│           │                                                 │
│           ▼                                                 │
│  4. 调度重组                                                 │
│     Recomposer 将无效的 scope 加入重组队列                   │
│           │                                                 │
│           ▼                                                 │
│  5. 执行重组                                                 │
│     在下一帧执行受影响的 Composable                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Snapshot 系统**：

Snapshot 是 Compose 的状态隔离机制，类似于数据库事务：

```kotlin
// Snapshot 核心概念
abstract class Snapshot {
    // 当前快照
    companion object {
        val current: Snapshot get() = currentSnapshot()
    }
    
    // 记录读取
    abstract fun recordReadOf(state: StateObject)
    
    // 记录写入
    abstract fun recordWriteOf(state: StateObject)
}

// MutableSnapshot - 可变快照
class MutableSnapshot : Snapshot {
    // 在快照中的修改是隔离的
    // 只有 apply() 后才对外可见
    fun apply(): SnapshotApplyResult
}

// 使用示例
val snapshot = Snapshot.takeMutableSnapshot()
snapshot.enter {
    // 在快照中修改状态
    state.value = newValue
}
// 应用快照，使修改可见
snapshot.apply()
```

**Snapshot 的作用**：
1. **状态隔离**：不同线程可以有独立的状态视图
2. **原子性**：多个状态修改可以原子地应用
3. **变更追踪**：自动追踪哪些状态被读取/写入
4. **冲突检测**：检测并发修改冲突

---

#### Q6: 解释 Compose 中的稳定性（Stability）概念，如何优化不稳定类型？

**答案要点**：

**稳定性定义**：
- 稳定类型：两个实例如果 `equals` 返回 true，则它们永远相等
- Compose 编译器使用稳定性判断是否可以跳过重组

**稳定类型**：
1. 基本类型：`Int`, `String`, `Float`, `Boolean` 等
2. 函数类型：`() -> Unit`, `(Int) -> String` 等
3. `@Stable` 或 `@Immutable` 标记的类
4. 所有属性都是稳定的 data class

**不稳定类型**：
```kotlin
// 不稳定：包含 var 属性
data class User(var name: String)

// 不稳定：包含集合类型
data class UserList(val users: List<User>)

// 不稳定：包含不稳定类型
data class State(val user: User)
```

**优化方法**：

```kotlin
// 1. 使用 @Immutable 标记不可变类
@Immutable
data class User(
    val id: String,
    val name: String
)

// 2. 使用 @Stable 标记可变但可追踪的类
@Stable
class UserState {
    var name by mutableStateOf("")
    var age by mutableStateOf(0)
}

// 3. 使用 ImmutableList
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.toImmutableList

@Immutable
data class UserList(
    val users: ImmutableList<User>
)

// 4. 配置编译器报告，分析稳定性
// build.gradle.kts
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
}
// 生成的报告会显示每个类的稳定性状态
```


### 5.3 实战应用题

#### Q7: LaunchedEffect、DisposableEffect、SideEffect 的区别和使用场景？

**答案要点**：

| API | 执行时机 | 清理机制 | 支持挂起 | 使用场景 |
|-----|---------|---------|---------|---------|
| LaunchedEffect | 首次组合 + key 变化 | 自动取消协程 | ✅ | 网络请求、动画 |
| DisposableEffect | 首次组合 + key 变化 | onDispose 回调 | ❌ | 注册/注销监听器 |
| SideEffect | 每次成功重组后 | 无 | ❌ | 同步状态到非 Compose 代码 |

**代码示例**：

```kotlin
@Composable
fun EffectsExample(userId: String) {
    // LaunchedEffect: 加载用户数据
    var user by remember { mutableStateOf<User?>(null) }
    LaunchedEffect(userId) {
        // userId 变化时，取消之前的请求，发起新请求
        user = repository.getUser(userId)
    }
    
    // DisposableEffect: 监听生命周期
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> analytics.logResume()
                Lifecycle.Event.ON_PAUSE -> analytics.logPause()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
    
    // SideEffect: 同步到分析系统
    SideEffect {
        analytics.setUserId(userId)
    }
}
```

**选择指南**：
- 需要协程？→ `LaunchedEffect`
- 需要清理资源？→ `DisposableEffect`
- 每次重组都要执行？→ `SideEffect`

---

#### Q8: 如何在 Compose 中实现列表性能优化？LazyColumn 的工作原理是什么？

**答案要点**：

**LazyColumn 工作原理**：
1. 只组合可见项（+ 预加载缓冲区）
2. 使用 `SubcomposeLayout` 实现按需组合
3. 复用 Composition 和 LayoutNode

```
┌─────────────────────────────────────────────────────────────┐
│                  LazyColumn 可视化                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  预加载区域（上方）                                  │  │
│   │  Item -2, Item -1                                   │  │
│   └─────────────────────────────────────────────────────┘  │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  可见区域                                           │  │
│   │  ┌─────────────────────────────────────────────┐   │  │
│   │  │  Item 0  ◀── 已组合                         │   │  │
│   │  ├─────────────────────────────────────────────┤   │  │
│   │  │  Item 1  ◀── 已组合                         │   │  │
│   │  ├─────────────────────────────────────────────┤   │  │
│   │  │  Item 2  ◀── 已组合                         │   │  │
│   │  └─────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────┘  │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  预加载区域（下方）                                  │  │
│   │  Item 3, Item 4                                     │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
│   Item 5 ~ Item N: 未组合，不占用内存                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**性能优化技巧**：

```kotlin
@Composable
fun OptimizedLazyColumn(items: List<Item>) {
    LazyColumn(
        // 1. 配置预加载
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            items = items,
            // 2. 使用稳定的 key
            key = { it.id },
            // 3. 使用 contentType 优化复用
            contentType = { it.type }
        ) { item ->
            // 4. 避免在 item 中创建新的 lambda
            ItemRow(
                item = item,
                onClick = remember(item.id) { 
                    { onItemClick(item.id) } 
                }
            )
        }
    }
}

// 5. 使用 LazyListState 监控滚动
@Composable
fun ScrollAwareLazyColumn(items: List<Item>) {
    val listState = rememberLazyListState()
    
    // 使用 derivedStateOf 减少重组
    val showScrollToTop by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }
    
    Box {
        LazyColumn(state = listState) {
            items(items, key = { it.id }) { item ->
                ItemRow(item)
            }
        }
        
        if (showScrollToTop) {
            ScrollToTopButton(
                onClick = {
                    scope.launch {
                        listState.animateScrollToItem(0)
                    }
                }
            )
        }
    }
}

// 6. 避免嵌套滚动
// ❌ 错误
LazyColumn {
    item {
        LazyRow { ... }  // 嵌套滚动
    }
}

// ✅ 正确
LazyColumn {
    item {
        Row(
            modifier = Modifier.horizontalScroll(rememberScrollState())
        ) { ... }
    }
}
```

---

#### Q9: Compose 与传统 View 如何互操作？有哪些注意事项？

**答案要点**：

**在 Compose 中使用 View**：

```kotlin
// AndroidView - 嵌入传统 View
@Composable
fun WebViewComposable(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                webViewClient = WebViewClient()
                settings.javaScriptEnabled = true
            }
        },
        update = { webView ->
            webView.loadUrl(url)
        },
        modifier = Modifier.fillMaxSize()
    )
}

// 处理 View 的生命周期
@Composable
fun LifecycleAwareView() {
    val lifecycle = LocalLifecycleOwner.current.lifecycle
    
    AndroidView(
        factory = { context ->
            MyCustomView(context)
        }
    )
    
    DisposableEffect(lifecycle) {
        val observer = LifecycleEventObserver { _, event ->
            // 处理生命周期事件
        }
        lifecycle.addObserver(observer)
        onDispose {
            lifecycle.removeObserver(observer)
        }
    }
}
```

**在 View 中使用 Compose**：

```kotlin
// ComposeView
class MyFragment : Fragment() {
    override fun onCreateView(...): View {
        return ComposeView(requireContext()).apply {
            // 重要：设置 ViewCompositionStrategy
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                MyTheme {
                    MyComposable()
                }
            }
        }
    }
}
```

**注意事项**：

1. **ViewCompositionStrategy**：必须正确设置，否则可能内存泄漏
2. **生命周期管理**：View 的生命周期需要手动处理
3. **性能考虑**：频繁切换 Compose/View 有性能开销
4. **主题一致性**：确保 Compose 和 View 使用一致的主题
5. **焦点处理**：Compose 和 View 的焦点系统不同

---

### 5.4 高级进阶题

#### Q10: 如何实现自定义 Layout？Compose 的测量和布局流程是怎样的？

**答案要点**：

**测量布局流程**：

```
┌─────────────────────────────────────────────────────────────┐
│                 Compose 渲染三阶段                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Composition（组合）                                    │
│      ┌─────────────────────────────────────────────────┐   │
│      │  执行 @Composable 函数                          │   │
│      │  构建 UI 树（LayoutNode 树）                    │   │
│      │  收集 State 依赖                                │   │
│      └─────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│   2. Layout（布局）                                         │
│      ┌─────────────────────────────────────────────────┐   │
│      │  Measure: 测量每个节点的大小                    │   │
│      │  Place: 确定每个节点的位置                      │   │
│      │  单次遍历，从上到下                             │   │
│      └─────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│   3. Drawing（绘制）                                        │
│      ┌─────────────────────────────────────────────────┐   │
│      │  遍历 LayoutNode 树                             │   │
│      │  调用 Canvas API 绘制                           │   │
│      └─────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**自定义 Layout 实现**：

```kotlin
// 自定义流式布局（FlowRow）
@Composable
fun FlowRow(
    modifier: Modifier = Modifier,
    horizontalSpacing: Dp = 0.dp,
    verticalSpacing: Dp = 0.dp,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        // 1. 测量所有子元素（不限制宽度）
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints.copy(minWidth = 0))
        }
        
        // 2. 计算行布局
        val rows = mutableListOf<List<Placeable>>()
        var currentRow = mutableListOf<Placeable>()
        var currentRowWidth = 0
        
        placeables.forEach { placeable ->
            if (currentRowWidth + placeable.width > constraints.maxWidth 
                && currentRow.isNotEmpty()) {
                rows.add(currentRow)
                currentRow = mutableListOf()
                currentRowWidth = 0
            }
            currentRow.add(placeable)
            currentRowWidth += placeable.width + horizontalSpacing.roundToPx()
        }
        if (currentRow.isNotEmpty()) {
            rows.add(currentRow)
        }
        
        // 3. 计算总高度
        val height = rows.sumOf { row ->
            row.maxOf { it.height }
        } + (rows.size - 1) * verticalSpacing.roundToPx()
        
        // 4. 布局
        layout(constraints.maxWidth, height) {
            var y = 0
            rows.forEach { row ->
                var x = 0
                val rowHeight = row.maxOf { it.height }
                row.forEach { placeable ->
                    placeable.placeRelative(x, y)
                    x += placeable.width + horizontalSpacing.roundToPx()
                }
                y += rowHeight + verticalSpacing.roundToPx()
            }
        }
    }
}

// 使用 SubcomposeLayout 实现按需组合
@Composable
fun MeasureFirst(
    content: @Composable () -> Unit,
    dependentContent: @Composable (IntSize) -> Unit
) {
    SubcomposeLayout { constraints ->
        // 先测量 content
        val contentPlaceable = subcompose("content") {
            content()
        }.first().measure(constraints)
        
        // 根据 content 的大小组合 dependentContent
        val dependentPlaceable = subcompose("dependent") {
            dependentContent(IntSize(
                contentPlaceable.width,
                contentPlaceable.height
            ))
        }.first().measure(constraints)
        
        layout(contentPlaceable.width, contentPlaceable.height) {
            contentPlaceable.placeRelative(0, 0)
            dependentPlaceable.placeRelative(0, 0)
        }
    }
}
```

---

#### Q11: Compose 中的 Modifier 是如何工作的？如何实现自定义 Modifier？

**答案要点**：

**Modifier 工作原理**：

```kotlin
// Modifier 是一个接口，通过 then 操作符链式组合
interface Modifier {
    fun <R> foldIn(initial: R, operation: (R, Element) -> R): R
    fun <R> foldOut(initial: R, operation: (Element, R) -> R): R
    infix fun then(other: Modifier): Modifier
}

// 链式调用形成 CombinedModifier 链
Modifier
    .fillMaxSize()      // Modifier1
    .padding(16.dp)     // Modifier2
    .background(Color.Red)  // Modifier3

// 内部结构
CombinedModifier(
    outer = CombinedModifier(
        outer = FillMaxSizeModifier,
        inner = PaddingModifier
    ),
    inner = BackgroundModifier
)
```

**自定义 Modifier**：

```kotlin
// 方式1：使用 Modifier.composed（有状态）
fun Modifier.shake(enabled: Boolean) = composed {
    val shakeOffset = remember { Animatable(0f) }
    
    LaunchedEffect(enabled) {
        if (enabled) {
            shakeOffset.animateTo(
                targetValue = 0f,
                animationSpec = keyframes {
                    durationMillis = 500
                    0f at 0
                    -10f at 100
                    10f at 200
                    -10f at 300
                    10f at 400
                    0f at 500
                }
            )
        }
    }
    
    this.offset { IntOffset(shakeOffset.value.roundToInt(), 0) }
}

// 方式2：使用 Modifier.Node（高性能，推荐）
fun Modifier.customBackground(color: Color) = this then CustomBackgroundElement(color)

private data class CustomBackgroundElement(
    val color: Color
) : ModifierNodeElement<CustomBackgroundNode>() {
    override fun create() = CustomBackgroundNode(color)
    override fun update(node: CustomBackgroundNode) {
        node.color = color
    }
}

private class CustomBackgroundNode(
    var color: Color
) : DrawModifierNode, Modifier.Node() {
    override fun ContentDrawScope.draw() {
        drawRect(color)
        drawContent()
    }
}

// 方式3：使用 layout 修改测量/布局
fun Modifier.customPadding(padding: Dp) = this.layout { measurable, constraints ->
    val paddingPx = padding.roundToPx()
    val placeable = measurable.measure(
        constraints.offset(-paddingPx * 2, -paddingPx * 2)
    )
    layout(
        placeable.width + paddingPx * 2,
        placeable.height + paddingPx * 2
    ) {
        placeable.placeRelative(paddingPx, paddingPx)
    }
}

// 方式4：使用 drawWithContent 自定义绘制
fun Modifier.debugBorder() = this.drawWithContent {
    drawContent()
    drawRect(
        color = Color.Red,
        style = Stroke(width = 2.dp.toPx())
    )
}
```

---

#### Q12: 如何在 Compose 中处理复杂的状态管理？ViewModel 与 Compose 如何配合？

**答案要点**：

**状态管理层次**：

```
┌─────────────────────────────────────────────────────────────┐
│                    状态管理层次                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  UI 状态（Composable 内部）                         │  │
│   │  remember { mutableStateOf(...) }                   │  │
│   │  例：展开/折叠、输入框焦点                          │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  Screen 状态（ViewModel）                           │  │
│   │  StateFlow、LiveData                                │  │
│   │  例：列表数据、加载状态、错误信息                    │  │
│   └─────────────────────────────────────────────────────┘  │
│                          │                                  │
│                          ▼                                  │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  App 状态（全局）                                   │  │
│   │  CompositionLocal、依赖注入                         │  │
│   │  例：用户登录状态、主题设置                          │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**ViewModel 与 Compose 配合**：

```kotlin
// ViewModel 定义
class UserViewModel : ViewModel() {
    // 使用 StateFlow
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
    
    // 或使用 mutableStateOf（Compose 专用）
    var uiState by mutableStateOf(UserUiState())
        private set
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val user = repository.getUser(userId)
                _uiState.update { it.copy(user = user, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }
}

// UI State 定义
@Immutable
data class UserUiState(
    val user: User? = null,
    val isLoading: Boolean = false,
    val error: String? = null
)

// Composable 使用
@Composable
fun UserScreen(
    viewModel: UserViewModel = viewModel()
) {
    // 收集 StateFlow
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    UserContent(
        uiState = uiState,
        onRetry = { viewModel.loadUser(userId) }
    )
}

// 无状态 Composable（易于测试和预览）
@Composable
fun UserContent(
    uiState: UserUiState,
    onRetry: () -> Unit
) {
    when {
        uiState.isLoading -> LoadingIndicator()
        uiState.error != null -> ErrorMessage(uiState.error, onRetry)
        uiState.user != null -> UserCard(uiState.user)
    }
}

// 使用 Hilt 注入 ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val userId: String = savedStateHandle["userId"] ?: ""
    // ...
}

@Composable
fun UserScreen(
    viewModel: UserViewModel = hiltViewModel()
) {
    // ...
}
```

---

## 6. 总结

### 6.1 Compose 核心知识点

```
┌─────────────────────────────────────────────────────────────┐
│                  Compose 知识体系                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  基础概念                                                   │
│  ├── 声明式 UI 原理                                        │
│  ├── @Composable 函数                                      │
│  └── Modifier 系统                                         │
│                                                             │
│  状态管理                                                   │
│  ├── remember / rememberSaveable                           │
│  ├── mutableStateOf / derivedStateOf                       │
│  ├── State Hoisting                                        │
│  └── ViewModel 集成                                        │
│                                                             │
│  副作用 API                                                 │
│  ├── LaunchedEffect                                        │
│  ├── DisposableEffect                                      │
│  ├── SideEffect                                            │
│  └── rememberCoroutineScope                                │
│                                                             │
│  性能优化                                                   │
│  ├── 稳定性（@Stable, @Immutable）                         │
│  ├── 智能重组跳过                                          │
│  ├── derivedStateOf                                        │
│  └── 延迟读取 State                                        │
│                                                             │
│  底层原理                                                   │
│  ├── Compose 编译器                                        │
│  ├── Slot Table                                            │
│  ├── Snapshot 系统                                         │
│  └── Recomposition 机制                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 面试准备建议

1. **理解原理**：不仅要会用，更要理解背后的原理
2. **动手实践**：自己实现自定义 Layout、Modifier
3. **性能意识**：了解性能优化的各种手段
4. **源码阅读**：阅读 Compose 核心源码
5. **对比思考**：与传统 View 系统对比，理解设计决策

---

*本文档持续更新中，如有问题欢迎讨论。*
