# PMS 原理

## 速记总结

> **一句话理解 PMS**：PMS 就像一个**应用商店管家**，管理着所有 App 的安装、卸载、权限和信息查询。手机上装了什么 App、每个 App 有什么权限、谁能响应什么 Intent，PMS 全知道。

### 核心概念速记表

| 概念 | 一句话记忆 | 关键类/文件 |
|------|-----------|------------|
| PMS | System Server 中的包管理总管，开机扫描所有 APK | `PackageManagerService` |
| PackageParser | APK 的翻译官，解析 AndroidManifest.xml 提取所有信息 | `PackageParser` |
| packages.xml | 所有已安装 App 的户口本，存储在 /data/system/ | `/data/system/packages.xml` |
| PackageSetting | 内存中每个包的配置档案（对应 packages.xml 中的一条记录） | `PackageSetting` |
| ComponentResolver | Intent 的匹配引擎，负责找到能响应某个 Intent 的组件 | `ComponentResolver` |
| PermissionManagerService | 权限管家，管理所有权限的声明、授予、撤销 | `PermissionManagerService` |
| PackageInstaller | 安装器门面，处理安装会话和用户交互 | `PackageInstallerService` |
| dex2oat | DEX 字节码的编译器，将 dex 编译为机器码（OAT 格式） | `dex2oat`（Native 工具） |
| Intent Filter | 组件的能力标签，声明能响应哪些 Action/Category/Data | `IntentFilter` |
| APK 签名 | App 的身份证，确保完整性和开发者身份 | `ApkSignatureVerifier` |

---

## 1. 概述

PackageManagerService（PMS）是 Android 系统的包管理服务，负责应用的安装、卸载、权限管理和 Intent 解析。

### PMS 核心职责

| 职责 | 说明 |
|------|------|
| 包管理 | APK 安装、卸载、更新 |
| 权限管理 | 权限声明、授予、检查 |
| Intent 解析 | 查找匹配的组件 |
| 包信息查询 | 获取应用信息 |
| 签名验证 | APK 签名校验 |

## 2. 核心原理

### 2.1 PMS 启动

```java
// PackageManagerService.java
public static PackageManagerService main(Context context, Installer installer,
        @NonNull DomainVerificationService domainVerificationService,
        boolean factoryTest) {
    
    // 创建 PMS
    PackageManagerService m = new PackageManagerService(injector, factoryTest,
            PackagePartitions.FINGERPRINT, Build.IS_ENG, Build.IS_USERDEBUG,
            Build.VERSION.SDK_INT, Build.VERSION.INCREMENTAL);
    
    // 注册到 ServiceManager
    ServiceManager.addService("package", m);
    
    return m;
}

// 构造函数
public PackageManagerService(Injector injector, boolean factoryTest,
        final String partitionsFingerprint, final boolean isEngBuild,
        final boolean isUserDebugBuild, final int sdkVersion,
        final String incrementalVersion) {
    
    // 1. 扫描系统应用
    scanDirTracedLI(systemAppDir, ...);
    
    // 2. 扫描用户应用
    scanDirTracedLI(dataAppDir, ...);
    
    // 3. 更新权限
    mPermissionManager.updateAllPermissions(StorageManager.UUID_PRIVATE_INTERNAL, false);
    
    // 4. 准备数据目录
    mPrepareAppDataFuture = SystemServerInitThreadPool.submit(() -> {
        prepareAppData();
    }, "prepareAppData");
}
```

### 2.2 APK 安装流程

