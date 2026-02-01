# JVM内存模型与GC

## 1. 概述

JVM（Java Virtual Machine）内存模型是 Java 程序运行的基础，理解 JVM 内存结构和垃圾回收机制对于编写高性能 Android 应用至关重要。本文将深入讲解 JVM 内存区域划分、对象创建过程、GC 算法与垃圾收集器，以及四种引用类型。

## 2. 核心原理

### 2.1 JVM 内存区域

JVM 运行时数据区主要分为以下几个部分：

```
┌─────────────────────────────────────────────────────────────┐
│                    JVM 运行时数据区                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    方法区 (Method Area)              │   │
│  │    - 类信息、常量、静态变量、JIT编译后的代码           │   │
│  │    - JDK8后改为元空间(Metaspace)，使用本地内存        │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                      堆 (Heap)                       │   │
│  │  ┌─────────────────┐  ┌─────────────────────────┐   │   │
│  │  │   新生代 Young   │  │       老年代 Old        │   │   │
│  │  │ ┌────┬────┬────┐│  │                         │   │   │
│  │  │ │Eden│ S0 │ S1 ││  │                         │   │   │
│  │  │ └────┴────┴────┘│  │                         │   │   │
│  │  └─────────────────┘  └─────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  线程私有区域（每个线程独立）                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │  虚拟机栈     │ │ 本地方法栈   │ │   程序计数器      │    │
│  │  (VM Stack)  │ │(Native Stack)│ │ (PC Register)    │    │
│  └──────────────┘ └──────────────┘ └──────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### 2.1.1 程序计数器 (Program Counter Register)

- **作用**：记录当前线程执行的字节码指令地址
- **特点**：
  - 线程私有，每个线程都有独立的程序计数器
  - 唯一不会发生 OutOfMemoryError 的区域
  - 如果执行 Native 方法，计数器值为空（Undefined）

#### 2.1.2 虚拟机栈 (VM Stack)

- **作用**：存储局部变量表、操作数栈、动态链接、方法出口等
- **特点**：
  - 线程私有，生命周期与线程相同
  - 每个方法执行时创建栈帧（Stack Frame）
  - 可能抛出 StackOverflowError（栈深度超限）或 OutOfMemoryError（无法申请足够内存）

```
┌─────────────────────────────────────┐
│            虚拟机栈                  │
├─────────────────────────────────────┤
│  ┌─────────────────────────────┐   │
│  │      栈帧 (Stack Frame)     │   │
│  │  ┌───────────────────────┐  │   │
│  │  │    局部变量表          │  │   │
│  │  │  (Local Variables)    │  │   │
│  │  └───────────────────────┘  │   │
│  │  ┌───────────────────────┐  │   │
│  │  │     操作数栈           │  │   │
│  │  │  (Operand Stack)      │  │   │
│  │  └───────────────────────┘  │   │
│  │  ┌───────────────────────┐  │   │
│  │  │     动态链接           │  │   │
│  │  │  (Dynamic Linking)    │  │   │
│  │  └───────────────────────┘  │   │
│  │  ┌───────────────────────┐  │   │
│  │  │     方法返回地址       │  │   │
│  │  │  (Return Address)     │  │   │
│  │  └───────────────────────┘  │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

#### 2.1.3 本地方法栈 (Native Method Stack)

- **作用**：为 Native 方法服务
- **特点**：与虚拟机栈类似，但服务于 Native 方法
- HotSpot 虚拟机将虚拟机栈和本地方法栈合二为一

#### 2.1.4 堆 (Heap)

- **作用**：存放对象实例和数组
- **特点**：
  - 所有线程共享
  - GC 管理的主要区域
  - 可通过 -Xms（初始大小）和 -Xmx（最大大小）设置

**堆内存分代结构**：

| 区域 | 占比 | 说明 |
|------|------|------|
| 新生代 (Young Generation) | 1/3 堆 | 新创建的对象首先分配在这里 |
| - Eden 区 | 8/10 新生代 | 对象首次分配的区域 |
| - Survivor 0 (From) | 1/10 新生代 | 存活对象复制区 |
| - Survivor 1 (To) | 1/10 新生代 | 存活对象复制区 |
| 老年代 (Old Generation) | 2/3 堆 | 长期存活的对象 |

