# ContentProvider 详解

## 1. 概述

ContentProvider 是 Android 四大组件之一，用于在不同应用之间共享数据。它提供了一套标准的接口，使得应用可以安全地访问其他应用的数据，而无需了解数据的存储方式。

**ContentProvider 的核心特点：**
- 跨进程数据共享的标准方式
- 基于 URI 的数据访问模型
- 支持权限控制
- 底层基于 Binder 机制

**典型应用场景：**
- 访问系统数据（联系人、媒体库、日历等）
- 应用间数据共享
- 使用 FileProvider 共享文件

## 2. 核心原理

### 2.1 ContentProvider 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端应用                                │
│  ContentResolver.query(uri, projection, selection, args, order) │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ContentResolver                             │
│  - 解析 URI，确定目标 ContentProvider                            │
│  - 通过 AMS 获取 ContentProvider 的 Binder 代理                  │
└─────────────────────────────┬───────────────────────────────────┘
                              │ Binder IPC
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   ActivityManagerService                         │
│  - 查找或启动目标 ContentProvider 所在进程                        │
│  - 返回 IContentProvider Binder 代理                             │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ContentProvider 进程                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                   ContentProvider                           ││
│  │  - onCreate()：初始化                                        ││
│  │  - query()：查询数据                                         ││
│  │  - insert()：插入数据                                        ││
│  │  - update()：更新数据                                        ││
│  │  - delete()：删除数据                                        ││
│  │  - getType()：返回 MIME 类型                                 ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              数据存储（SQLite、文件等）                       ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 URI 结构

```
content://com.example.provider/users/123
   │              │              │    │
   │              │              │    └── ID（可选）
   │              │              └── Path（表名或路径）
   │              └── Authority（唯一标识 Provider）
   └── Scheme（固定为 content）

完整 URI 格式：
content://authority/path/id?query#fragment
```

**URI 匹配规则：**
```kotlin
// UriMatcher 用于匹配 URI
private val uriMatcher = UriMatcher(UriMatcher.NO_MATCH).apply {
    // content://com.example.provider/users
    addURI("com.example.provider", "users", USERS)
    // content://com.example.provider/users/123
    addURI("com.example.provider", "users/#", USER_ID)
    // content://com.example.provider/users/name/john
    addURI("com.example.provider", "users/name/*", USER_NAME)
}

companion object {
    const val USERS = 1
    const val USER_ID = 2
    const val USER_NAME = 3
}
```

### 2.3 MIME 类型

ContentProvider 使用 MIME 类型描述数据格式：

```kotlin
override fun getType(uri: Uri): String? {
    return when (uriMatcher.match(uri)) {
        // 多条记录
        USERS -> "vnd.android.cursor.dir/vnd.com.example.provider.users"
        // 单条记录
        USER_ID -> "vnd.android.cursor.item/vnd.com.example.provider.users"
        else -> null
    }
}
```

**MIME 类型格式：**
- 多条记录：`vnd.android.cursor.dir/vnd.<authority>.<path>`
- 单条记录：`vnd.android.cursor.item/vnd.<authority>.<path>`

## 3. 关键源码解析

### 3.1 ContentResolver 查询流程

```java
// frameworks/base/core/java/android/content/ContentResolver.java
public final @Nullable Cursor query(@NonNull Uri uri, @Nullable String[] projection,
        @Nullable String selection, @Nullable String[] selectionArgs,
        @Nullable String sortOrder) {
    return query(uri, projection, selection, selectionArgs, sortOrder, null);
}

public final @Nullable Cursor query(@NonNull Uri uri, @Nullable String[] projection,
        @Nullable String selection, @Nullable String[] selectionArgs,
        @Nullable String sortOrder, @Nullable CancellationSignal cancellationSignal) {
    
    // 1. 获取 ContentProvider 的 Binder 代理
    IContentProvider unstableProvider = acquireUnstableProvider(uri);
    if (unstableProvider == null) {
        return null;
    }
    
    IContentProvider stableProvider = null;
    Cursor qCursor = null;
    try {
        try {
            // 2. 通过 Binder 调用 ContentProvider.query()
            qCursor = unstableProvider.query(mPackageName, uri, projection,
                    selection, selectionArgs, sortOrder, remoteCancellationSignal);
        } catch (DeadObjectException e) {
            // Provider 进程死亡，尝试重新获取
            unstableProviderDied(unstableProvider);
            stableProvider = acquireProvider(uri);
            if (stableProvider == null) {
                return null;
            }
            qCursor = stableProvider.query(mPackageName, uri, projection,
                    selection, selectionArgs, sortOrder, remoteCancellationSignal);
        }
        
        if (qCursor == null) {
            return null;
        }
        
        // 3. 包装 Cursor，确保跨进程安全
        final IContentProvider provider = (stableProvider != null) ? stableProvider
                : acquireProvider(uri);
        final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
        stableProvider = null;
        qCursor = null;
        return wrapper;
    } finally {
        // 释放资源
        if (qCursor != null) {
            qCursor.close();
        }
        if (unstableProvider != null) {
            releaseUnstableProvider(unstableProvider);
        }
        if (stableProvider != null) {
            releaseProvider(stableProvider);
        }
    }
}
```