```
┌─────────────────────────────────────────────────────────────┐
│                    APK 安装流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐                                          │
│   │ 安装请求      │  (adb install / 应用商店 / 文件管理器)    │
│   └──────┬───────┘                                          │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │ 复制 APK     │  复制到 /data/app/                        │
│   └──────┬───────┘                                          │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │ 解析 APK     │  解析 AndroidManifest.xml                 │
│   └──────┬───────┘                                          │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │ 签名验证     │  验证 APK 签名                            │
│   └──────┬───────┘                                          │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │ DEX 优化     │  dex2oat 编译                            │
│   └──────┬───────┘                                          │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │ 创建数据目录  │  /data/data/<package>/                   │
│   └──────┬───────┘                                          │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │ 更新包信息   │  更新 packages.xml                        │
│   └──────┬───────┘                                          │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │ 发送广播     │  ACTION_PACKAGE_ADDED                     │
│   └──────────────┘                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 APK 安装完整流程详解

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    APK 安装全链路（详细版）                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ① 安装触发                                                              │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ adb install / 应用商店 / 文件管理器 / Session API │                    │
│  │ → 统一走 PackageInstallerService.createSession()  │                   │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ② 权限检查                                                              │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ 检查调用者是否有 INSTALL_PACKAGES 权限             │                    │
│  │ • 系统应用（如应用商店）→ 有权限 → 静默安装        │                    │
│  │ • 普通应用 → 无权限 → 弹出安装确认界面            │                    │
│  │ • adb → shell uid → 有权限                        │                   │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ③ 复制 APK                                                              │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ 复制到 /data/app/<packageName>-<random>/base.apk │                    │
│  │ • 临时目录 → 最终目录（原子操作）                  │                    │
│  │ • 提取 native .so 库到 lib/ 目录                  │                    │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ④ 解析 APK（PackageParser）                                             │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ 解析 AndroidManifest.xml，提取：                   │                    │
│  │ • 包名、版本、minSdkVersion                       │                    │
│  │ • 四大组件声明                                     │                    │
│  │ • 权限声明和使用                                   │                    │
│  │ • IntentFilter                                     │                   │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ⑤ 签名验证（APK Signature Scheme）                                      │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ • v1: JAR 签名（最初的签名方案）                   │                    │
│  │ • v2: APK 签名方案（Android 7.0+，整个 APK 校验） │                    │
│  │ • v3: 支持密钥轮换（Android 9.0+）               │                    │
│  │ • v4: 增量安装签名（Android 11+）                 │                    │
│  │ 更新安装时：新签名必须与旧签名一致                 │                    │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ⑥ DEX 优化（dex2oat）                                                   │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ 将 classes.dex → OAT 文件（机器码）               │                    │
│  │ • 存储在 /data/dalvik-cache/ 或 /data/app/oat/   │                    │
│  │ • 编译模式：speed / speed-profile / quicken       │                    │
│  │ • ⚠️ 这就是安装时"正在优化应用"的过程              │                    │
│  │ • 首次安装可能较慢（特别是大型 App）               │                    │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ⑦ 创建数据目录                                                          │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ • /data/data/<packageName>/  (内部存储)           │                    │
│  │ • 设置 UID、GID 和目录权限（Linux 沙箱隔离）      │                    │
│  │ • 每个 App 分配唯一 UID（如 u0_a123 = 10123）    │                    │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ⑧ 更新系统记录                                                          │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ • 更新 /data/system/packages.xml（持久化）        │                    │
│  │ • 更新内存中的 mPackages HashMap                   │                    │
│  │ • 注册所有四大组件到 ComponentResolver             │                    │
│  │ • 注册权限信息                                     │                    │
│  └──────────────────────┬───────────────────────────┘                    │
│                         ▼                                                │
│  ⑨ 发送广播                                                              │
│  ┌──────────────────────────────────────────────────┐                    │
│  │ ACTION_PACKAGE_ADDED (新装)                        │                   │
│  │ ACTION_PACKAGE_REPLACED (覆盖安装)                 │                   │
│  │ → Launcher 收到广播后刷新桌面图标                  │                    │
│  └──────────────────────────────────────────────────┘                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 安装源码分析

```java
// PackageInstallerSession.java
private void installNonStaged() throws PackageManagerException {
    // 1. 验证签名
    final PackageParser.Package pkg = parsePackage(stageDir, parseFlags);
    
    // 2. 验证权限
    verifyPackage(pkg);
    
    // 3. 安装
    mPm.installStage(params, pkg, ...);
}

// PackageManagerService.java
void installStage(InstallParams params, ParsedPackage pkg, ...) {
    // 1. 复制 APK
    final File codeFile = new File(pkg.getPath());
    
    // 2. 扫描包
    final ScanResult scanResult = scanPackageTracedLI(pkg, parseFlags, scanFlags,
            currentTime, null);
    
    // 3. 更新设置
    synchronized (mLock) {
        commitPackageSettings(pkg, oldPkg, pkgSetting, scanFlags, ...);
    }
    
    // 4. 准备数据目录
    prepareAppDataAfterInstallLIF(pkg);
    
    // 5. DEX 优化
    mPackageDexOptimizer.performDexOpt(pkg, ...);
    
    // 6. 发送广播
    sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName, ...);
}
```

### 2.5 权限管理

```java
// PermissionManagerService.java
public class PermissionManagerService {
    
    // 检查权限
    public int checkPermission(String permName, String pkgName, int userId) {
        // 获取包信息
        final PackageParser.Package pkg = mPackageManagerInt.getPackage(pkgName);
        if (pkg == null) {
            return PackageManager.PERMISSION_DENIED;
        }
        
        // 检查是否授予
        final PackageSetting ps = (PackageSetting) pkg.mExtras;
        final PermissionsState permissionsState = ps.getPermissionsState();
        
        if (permissionsState.hasPermission(permName, userId)) {
            return PackageManager.PERMISSION_GRANTED;
        }
        
        return PackageManager.PERMISSION_DENIED;
    }
    
