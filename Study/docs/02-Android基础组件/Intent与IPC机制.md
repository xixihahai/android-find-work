# Intent 与 IPC 机制

## 1. 概述

Intent 是 Android 中组件间通信的核心机制，用于启动 Activity、Service、发送广播等。IPC（Inter-Process Communication，进程间通信）是 Android 系统的基础，Binder 是 Android 特有的 IPC 机制，具有高效、安全的特点。

**本章核心内容：**
- Intent 的匹配规则和使用方式
- Binder 机制的原理（驱动、内存映射、一次拷贝）
- AIDL 的使用与原理
- 各种 IPC 方式的对比

## 2. Intent 详解

### 2.1 Intent 的组成

```
┌─────────────────────────────────────────────────────────────────┐
│                          Intent                                  │
├─────────────────────────────────────────────────────────────────┤
│  Component (组件名)    - 显式指定目标组件                         │
│  Action (动作)         - 要执行的动作                            │
│  Category (类别)       - 组件的类别信息                          │
│  Data (数据)           - 操作的数据 URI                          │
│  Type (类型)           - 数据的 MIME 类型                        │
│  Extras (附加数据)     - 键值对形式的额外数据                     │
│  Flags (标志)          - 启动模式等标志位                        │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 显式 Intent vs 隐式 Intent

```kotlin
// 显式 Intent - 明确指定目标组件
val explicitIntent = Intent(this, TargetActivity::class.java).apply {
    putExtra("key", "value")
}
startActivity(explicitIntent)

// 隐式 Intent - 通过 Action、Category、Data 匹配
val implicitIntent = Intent().apply {
    action = Intent.ACTION_VIEW
    data = Uri.parse("https://www.example.com")
    // 系统会查找能处理此 Intent 的组件
}
startActivity(implicitIntent)

// 检查是否有组件能处理隐式 Intent
if (implicitIntent.resolveActivity(packageManager) != null) {
    startActivity(implicitIntent)
}
```

### 2.3 Intent 匹配规则

#### Action 匹配

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MyActivity">
    <intent-filter>
        <action android:name="com.example.ACTION_VIEW" />
        <action android:name="com.example.ACTION_EDIT" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

**规则：**
- Intent 的 Action 必须与 IntentFilter 中的**某一个** Action 匹配
- IntentFilter 可以声明多个 Action
- Intent 必须指定 Action（隐式 Intent）

#### Category 匹配

```xml
<intent-filter>
    <action android:name="com.example.ACTION" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
</intent-filter>
```

**规则：**
- Intent 中的**所有** Category 都必须在 IntentFilter 中存在
- IntentFilter 可以有额外的 Category
- 使用 startActivity 时，系统会自动添加 `DEFAULT` Category
- 因此 IntentFilter 必须包含 `DEFAULT` Category

#### Data 匹配

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data
        android:scheme="https"
        android:host="www.example.com"
        android:pathPrefix="/article" />
</intent-filter>
```

**Data 的组成：**
```
scheme://host:port/path
https://www.example.com:443/article/123

- scheme: 协议（http、https、content、file 等）
- host: 主机名
- port: 端口号
- path: 路径
- pathPrefix: 路径前缀
- pathPattern: 路径模式（支持通配符）
```

**匹配规则：**
- 如果 IntentFilter 指定了 Data，Intent 必须包含匹配的 Data
- scheme、host、port、path 都要匹配
- 如果只指定 scheme，则只匹配 scheme

### 2.4 常用系统 Intent

```kotlin
// 打开网页
val webIntent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.example.com"))

// 拨打电话
val dialIntent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:10086"))

// 发送短信
val smsIntent = Intent(Intent.ACTION_SENDTO, Uri.parse("smsto:10086")).apply {
    putExtra("sms_body", "Hello")
}

// 发送邮件
val emailIntent = Intent(Intent.ACTION_SENDTO, Uri.parse("mailto:test@example.com")).apply {
    putExtra(Intent.EXTRA_SUBJECT, "Subject")
    putExtra(Intent.EXTRA_TEXT, "Body")
}

// 选择图片
val pickIntent = Intent(Intent.ACTION_PICK).apply {
    type = "image/*"
}

// 拍照
val cameraIntent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)

// 分享文本
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Share content")
}

// 安装 APK（Android 7.0+ 需要 FileProvider）
val installIntent = Intent(Intent.ACTION_VIEW).apply {
    setDataAndType(apkUri, "application/vnd.android.package-archive")
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
```