### 3.2 获取 ContentProvider

```java
// frameworks/base/core/java/android/app/ActivityThread.java
public final IContentProvider acquireProvider(
        Context c, String auth, int userId, boolean stable) {
    
    // 1. 先从本地缓存查找
    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
    if (provider != null) {
        return provider;
    }
    
    // 2. 通过 AMS 获取
    ContentProviderHolder holder = null;
    try {
        synchronized (getGetProviderLock(auth, userId)) {
            holder = ActivityManager.getService().getContentProvider(
                    getApplicationThread(), c.getOpPackageName(), auth, userId, stable);
        }
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    
    if (holder == null) {
        return null;
    }
    
    // 3. 安装 ContentProvider
    holder = installProvider(c, holder, holder.info,
            true /*noisy*/, holder.noReleaseNeeded, stable);
    return holder.provider;
}
```

### 3.3 ContentProvider 创建过程

```java
// frameworks/base/core/java/android/app/ActivityThread.java
private void handleBindApplication(AppBindData data) {
    // ... 省略其他初始化代码
    
    // 安装 ContentProviders
    if (!data.restrictedBackupMode) {
        if (!ArrayUtils.isEmpty(data.providers)) {
            installContentProviders(app, data.providers);
        }
    }
    
    // 调用 Application.onCreate()
    mInstrumentation.callApplicationOnCreate(app);
}

private void installContentProviders(
        Context context, List<ProviderInfo> providers) {
    
    final ArrayList<ContentProviderHolder> results = new ArrayList<>();
    
    for (ProviderInfo cpi : providers) {
        // 安装每个 ContentProvider
        ContentProviderHolder cph = installProvider(context, null, cpi,
                false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
        if (cph != null) {
            cph.noReleaseNeeded = true;
            results.add(cph);
        }
    }
    
    // 发布到 AMS
    try {
        ActivityManager.getService().publishContentProviders(
            getApplicationThread(), results);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}

private ContentProviderHolder installProvider(Context context,
        ContentProviderHolder holder, ProviderInfo info,
        boolean noisy, boolean noReleaseNeeded, boolean stable) {
    
    ContentProvider localProvider = null;
    IContentProvider provider;
    
    if (holder == null || holder.provider == null) {
        // 创建 ContentProvider 实例
        try {
            final java.lang.ClassLoader cl = c.getClassLoader();
            LoadedApk packageInfo = peekPackageInfo(info.packageName, true);
            
            // 通过反射创建实例
            localProvider = packageInfo.getAppFactory()
                    .instantiateProvider(cl, info.name);
        } catch (Exception e) {
            // ...
        }
        
        // 调用 attachInfo，内部会调用 onCreate()
        localProvider.attachInfo(c, info);
    }
    
    // ... 缓存 Provider
    return retHolder;
}
```

### 3.4 ContentProvider.attachInfo

```java
// frameworks/base/core/java/android/content/ContentProvider.java
public void attachInfo(Context context, ProviderInfo info) {
    attachInfo(context, info, false);
}

private void attachInfo(Context context, ProviderInfo info, boolean testing) {
    mNoPerms = testing;
    
    if (mContext == null) {
        mContext = context;
        if (context != null && mTransport != null) {
            mTransport.mAppOpsManager = (AppOpsManager) context.getSystemService(
                    Context.APP_OPS_SERVICE);
        }
        mMyUid = Process.myUid();
        if (info != null) {
            setReadPermission(info.readPermission);
            setWritePermission(info.writePermission);
            setPathPermissions(info.pathPermissions);
            mExported = info.exported;
            mSingleUser = (info.flags & ProviderInfo.FLAG_SINGLE_USER) != 0;
            setAuthorities(info.authority);
        }
        // 调用 onCreate()
        ContentProvider.this.onCreate();
    }
}
```