    // 授予权限
    public void grantRuntimePermission(String packageName, String permName, int userId) {
        // 获取包设置
        final PackageSetting ps = mPackageManagerInt.getPackageSetting(packageName);
        
        // 获取权限信息
        final BasePermission bp = mSettings.getPermissionLocked(permName);
        
        // 授予权限
        final PermissionsState permissionsState = ps.getPermissionsState();
        final int result = permissionsState.grantRuntimePermission(bp, userId);
        
        if (result != PERMISSION_OPERATION_FAILURE) {
            // 持久化
            mPackageManagerInt.writeSettings(false);
            
            // 通知变化
            mOnPermissionChangeListeners.onPermissionsChanged(uid);
        }
    }
}

// Android 6.0+ 运行时权限
public class RuntimePermissionExample : AppCompatActivity() {
    
    private val requestPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            // 权限已授予
        } else {
            // 权限被拒绝
        }
    }
    
    fun checkAndRequestPermission() {
        when {
            ContextCompat.checkSelfPermission(
                this, Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> {
                // 已有权限
            }
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
                // 显示解释
                showRationale()
            }
            else -> {
                // 请求权限
                requestPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
        }
    }
}
```

### 2.6 Intent 解析

```java
// PackageManagerService.java
public ResolveInfo resolveIntent(Intent intent, String resolvedType, int flags, int userId) {
    // 查询匹配的 Activity
    final List<ResolveInfo> query = queryIntentActivitiesInternal(intent, resolvedType,
            flags, userId);
    
    // 选择最佳匹配
    return chooseBestActivity(intent, resolvedType, flags, query, userId);
}

// 查询匹配的 Activity
private List<ResolveInfo> queryIntentActivitiesInternal(Intent intent,
        String resolvedType, int flags, int userId) {
    
    // 显式 Intent
    ComponentName comp = intent.getComponent();
    if (comp != null) {
        final List<ResolveInfo> list = new ArrayList<>(1);
        final ActivityInfo ai = getActivityInfo(comp, flags, userId);
        if (ai != null) {
            final ResolveInfo ri = new ResolveInfo();
            ri.activityInfo = ai;
            list.add(ri);
        }
        return list;
    }
    
    // 隐式 Intent
    final String pkgName = intent.getPackage();
    if (pkgName == null) {
        // 查询所有匹配的 Activity
        return mComponentResolver.queryActivities(intent, resolvedType, flags, userId);
    } else {
        // 查询指定包的 Activity
        final PackageParser.Package pkg = mPackages.get(pkgName);
        if (pkg != null) {
            return mComponentResolver.queryActivities(intent, resolvedType, flags,
                    pkg.activities, userId);
        }
    }
    
    return Collections.emptyList();
}

// IntentFilter 匹配
// ComponentResolver.java
private static boolean intentFilterMatches(IntentFilter filter, Intent intent,
        String resolvedType) {
    // 匹配 Action
    if (!filter.matchAction(intent.getAction())) {
        return false;
    }
    
    // 匹配 Category
    if (!filter.matchCategories(intent.getCategories())) {
        return false;
    }
    
    // 匹配 Data
    int match = filter.matchData(intent.getType(), intent.getScheme(), intent.getData());
    if (match < 0) {
        return false;
    }
    
    return true;
}
```

## 3. 关键源码解析

### 3.1 packages.xml 结构与关键字段

```xml
<!-- /data/system/packages.xml -->
<packages>
    <!-- 权限定义 -->
    <permissions>
        <item name="android.permission.CAMERA" package="android" protection="1" />
    </permissions>
    
    <!-- 包信息 -->
    <package name="com.example.app" 
             codePath="/data/app/com.example.app-1"
             nativeLibraryPath="/data/app/com.example.app-1/lib"
             primaryCpuAbi="arm64-v8a"
             publicFlags="944291396"
             privateFlags="0"
             ft="18a1b2c3d4e"
             it="18a1b2c3d4e"
             ut="18a1b2c3d4e"
             version="1"
             userId="10123">
        <sigs count="1">
            <cert index="0" key="..." />
        </sigs>
        <perms>
            <item name="android.permission.INTERNET" granted="true" flags="0" />
        </perms>
    </package>
