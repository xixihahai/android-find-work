# 新技术与 Android W

## 速记总结：2025-2026 核心技术趋势概览

> **面试前 5 分钟速览表** — 快速建立全局认知

| 技术 | 成熟度 | 面试频率 | 一句话理解 |
|------|--------|---------|-----------|
| **Jetpack Compose** | ★★★★☆ 生产就绪 | 🔥🔥🔥 极高 | 声明式 UI 框架，用函数描述界面，状态变化自动驱动 UI 更新 |
| **KMM / Compose Multiplatform** | ★★★☆☆ 稳定但生态发展中 | 🔥🔥 中高 | Kotlin 跨平台方案，共享业务逻辑，CMP 进一步共享 UI |
| **Gradle 构建优化** | ★★★★★ 成熟 | 🔥🔥 中高 | 并行构建 + 缓存 + 增量编译 + Version Catalog，大幅缩短构建时间 |
| **AI on Device** | ★★☆☆☆ 早期阶段 | 🔥 低（趋势题） | Gemini Nano / ML Kit / TFLite 让模型在端侧运行，主打隐私和离线 |
| **Android 新版本特性** | ★★★★☆ 跟随版本迭代 | 🔥🔥🔥 高 | 每年收紧权限、增强隐私，Photo Picker / 精确闹钟 / 前台服务类型 |

> **记忆口诀**：Compose 必考、KMM 要懂、Gradle 要会调、AI 知趋势、新版本看权限

---

## 1. 概述

本文介绍 Android 最新技术趋势，包括 Android W 新特性、Compose 编译器、KMM、Gradle 优化、端侧 AI 等。

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

### 3.0 Compose vs 传统 View 核心差异对比

| 维度 | Jetpack Compose | 传统 View 系统 |
|------|----------------|---------------|
| **编程范式** | 声明式：描述"UI 应该是什么样" | 命令式：描述"如何一步步构建 UI" |
| **渲染机制** | Compose Runtime → Slot Table → Canvas 直接绘制 | XML 膨胀 → View Tree → measure/layout/draw 三步走 |
| **状态管理** | `State<T>` + 单向数据流，状态变化自动触发重组 | 手动 `setText()`、`setVisibility()` 等命令式更新 |
| **性能特点** | 智能重组（仅变化部分重组）、跳过优化（$changed 参数） | 整棵 View 树 invalidate，需手动优化（ViewHolder 等） |
| **学习曲线** | 需理解重组、副作用、状态提升等新概念 | 概念成熟、资料丰富，但 XML + Java/Kotlin 双文件切换 |
| **互操作性** | `AndroidView` 嵌入传统 View；`ComposeView` 嵌入 Compose | 原生支持所有系统组件 |
| **测试** | `ComposeTestRule` 语义树测试，天然适合 UI 测试 | Espresso / UI Automator，依赖 Activity 生命周期 |
| **生态成熟度** | 核心库稳定，Material 3 完善，部分三方库仍在迁移中 | 十余年积累，几乎所有场景都有成熟方案 |

### 3.0.1 Compose 重组（Recomposition）原理图解

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    Compose 重组流程                               │
  ├──────────────────────────────────────────────────────────────────┤
  │                                                                  │
  │   State 变化                                                     │
  │     │                                                            │
  │     ▼                                                            │
  │   ┌──────────────────┐     ┌───────────────────────────────┐    │
  │   │ Snapshot 系统检测  │────▶│ 标记依赖该 State 的 Scope 为   │    │
  │   │ 状态读写          │     │ Invalid（脏）                  │    │
  │   └──────────────────┘     └──────────────┬────────────────┘    │
  │                                            │                     │
  │                                            ▼                     │
  │                            ┌───────────────────────────────┐    │
  │                            │  Recomposer 调度重组           │    │
  │                            │  （下一帧开始前）               │    │
  │                            └──────────────┬────────────────┘    │
  │                                            │                     │
  │                        ┌──────────────────┴──────────────────┐  │
  │                        │                                      │  │
  │                        ▼                                      ▼  │
  │              ┌──────────────────┐               ┌────────────┐  │
  │              │ $changed 参数检查 │               │ 无变化      │  │
  │              │ 参数是否改变？    │               │ → 跳过整个  │  │
  │              └────────┬─────────┘               │   Group     │  │
  │                       │                          └────────────┘  │
  │           ┌───────────┴───────────┐                              │
  │           ▼                       ▼                              │
  │   ┌──────────────┐      ┌──────────────┐                        │
  │   │ 参数已变化    │      │ 参数未变化    │                        │
  │   │ → 重新执行    │      │ → skipToEnd  │                        │
  │   │   Composable  │      │   跳过该组   │                        │
  │   └──────┬───────┘      └──────────────┘                        │
  │          │                                                       │
  │          ▼                                                       │
  │   ┌──────────────────────────┐                                  │
  │   │ 更新 Slot Table          │                                  │
  │   │ → 对比前后差异            │                                  │
  │   │ → 发射变更到渲染层        │                                  │
  │   └──────────────────────────┘                                  │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘

  关键概念：
  • Slot Table：存储组合树结构的线性数组，类似虚拟 DOM
  • Scope：一个可被独立重组的最小单元（通常对应一个 Composable 函数）
  • $changed 参数：编译器注入，位运算判断参数是否变化，实现"跳过优化"