## 4. 实战应用

### 4.1 自定义 ContentProvider

```kotlin
class UserProvider : ContentProvider() {
    
    private lateinit var dbHelper: UserDbHelper
    
    companion object {
        const val AUTHORITY = "com.example.provider"
        val CONTENT_URI: Uri = Uri.parse("content://$AUTHORITY/users")
        
        private const val USERS = 1
        private const val USER_ID = 2
        
        private val uriMatcher = UriMatcher(UriMatcher.NO_MATCH).apply {
            addURI(AUTHORITY, "users", USERS)
            addURI(AUTHORITY, "users/#", USER_ID)
        }
    }
    
    override fun onCreate(): Boolean {
        // 注意：onCreate 在主线程调用，避免耗时操作
        dbHelper = UserDbHelper(context!!)
        return true
    }
    
    override fun query(
        uri: Uri,
        projection: Array<out String>?,
        selection: String?,
        selectionArgs: Array<out String>?,
        sortOrder: String?
    ): Cursor? {
        val db = dbHelper.readableDatabase
        
        val cursor = when (uriMatcher.match(uri)) {
            USERS -> {
                db.query("users", projection, selection, selectionArgs, 
                    null, null, sortOrder)
            }
            USER_ID -> {
                val id = ContentUris.parseId(uri)
                db.query("users", projection, "_id = ?", arrayOf(id.toString()),
                    null, null, sortOrder)
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
        
        // 设置通知 URI，数据变化时通知观察者
        cursor?.setNotificationUri(context?.contentResolver, uri)
        return cursor
    }
    
    override fun insert(uri: Uri, values: ContentValues?): Uri? {
        val db = dbHelper.writableDatabase
        
        return when (uriMatcher.match(uri)) {
            USERS -> {
                val id = db.insert("users", null, values)
                if (id > 0) {
                    val newUri = ContentUris.withAppendedId(CONTENT_URI, id)
                    // 通知数据变化
                    context?.contentResolver?.notifyChange(newUri, null)
                    newUri
                } else {
                    throw SQLException("Failed to insert row into $uri")
                }
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
    }
    
    override fun update(
        uri: Uri,
        values: ContentValues?,
        selection: String?,
        selectionArgs: Array<out String>?
    ): Int {
        val db = dbHelper.writableDatabase
        
        val count = when (uriMatcher.match(uri)) {
            USERS -> {
                db.update("users", values, selection, selectionArgs)
            }
            USER_ID -> {
                val id = ContentUris.parseId(uri)
                db.update("users", values, "_id = ?", arrayOf(id.toString()))
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
        
        if (count > 0) {
            context?.contentResolver?.notifyChange(uri, null)
        }
        return count
    }
    
    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<out String>?): Int {
        val db = dbHelper.writableDatabase
        
        val count = when (uriMatcher.match(uri)) {
            USERS -> {
                db.delete("users", selection, selectionArgs)
            }
            USER_ID -> {
                val id = ContentUris.parseId(uri)
                db.delete("users", "_id = ?", arrayOf(id.toString()))
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
        
        if (count > 0) {
            context?.contentResolver?.notifyChange(uri, null)
        }
        return count
    }
    
    override fun getType(uri: Uri): String? {
        return when (uriMatcher.match(uri)) {
            USERS -> "vnd.android.cursor.dir/vnd.$AUTHORITY.users"
            USER_ID -> "vnd.android.cursor.item/vnd.$AUTHORITY.users"
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
    }
}
```

**Manifest 声明：**
```xml
<provider
    android:name=".UserProvider"
    android:authorities="com.example.provider"
    android:exported="true"
    android:readPermission="com.example.READ_USERS"
    android:writePermission="com.example.WRITE_USERS">
    
    <!-- 路径级别权限 -->
    <path-permission
        android:pathPrefix="/public"
        android:readPermission="com.example.READ_PUBLIC" />
</provider>
```

### 4.2 使用 ContentResolver 访问数据