</packages>
```

**packages.xml 关键字段说明**：

| 字段 | 说明 | 示例 |
|------|------|------|
| `name` | 包名 | `com.example.app` |
| `codePath` | APK 安装路径 | `/data/app/com.example.app-1` |
| `nativeLibraryPath` | native .so 库路径 | `/data/app/com.example.app-1/lib` |
| `primaryCpuAbi` | CPU 架构 | `arm64-v8a` / `armeabi-v7a` |
| `ft` | APK 文件最后修改时间 (hex) | `18a1b2c3d4e` |
| `it` | 首次安装时间 (hex) | `18a1b2c3d4e` |
| `ut` | 最后更新时间 (hex) | `18a1b2c3d4e` |
| `version` | versionCode | `1` |
| `userId` | 分配的 Linux UID（10000+） | `10123` → 对应 `u0_a123` |
| `publicFlags` | 公开标志位（debuggable、系统app等） | `944291396` |
| `privateFlags` | 私有标志位 | `0` |
| `<sigs>` | APK 签名证书信息 | 用于更新时验证签名一致 |
| `<perms>` | 已授予的权限列表 | `granted="true"` 表示已授权 |

> **面试重点**：packages.xml 是 PMS 的"数据库"。PMS 启动时读取它恢复所有包信息到内存，安装/卸载时更新它进行持久化。手机恢复出厂设置就是删除这些数据。

### 3.2 Android 运行时权限模型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                Android 权限分类模型                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ 普通权限 (Normal Permission)                                 │        │
│  │ protection = "normal"                                        │        │
│  │ • 安装时自动授予，用户无感知                                  │        │
│  │ • 例：INTERNET, BLUETOOTH, ACCESS_WIFI_STATE                │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ 危险权限 (Dangerous Permission) — Android 6.0+ 运行时请求    │        │
│  │ protection = "dangerous"                                     │        │
│  │                                                              │        │
│  │  Permission Group       包含的权限                            │        │
│  │  ──────────────────     ───────────────────────              │        │
│  │  CALENDAR               READ_CALENDAR, WRITE_CALENDAR       │        │
│  │  CAMERA                 CAMERA                               │        │
│  │  CONTACTS               READ/WRITE_CONTACTS, GET_ACCOUNTS   │        │
│  │  LOCATION               ACCESS_FINE/COARSE_LOCATION         │        │
│  │  MICROPHONE             RECORD_AUDIO                         │        │
│  │  PHONE                  READ_PHONE_STATE, CALL_PHONE, ...   │        │
│  │  SENSORS                BODY_SENSORS                         │        │
│  │  SMS                    SEND/READ/RECEIVE_SMS, ...           │        │
│  │  STORAGE                READ/WRITE_EXTERNAL_STORAGE          │        │
│  │                                                              │        │
│  │  授予流程：                                                   │        │
│  │  App 请求 → 系统弹框 → 用户选择 Grant / Deny                │        │
│  │  ⚠️ 同组内授予一个，其他自动授予（Android 11 前）             │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ 签名权限 (Signature Permission)                              │        │
│  │ protection = "signature"                                     │        │
│  │ • 只有与声明权限的 App 相同签名才能获得                       │        │
│  │ • 例：系统签名权限，OEM 厂商自定义权限                       │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │ 特殊权限 (Special Permission)                                │        │
│  │ • 需要用户到设置中手动开启                                    │        │
│  │ • SYSTEM_ALERT_WINDOW → Settings.canDrawOverlays()          │        │
│  │ • WRITE_SETTINGS → Settings.System.canWrite()               │        │
│  │ • REQUEST_INSTALL_PACKAGES → 允许安装未知来源                │        │
│  │ • MANAGE_EXTERNAL_STORAGE (Android 11+) → 所有文件访问      │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  💡 权限检查流程（运行时）：                                             │
│  checkSelfPermission()                                                  │
│    → PMS.checkPermission()                                              │
│      → PermissionManagerService.checkPermission()                       │
│        → PackageSetting.getPermissionsState()                           │
│          → 返回 GRANTED / DENIED                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Intent 解析流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Intent 解析流程                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  startActivity(intent)                                                  │
│     │                                                                   │
│     ▼                                                                   │
│  PMS.resolveIntent(intent, ...)                                         │
│     │                                                                   │
│     ├── intent.getComponent() != null ?                                 │
│     │                                                                   │
│     ├── YES → 显式 Intent                                               │
│     │   │                                                               │
│     │   ▼                                                               │
│     │   直接根据 ComponentName (包名+类名) 查找                         │
│     │   │                                                               │
│     │   ▼                                                               │
│     │   getActivityInfo(componentName) → 返回目标组件                   │
│     │   ✅ 精确匹配，效率高                                             │
│     │                                                                   │
│     └── NO → 隐式 Intent                                                │
│         │                                                               │
│         ▼                                                               │
│         ComponentResolver.queryActivities(intent, ...)                  │
│         │                                                               │
│         │ 遍历所有已注册的 IntentFilter，逐项匹配：                     │
│         │                                                               │
│         ├── ① Action 匹配                                               │
│         │   IntentFilter 必须包含 Intent 的 Action                      │
│         │   filter.matchAction(intent.getAction())                      │
│         │                                                               │
│         ├── ② Category 匹配                                             │
│         │   IntentFilter 必须包含 Intent 的所有 Category                │
│         │   ⚠️ 系统会自动添加 CATEGORY_DEFAULT                          │
│         │   所以 IntentFilter 必须声明 <category DEFAULT>               │
│         │                                                               │
│         ├── ③ Data 匹配 (scheme://authority/path?type)                  │
│         │   匹配 URI 的 scheme、authority、path                         │
│         │   匹配 MIME type                                              │
│         │                                                               │
│         └── 全部匹配通过 → 加入候选列表                                 │
│                                                                         │
│         ▼                                                               │
│         chooseBestActivity(candidates)                                  │
│         │                                                               │
│         ├── 只有一个匹配 → 直接使用                                     │
│         ├── 有 preferred activity → 使用首选                            │
│         └── 多个匹配 → 弹出"选择打开方式"对话框                        │
│                                                                         │
│  💡 面试重点：                                                           │
│  • 显式 Intent 直接定位，隐式 Intent 靠 IntentFilter 匹配              │
│  • 隐式 Intent 必须同时匹配 Action + Category + Data 才算通过          │
│  • 隐式 Intent 的 Activity 必须声明 CATEGORY_DEFAULT                   │
│  • Android 11+ Package Visibility 限制了隐式查询的范围                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 APK 解析

```java
// PackageParser.java
public Package parsePackage(File packageFile, int flags) throws PackageParserException {
    if (packageFile.isDirectory()) {
        return parseClusterPackage(packageFile, flags);
    } else {
        return parseMonolithicPackage(packageFile, flags);
    }
}