```

### 3.0.2 Compose vs View 使用场景选择

| 场景 | 推荐方案 | 依据 |
|------|---------|------|
| 全新项目，无历史包袱 | **Compose** | 开发效率高，代码量少 30-50% |
| 复杂动态列表（聊天、Feed） | **Compose LazyColumn** | 天然支持 Diff，无需 Adapter/ViewHolder |
| 动画密集型界面 | **Compose** | Animation API 声明式，比属性动画更直观 |
| 已有大型项目，渐进迁移 | **混合使用** | ComposeView 嵌入 Fragment，逐步替换 |
| 高度自定义 View（地图、视频播放器） | **传统 View** | 依赖原生 SurfaceView / TextureView |
| 依赖大量三方 View 库（图表、富文本编辑） | **传统 View** | 部分库尚未提供 Compose 版本 |
| WebView 深度集成 | **传统 View** | WebView 在 Compose 中需 AndroidView 包裹，交互复杂 |
| 团队全员熟悉 XML 布局，短期交付 | **传统 View** | 学习成本是现实约束 |

> **面试回答策略**：不要极端，答"新项目优先 Compose，老项目渐进迁移，特殊场景保留 View"最稳妥。

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

## 4. Kotlin Multiplatform (KMM) / Compose Multiplatform (CMP)

### 4.0 KMM 项目结构详解

```
my-kmm-project/
├── build.gradle.kts                  ← 根构建脚本
├── settings.gradle.kts
│
├── shared/                           ← 共享模块（核心）
│   ├── build.gradle.kts              ← KMM 插件配置
│   └── src/
│       ├── commonMain/               ← 【跨平台共享代码】
│       │   └── kotlin/
│       │       ├── model/            ← 数据模型（data class）
│       │       ├── repository/       ← 仓库层（expect 声明）
│       │       ├── usecase/          ← 业务逻辑
│       │       └── network/          ← Ktor 网络请求
│       │
│       ├── commonTest/               ← 共享单元测试
│       │   └── kotlin/
│       │
│       ├── androidMain/              ← 【Android 平台实现】
│       │   └── kotlin/
│       │       ├── actual/           ← actual 实现（如数据库、文件IO）
│       │       └── platform/         ← Android Context 相关
│       │
│       ├── androidUnitTest/          ← Android 特定测试
│       │
│       ├── iosMain/                  ← 【iOS 平台实现】
│       │   └── kotlin/
│       │       ├── actual/           ← actual 实现（NSUserDefaults 等）
│       │       └── platform/         ← iOS 特定 API 调用
│       │
│       └── iosTest/                  ← iOS 特定测试
│
├── androidApp/                       ← Android 应用壳
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       └── kotlin/                   ← Android UI（Compose / XML）
│
└── iosApp/                           ← iOS 应用壳（Xcode 项目）
    ├── iosApp.xcodeproj
    └── iosApp/                       ← SwiftUI / UIKit 界面
