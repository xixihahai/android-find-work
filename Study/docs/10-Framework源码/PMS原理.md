# PMS 原理

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

### 2.3 安装源码分析

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

### 2.4 权限管理

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

### 2.5 Intent 解析

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

### 3.1 packages.xml 结构

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

### 3.2 APK 解析

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

## 5. 常见面试题

### 问题1：APK 安装流程是怎样的？

**答案要点**：
1. 复制 APK 到 /data/app/
2. 解析 AndroidManifest.xml
3. 验证签名
4. DEX 优化（dex2oat）
5. 创建数据目录
6. 更新 packages.xml
7. 发送 ACTION_PACKAGE_ADDED 广播

### 问题2：PMS 如何解析 Intent？

**答案要点**：
- 显式 Intent：直接根据 ComponentName 查找
- 隐式 Intent：匹配 Action、Category、Data
- 返回所有匹配的组件，选择最佳匹配

### 问题3：Android 权限机制是怎样的？

**答案要点**：
- 普通权限：安装时自动授予
- 危险权限：运行时请求（Android 6.0+）
- 签名权限：相同签名才能授予
- 特殊权限：需要用户在设置中手动开启

### 问题4：packages.xml 的作用是什么？

**答案要点**：
- 存储所有已安装应用的信息
- 包括包名、路径、签名、权限等
- PMS 启动时读取，更新时写入
- 位于 /data/system/packages.xml

### 问题5：APK 签名的作用是什么？

**答案要点**：
- 验证 APK 完整性
- 确认开发者身份
- 升级时验证签名一致性
- 签名权限的基础