private Package parseMonolithicPackage(File apkFile, int flags)
        throws PackageParserException {
    // 解析 APK
    final AssetManager assets = newConfiguredAssetManager();
    final int cookie = assets.addAssetPath(apkFile.getAbsolutePath());
    
    // 解析 AndroidManifest.xml
    final Package pkg = parseBaseApk(apkFile, assets, flags);
    
    return pkg;
}

private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
        throws PackageParserException {
    // 获取 AndroidManifest.xml
    final XmlResourceParser parser = assets.openXmlResourceParser(cookie,
            ANDROID_MANIFEST_FILENAME);
    
    // 解析
    final Package pkg = parseBaseApk(apkPath, res, parser, flags, outError);
    
    // 解析组件
    parseBaseApplication(pkg, res, parser, flags, outError);
    
    return pkg;
}
```

## 4. 实战应用

### 4.1 获取包信息

```kotlin
// 获取已安装应用列表
fun getInstalledApps(context: Context): List<ApplicationInfo> {
    val pm = context.packageManager
    return pm.getInstalledApplications(PackageManager.GET_META_DATA)
}

// 获取应用信息
fun getAppInfo(context: Context, packageName: String): ApplicationInfo? {
    return try {
        context.packageManager.getApplicationInfo(packageName, 0)
    } catch (e: PackageManager.NameNotFoundException) {
        null
    }
}

// 获取 Activity 信息
fun getActivityInfo(context: Context, componentName: ComponentName): ActivityInfo? {
    return try {
        context.packageManager.getActivityInfo(componentName, 0)
    } catch (e: PackageManager.NameNotFoundException) {
        null
    }
}