#### 2.1.5 方法区 (Method Area)

- **作用**：存储类信息、常量、静态变量、即时编译器编译后的代码
- **演进**：
  - JDK 7 及之前：永久代（PermGen），使用堆内存
  - JDK 8 及之后：元空间（Metaspace），使用本地内存

```
JDK 7:
┌─────────────────────────────────────┐
│              堆内存                  │
│  ┌─────────────────────────────┐   │
│  │         永久代 PermGen       │   │
│  │  - 类元数据                  │   │
│  │  - 字符串常量池              │   │
│  │  - 静态变量                  │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘

JDK 8+:
┌─────────────────────────────────────┐
│              堆内存                  │
│  - 字符串常量池（移到堆中）          │
│  - 静态变量（移到堆中）              │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│           本地内存 (Native)          │
│  ┌─────────────────────────────┐   │
│  │       元空间 Metaspace       │   │
│  │  - 类元数据                  │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### 2.2 对象创建与内存分配

#### 2.2.1 对象创建过程

```
1. 类加载检查
   ↓
2. 分配内存
   ├── 指针碰撞 (Bump the Pointer) - 内存规整时
   └── 空闲列表 (Free List) - 内存不规整时
   ↓
3. 初始化零值
   ↓
4. 设置对象头
   ↓
5. 执行 <init> 方法
```

#### 2.2.2 对象内存布局

```
┌─────────────────────────────────────────────────────────┐
│                      对象内存布局                        │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────┐   │
│  │              对象头 (Object Header)              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │         Mark Word (标记字段)             │   │   │
│  │  │  - 哈希码、GC分代年龄、锁状态标志         │   │   │
│  │  │  - 线程持有的锁、偏向线程ID              │   │   │
│  │  │  - 偏向时间戳                           │   │   │
│  │  │  32位: 4字节 / 64位: 8字节              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │       类型指针 (Class Pointer)           │   │   │
│  │  │  - 指向类元数据的指针                    │   │   │
│  │  │  - 开启压缩指针: 4字节                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │       数组长度 (仅数组对象)              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │              实例数据 (Instance Data)           │   │
│  │  - 对象真正存储的有效信息                       │   │
│  │  - 各种类型的字段内容                          │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │              对齐填充 (Padding)                 │   │
│  │  - HotSpot要求对象大小必须是8字节的整数倍       │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

#### 2.2.3 内存分配策略

1. **对象优先在 Eden 区分配**
2. **大对象直接进入老年代**（-XX:PretenureSizeThreshold）
3. **长期存活的对象进入老年代**（默认年龄阈值 15，-XX:MaxTenuringThreshold）
4. **动态对象年龄判定**：Survivor 中相同年龄对象大小总和 > Survivor 空间一半，则年龄 >= 该年龄的对象直接进入老年代
5. **空间分配担保**：Minor GC 前检查老年代最大可用连续空间是否大于新生代所有对象总空间

### 2.3 GC 算法

#### 2.3.1 标记-清除算法 (Mark-Sweep)

```
标记阶段:                    清除阶段:
┌───┬───┬───┬───┬───┐      ┌───┬───┬───┬───┬───┐
│ A │ B │ C │ D │ E │  →   │   │ B │   │ D │   │
│ ✓ │   │ ✓ │   │ ✓ │      │   │   │   │   │   │
└───┴───┴───┴───┴───┘      └───┴───┴───┴───┴───┘
  ↑       ↑       ↑          ↑       ↑       ↑
 标记    标记    标记        清除    清除    清除
```

- **优点**：实现简单
- **缺点**：
  - 效率不高（标记和清除效率都不高）
  - 产生内存碎片

#### 2.3.2 复制算法 (Copying)

```
复制前:                      复制后:
┌─────────────┬─────────────┐
│  From 区    │   To 区     │
│ A B C D E F │             │
│ ✓   ✓   ✓   │             │
└─────────────┴─────────────┘
        ↓
┌─────────────┬─────────────┐
│  From 区    │   To 区     │
│             │ A C E       │
│             │ (紧凑排列)   │
└─────────────┴─────────────┘
```