```kotlin
class UserRepository(private val context: Context) {
    
    private val contentResolver = context.contentResolver
    
    // 查询所有用户
    fun getAllUsers(): List<User> {
        val users = mutableListOf<User>()
        
        contentResolver.query(
            UserProvider.CONTENT_URI,
            arrayOf("_id", "name", "email"),
            null,
            null,
            "name ASC"
        )?.use { cursor ->
            while (cursor.moveToNext()) {
                val id = cursor.getLong(cursor.getColumnIndexOrThrow("_id"))
                val name = cursor.getString(cursor.getColumnIndexOrThrow("name"))
                val email = cursor.getString(cursor.getColumnIndexOrThrow("email"))
                users.add(User(id, name, email))
            }
        }
        
        return users
    }
    
    // 插入用户
    fun insertUser(name: String, email: String): Uri? {
        val values = ContentValues().apply {
            put("name", name)
            put("email", email)
        }
        return contentResolver.insert(UserProvider.CONTENT_URI, values)
    }
    
    // 更新用户
    fun updateUser(id: Long, name: String, email: String): Int {
        val values = ContentValues().apply {
            put("name", name)
            put("email", email)
        }
        val uri = ContentUris.withAppendedId(UserProvider.CONTENT_URI, id)
        return contentResolver.update(uri, values, null, null)
    }
    
    // 删除用户
    fun deleteUser(id: Long): Int {
        val uri = ContentUris.withAppendedId(UserProvider.CONTENT_URI, id)
        return contentResolver.delete(uri, null, null)
    }
    
    // 批量操作
    fun batchInsert(users: List<User>): Int {
        val operations = ArrayList<ContentProviderOperation>()
        
        for (user in users) {
            operations.add(
                ContentProviderOperation.newInsert(UserProvider.CONTENT_URI)
                    .withValue("name", user.name)
                    .withValue("email", user.email)
                    .build()
            )
        }
        
        val results = contentResolver.applyBatch(UserProvider.AUTHORITY, operations)
        return results.size
    }
    
    // 监听数据变化
    fun observeUsers(onChange: () -> Unit): ContentObserver {
        val observer = object : ContentObserver(Handler(Looper.getMainLooper())) {
            override fun onChange(selfChange: Boolean) {
                onChange()
            }
        }
        
        contentResolver.registerContentObserver(
            UserProvider.CONTENT_URI,
            true,  // notifyForDescendants
            observer
        )
        
        return observer
    }
}
```

### 4.3 FileProvider

FileProvider 是 ContentProvider 的子类，用于安全地共享文件。

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <!-- 内部存储 files 目录 -->
    <files-path name="internal_files" path="images/" />
    
    <!-- 内部存储 cache 目录 -->
    <cache-path name="internal_cache" path="temp/" />
    
    <!-- 外部存储 files 目录 -->
    <external-files-path name="external_files" path="documents/" />
    
    <!-- 外部存储 cache 目录 -->
    <external-cache-path name="external_cache" path="cache/" />
    
    <!-- 外部存储根目录 -->
    <external-path name="external" path="Download/" />
</paths>
```

```kotlin
// 使用 FileProvider 分享文件
fun shareFile(context: Context, file: File) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        file
    )
    
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "image/*"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    
    context.startActivity(Intent.createChooser(intent, "Share Image"))
}

// 使用 FileProvider 拍照
class CameraActivity : AppCompatActivity() {
    
    private lateinit var photoUri: Uri
    private lateinit var photoFile: File
    
    private val takePicture = registerForActivityResult(
        ActivityResultContracts.TakePicture()
    ) { success ->
        if (success) {
            // 照片保存成功
            displayPhoto(photoUri)
        }
    }
    
    fun takePhoto() {
        photoFile = createImageFile()
        photoUri = FileProvider.getUriForFile(
            this,
            "${packageName}.fileprovider",
            photoFile
        )
        takePicture.launch(photoUri)
    }
    
    private fun createImageFile(): File {
        val timestamp = SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault())
            .format(Date())
        val storageDir = getExternalFilesDir(Environment.DIRECTORY_PICTURES)
        return File.createTempFile("JPEG_${timestamp}_", ".jpg", storageDir)
    }
}
```

### 4.4 权限控制

```xml
<!-- 定义权限 -->
<permission
    android:name="com.example.READ_USERS"
    android:protectionLevel="signature" />

<permission
    android:name="com.example.WRITE_USERS"
    android:protectionLevel="signature" />

<!-- Provider 声明 -->
<provider
    android:name=".UserProvider"
    android:authorities="com.example.provider"
    android:exported="true"
    android:readPermission="com.example.READ_USERS"
    android:writePermission="com.example.WRITE_USERS">
    
    <!-- 临时授权 -->
    <grant-uri-permission android:pathPattern=".*" />
</provider>
```

```kotlin
// 临时授权
fun grantTemporaryAccess(context: Context, uri: Uri, targetPackage: String) {
    context.grantUriPermission(
        targetPackage,
        uri,
        Intent.FLAG_GRANT_READ_URI_PERMISSION
    )
}