// 查询可处理 Intent 的 Activity
fun queryActivities(context: Context, intent: Intent): List<ResolveInfo> {
    return context.packageManager.queryIntentActivities(intent, 0)
}
```

## 5. 场景案例深度解析

### 5.1 安装时"正在优化应用"是什么？

**现象**：安装 App 或系统更新后，经常看到"正在优化应用 x/n..."。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  DEX 优化（dex2oat）原理                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  APK 中的代码格式演进：                                                  │
│                                                                         │
│  .java → .class → .dex → .oat/.odex/.vdex                              │
│   源码    字节码    Dalvik   ART 机器码                                   │
│                    字节码                                                │
│                                                                         │
│  Android 运行时演进：                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                              │
│  │ Dalvik   │→ │ ART      │→ │ ART +    │                              │
│  │ (解释+JIT)│  │ (AOT)    │  │ Profile  │                              │
│  │ 4.4 以前 │  │ 5.0~6.x  │  │ 7.0+     │                              │
│  └──────────┘  └──────────┘  └──────────┘                              │
│                                                                         │
│  • Dalvik：解释执行+JIT（2.2起），运行时翻译，安装快但运行较慢           │
│  • ART (5.0)：AOT（预编译），安装时全量编译，安装慢但运行快               │
│  • ART (7.0+)：混合模式                                                 │
│    - 安装时只做轻量级 verify                                             │
│    - 首次运行使用 JIT + 收集热点方法 Profile                             │
│    - 空闲充电时后台执行 dex2oat（profile-guided AOT）                    │
│    - 只编译热点代码 → 空间和时间的平衡                                   │
│                                                                         │
│  dex2oat 编译级别：                                                      │
│  ┌─────────────┬──────────────────────────────────────┐                 │
│  │ 编译级别     │ 说明                                  │                 │
│  ├─────────────┼──────────────────────────────────────┤                 │
│  │ verify       │ 只验证字节码，不编译（最快）           │                 │
│  │ quicken      │ 轻量优化（内联简单方法）               │                 │
│  │ speed-profile│ 只编译 Profile 中的热点方法（推荐）    │                 │
│  │ speed        │ 全量编译所有方法（最慢，占空间最大）    │                 │
│  └─────────────┴──────────────────────────────────────┘                 │
│                                                                         │
│  📱 "正在优化应用" = 系统更新后对所有 App 重新执行 dex2oat                │
│  因为 ART 运行时版本变了，之前编译的 OAT 文件不兼容                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 静默安装 vs 用户安装

```
┌─────────────────────────────────────────────────────────────────────────┐
│                不同安装方式对比                                           │
├──────────────┬─────────────────────┬────────────────────────────────────┤
│ 安装方式      │ 权限要求             │ 典型场景                          │
├──────────────┼─────────────────────┼────────────────────────────────────┤
│ adb install  │ shell uid (2000)    │ 开发调试                           │
│              │ 无需额外权限         │ adb install -r app.apk            │
├──────────────┼─────────────────────┼────────────────────────────────────┤
│ 系统应用商店  │ INSTALL_PACKAGES    │ Google Play / 华为应用市场         │
│ (静默安装)   │ 系统签名应用才有     │ 用户点"安装"后无确认弹窗          │
├──────────────┼─────────────────────┼────────────────────────────────────┤
│ 第三方应用    │ REQUEST_INSTALL_    │ 浏览器下载 APK 安装               │
│ (用户确认)   │ PACKAGES 权限       │ 需要用户手动开启"未知来源"        │
│              │ + 用户确认安装       │ Android 8.0+ 按应用源细分管理     │
├──────────────┼─────────────────────┼────────────────────────────────────┤
│ PackageInstaller │ createSession()  │ 应用内更新、插件化框架             │
│ Session API  │ 普通权限即可创建     │ 但安装仍需用户确认（非系统app）   │
├──────────────┼─────────────────────┼────────────────────────────────────┤
│ 设备管理器    │ Device Owner /      │ 企业 MDM 管理的设备               │
│ (完全静默)   │ Profile Owner       │ 无需用户交互                       │
├──────────────┴─────────────────────┴────────────────────────────────────┤
│                                                                         │
│  💡 面试要点：                                                           │
│  • 静默安装 = 无需用户确认，需要 INSTALL_PACKAGES 权限或 Device Owner   │
│  • 普通 App 无法实现真正的静默安装（安全设计）                           │
│  • Android 8.0+ 取消全局"未知来源"开关，改为按应用源细分管理            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```kotlin
// 使用 PackageInstaller Session API 安装（Android 5.0+）
fun installApk(context: Context, apkFile: File) {
    val packageInstaller = context.packageManager.packageInstaller
    val params = PackageInstaller.SessionParams(
        PackageInstaller.SessionParams.MODE_FULL_INSTALL
    )
    
    val sessionId = packageInstaller.createSession(params)
    val session = packageInstaller.openSession(sessionId)
    
    // 写入 APK 数据
    session.openWrite("base.apk", 0, apkFile.length()).use { output ->
        apkFile.inputStream().use { input ->
            input.copyTo(output)
        }
        session.fsync(output)
    }
    
    // 提交安装（会弹出安装确认界面）
    val intent = Intent(context, InstallResultReceiver::class.java)
    val pendingIntent = PendingIntent.getBroadcast(
        context, sessionId, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
    )
    session.commit(pendingIntent.intentSender)
}
```