- **优点**：
  - 效率高
  - 无内存碎片
- **缺点**：内存利用率只有 50%
- **应用**：新生代（Eden + Survivor）

#### 2.3.3 标记-整理算法 (Mark-Compact)

```
标记阶段:                    整理阶段:
┌───┬───┬───┬───┬───┐      ┌───┬───┬───┬───┬───┐
│ A │   │ C │   │ E │  →   │ A │ C │ E │   │   │
│ ✓ │   │ ✓ │   │ ✓ │      │   │   │   │   │   │
└───┴───┴───┴───┴───┘      └───┴───┴───┴───┴───┘
```

- **优点**：无内存碎片，内存利用率高
- **缺点**：移动对象需要更新引用，效率较低
- **应用**：老年代

#### 2.3.4 分代收集算法 (Generational Collection)

```
┌─────────────────────────────────────────────────────────┐
│                       堆内存                            │
├─────────────────────────────────────────────────────────┤
│  新生代 (Young Generation)                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │  使用复制算法                                    │   │
│  │  - 对象存活率低，复制成本小                      │   │
│  │  - Minor GC / Young GC                          │   │
│  └─────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│  老年代 (Old Generation)                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │  使用标记-清除或标记-整理算法                    │   │
│  │  - 对象存活率高，没有额外空间担保                │   │
│  │  - Major GC / Full GC                           │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 2.4 垃圾收集器

#### 2.4.1 垃圾收集器概览

```
┌─────────────────────────────────────────────────────────────┐
│                      新生代收集器                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │  Serial  │  │ ParNew   │  │ Parallel │                  │
│  │          │  │          │  │ Scavenge │                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
│       │             │             │                         │
│       │    可配合   │    可配合   │                         │
│       ↓             ↓             ↓                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │Serial Old│  │   CMS    │  │Parallel  │                  │
│  │          │  │          │  │   Old    │                  │
│  └──────────┘  └──────────┘  └──────────┘                  │
│                      老年代收集器                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    整堆收集器                                │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │        G1        │  │       ZGC        │                │
│  │   (JDK 9 默认)   │  │   (JDK 15 正式)  │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

#### 2.4.2 Serial 收集器

- **特点**：单线程，Stop-The-World
- **算法**：新生代复制算法，老年代标记-整理
- **适用**：Client 模式，单核 CPU

```
应用线程: ═══════════╗          ╔═══════════
                     ║   STW    ║
GC 线程:             ╠══════════╣
                     ║  Serial  ║
                     ╚══════════╝
```

#### 2.4.3 Parallel Scavenge 收集器

- **特点**：多线程并行，关注吞吐量
- **吞吐量** = 运行用户代码时间 / (运行用户代码时间 + GC时间)
- **参数**：
  - -XX:MaxGCPauseMillis：最大 GC 停顿时间
  - -XX:GCTimeRatio：吞吐量大小

```
应用线程: ═══════════╗              ╔═══════════
                     ║     STW      ║
GC 线程:             ╠══════════════╣
                     ║ Thread 1     ║
                     ║ Thread 2     ║
                     ║ Thread 3     ║
                     ║ Thread 4     ║
                     ╚══════════════╝
```

#### 2.4.4 CMS 收集器 (Concurrent Mark Sweep)

- **特点**：并发收集，低停顿
- **算法**：标记-清除

```
初始标记    并发标记      重新标记    并发清除
(STW)      (并发)        (STW)      (并发)
  ↓          ↓            ↓          ↓
┌───┐    ┌───────┐    ┌─────┐    ┌───────┐
│   │    │       │    │     │    │       │
└───┘    └───────┘    └─────┘    └───────┘
  ↑          ↑            ↑          ↑
 短暂      与用户        短暂      与用户
 停顿      线程并发      停顿      线程并发
```

