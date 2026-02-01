# Kotlin特性

## 1. 概述

Kotlin 是一门现代化的编程语言，提供了许多强大的特性来提高开发效率和代码质量。本文将深入讲解 Kotlin 的核心特性，包括扩展函数、内联函数、空安全、委托属性、密封类等，以及它们的实现原理。

## 2. 核心原理

### 2.1 扩展函数

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          扩展函数原理                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  什么是扩展函数:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 在不修改类源码的情况下，为类添加新函数                        │   │
│  │  - 编译后变成静态方法，接收者作为第一个参数                      │   │
│  │  - 不能访问类的私有成员                                         │   │
│  │  - 静态解析，不支持多态                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  编译原理:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // Kotlin 代码                                                 │   │
│  │  fun String.addExclamation(): String {                          │   │
│  │      return this + "!"                                          │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 编译后的 Java 代码                                          │   │
│  │  public static String addExclamation(String $this$addExclamation) {│  │
│  │      return $this$addExclamation + "!";                         │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 调用                                                        │   │
│  │  "Hello".addExclamation()  // Kotlin                            │   │
│  │  StringExtKt.addExclamation("Hello")  // Java                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  扩展属性:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 扩展属性没有 backing field，必须提供 getter                 │   │
│  │  val String.lastChar: Char                                      │   │
│  │      get() = this[length - 1]                                   │   │
│  │                                                                 │   │
│  │  // 可变扩展属性                                                │   │
│  │  var StringBuilder.lastChar: Char                               │   │
│  │      get() = this[length - 1]                                   │   │
│  │      set(value) { this.setCharAt(length - 1, value) }           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  注意事项:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. 扩展函数是静态解析的，不支持多态                            │   │
│  │  2. 成员函数优先于扩展函数                                      │   │
│  │  3. 可以为可空类型定义扩展函数                                  │   │
│  │  4. 扩展函数可以重载                                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 内联函数

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          内联函数 (inline)                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  什么是内联函数:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 编译时将函数体直接插入到调用处                               │   │
│  │  - 避免函数调用开销和 Lambda 对象创建                           │   │
│  │  - 适用于高阶函数                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  inline 关键字:                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 定义内联函数                                                │   │
│  │  inline fun <T> measureTime(block: () -> T): T {                │   │
│  │      val start = System.currentTimeMillis()                     │   │
│  │      val result = block()                                       │   │
│  │      println("Time: ${System.currentTimeMillis() - start}ms")   │   │
│  │      return result                                              │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 调用                                                        │   │
│  │  val result = measureTime {                                     │   │
│  │      // 这个 Lambda 不会创建对象                                │   │
│  │      heavyOperation()                                           │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 编译后 (内联展开)                                           │   │
│  │  val start = System.currentTimeMillis()                         │   │
│  │  val result = heavyOperation()                                  │   │
│  │  println("Time: ${System.currentTimeMillis() - start}ms")       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  noinline 关键字:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 阻止某个 Lambda 参数被内联                                  │   │
│  │  inline fun foo(inlined: () -> Unit, noinline notInlined: () -> Unit) {│
│  │      inlined()      // 会被内联                                 │   │
│  │      notInlined()   // 不会被内联，可以存储或传递               │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 使用场景：需要将 Lambda 存储或传递给其他函数                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  crossinline 关键字:                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 禁止 Lambda 中的非局部返回                                  │   │
│  │  inline fun foo(crossinline block: () -> Unit) {                │   │
│  │      val runnable = Runnable {                                  │   │
│  │          block()  // 这里不能使用 return                        │   │
│  │      }                                                          │   │
│  │      runnable.run()                                             │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 使用场景：Lambda 在另一个执行上下文中调用                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  reified 关键字:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 具体化类型参数，在运行时保留泛型类型信息                    │   │
│  │  inline fun <reified T> isInstance(value: Any): Boolean {       │   │
│  │      return value is T  // 可以使用 is 检查                     │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  inline fun <reified T : Activity> Context.startActivity() {    │   │
│  │      val intent = Intent(this, T::class.java)                   │   │
│  │      startActivity(intent)                                      │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 使用                                                        │   │
│  │  context.startActivity<MainActivity>()                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 空安全机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          空安全机制                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  可空类型与非空类型:                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  var name: String = "Kotlin"   // 非空类型，不能赋值 null       │   │
│  │  var name: String? = null      // 可空类型，可以赋值 null       │   │
│  │                                                                 │   │
│  │  // 编译时检查                                                  │   │
│  │  name.length        // 非空类型，直接调用                       │   │
│  │  name?.length       // 可空类型，安全调用                       │   │
│  │  name!!.length      // 强制非空，可能抛出 NPE                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  安全调用操作符 (?.)：                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  val length = name?.length  // name 为 null 时返回 null         │   │
│  │                                                                 │   │
│  │  // 链式调用                                                    │   │
│  │  val city = user?.address?.city                                 │   │
│  │                                                                 │   │
│  │  // 编译后                                                      │   │
│  │  val length = if (name != null) name.length else null           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Elvis 操作符 (?:)：                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  val length = name?.length ?: 0  // null 时使用默认值           │   │
│  │                                                                 │   │
│  │  // 提前返回                                                    │   │
│  │  val name = user?.name ?: return                                │   │
│  │  val name = user?.name ?: throw IllegalArgumentException()      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  非空断言 (!!)：                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  val length = name!!.length  // 断言非空，null 时抛出 NPE       │   │
│  │                                                                 │   │
│  │  // 谨慎使用，可能导致 NPE                                      │   │
│  │  // 适用于确定不为 null 的场景                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  安全类型转换 (as?)：                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  val str: String? = value as? String  // 转换失败返回 null      │   │
│  │                                                                 │   │
│  │  // 等价于                                                      │   │
│  │  val str: String? = if (value is String) value else null        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  let 函数：                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 只在非空时执行                                              │   │
│  │  name?.let { nonNullName ->                                     │   │
│  │      println(nonNullName.length)                                │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 链式处理                                                    │   │
│  │  user?.name?.let { name ->                                      │   │
│  │      processName(name)                                          │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 委托属性

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          委托属性 (Delegated Properties)                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  什么是委托属性:                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 将属性的 getter/setter 委托给另一个对象                      │   │
│  │  - 使用 by 关键字                                               │   │
│  │  - 委托对象需要提供 getValue/setValue 方法                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  by lazy - 延迟初始化:                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 第一次访问时才初始化                                        │   │
│  │  val heavyObject: HeavyObject by lazy {                         │   │
│  │      println("Initializing...")                                 │   │
│  │      HeavyObject()                                              │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 线程安全模式                                                │   │
│  │  val obj by lazy(LazyThreadSafetyMode.SYNCHRONIZED) { ... }     │   │
│  │  val obj by lazy(LazyThreadSafetyMode.PUBLICATION) { ... }      │   │
│  │  val obj by lazy(LazyThreadSafetyMode.NONE) { ... }             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  observable - 可观察属性:                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  var name: String by Delegates.observable("initial") {          │   │
│  │      property, oldValue, newValue ->                            │   │
│  │      println("$oldValue -> $newValue")                          │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // vetoable - 可拒绝的属性                                     │   │
│  │  var age: Int by Delegates.vetoable(0) { _, old, new ->         │   │
│  │      new >= 0  // 返回 false 则拒绝修改                         │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Map 委托:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  class User(map: Map<String, Any?>) {                           │   │
│  │      val name: String by map                                    │   │
│  │      val age: Int by map                                        │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  val user = User(mapOf("name" to "John", "age" to 25))          │   │
│  │  println(user.name)  // John                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  自定义委托:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  class LoggingDelegate<T>(private var value: T) {               │   │
│  │      operator fun getValue(thisRef: Any?, property: KProperty<*>): T {│
│  │          println("Getting ${property.name}: $value")            │   │
│  │          return value                                           │   │
│  │      }                                                          │   │
│  │                                                                 │   │
│  │      operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {│
│  │          println("Setting ${property.name}: $value -> $newValue")│  │
│  │          value = newValue                                       │   │
│  │      }                                                          │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  var name: String by LoggingDelegate("initial")                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.5 密封类与数据类

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    密封类 (sealed class) 与数据类 (data class)           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  密封类 (sealed class):                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 限制类的继承，所有子类必须在同一文件中定义                   │   │
│  │  - 适合表示有限的状态集合                                       │   │
│  │  - when 表达式可以穷尽所有情况，无需 else                       │   │
│  │                                                                 │   │
│  │  sealed class Result<out T> {                                   │   │
│  │      data class Success<T>(val data: T) : Result<T>()           │   │
│  │      data class Error(val message: String) : Result<Nothing>()  │   │
│  │      object Loading : Result<Nothing>()                         │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // when 表达式                                                 │   │
│  │  fun handleResult(result: Result<String>) = when (result) {     │   │
│  │      is Result.Success -> println(result.data)                  │   │
│  │      is Result.Error -> println(result.message)                 │   │
│  │      is Result.Loading -> println("Loading...")                 │   │
│  │      // 不需要 else，编译器知道所有情况                         │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  密封接口 (sealed interface) - Kotlin 1.5+:                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  sealed interface Error {                                       │   │
│  │      data class NetworkError(val code: Int) : Error             │   │
│  │      data class DatabaseError(val message: String) : Error      │   │
│  │      object UnknownError : Error                                │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  数据类 (data class):                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 自动生成 equals(), hashCode(), toString(), copy()            │   │
│  │  - 自动生成 componentN() 函数用于解构                           │   │
│  │                                                                 │   │
│  │  data class User(                                               │   │
│  │      val name: String,                                          │   │
│  │      val age: Int                                               │   │
│  │  )                                                              │   │
│  │                                                                 │   │
│  │  // 自动生成的方法                                              │   │
│  │  val user1 = User("John", 25)                                   │   │
│  │  val user2 = User("John", 25)                                   │   │
│  │  println(user1 == user2)  // true (equals)                      │   │
│  │  println(user1)  // User(name=John, age=25) (toString)          │   │
│  │                                                                 │   │
│  │  // copy                                                        │   │
│  │  val user3 = user1.copy(age = 26)                               │   │
│  │                                                                 │   │
│  │  // 解构                                                        │   │
│  │  val (name, age) = user1                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  数据类的限制:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - 主构造函数至少有一个参数                                     │   │
│  │  - 主构造函数参数必须是 val 或 var                              │   │
│  │  - 不能是 abstract, open, sealed, inner                         │   │
│  │  - 只有主构造函数中的属性参与 equals/hashCode                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.6 DSL 构建

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          DSL 构建                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  什么是 DSL:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  - Domain Specific Language，领域特定语言                       │   │
│  │  - 利用 Kotlin 的语法特性构建声明式 API                         │   │
│  │  - 例如: Gradle Kotlin DSL, Anko, Jetpack Compose               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  构建 DSL 的关键特性:                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. 带接收者的 Lambda                                           │   │
│  │  2. 扩展函数                                                    │   │
│  │  3. 中缀函数                                                    │   │
│  │  4. 运算符重载                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  带接收者的 Lambda:                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  // 定义                                                        │   │
│  │  fun buildString(action: StringBuilder.() -> Unit): String {    │   │
│  │      val sb = StringBuilder()                                   │   │
│  │      sb.action()  // 或 action(sb)                              │   │
│  │      return sb.toString()                                       │   │
│  │  }                                                              │   │
│  │                                                                 │   │
│  │  // 使用                                                        │   │
│  │  val result = buildString {                                     │   │
│  │      append("Hello, ")                                          │   │
│  │      append("World!")                                           │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  HTML DSL 示例:                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  html {                                                         │   │
│  │      head {                                                     │   │
│  │          title { +"My Page" }                                   │   │
│  │      }                                                          │   │
│  │      body {                                                     │   │
│  │          h1 { +"Welcome" }                                      │   │
│  │          p { +"This is a paragraph" }                           │   │
│  │      }                                                          │   │
│  │  }                                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. 关键源码解析

### 3.1 by lazy 实现

```kotlin
/**
 * lazy 函数定义
 */
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = 
    SynchronizedLazyImpl(initializer)

public actual fun <T> lazy(
    mode: LazyThreadSafetyMode, 
    initializer: () -> T
): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }

/**
 * SynchronizedLazyImpl - 线程安全的延迟初始化
 */
private class SynchronizedLazyImpl<out T>(initializer: () -> T) : Lazy<T> {
    private var initializer: (() -> T)? = initializer
    
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    
    // 锁对象
    private val lock = this
    
    override val value: T
        get() {
            val _v1 = _value
            // 已初始化，直接返回
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }
            
            // 双重检查锁定
            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST")
                    _v2 as T
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null  // 释放引用
                    typedValue
                }
            }
        }
    
    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE
}
```

### 3.2 扩展函数编译后的代码

```kotlin
/**
 * Kotlin 扩展函数
 */
// StringExtensions.kt
fun String.addPrefix(prefix: String): String {
    return prefix + this
}

fun String.isEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}

/**
 * 编译后的 Java 代码
 */
// StringExtensionsKt.java
public final class StringExtensionsKt {
    
    @NotNull
    public static final String addPrefix(
        @NotNull String $this$addPrefix, 
        @NotNull String prefix
    ) {
        Intrinsics.checkNotNullParameter($this$addPrefix, "$this$addPrefix");
        Intrinsics.checkNotNullParameter(prefix, "prefix");
        return prefix + $this$addPrefix;
    }
    
    public static final boolean isEmail(@NotNull String $this$isEmail) {
        Intrinsics.checkNotNullParameter($this$isEmail, "$this$isEmail");
        return $this$isEmail.contains("@") && $this$isEmail.contains(".");
    }
}

// Java 中调用
String result = StringExtensionsKt.addPrefix("World", "Hello ");
boolean isEmail = StringExtensionsKt.isEmail("test@example.com");
```

### 3.3 内联函数编译后的代码

```kotlin
/**
 * 内联函数示例
 */
inline fun <T> lock(lock: Lock, action: () -> T): T {
    lock.lock()
    try {
        return action()
    } finally {
        lock.unlock()
    }
}

// 调用
fun main() {
    val lock = ReentrantLock()
    val result = lock(lock) {
        "Hello"
    }
}

/**
 * 编译后 (内联展开)
 */
fun main() {
    val lock = ReentrantLock()
    
    // lock 函数被内联展开
    lock.lock()
    val result: String
    try {
        result = "Hello"  // Lambda 也被内联
    } finally {
        lock.unlock()
    }
}

/**
 * reified 类型参数示例
 */
inline fun <reified T> Gson.fromJson(json: String): T {
    return fromJson(json, T::class.java)
}

// 使用
val user: User = gson.fromJson(jsonString)

// 编译后
val user: User = gson.fromJson(jsonString, User::class.java)
```

### 3.4 委托属性编译后的代码

```kotlin
/**
 * 委托属性示例
 */
class Example {
    var name: String by Delegates.observable("initial") { prop, old, new ->
        println("$old -> $new")
    }
}

/**
 * 编译后的代码 (简化)
 */
class Example {
    // 委托对象
    private val name$delegate = Delegates.observable("initial") { prop, old, new ->
        println("$old -> $new")
    }
    
    // 属性元数据
    private val name$metadata = ::name
    
    var name: String
        get() = name$delegate.getValue(this, name$metadata)
        set(value) = name$delegate.setValue(this, name$metadata, value)
}
```

## 4. 实战应用

### 4.1 Android 中的扩展函数

```kotlin
/**
 * View 扩展函数
 */
fun View.visible() {
    visibility = View.VISIBLE
}

fun View.gone() {
    visibility = View.GONE
}

fun View.invisible() {
    visibility = View.INVISIBLE
}

// 点击防抖
fun View.setOnSingleClickListener(interval: Long = 500, action: (View) -> Unit) {
    var lastClickTime = 0L
    setOnClickListener { view ->
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastClickTime > interval) {
            lastClickTime = currentTime
            action(view)
        }
    }
}

/**
 * Context 扩展函数
 */
inline fun <reified T : Activity> Context.startActivity(
    vararg pairs: Pair<String, Any?>
) {
    val intent = Intent(this, T::class.java).apply {
        pairs.forEach { (key, value) ->
            when (value) {
                is Int -> putExtra(key, value)
                is String -> putExtra(key, value)
                is Boolean -> putExtra(key, value)
                is Parcelable -> putExtra(key, value)
                // ... 其他类型
            }
        }
    }
    startActivity(intent)
}

// 使用
context.startActivity<DetailActivity>("id" to 123, "name" to "John")

/**
 * Toast 扩展
 */
fun Context.toast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}

fun Fragment.toast(message: String) {
    context?.toast(message)
}
```

### 4.2 使用密封类处理 UI 状态

```kotlin
/**
 * UI 状态密封类
 */
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val exception: Throwable) : UiState<Nothing>()
    object Empty : UiState<Nothing>()
}

/**
 * ViewModel 中使用
 */
class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<User>>(UiState.Loading)
    val uiState: StateFlow<UiState<User>> = _uiState.asStateFlow()
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val user = userRepository.getUser(id)
                _uiState.value = if (user != null) {
                    UiState.Success(user)
                } else {
                    UiState.Empty
                }
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e)
            }
        }
    }
}

/**
 * UI 层处理
 */
lifecycleScope.launch {
    viewModel.uiState.collect { state ->
        when (state) {
            is UiState.Loading -> showLoading()
            is UiState.Success -> showUser(state.data)
            is UiState.Error -> showError(state.exception.message)
            is UiState.Empty -> showEmpty()
        }
    }
}
```

### 4.3 DSL 构建示例

```kotlin
/**
 * 构建 RecyclerView Adapter DSL
 */
class AdapterDsl<T> {
    var items: List<T> = emptyList()
    var layoutId: Int = 0
    private var bindAction: ((View, T, Int) -> Unit)? = null
    private var clickAction: ((T, Int) -> Unit)? = null
    
    fun bind(action: (View, T, Int) -> Unit) {
        bindAction = action
    }
    
    fun onClick(action: (T, Int) -> Unit) {
        clickAction = action
    }
    
    fun build(): RecyclerView.Adapter<*> {
        return object : RecyclerView.Adapter<RecyclerView.ViewHolder>() {
            override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
                val view = LayoutInflater.from(parent.context)
                    .inflate(layoutId, parent, false)
                return object : RecyclerView.ViewHolder(view) {}
            }
            
            override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
                val item = items[position]
                bindAction?.invoke(holder.itemView, item, position)
                holder.itemView.setOnClickListener {
                    clickAction?.invoke(item, position)
                }
            }
            
            override fun getItemCount() = items.size
        }
    }
}

fun <T> adapter(init: AdapterDsl<T>.() -> Unit): RecyclerView.Adapter<*> {
    return AdapterDsl<T>().apply(init).build()
}

// 使用
val adapter = adapter<User> {
    items = userList
    layoutId = R.layout.item_user
    
    bind { view, user, position ->
        view.findViewById<TextView>(R.id.name).text = user.name
        view.findViewById<TextView>(R.id.age).text = user.age.toString()
    }
    
    onClick { user, position ->
        showUserDetail(user)
    }
}
```

### 4.4 常见坑点

1. **扩展函数不支持多态**：静态解析，调用哪个扩展函数取决于变量的声明类型
2. **内联函数不能递归**：会导致无限展开
3. **by lazy 默认是线程安全的**：单线程场景可以使用 NONE 模式提高性能
4. **数据类的 equals 只比较主构造函数参数**：其他属性不参与比较
5. **密封类的子类必须在同一文件**：Kotlin 1.5+ 可以在同一模块

## 5. 常见面试题

### 问题1：扩展函数的原理是什么？

**答案要点**：
- 编译后变成静态方法，接收者作为第一个参数
- 不能访问类的私有成员
- 静态解析，不支持多态
- 成员函数优先于扩展函数

### 问题2：inline、noinline、crossinline 的区别？

**答案要点**：
| 关键字 | 作用 | 使用场景 |
|--------|------|----------|
| inline | 内联函数和 Lambda | 高阶函数，避免对象创建 |
| noinline | 阻止 Lambda 内联 | 需要存储或传递 Lambda |
| crossinline | 禁止非局部返回 | Lambda 在其他上下文执行 |

### 问题3：reified 关键字的作用？

**答案要点**：
- 具体化类型参数，在运行时保留泛型类型信息
- 只能用于内联函数
- 可以使用 is 检查、获取 Class 对象
- 常用于简化 API，如 `startActivity<MainActivity>()`

### 问题4：by lazy 的线程安全模式有哪些？

**答案要点**：
- **SYNCHRONIZED**（默认）：双重检查锁定，线程安全
- **PUBLICATION**：允许多个线程初始化，但只使用第一个完成的值
- **NONE**：不保证线程安全，性能最好

### 问题5：密封类和枚举类的区别？

**答案要点**：
| 特性 | 密封类 | 枚举类 |
|------|--------|--------|
| 实例 | 可以有多个实例 | 每个值只有一个实例 |
| 状态 | 可以持有不同状态 | 所有值状态相同 |
| 继承 | 子类可以是类或对象 | 只能是枚举值 |
| 用途 | 复杂状态建模 | 简单枚举 |

### 问题6：Kotlin 的空安全是如何实现的？

**答案要点**：
- 编译时类型检查，区分可空类型和非空类型
- 安全调用操作符 `?.` 编译为 if-null 检查
- Elvis 操作符 `?:` 提供默认值
- 非空断言 `!!` 可能抛出 NPE
- 平台类型处理 Java 互操作

### 问题7：什么是带接收者的 Lambda？

**答案要点**：
- Lambda 内部可以直接访问接收者的成员
- 类似于扩展函数
- 用于构建 DSL
- 例如：`StringBuilder.() -> Unit`

### 问题8：data class 自动生成哪些方法？

**答案要点**：
- `equals()` / `hashCode()`：基于主构造函数参数
- `toString()`：格式化输出
- `copy()`：复制对象，可修改部分属性
- `componentN()`：解构声明
- 注意：只有主构造函数中的属性参与这些方法