## 3. Binder 机制原理

### 3.1 为什么使用 Binder

| IPC 方式 | 拷贝次数 | 特点 |
|---------|---------|------|
| 共享内存 | 0 | 复杂，需要同步机制 |
| Binder | 1 | 高效，安全，支持身份验证 |
| Socket | 2 | 通用，但效率较低 |
| 管道/消息队列 | 2 | 传统 Linux IPC |

**Binder 的优势：**
1. **性能**：只需一次数据拷贝
2. **安全**：支持 UID/PID 身份验证
3. **易用**：C/S 架构，面向对象

### 3.2 Binder 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户空间                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │   Client 进程    │              │   Server 进程    │          │
│  │                 │              │                 │          │
│  │  ┌───────────┐  │              │  ┌───────────┐  │          │
│  │  │  Proxy    │  │              │  │   Stub    │  │          │
│  │  │ (代理对象) │  │              │  │ (存根对象) │  │          │
│  │  └─────┬─────┘  │              │  └─────┬─────┘  │          │
│  │        │        │              │        │        │          │
│  │  ┌─────▼─────┐  │              │  ┌─────▼─────┐  │          │
│  │  │ IPCThread │  │              │  │ IPCThread │  │          │
│  │  │ Pool      │  │              │  │ Pool      │  │          │
│  │  └─────┬─────┘  │              │  └─────┬─────┘  │          │
│  └────────┼────────┘              └────────┼────────┘          │
│           │                                │                    │
├───────────┼────────────────────────────────┼────────────────────┤
│           │         内核空间               │                    │
│           │                                │                    │
│           ▼                                ▼                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Binder 驱动                              ││
│  │  /dev/binder                                                ││
│  │                                                             ││
│  │  ┌─────────────────────────────────────────────────────┐   ││
│  │  │              内核缓冲区（mmap 映射）                   │   ││
│  │  │                                                     │   ││
│  │  │  数据从 Client 用户空间 copy_from_user 到内核缓冲区    │   ││
│  │  │  Server 通过 mmap 直接访问内核缓冲区（零拷贝）         │   ││
│  │  └─────────────────────────────────────────────────────┘   ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Binder 通信流程

```
Client 进程                 Binder 驱动                 Server 进程
    │                           │                           │
    │  1. 调用 Proxy 方法        │                           │
    │  2. 序列化参数到 Parcel    │                           │
    │                           │                           │
    │  3. transact()            │                           │
    ├──────────────────────────►│                           │
    │                           │                           │
    │     4. copy_from_user     │                           │
    │     (用户空间→内核空间)    │                           │
    │                           │                           │
    │                           │  5. 唤醒 Server 线程       │
    │                           ├──────────────────────────►│
    │                           │                           │
    │                           │  6. Server 通过 mmap      │
    │                           │     直接读取数据          │
    │                           │                           │
    │                           │  7. onTransact()          │
    │                           │     反序列化，执行方法     │
    │                           │                           │
    │                           │  8. 写入返回值            │
    │                           │◄──────────────────────────┤
    │                           │                           │
    │  9. 返回结果               │                           │
    │◄──────────────────────────┤                           │
    │                           │                           │
    │  10. 反序列化返回值        │                           │
    │                           │                           │
```

### 3.4 一次拷贝原理

```
┌─────────────────────────────────────────────────────────────────┐
│                     传统 IPC（两次拷贝）                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client 用户空间  ──copy_from_user──►  内核空间                  │
│                                           │                     │
│                                    copy_to_user                 │
│                                           │                     │
│                                           ▼                     │
│                                    Server 用户空间              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Binder（一次拷贝）                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client 用户空间  ──copy_from_user──►  内核缓冲区                │
│                                           │                     │
│                                         mmap                    │
│                                      (内存映射)                  │
│                                           │                     │
│                                           ▼                     │
│                                    Server 用户空间              │
│                                    (直接访问内核缓冲区)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**关键点：**
- Server 进程通过 `mmap()` 将内核缓冲区映射到用户空间
- Client 数据通过 `copy_from_user()` 拷贝到内核缓冲区
- Server 直接访问映射的内存，无需再次拷贝
- 因此只有一次数据拷贝

## 4. AIDL 使用与原理

### 4.1 AIDL 基本使用

**1. 定义 AIDL 接口**

```java
// IBookManager.aidl
package com.example.aidl;