**四个阶段**：
1. **初始标记**：标记 GC Roots 直接关联的对象，速度快，STW
2. **并发标记**：从 GC Roots 遍历整个对象图，耗时长，与用户线程并发
3. **重新标记**：修正并发标记期间变动的对象，STW
4. **并发清除**：清除标记的对象，与用户线程并发

**缺点**：
- CPU 敏感（并发阶段占用 CPU）
- 无法处理浮动垃圾
- 产生内存碎片

#### 2.4.5 G1 收集器 (Garbage First)

- **特点**：面向服务端，可预测停顿时间
- **算法**：整体标记-整理，局部复制

```
┌─────────────────────────────────────────────────────────┐
│                    G1 堆内存布局                         │
├─────────────────────────────────────────────────────────┤
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐    │
│  │ E │ S │ O │ E │ H │ O │ E │ S │ O │ E │ O │ H │    │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘    │
│                                                         │
│  E = Eden    S = Survivor    O = Old    H = Humongous  │
│                                                         │
│  - 将堆划分为多个大小相等的 Region (1MB~32MB)           │
│  - 每个 Region 可以是 Eden/Survivor/Old/Humongous      │
│  - Humongous 用于存储大对象 (>Region 50%)              │
└─────────────────────────────────────────────────────────┘
```

**G1 收集过程**：
1. **Young GC**：收集所有 Eden 和 Survivor Region
2. **并发标记周期**：
   - 初始标记（STW）
   - 根区域扫描
   - 并发标记
   - 重新标记（STW）
   - 清理（STW）
3. **Mixed GC**：收集所有 Young Region + 部分 Old Region

#### 2.4.6 ZGC 收集器

- **特点**：超低延迟（< 10ms），支持 TB 级堆
- **算法**：基于 Region，使用读屏障、染色指针、内存多重映射

```
┌─────────────────────────────────────────────────────────┐
│                    ZGC 特性                             │
├─────────────────────────────────────────────────────────┤
│  - 停顿时间不超过 10ms                                  │
│  - 停顿时间不会随堆大小增加而增加                       │
│  - 支持 8MB ~ 16TB 堆大小                              │
│  - JDK 15 正式发布                                     │
└─────────────────────────────────────────────────────────┘

染色指针 (Colored Pointer):
┌─────────────────────────────────────────────────────────┐
│  64位指针                                               │
│  ┌────────┬────┬────┬────┬────┬──────────────────────┐ │
│  │ 未使用  │ F  │ R  │ M1 │ M0 │     对象地址 (44位)  │ │
│  │ (16位) │(1) │(1) │(1) │(1) │                      │ │
│  └────────┴────┴────┴────┴────┴──────────────────────┘ │
│  F=Finalizable  R=Remapped  M0/M1=Marked              │
└─────────────────────────────────────────────────────────┘
```

### 2.5 四种引用类型

```
┌─────────────────────────────────────────────────────────────┐
│                    Java 四种引用类型                         │
├─────────────────────────────────────────────────────────────┤
│  引用类型    │  回收时机           │  用途                   │
├─────────────────────────────────────────────────────────────┤
│  强引用      │  永不回收           │  普通对象引用           │
│  (Strong)   │  (除非置null)       │  Object obj = new X()  │
├─────────────────────────────────────────────────────────────┤
│  软引用      │  内存不足时回收      │  缓存                   │
│  (Soft)     │                     │  SoftReference<T>      │
├─────────────────────────────────────────────────────────────┤
│  弱引用      │  下次GC时回收       │  WeakHashMap           │
│  (Weak)     │                     │  WeakReference<T>      │
├─────────────────────────────────────────────────────────────┤
│  虚引用      │  随时可能被回收      │  跟踪对象被回收的状态    │
│  (Phantom)  │  必须配合引用队列    │  PhantomReference<T>   │
└─────────────────────────────────────────────────────────────┘
```

## 3. 关键源码解析

### 3.1 软引用使用示例