### 5.3 Package Visibility（Android 11+）

**问题**：升级到 Android 11 后，`queryIntentActivities()` 返回空列表？`getPackageInfo()` 抛出 NameNotFoundException？

```
┌─────────────────────────────────────────────────────────────────────────┐
│            Package Visibility 限制（Android 11, API 30+）                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  为什么引入？                                                            │
│  • 隐私保护：防止 App 通过查询已安装应用列表来做用户画像                  │
│  • 之前任何 App 都能调用 getInstalledPackages() 获取全部已装 App         │
│                                                                         │
│  变化：                                                                  │
│  • 默认情况下，App 只能"看到"部分其他 App                               │
│  • 自动可见：系统应用、自己、通过 Intent 交互的 App                      │
│  • 其他 App 需要在 AndroidManifest.xml 中显式声明                       │
│                                                                         │
│  解决方案：在 AndroidManifest.xml 中添加 <queries>                       │
│                                                                         │
│  方式 1：声明特定包名                                                    │
│  <queries>                                                              │
│      <package android:name="com.example.target" />                      │
│  </queries>                                                             │
│                                                                         │
│  方式 2：声明 Intent Filter                                              │
│  <queries>                                                              │
│      <intent>                                                           │
│          <action android:name="android.intent.action.SEND" />           │
│          <data android:mimeType="image/*" />                            │
│      </intent>                                                          │
│  </queries>                                                             │
│                                                                         │
│  方式 3：（不推荐）QUERY_ALL_PACKAGES 权限                               │
│  • Google Play 审核严格，需要提供正当理由                                │
│  • 只有杀毒、文件管理器等特定类型 App 才能通过审核                       │
│                                                                         │
│  💡 面试要点：                                                           │
│  • Android 11+ 默认限制 App 的包可见性                                  │
│  • 需要在 Manifest 中通过 <queries> 声明需要查询的包                    │
│  • 这是 PMS 在 resolveIntent / queryIntentActivities 时增加的过滤      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.4 权限被拒绝后如何引导用户

```kotlin
/**
 * 完整的运行时权限请求最佳实践
 * 
 * shouldShowRequestPermissionRationale() 返回值含义：
 * ┌──────────────────────┬────────────────────────────────────┐
 * │ 返回值                │ 含义                               │
 * ├──────────────────────┼────────────────────────────────────┤
 * │ true                  │ 用户之前拒绝过，但没有勾选"不再询问" │
 * │                       │ → 应该先解释为什么需要权限再请求    │
 * ├──────────────────────┼────────────────────────────────────┤
 * │ false (首次请求)      │ 用户从未被请求过该权限               │
 * │                       │ → 直接请求即可                      │
 * ├──────────────────────┼────────────────────────────────────┤
 * │ false (拒绝+不再询问) │ 用户拒绝并勾选了"不再询问"          │
 * │                       │ → requestPermission 不会弹框       │
 * │                       │ → 只能引导用户去设置页手动开启       │
 * └──────────────────────┴────────────────────────────────────┘
 */
class PermissionActivity : AppCompatActivity() {