import com.example.aidl.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
    void registerCallback(IBookCallback callback);
    void unregisterCallback(IBookCallback callback);
}

// Book.aidl
package com.example.aidl;

parcelable Book;

// IBookCallback.aidl
package com.example.aidl;

interface IBookCallback {
    void onBookAdded(in Book book);
}
```

**2. 实现 Parcelable 数据类**

```kotlin
// Book.kt
@Parcelize
data class Book(
    val id: Int,
    val name: String
) : Parcelable
```

**3. 实现服务端**

```kotlin
class BookService : Service() {
    
    private val bookList = mutableListOf<Book>()
    private val callbacks = RemoteCallbackList<IBookCallback>()
    
    private val binder = object : IBookManager.Stub() {
        
        override fun getBookList(): List<Book> {
            return bookList.toList()
        }
        
        override fun addBook(book: Book) {
            bookList.add(book)
            notifyBookAdded(book)
        }
        
        override fun registerCallback(callback: IBookCallback) {
            callbacks.register(callback)
        }
        
        override fun unregisterCallback(callback: IBookCallback) {
            callbacks.unregister(callback)
        }
    }
    
    private fun notifyBookAdded(book: Book) {
        val count = callbacks.beginBroadcast()
        for (i in 0 until count) {
            try {
                callbacks.getBroadcastItem(i).onBookAdded(book)
            } catch (e: RemoteException) {
                e.printStackTrace()
            }
        }
        callbacks.finishBroadcast()
    }
    
    override fun onBind(intent: Intent): IBinder = binder
}
```

**4. 实现客户端**

```kotlin
class BookActivity : AppCompatActivity() {
    
    private var bookManager: IBookManager? = null
    
    private val callback = object : IBookCallback.Stub() {
        override fun onBookAdded(book: Book) {
            runOnUiThread {
                // 更新 UI
                Log.d("TAG", "New book: ${book.name}")
            }
        }
    }
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            bookManager = IBookManager.Stub.asInterface(service)
            
            // 注册回调
            bookManager?.registerCallback(callback)
            
            // 获取数据
            val books = bookManager?.bookList
            Log.d("TAG", "Books: $books")
        }
        
        override fun onServiceDisconnected(name: ComponentName) {
            bookManager = null
        }
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 绑定服务
        val intent = Intent(this, BookService::class.java)
        bindService(intent, connection, Context.BIND_AUTO_CREATE)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        bookManager?.unregisterCallback(callback)
        unbindService(connection)
    }
    
    fun addBook() {
        val book = Book(1, "Android 开发艺术探索")
        bookManager?.addBook(book)
    }
}
```

### 4.2 AIDL 生成代码分析

```java
// 编译器自动生成的 IBookManager.java
public interface IBookManager extends android.os.IInterface {
    
    // Stub 类 - 服务端实现
    public static abstract class Stub extends android.os.Binder 
            implements IBookManager {
        
        private static final String DESCRIPTOR = "com.example.aidl.IBookManager";
        
        // 方法标识
        static final int TRANSACTION_getBookList = IBinder.FIRST_CALL_TRANSACTION + 0;
        static final int TRANSACTION_addBook = IBinder.FIRST_CALL_TRANSACTION + 1;
        
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        
        // 将 IBinder 转换为接口
        public static IBookManager asInterface(android.os.IBinder obj) {
            if (obj == null) {
                return null;
            }
            // 查询本地接口（同进程）
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (iin != null && iin instanceof IBookManager) {
                return (IBookManager) iin;  // 同进程，直接返回
            }
            // 跨进程，返回 Proxy
            return new Proxy(obj);
        }
        
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }
        