```java
/**
 * 软引用示例 - 实现图片缓存
 * 当内存不足时，GC 会回收软引用指向的对象
 */
public class ImageCache {
    // 使用软引用缓存图片，内存不足时自动释放
    private Map<String, SoftReference<Bitmap>> cache = new HashMap<>();
    
    public void put(String key, Bitmap bitmap) {
        // 将 Bitmap 包装成软引用存入缓存
        cache.put(key, new SoftReference<>(bitmap));
    }
    
    public Bitmap get(String key) {
        SoftReference<Bitmap> ref = cache.get(key);
        if (ref != null) {
            // get() 可能返回 null，说明对象已被 GC 回收
            return ref.get();
        }
        return null;
    }
}
```

### 3.2 弱引用与 WeakHashMap

```java
/**
 * WeakHashMap 源码分析 (JDK 8)
 * Key 使用弱引用，当 Key 没有强引用时，Entry 会被自动清除
 */
public class WeakHashMap<K,V> extends AbstractMap<K,V> {
    
    /**
     * Entry 继承 WeakReference，Key 作为弱引用的 referent
     */
    private static class Entry<K,V> extends WeakReference<Object> 
            implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;
        
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            // 调用 WeakReference 构造函数，key 作为弱引用对象
            // queue 用于接收被 GC 回收的引用通知
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
    }
    
    // 引用队列，当弱引用对象被 GC 回收时，引用会被加入此队列
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
    
    /**
     * 清除已被 GC 回收的 Entry
     * 在 get/put/size 等操作前调用
     */
    private void expungeStaleEntries() {
        // 从引用队列中取出已被回收的引用
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);
                // 从链表中移除该 Entry
                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        e.value = null; // 帮助 GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
}
```

### 3.3 虚引用使用示例

```java
/**
 * 虚引用示例 - 跟踪对象回收状态
 * 虚引用必须配合引用队列使用
 */
public class PhantomReferenceDemo {
    
    public static void main(String[] args) {
        // 创建引用队列
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        
        // 创建对象
        Object obj = new Object();
        
        // 创建虚引用，关联引用队列
        PhantomReference<Object> phantomRef = new PhantomReference<>(obj, queue);
        
        // 虚引用的 get() 方法始终返回 null
        System.out.println(phantomRef.get()); // 输出: null
        
        // 移除强引用
        obj = null;
        
        // 触发 GC
        System.gc();
        
        // 等待一段时间让 GC 完成
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        // 从引用队列中获取被回收的引用
        Reference<?> ref = queue.poll();
        if (ref != null) {
            System.out.println("对象已被回收，可以进行资源清理");
            // 这里可以进行一些清理工作，如释放直接内存
        }
    }
}
```

### 3.4 对象内存分配源码 (HotSpot)

```cpp
/**
 * HotSpot 对象分配核心代码 (简化版)
 * 源码位置: src/hotspot/share/gc/shared/collectedHeap.cpp
 */
HeapWord* CollectedHeap::obj_allocate(Klass* klass, int size, TRAPS) {
    // 1. 尝试在 TLAB (Thread Local Allocation Buffer) 中分配
    HeapWord* result = allocate_from_tlab(klass, THREAD, size);
    if (result != NULL) {
        return result;
    }
    
    // 2. TLAB 分配失败，尝试在 Eden 区分配
    result = allocate_outside_tlab(klass, size, THREAD);
    return result;
}

/**
 * TLAB 分配 - 无锁快速分配
 * 每个线程有自己的 TLAB，避免多线程竞争
 */
HeapWord* CollectedHeap::allocate_from_tlab(Klass* klass, Thread* thread, size_t size) {
    // 获取当前线程的 TLAB
    ThreadLocalAllocBuffer& tlab = thread->tlab();
    
    // 指针碰撞分配
    HeapWord* obj = tlab.allocate(size);
    return obj;
}
```

## 4. 实战应用

### 4.1 Android 中的内存优化

#### 4.1.1 使用软引用实现图片缓存