    private val cameraPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            openCamera()
        } else {
            // 区分"拒绝"和"拒绝且不再询问"
            if (!shouldShowRequestPermissionRationale(Manifest.permission.CAMERA)) {
                // 用户选择了"不再询问" → 引导到设置页
                showGoToSettingsDialog()
            } else {
                // 只是普通拒绝
                showPermissionDeniedMessage()
            }
        }
    }

    fun requestCameraPermission() {
        when {
            // 已有权限
            ContextCompat.checkSelfPermission(
                this, Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> {
                openCamera()
            }
            // 之前拒绝过，需要解释
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
                AlertDialog.Builder(this)
                    .setTitle("需要相机权限")
                    .setMessage("拍照功能需要使用相机，请授予相机权限")
                    .setPositiveButton("授予") { _, _ ->
                        cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
                    }
                    .setNegativeButton("取消", null)
                    .show()
            }
            // 首次请求，直接请求
            else -> {
                cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
        }
    }

    private fun showGoToSettingsDialog() {
        AlertDialog.Builder(this)
            .setTitle("权限被永久拒绝")
            .setMessage("请在设置中手动开启相机权限")
            .setPositiveButton("去设置") { _, _ ->
                val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                    data = Uri.parse("package:$packageName")
                }
                startActivity(intent)
            }
            .setNegativeButton("取消", null)
            .show()
    }
}
```

---

## 6. 常见面试题

### 问题1：APK 安装流程是怎样的？

**答案要点**：
1. 权限检查（调用者是否有安装权限）
2. 复制 APK 到 /data/app/
3. 解析 AndroidManifest.xml（PackageParser）
4. 验证 APK 签名（v1/v2/v3/v4）
5. DEX 优化（dex2oat 编译为机器码）
6. 创建数据目录（/data/data/<pkg>/，分配 UID）
7. 更新 packages.xml 和内存数据
8. 发送 ACTION_PACKAGE_ADDED 广播

> **记忆口诀**：权复解签优建更广（权限检查、复制、解析、签名、优化、建目录、更新记录、广播）

### 问题2：PMS 如何解析 Intent？

**答案要点**：
- **显式 Intent**：直接根据 ComponentName（包名+类名）查找，精确匹配
- **隐式 Intent**：遍历所有 IntentFilter，匹配 Action + Category + Data 三要素
- 三要素必须全部匹配才算通过
- 多个匹配时通过 `chooseBestActivity()` 选择最佳或弹出选择框
- Android 11+ 受 Package Visibility 限制，查询范围受限

> **记忆要点**：显式靠名字，隐式靠三要素（Action + Category + Data），11+ 还要 queries。

### 问题3：Android 权限机制是怎样的？

**答案要点**：
- **普通权限（normal）**：安装时自动授予，如 INTERNET
- **危险权限（dangerous）**：运行时请求（Android 6.0+），如 CAMERA、LOCATION，按权限组管理
- **签名权限（signature）**：相同签名才能授予，如系统 API 权限
- **特殊权限**：需要用户到设置页手动开启，如 SYSTEM_ALERT_WINDOW

> **记忆要点**：普通自动给，危险要问人，签名看身份，特殊去设置。

### 问题4：packages.xml 的作用是什么？

**答案要点**：
- PMS 的"持久化数据库"，存储所有已安装应用的信息
- 包括包名、路径、签名、权限、UID、安装时间等
- PMS 启动时读取它恢复内存数据结构
- 安装/卸载/更新时写入它进行持久化
- 位于 /data/system/packages.xml

> **记忆要点**：packages.xml 是 App 的户口本，PMS 开机读、变更写。

### 问题5：APK 签名的作用是什么？

**答案要点**：
- 验证 APK 完整性（防篡改）
- 确认开发者身份（不可伪造）
- 升级时验证签名一致性（防止被恶意替换）
- 签名权限的基础（signature permission 的授予条件）
- v1 → v2 → v3 → v4 逐步增强安全性

> **记忆要点**：签名 = APK 的身份证 + 防伪标签。升级必须签名一致，否则安装失败。

### 问题6：安装 App 时"正在优化"是在做什么？

**答案要点**：
- 执行 dex2oat，将 DEX 字节码编译为 ART 运行时的机器码（OAT 格式）
- Android 5.0~6.0：安装时全量 AOT 编译（安装慢，运行快）
- Android 7.0+：混合模式 — 安装时只做 verify，运行时 JIT + 收集 Profile，空闲时按 Profile AOT 编译热点代码
- 系统 OTA 更新后要对所有 App 重新 dex2oat（ART 版本变了），所以出现"正在优化应用 1/n"

> **记忆要点**：dex2oat = 把"通用语言"翻译成"本地方言"，运行更快。7.0+ 只翻译常用的。

### 问题7：Android 11 的 Package Visibility 是什么？

**答案要点**：
- Android 11（API 30）起，App 默认无法看到设备上所有已安装 App
- 目的是隐私保护，防止 App 通过查询安装列表做用户画像
- 自动可见的：系统 App、自身、通过 Intent 实际交互的 App
- 需要在 Manifest `<queries>` 中声明需要查询的包名或 Intent
- `QUERY_ALL_PACKAGES` 权限可绕过限制但 Google Play 审核严格

> **记忆要点**：11+ 默认看不到别人，想看要在 Manifest 里声明 queries。

### 问题8：shouldShowRequestPermissionRationale 的返回值如何理解？

**答案要点**：
- 返回 `true`：用户之前拒绝过，但没有勾选"不再询问" → 应该先解释再请求
- 返回 `false`（首次）：从未请求过 → 直接请求
- 返回 `false`（非首次）：用户拒绝并勾选了"不再询问" → 只能引导到设置页
- 最佳实践：请求前先检查 → rationale 弹解释框 → 请求 → 被永久拒绝则引导设置页
- 使用 `ActivityResultContracts.RequestPermission()` 代替废弃的 `onRequestPermissionsResult`

> **记忆要点**：true = 上次拒绝了但还有机会再问；false 要区分首次还是永久拒绝，永久拒绝只能去设置页。