        // 处理客户端请求
        @Override
        public boolean onTransact(int code, android.os.Parcel data,
                android.os.Parcel reply, int flags) throws RemoteException {
            
            switch (code) {
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    // 调用实际方法
                    List<Book> result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    // 反序列化参数
                    Book book = data.readTypedObject(Book.CREATOR);
                    // 调用实际方法
                    this.addBook(book);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }
        
        // Proxy 类 - 客户端代理
        private static class Proxy implements IBookManager {
            
            private android.os.IBinder mRemote;
            
            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }
            
            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }
            
            @Override
            public List<Book> getBookList() throws RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                List<Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    // 发起跨进程调用
                    mRemote.transact(TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    // 反序列化返回值
                    _result = _reply.createTypedArrayList(Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            
            @Override
            public void addBook(Book book) throws RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    // 序列化参数
                    _data.writeTypedObject(book, 0);
                    // 发起跨进程调用
                    mRemote.transact(TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }
    }
    
    // 接口方法声明
    List<Book> getBookList() throws RemoteException;
    void addBook(Book book) throws RemoteException;
}
```

### 4.3 AIDL 参数方向

```java
// in: 客户端 → 服务端（默认）
void addBook(in Book book);

// out: 服务端 → 客户端
void getBook(out Book book);

// inout: 双向传递
void updateBook(inout Book book);
```

**注意：**
- `in` 参数在服务端修改不会影响客户端
- `out` 参数客户端传入的值服务端收不到
- `inout` 双向同步，但性能开销最大

## 5. Messenger

Messenger 是基于 AIDL 的轻量级 IPC 方案，适合简单的消息传递。

```kotlin
// 服务端
class MessengerService : Service() {
    
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                MSG_FROM_CLIENT -> {
                    val data = msg.data.getString("data")
                    Log.d("TAG", "Received: $data")
                    
                    // 回复客户端
                    val replyMessenger = msg.replyTo
                    val replyMsg = Message.obtain(null, MSG_FROM_SERVER)
                    replyMsg.data = Bundle().apply {
                        putString("reply", "Server received: $data")
                    }
                    replyMessenger?.send(replyMsg)
                }
            }
        }
    }
    
    private val messenger = Messenger(handler)
    
    override fun onBind(intent: Intent): IBinder = messenger.binder
    
    companion object {
        const val MSG_FROM_CLIENT = 1
        const val MSG_FROM_SERVER = 2
    }
}

// 客户端
class MessengerActivity : AppCompatActivity() {
    
    private var serverMessenger: Messenger? = null
    
    private val clientHandler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                MessengerService.MSG_FROM_SERVER -> {
                    val reply = msg.data.getString("reply")
                    Log.d("TAG", "Reply: $reply")
                }
            }
        }
    }
    
    private val clientMessenger = Messenger(clientHandler)
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            serverMessenger = Messenger(service)
            sendMessage("Hello from client")
        }
        
        override fun onServiceDisconnected(name: ComponentName) {
            serverMessenger = null
        }
    }
    
    private fun sendMessage(data: String) {
        val msg = Message.obtain(null, MessengerService.MSG_FROM_CLIENT)
        msg.data = Bundle().apply {
            putString("data", data)
        }
        msg.replyTo = clientMessenger  // 设置回复 Messenger
        serverMessenger?.send(msg)
    }
}
```

## 6. Bundle 大小限制

```kotlin
// Bundle 通过 Binder 传输，有大小限制
// Binder 事务缓冲区大小约 1MB（所有进程共享）
// 单次传输建议不超过 500KB

// ❌ 错误：传输大数据
val intent = Intent(this, TargetActivity::class.java).apply {
    putExtra("large_data", veryLargeByteArray)  // 可能导致 TransactionTooLargeException
}