```java
/**
 * Android 图片缓存最佳实践
 * 结合 LruCache 和软引用实现二级缓存
 */
public class ImageMemoryCache {
    // 一级缓存：LruCache 强引用缓存
    private LruCache<String, Bitmap> lruCache;
    
    // 二级缓存：软引用缓存，存放从 LruCache 移除的图片
    private Map<String, SoftReference<Bitmap>> softCache;
    
    public ImageMemoryCache() {
        // 获取应用可用最大内存的 1/8 作为缓存大小
        int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
        int cacheSize = maxMemory / 8;
        
        softCache = new ConcurrentHashMap<>();
        
        lruCache = new LruCache<String, Bitmap>(cacheSize) {
            @Override
            protected int sizeOf(String key, Bitmap bitmap) {
                // 返回图片大小，单位 KB
                return bitmap.getByteCount() / 1024;
            }
            
            @Override
            protected void entryRemoved(boolean evicted, String key,
                    Bitmap oldValue, Bitmap newValue) {
                if (evicted && oldValue != null) {
                    // 被移除的图片放入软引用缓存
                    softCache.put(key, new SoftReference<>(oldValue));
                }
            }
        };
    }
    
    public Bitmap get(String key) {
        // 先从 LruCache 获取
        Bitmap bitmap = lruCache.get(key);
        if (bitmap != null) {
            return bitmap;
        }
        
        // 再从软引用缓存获取
        SoftReference<Bitmap> ref = softCache.get(key);
        if (ref != null) {
            bitmap = ref.get();
            if (bitmap != null) {
                // 重新放入 LruCache
                lruCache.put(key, bitmap);
                softCache.remove(key);
                return bitmap;
            } else {
                // 已被 GC 回收，移除无效引用
                softCache.remove(key);
            }
        }
        return null;
    }
}
```

#### 4.1.2 避免内存泄漏

```java
/**
 * 常见内存泄漏场景及解决方案
 */

// ❌ 错误示例：非静态内部类持有外部类引用
public class LeakActivity extends Activity {
    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // 非静态内部类隐式持有 Activity 引用
            // 如果 Handler 有延迟消息，Activity 无法被回收
        }
    };
}

// ✅ 正确示例：使用静态内部类 + 弱引用
public class SafeActivity extends Activity {
    private SafeHandler handler = new SafeHandler(this);
    
    private static class SafeHandler extends Handler {
        // 使用弱引用持有 Activity
        private WeakReference<Activity> activityRef;
        
        SafeHandler(Activity activity) {
            activityRef = new WeakReference<>(activity);
        }
        
        @Override
        public void handleMessage(Message msg) {
            Activity activity = activityRef.get();
            if (activity != null && !activity.isFinishing()) {
                // 安全地使用 Activity
            }
        }
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 移除所有回调和消息
        handler.removeCallbacksAndMessages(null);
    }
}
```

### 4.2 GC 调优实践

#### 4.2.1 常用 JVM 参数

```bash
# 堆内存设置
-Xms512m          # 初始堆大小
-Xmx1024m         # 最大堆大小
-Xmn256m          # 新生代大小

# 元空间设置 (JDK 8+)
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=256m

# GC 收集器选择
-XX:+UseSerialGC           # Serial 收集器
-XX:+UseParallelGC         # Parallel 收集器
-XX:+UseConcMarkSweepGC    # CMS 收集器
-XX:+UseG1GC               # G1 收集器 (JDK 9 默认)
-XX:+UseZGC                # ZGC 收集器 (JDK 15+)

# GC 日志
-Xlog:gc*:file=gc.log      # JDK 9+ 统一日志
-XX:+PrintGCDetails        # JDK 8 打印 GC 详情
-XX:+PrintGCDateStamps     # JDK 8 打印 GC 时间戳

# G1 收集器调优
-XX:MaxGCPauseMillis=200   # 目标最大停顿时间
-XX:G1HeapRegionSize=16m   # Region 大小
```

#### 4.2.2 Android ART 虚拟机 GC

