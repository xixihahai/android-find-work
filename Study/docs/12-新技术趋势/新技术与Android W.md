# 新技术与 Android W

## 1. 概述

本文介绍 Android 最新技术趋势，包括 Android W 新特性、Compose 编译器、KMM、Gradle 优化等。

## 2. Android W 新特性

### 2.1 主要更新

| 特性 | 说明 |
|------|------|
| 隐私增强 | 更严格的权限控制 |
| 性能优化 | ART 运行时改进 |
| UI 更新 | Material You 增强 |
| 安全性 | 更强的应用沙箱 |
| 开发者工具 | 新的调试和分析工具 |

### 2.2 隐私与权限

```kotlin
// 照片选择器（无需权限）
val pickMedia = registerForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
    uri?.let { handleSelectedMedia(it) }
}

// 启动选择器
pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageAndVideo))

// 精确闹钟权限
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    val alarmManager = getSystemService(AlarmManager::class.java)
    if (!alarmManager.canScheduleExactAlarms()) {
        // 引导用户授权
        startActivity(Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM))
    }
}

// 后台位置权限
// 需要先获取前台位置权限，再单独请求后台权限
```

## 3. Compose 编译器原理

### 3.1 编译过程

```
┌─────────────────────────────────────────────────────────────┐
│                 Compose 编译过程                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Kotlin 源码                                                │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │           Compose Compiler Plugin                   │   │
│   │                                                     │   │
│   │   1. 识别 @Composable 函数                          │   │
│   │   2. 添加 Composer 参数                             │   │
│   │   3. 生成 Group 调用                                │   │
│   │   4. 添加 $changed 参数（跳过优化）                  │   │
│   │   5. 生成 remember 调用                             │   │
│   │                                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│       │                                                     │
│       ▼                                                     │
│   编译后的 Kotlin 代码                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 编译前后对比

```kotlin
// 编译前
@Composable
fun Greeting(name: String) {
    Text("Hello, $name!")
}

// 编译后（简化）
fun Greeting(name: String, $composer: Composer, $changed: Int) {
    $composer.startRestartGroup(123)
    
    // 检查是否需要重组
    if ($changed and 0b0001 != 0 || !$composer.skipping) {
        Text("Hello, $name!", $composer, 0)
    } else {
        $composer.skipToGroupEnd()
    }
    
    $composer.endRestartGroup()?.updateScope { nc, _ ->
        Greeting(name, nc, $changed or 0b0001)
    }
}
```

## 4. Kotlin Multiplatform (KMM)

### 4.1 KMM 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    KMM 架构                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                   共享代码 (commonMain)              │   │
│   │                                                     │   │
│   │   - 业务逻辑                                        │   │
│   │   - 数据模型                                        │   │
│   │   - 网络请求 (Ktor)                                 │   │
│   │   - 数据库 (SQLDelight)                             │   │
│   │                                                     │   │
│   └───────────────────────┬─────────────────────────────┘   │
│                           │                                 │
│           ┌───────────────┴───────────────┐                │
│           │                               │                │
│           ▼                               ▼                │
│   ┌───────────────┐               ┌───────────────┐       │
│   │  androidMain  │               │    iosMain    │       │
│   │               │               │               │       │
│   │  - Android    │               │  - iOS 特定   │       │
│   │    特定实现    │               │    实现       │       │
│   └───────────────┘               └───────────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 KMM 示例

```kotlin
// commonMain - 共享代码
expect class Platform() {
    val name: String
}

class Greeting {
    private val platform = Platform()
    fun greet(): String = "Hello, ${platform.name}!"
}

// androidMain - Android 实现
actual class Platform actual constructor() {
    actual val name: String = "Android ${Build.VERSION.SDK_INT}"
}

// iosMain - iOS 实现
actual class Platform actual constructor() {
    actual val name: String = UIDevice.currentDevice.systemName() + " " +
            UIDevice.currentDevice.systemVersion
}
```

## 5. Gradle 构建优化

### 5.1 优化配置

```kotlin
// gradle.properties
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configureondemand=true
kotlin.incremental=true
kotlin.caching.enabled=true

// build.gradle.kts
android {
    buildFeatures {
        buildConfig = false  // 禁用不需要的功能
        aidl = false
        renderScript = false
    }
    
    // 使用 Version Catalog
    dependencies {
        implementation(libs.androidx.core)
        implementation(libs.compose.ui)
    }
}
```

### 5.2 AGP 新特性

```kotlin
// 使用 Baseline Profile
android {
    defaultConfig {
        // 启用 Baseline Profile
    }
}

dependencies {
    implementation("androidx.profileinstaller:profileinstaller:1.3.0")
    baselineProfile(project(":baselineprofile"))
}

// 使用 R8 全模式
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

## 6. 跨平台方案对比

| 特性 | Flutter | React Native | KMM |
|------|---------|--------------|-----|
| 语言 | Dart | JavaScript | Kotlin |
| UI 共享 | 是 | 是 | 否 |
| 逻辑共享 | 是 | 是 | 是 |
| 原生性能 | 接近原生 | 较低 | 原生 |
| 学习成本 | 中 | 低 | 低（Kotlin 开发者）|
| 生态 | 丰富 | 丰富 | 发展中 |
| 适用场景 | 全新项目 | 快速开发 | 已有原生项目 |

## 7. 常见面试题

### 问题1：Compose 编译器做了什么？

**答案要点**：
- 识别 @Composable 函数
- 添加 Composer 参数
- 生成 Group 调用（用于 Slot Table）
- 添加 $changed 参数实现跳过优化
- 处理 remember 等状态管理

### 问题2：KMM 的优势和局限是什么？

**答案要点**：
- 优势：共享业务逻辑、Kotlin 语言、原生性能
- 局限：UI 不共享、iOS 生态不如 Android、工具链不够成熟

### 问题3：如何优化 Gradle 构建速度？

**答案要点**：
- 启用并行构建和缓存
- 增加 JVM 内存
- 使用增量编译
- 禁用不需要的功能
- 使用 Version Catalog 管理依赖

### 问题4：Flutter 和原生开发如何选择？

**答案要点**：
- Flutter：快速开发、UI 一致性、跨平台
- 原生：性能要求高、深度系统集成、已有原生代码
- 混合：Flutter 做 UI，原生做核心功能

### 问题5：Android 隐私政策的发展趋势是什么？

**答案要点**：
- 权限更细粒度
- 后台访问更受限
- 用户控制更强
- 数据最小化原则
- 透明度要求更高