// ✅ 正确：使用其他方式传输大数据
// 1. 写入文件，传递文件路径
// 2. 使用 ContentProvider
// 3. 使用数据库
// 4. 使用 ViewModel（同进程）
```

## 7. IPC 方式对比

| IPC 方式 | 优点 | 缺点 | 使用场景 |
|---------|------|------|---------|
| Bundle | 简单易用 | 大小限制，只能传 Parcelable | Activity/Service 间传参 |
| 文件共享 | 简单 | 不适合高并发，无同步机制 | 简单数据共享 |
| Messenger | 简单，支持回调 | 串行处理，不支持并发 | 简单消息传递 |
| AIDL | 功能强大，支持并发 | 复杂 | 复杂跨进程通信 |
| ContentProvider | 标准接口，支持权限 | 主要用于数据共享 | 数据共享 |
| Socket | 通用，支持网络 | 效率较低 | 网络通信 |
| BroadcastReceiver | 一对多 | 效率低，不适合大数据 | 广播通知 |

## 8. 常见面试题

### 问题1：Intent 的匹配规则是什么？

**答案要点：**
- **Action**：Intent 的 Action 必须与 IntentFilter 中的某一个 Action 匹配
- **Category**：Intent 中的所有 Category 都必须在 IntentFilter 中存在
- **Data**：
  - 如果 IntentFilter 指定了 Data，Intent 必须包含匹配的 Data
  - scheme、host、port、path 都要匹配
- **注意**：startActivity 会自动添加 DEFAULT Category

### 问题2：Binder 的优势是什么？为什么 Android 选择 Binder？

**答案要点：**
1. **性能高**：只需一次数据拷贝（通过 mmap）
2. **安全**：支持 UID/PID 身份验证，可以验证调用者身份
3. **易用**：C/S 架构，面向对象的调用方式
4. **稳定**：内核级实现，稳定可靠
- 对比：Socket 需要两次拷贝，共享内存需要复杂的同步机制

### 问题3：Binder 的一次拷贝是如何实现的？

**答案要点：**
1. Server 进程通过 `mmap()` 将内核缓冲区映射到用户空间
2. Client 发送数据时，通过 `copy_from_user()` 拷贝到内核缓冲区
3. Server 直接访问映射的内存区域，无需再次拷贝
4. 因此整个过程只有一次数据拷贝（Client → 内核）

### 问题4：AIDL 中 Stub 和 Proxy 的作用是什么？

**答案要点：**
- **Stub（存根）**：
  - 服务端的抽象类
  - 继承 Binder，实现 AIDL 接口
  - 在 `onTransact()` 中处理客户端请求
  - 服务端需要继承 Stub 实现具体方法
- **Proxy（代理）**：
  - 客户端的代理类
  - 实现 AIDL 接口
  - 在方法中序列化参数，调用 `transact()` 发起跨进程调用
  - 客户端通过 Proxy 调用远程方法

### 问题5：AIDL 中 in、out、inout 的区别？

**答案要点：**
- **in**（默认）：数据从客户端传到服务端，服务端修改不影响客户端
- **out**：数据从服务端传到客户端，客户端传入的值服务端收不到
- **inout**：双向传递，客户端和服务端都能修改
- **性能**：in < out < inout，尽量使用 in

### 问题6：Messenger 和 AIDL 的区别？

**答案要点：**
| 特性 | Messenger | AIDL |
|-----|-----------|------|
| 实现复杂度 | 简单 | 复杂 |
| 并发处理 | 串行（Handler） | 支持并发 |
| 数据类型 | Message/Bundle | 自定义 Parcelable |
| 回调支持 | 通过 replyTo | 支持回调接口 |
| 使用场景 | 简单消息传递 | 复杂跨进程通信 |

### 问题7：Bundle 传输数据有什么限制？

**答案要点：**
- Binder 事务缓冲区大小约 **1MB**（所有进程共享）
- 单次传输建议不超过 **500KB**
- 超过限制会抛出 `TransactionTooLargeException`
- **解决方案**：
  1. 写入文件，传递文件路径
  2. 使用 ContentProvider
  3. 使用数据库
  4. 使用 ViewModel（同进程）

### 问题8：如何实现跨进程回调？

**答案要点：**
1. **AIDL 回调接口**：
   - 定义回调 AIDL 接口
   - 服务端使用 `RemoteCallbackList` 管理回调
   - 客户端实现回调接口的 Stub
2. **Messenger**：
   - 客户端创建 Messenger
   - 通过 `Message.replyTo` 传递给服务端
   - 服务端通过 replyTo 发送回复
3. **注意**：回调在 Binder 线程执行，更新 UI 需要切换到主线程