```

```
expect/actual 机制示意：

  commonMain                  androidMain               iosMain
  ┌────────────────┐          ┌────────────────┐        ┌────────────────┐
  │ expect class   │          │ actual class   │        │ actual class   │
  │   Platform {   │ ───▶     │   Platform {   │        │   Platform {   │
  │   val name     │          │   val name =   │        │   val name =   │
  │ }              │          │   "Android"    │        │   "iOS"        │
  └────────────────┘          └────────────────┘        └────────────────┘
```

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

### 6.1 KMM vs Flutter vs React Native 详细选型对比

| 维度 | KMM / CMP | Flutter | React Native |
|------|-----------|---------|--------------|
| **语言** | Kotlin | Dart | JavaScript / TypeScript |
| **UI 共享** | KMM 不共享；CMP 可共享 | 完全共享（自绘引擎） | 共享（映射到原生组件） |
| **逻辑共享** | ✅ 核心优势 | ✅ | ✅ |
| **性能** | 原生性能（编译为平台代码） | 接近原生（Skia 引擎） | 较低（JS Bridge 通信） |
| **包体积增量** | 极小（仅共享逻辑） | +5~10 MB（引擎） | +3~7 MB（JS 引擎） |
| **热更新** | ❌ 不支持 | ❌ 官方不支持 | ✅ CodePush 等 |
| **学习成本** | 低（Kotlin 开发者）/ 高（iOS 开发者） | 中（需学 Dart） | 低（前端开发者友好） |
| **调试体验** | Android Studio + Xcode | DevTools 优秀 | Chrome DevTools |
| **生态成熟度** | 发展中（Ktor, SQLDelight） | 丰富（pub.dev 20w+ 包） | 丰富（npm 生态） |
| **原生集成** | 无缝（本身就是原生） | 需 Platform Channel | 需 Native Module |
| **大厂采用** | Netflix, Cash App, Philips | Google Pay, BMW, Alibaba | Meta, Shopify, Discord |
| **适用场景** | 已有原生团队，逻辑复用 | 全新项目，UI 一致性 | 前端团队做移动端 |

### 6.2 什么项目适合用 KMM？

| 适合 KMM 的场景 | 不适合 KMM 的场景 |
|----------------|------------------|
| ✅ 已有成熟的 Android + iOS 原生团队 | ❌ 团队没有 Kotlin 经验 |
| ✅ 业务逻辑复杂（金融计算、协议解析） | ❌ 业务极简，UI 占比 90%+ |
| ✅ 两端 UI 差异大（各端有独立设计语言） | ❌ 两端 UI 完全一致（Flutter 更合适） |
| ✅ 对性能有极致要求（编译为原生代码） | ❌ 需要频繁热更新 |
| ✅ 渐进式迁移（先共享一个模块试水） | ❌ 需要快速从零搭建 MVP |
| ✅ 需要共享到桌面端（CMP 支持 JVM / WASM） | ❌ 只有移动端，且只需支持 Android |

> **面试回答模板**：KMM 的核心价值是"逻辑复用，UI 自由"。适合业务逻辑重、两端 UI 差异大、已有原生团队的项目。与 Flutter 的区别在于：Flutter 是"全盘共享"，KMM 是"选择性共享"。

## 7. AI on Device（端侧 AI）

### 7.1 核心技术栈

| 技术 | 提供方 | 定位 | 典型应用场景 |
|------|--------|------|-------------|
| **Gemini Nano** | Google | 端侧大语言模型（SLM） | 智能回复、文本摘要、上下文理解 |
| **ML Kit** | Google | 开箱即用的机器学习 SDK | 人脸检测、文字识别（OCR）、条码扫描、姿态检测 |
| **TensorFlow Lite (TFLite)** | Google | 轻量级推理框架 | 自定义模型部署（图像分类、语音识别等） |
| **ONNX Runtime Mobile** | Microsoft | 跨框架推理引擎 | PyTorch / TF 模型统一部署 |
| **MediaPipe** | Google | 实时多媒体 AI 管线 | 手势追踪、物体检测、图像分割 |

### 7.2 端侧 AI 的优势与限制

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    端侧 AI vs 云端 AI                        │
  ├────────────────────────────┬────────────────────────────────┤
  │        端侧（On-Device）    │         云端（Cloud）           │
  ├────────────────────────────┼────────────────────────────────┤
  │ ✅ 隐私：数据不离开设备      │ ❌ 数据需上传服务器             │
  │ ✅ 速度：无网络延迟（~10ms） │ ❌ 网络延迟（100ms-2s）        │
  │ ✅ 离线：无需网络连接        │ ❌ 必须联网                     │
  │ ✅ 成本：无服务器费用        │ ❌ API 调用按量计费             │
  │ ❌ 模型小：参数量受限        │ ✅ 可用超大模型                 │
  │ ❌ 算力弱：手机 GPU/NPU 有限 │ ✅ 算力几乎无限                │
  │ ❌ 更新慢：需随 App 更新     │ ✅ 服务端随时更新模型           │
  └────────────────────────────┴────────────────────────────────┘
```

### 7.3 Gemini Nano 集成示例

```kotlin
// 检查设备是否支持 Gemini Nano
val generativeModel = GenerativeModel(
    modelName = "gemini-nano",
    // Gemini Nano 运行在设备端，无需 API Key
)

// 使用 Gemini Nano 生成文本
suspend fun generateSmartReply(context: String): String {
    val response = generativeModel.generateContent(context)
    return response.text ?: "无法生成回复"
}

// ML Kit 文字识别示例
val recognizer = TextRecognition.getClient(ChineseTextRecognizerOptions.Builder().build())
val image = InputImage.fromBitmap(bitmap, 0)
recognizer.process(image)
    .addOnSuccessListener { text ->
        // text.text 包含识别结果
        Log.d("OCR", "识别结果: ${text.text}")
    }
    .addOnFailureListener { e ->
        Log.e("OCR", "识别失败", e)
    }
```

### 7.4 面试角度

> **当前定位**：端侧 AI 在 2025-2026 年处于快速发展期，面试中属于"加分项"。
> 面试官考察的是你对技术趋势的敏感度，而非深入的模型原理。
>
> **建议回答深度**：
> - 知道有哪些方案（Gemini Nano / ML Kit / TFLite）
> - 能说清端侧 vs 云端的取舍
> - 有实际集成经验是加分项（哪怕只是 ML Kit 的 OCR）

---

## 8. 常见面试题

### 问题1：Compose 编译器做了什么？

**答案要点**：
- 识别 @Composable 函数
- 添加 Composer 参数
- 生成 Group 调用（用于 Slot Table）
- 添加 $changed 参数实现跳过优化
- 处理 remember 等状态管理

**记忆要点**：编译器做了 5 件事 → "识、加、生、跳、记"（识别注解、加 Composer、生成 Group、跳过优化、记住状态）

### 问题2：KMM 的优势和局限是什么？

**答案要点**：
- 优势：共享业务逻辑、Kotlin 语言、原生性能
- 局限：UI 不共享、iOS 生态不如 Android、工具链不够成熟

**记忆要点**：优势三个字 → "共、Kotlin、原生"；局限 → "UI 不共享 + iOS 生态弱 + 工具链不成熟"

### 问题3：如何优化 Gradle 构建速度？

**答案要点**：
- 启用并行构建和缓存
- 增加 JVM 内存
- 使用增量编译
- 禁用不需要的功能
- 使用 Version Catalog 管理依赖

**记忆要点**：五步优化 → "并行、缓存、加内存、增量、砍功能"，再加一个 Version Catalog 统一依赖

### 问题4：Flutter 和原生开发如何选择？

**答案要点**：
- Flutter：快速开发、UI 一致性、跨平台
- 原生：性能要求高、深度系统集成、已有原生代码
- 混合：Flutter 做 UI，原生做核心功能

**记忆要点**：三种选择对应三种场景 → "新项目选 Flutter、性能敏感选原生、大项目选混合"

### 问题5：Android 隐私政策的发展趋势是什么？

**答案要点**：
- 权限更细粒度
- 后台访问更受限
- 用户控制更强
- 数据最小化原则
- 透明度要求更高

**记忆要点**：五个关键词 → "细粒度、限后台、强控制、最小化、透明化"

### 问题6：Compose 的重组（Recomposition）是什么？如何避免不必要的重组？

**答案要点**：
- 重组是 State 变化时，Compose 重新执行受影响的 Composable 函数来更新 UI
- 重组是智能的：只有读取了变化 State 的 Scope 才会重组
- 避免不必要重组的方法：
  - 使用 `remember` 缓存计算结果
  - 使用 `derivedStateOf` 减少状态变化频率
  - 传递稳定类型（`@Stable` / `@Immutable` 注解）
  - 将频繁变化的状态下推到最小 Scope
  - 使用 `key()` 帮助 Compose 识别列表项

**记忆要点**：重组 = "State 变 → Scope 脏 → 重新执行"；优化五招 → "remember、derived、Stable、下推、key"

### 问题7：端侧 AI（AI on Device）有什么优势？Android 上有哪些方案？

**答案要点**：
- 优势：隐私保护（数据不离开设备）、低延迟（无网络往返）、离线可用、无服务器成本
- 限制：模型大小受限、算力有限、更新不如云端灵活
- Android 方案：
  - Gemini Nano：端侧大语言模型，适合文本生成/摘要
  - ML Kit：开箱即用，人脸/文字/条码识别
  - TFLite：自定义模型部署
  - MediaPipe：实时多媒体 AI

**记忆要点**：优势三个字 → "隐私、速度、离线"；方案四个 → "Gemini Nano（文本）、ML Kit（识别）、TFLite（自定义）、MediaPipe（实时）"

### 问题8：Compose Multiplatform (CMP) 和 KMM 是什么关系？

**答案要点**：
- KMM（Kotlin Multiplatform）：共享业务逻辑层（网络、数据库、业务规则），UI 各端独立实现
- CMP（Compose Multiplatform）：在 KMM 基础上，进一步用 Compose 共享 UI 层
- 关系：CMP 是 KMM 的上层扩展，KMM 是基础，CMP 在其上添加了跨平台 UI 能力
- CMP 当前状态：Android/Desktop 稳定，iOS 处于 Beta 阶段
- 与 Flutter 的区别：CMP 使用 Kotlin + Compose，Flutter 使用 Dart；CMP 可以渐进式采用

**记忆要点**：KMM = 共享逻辑，CMP = KMM + 共享 UI。一句话区分 → "KMM 共享大脑，CMP 连脸也共享了"