// 撤销授权
fun revokeAccess(context: Context, uri: Uri) {
    context.revokeUriPermission(uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
```

## 5. 常见面试题

### 问题1：ContentProvider 的作用是什么？为什么要使用它？

**答案要点：**
- **作用**：跨进程数据共享的标准方式
- **优势**：
  1. 统一的数据访问接口（CRUD）
  2. 支持权限控制，保证数据安全
  3. 支持数据变化通知
  4. 解耦数据存储和数据访问
- **使用场景**：
  - 访问系统数据（联系人、媒体库）
  - 应用间数据共享
  - 使用 FileProvider 共享文件

### 问题2：ContentProvider 的 onCreate 在什么时候调用？

**答案要点：**
- 在 Application.onCreate() **之前**调用
- 在应用进程启动时，由 ActivityThread.handleBindApplication() 触发
- 调用顺序：
  1. Application.attachBaseContext()
  2. ContentProvider.onCreate()
  3. Application.onCreate()
- **注意**：onCreate 在主线程执行，避免耗时操作

### 问题3：ContentProvider 是如何实现跨进程通信的？

**答案要点：**
- 底层基于 **Binder 机制**
- 流程：
  1. 客户端通过 ContentResolver 发起请求
  2. ContentResolver 通过 AMS 获取目标 Provider 的 Binder 代理
  3. 通过 Binder IPC 调用 Provider 的方法
  4. 数据通过 Cursor（CursorWindow）跨进程传递
- Cursor 使用**匿名共享内存（Ashmem）**传递大量数据

### 问题4：URI 的结构是怎样的？UriMatcher 如何使用？

**答案要点：**
- **URI 结构**：`content://authority/path/id`
  - scheme：固定为 content
  - authority：唯一标识 Provider
  - path：数据路径（表名）
  - id：具体记录 ID（可选）
- **UriMatcher 使用**：
  ```kotlin
  val matcher = UriMatcher(UriMatcher.NO_MATCH)
  matcher.addURI("authority", "path", CODE)
  matcher.addURI("authority", "path/#", CODE_ID)  // # 匹配数字
  matcher.addURI("authority", "path/*", CODE_NAME) // * 匹配字符串
  ```

### 问题5：FileProvider 是什么？为什么需要它？

**答案要点：**
- **FileProvider**：ContentProvider 的子类，用于安全共享文件
- **为什么需要**：
  - Android 7.0+ 禁止通过 file:// URI 跨应用共享文件
  - 直接暴露文件路径存在安全风险
- **使用方式**：
  1. 在 Manifest 中声明 FileProvider
  2. 配置 file_paths.xml 指定可共享目录
  3. 使用 FileProvider.getUriForFile() 获取 content:// URI
  4. 通过 Intent.FLAG_GRANT_READ_URI_PERMISSION 授权

### 问题6：ContentProvider 如何实现权限控制？

**答案要点：**
1. **Provider 级别权限**：
   ```xml
   android:readPermission="..."
   android:writePermission="..."
   ```
2. **路径级别权限**：
   ```xml
   <path-permission android:pathPrefix="/public" ... />
   ```
3. **临时授权**：
   ```kotlin
   context.grantUriPermission(package, uri, flags)
   intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
   ```
4. **签名级权限**：`android:protectionLevel="signature"`

### 问题7：ContentObserver 的作用是什么？如何使用？

**答案要点：**
- **作用**：监听 ContentProvider 数据变化
- **使用方式**：
  ```kotlin
  // 注册观察者
  contentResolver.registerContentObserver(uri, true, observer)
  
  // Provider 端通知变化
  contentResolver.notifyChange(uri, null)
  
  // 注销观察者
  contentResolver.unregisterContentObserver(observer)
  ```
- **注意**：
  - 第二个参数 notifyForDescendants 表示是否监听子路径
  - 需要在适当时机注销，避免内存泄漏

### 问题8：ContentProvider 和 SQLite 的关系是什么？

**答案要点：**
- ContentProvider 是**数据访问接口**，SQLite 是**数据存储方式**
- ContentProvider 可以使用任何存储方式：
  - SQLite 数据库
  - 文件
  - SharedPreferences
  - 网络数据
  - 内存数据
- ContentProvider 封装了数据存储细节，对外提供统一接口
- 典型实现：ContentProvider + SQLiteOpenHelper