```
Android ART GC 类型:
┌─────────────────────────────────────────────────────────────┐
│  GC 类型              │  触发条件           │  特点          │
├─────────────────────────────────────────────────────────────┤
│  Concurrent GC        │  堆内存达到阈值      │  并发执行      │
│  Alloc GC            │  分配失败时          │  阻塞分配线程  │
│  Explicit GC         │  System.gc()        │  可被忽略      │
│  NativeAlloc GC      │  Native 内存压力     │  回收 Native   │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 常见坑点

1. **软引用不是万能的**：在内存充足时，软引用对象不会被回收，可能导致内存占用过高
2. **WeakHashMap 的 Key 必须没有强引用**：如果 Key 有其他强引用，Entry 不会被清除
3. **虚引用的 get() 始终返回 null**：不能通过虚引用获取对象
4. **System.gc() 只是建议**：JVM 可能忽略这个请求
5. **大对象直接进入老年代**：频繁创建大对象会导致频繁 Full GC

## 5. 常见面试题

### 问题1：JVM 内存区域有哪些？哪些是线程私有的？

**答案要点**：
- **线程共享**：堆、方法区（元空间）
- **线程私有**：虚拟机栈、本地方法栈、程序计数器
- 堆是 GC 的主要区域，存放对象实例
- 方法区存放类信息、常量、静态变量
- 虚拟机栈存放栈帧（局部变量表、操作数栈等）
- 程序计数器记录当前执行的字节码指令地址

### 问题2：对象在内存中的布局是怎样的？

**答案要点**：
- **对象头**：Mark Word（哈希码、GC 年龄、锁状态）+ 类型指针 + 数组长度（仅数组）
- **实例数据**：对象的字段内容
- **对齐填充**：保证对象大小是 8 字节的整数倍
- Mark Word 在不同锁状态下存储内容不同（无锁、偏向锁、轻量级锁、重量级锁）

### 问题3：GC 算法有哪些？各自的优缺点？

**答案要点**：
- **标记-清除**：简单但产生碎片，效率不高
- **复制算法**：无碎片、效率高，但内存利用率只有 50%，适合新生代
- **标记-整理**：无碎片、内存利用率高，但移动对象效率较低，适合老年代
- **分代收集**：根据对象存活周期选择不同算法，新生代用复制，老年代用标记-整理

### 问题4：G1 和 CMS 的区别？

**答案要点**：
| 特性 | CMS | G1 |
|------|-----|-----|
| 算法 | 标记-清除 | 标记-整理 + 复制 |
| 内存布局 | 传统分代 | Region 分区 |
| 碎片 | 会产生碎片 | 无碎片 |
| 停顿预测 | 不可预测 | 可设置目标停顿时间 |
| 适用场景 | 中小堆 | 大堆（6GB+） |
| 状态 | JDK 14 移除 | JDK 9 默认 |

### 问题5：四种引用类型的区别和使用场景？

**答案要点**：
- **强引用**：普通引用，不会被 GC 回收，除非置为 null
- **软引用**：内存不足时回收，适合做缓存（如图片缓存）
- **弱引用**：下次 GC 时回收，适合做临时缓存（如 WeakHashMap）
- **虚引用**：随时可能被回收，必须配合引用队列，用于跟踪对象回收状态

### 问题6：什么情况下会发生 Full GC？

**答案要点**：
1. 老年代空间不足
2. 方法区（元空间）空间不足
3. 调用 System.gc()（建议，不保证执行）
4. Minor GC 后存活对象大于老年代剩余空间（空间分配担保失败）
5. CMS GC 时出现 promotion failed 或 concurrent mode failure

### 问题7：如何判断对象是否可以被回收？

**答案要点**：
- **引用计数法**：对象被引用计数+1，引用失效计数-1，为0时可回收。缺点：无法解决循环引用
- **可达性分析**：从 GC Roots 出发，不可达的对象可回收
- **GC Roots 包括**：
  - 虚拟机栈中引用的对象
  - 方法区中静态属性引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中 JNI 引用的对象
  - 同步锁持有的对象

### 问题8：Android 中如何避免内存泄漏？

**答案要点**：
1. 使用静态内部类 + 弱引用代替非静态内部类
2. 及时注销监听器、广播接收器
3. Handler 使用静态内部类 + 弱引用，onDestroy 时移除消息
4. 避免 Context 泄漏，使用 ApplicationContext
5. 使用 LeakCanary 检测内存泄漏
6. 及时关闭 Cursor、Stream 等资源
7. 避免在集合中添加对象后不移除
