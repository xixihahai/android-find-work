# RecyclerView 深度解析

## 速记总结

### 一句话理解
> RecyclerView 就像一个**智能传送带工厂**：传送带（LayoutManager）决定产品怎么摆，工人（Adapter）负责组装产品，仓库（Recycler四级缓存）负责回收和复用零件，质检员（DiffUtil）只更新有变化的产品。

### 核心知识点速记
| 知识点 | 一句话记忆 | 面试频率 |
|--------|-----------|---------|
| 设计哲学 | 职责分离 + 组合模式，一个组件只做一件事 | ★★★★★ |
| 四级缓存 | Scrap(暂存)→Cache(精确复用)→Extension(自定义)→Pool(类型复用) | ★★★★★ |
| Cache vs Pool | Cache 按 position 匹配不需要 bind；Pool 按 viewType 匹配需要 bind | ★★★★★ |
| DiffUtil 三层比较 | areItemsTheSame(是不是同一个)→areContentsTheSame(内容变没变)→getChangePayload(哪里变了) | ★★★★☆ |
| 预取机制 | GapWorker 利用帧间空闲时间提前 create+bind 下一个 item | ★★★☆☆ |
| 三步布局 | 预布局(记录动画前状态)→实际布局→触发动画 | ★★★★☆ |
| 嵌套优化 | 共享 Pool + initialPrefetchItemCount + 保存滚动位置 | ★★★★☆ |

### 与其他知识的串联
```
RecyclerView 在 Android 知识体系中的位置
┌─────────────────────────────────────────────────────────┐
│                    RecyclerView                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── 向上关联 ───────────────────────────────────┐     │
│  │ View绘制流程: measure→layout→draw 是RV布局的基础│     │
│  │ 事件分发: RV的滚动处理基于 onTouchEvent         │     │
│  │ 嵌套滑动: NestedScrollingChild 接口实现         │     │
│  └────────────────────────────────────────────────┘     │
│                                                         │
│  ┌─── 向下关联 ───────────────────────────────────┐     │
│  │ 性能优化: 启动优化(预加载)、渲染优化(减少层级)   │     │
│  │ 内存管理: ViewHolder泄漏、Bitmap回收             │     │
│  │ Handler: 预取机制通过 Choreographer 调度          │     │
│  └────────────────────────────────────────────────┘     │
│                                                         │
│  ┌─── 横向关联 ───────────────────────────────────┐     │
│  │ Jetpack: ListAdapter 基于 DiffUtil              │     │
│  │ Glide: 图片加载与RV滚动状态联动                  │     │
│  │ Compose: LazyColumn 是 Compose 版 RecyclerView   │     │
│  └────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### 面试答题公式
> **RecyclerView 题万能公式**：先说设计思想(职责分离) → 再说核心机制(四级缓存/DiffUtil) → 最后说实战优化(Trace定位+针对性优化)

---

> 本文从设计哲学出发，串联 RecyclerView 的核心架构、缓存机制、布局流程、数据更新与性能优化，帮助你建立完整的知识体系。每个知识点都回答三个问题：**是什么、为什么这么设计、实战中怎么用**。

---

## 一、设计哲学：RecyclerView 到底在解决什么问题？

### 1.1 从 ListView 的痛点说起

要理解 RecyclerView，必须先理解 ListView 的问题。ListView 是一个"大而全"的控件——它同时负责**布局、滚动、回收、动画、分割线**等所有事情。这导致了两个根本性问题：

**问题一：扩展性差。** 想做网格布局？换 GridView。想做瀑布流？没有官方方案。想做横向列表？自己写。每换一种布局方式就需要一个新控件，而底层的回收逻辑、滚动逻辑又要重复实现一遍。

**问题二：性能优化手段有限。** ListView 的两级缓存（ActiveViews + ScrapViews）粒度太粗。`notifyDataSetChanged()` 一调用，所有 item 全量刷新，没有局部更新能力。ViewHolder 模式是"建议"而非强制，导致很多开发者根本不用。

### 1.2 RecyclerView 的核心设计思想：职责分离

RecyclerView 用**组合模式**彻底拆解了 ListView 的"上帝类"问题。它的核心哲学是：

> **每个组件只做一件事，通过组合实现复杂功能。**

```
                    ┌─────────────────────────┐
                    │      RecyclerView       │
                    │   (容器 + 滚动协调者)     │
                    └────────────┬────────────┘
                                 │
          ┌──────────┬───────────┼───────────┬──────────┐
          ▼          ▼           ▼           ▼          ▼
   LayoutManager   Adapter    Recycler   ItemAnimator  ItemDecoration
   "在哪里放"     "放什么"   "怎么回收"   "怎么动"      "怎么装饰"
```

| 组件 | 单一职责 | 为什么要分离 |
|------|----------|-------------|
| **LayoutManager** | 决定 item 的测量和摆放位置 | 换布局只需换 LayoutManager，不影响数据和回收逻辑 |
| **Adapter** | 将数据绑定到 ViewHolder | 数据层和展示层解耦，同一套数据可以用不同布局展示 |
| **Recycler** | 管理 ViewHolder 的缓存和回收 | 缓存策略独立于布局策略，可以针对性优化 |
| **ItemAnimator** | 处理 item 增删改的动画 | 动画逻辑不侵入布局和数据逻辑 |
| **ItemDecoration** | 绘制分割线、边距等装饰 | 装饰效果可叠加，不修改 item 本身 |

**这个设计的直接好处：**
- 想做瀑布流？换成 `StaggeredGridLayoutManager`，其他代码一行不改
- 想共享缓存？多个 RecyclerView 共用一个 `RecycledViewPool`
- 想自定义动画？继承 `ItemAnimator`，不影响布局和数据
- 想加分割线？`addItemDecoration()`，可叠加多个

### 1.3 强制 ViewHolder 模式：把"建议"变成"约束"

ListView 时代，ViewHolder 是一个最佳实践，但框架不强制。结果大量代码在 `getView()` 里反复 `findViewById()`。

RecyclerView 直接把 ViewHolder 变成了架构的一部分：

```java
// Adapter 的泛型直接绑定 ViewHolder 类型，你不用都不行
public abstract static class Adapter<VH extends ViewHolder> {
    public abstract VH onCreateViewHolder(ViewGroup parent, int viewType);
    public abstract void onBindViewHolder(VH holder, int position);
}
```

更关键的是，ViewHolder 不仅仅是"缓存 View 引用"。它同时承担了**缓存元数据**的角色：

```java
public abstract static class ViewHolder {
    public final View itemView;       // 对应的 View
    int mPosition;                    // 当前位置
    int mItemViewType;                // View 类型
    int mFlags;                       // 状态标记（是否需要更新、是否无效等）
    long mItemId;                     // 稳定 ID（用于动画和缓存匹配）
    // ...
}
```

这些元数据是整个缓存机制的基础——Recycler 通过 position、viewType、flags 来决定一个 ViewHolder 能否复用、是否需要重新绑定。

---

## 二、四级缓存机制：RecyclerView 性能的核心引擎

### 2.1 先建立直觉：缓存到底在做什么？

RecyclerView 面对的核心问题是：**屏幕只能显示有限个 item，但数据可能有成千上万条。** 如果每次滑动都要创建新 View、绑定新数据，性能一定很差。

所以缓存机制的本质是：**用空间换时间，尽量减少 onCreateViewHolder 和 onBindViewHolder 的调用次数。**

四级缓存的设计遵循一个递进逻辑：

```
离屏幕越近 → 数据越"新鲜" → 复用成本越低
离屏幕越远 → 数据越"过时" → 复用成本越高（但比重新创建便宜）
```

### 2.2 四级缓存全景图

```
┌──────────────────────────────────────────────────────────────────────┐
│                     ViewHolder 获取的完整链路                         │
│                                                                      │
│  需要一个 position=5 的 ViewHolder                                   │
│       │                                                              │
│       ▼                                                              │
│  ① Scrap (mAttachedScrap / mChangedScrap)                           │
│     问："屏幕上有没有 position=5 的 ViewHolder？"                      │
│     命中 → 直接用，无需 create 也无需 bind                             │
│     场景：layout 过程中的临时存储                                      │
│       │ 没有                                                         │
│       ▼                                                              │
│  ② Cache (mCachedViews，默认 2 个)                                   │
│     问："刚滑出屏幕的有没有 position=5 的？"                           │
│     命中 → 直接用，无需 bind（position + viewType 双匹配）             │
│     场景：用户来回小幅滑动                                             │
│       │ 没有                                                         │
│       ▼                                                              │
│  ③ ViewCacheExtension (开发者自定义，通常不用)                        │
│     问："开发者有没有自己缓存这个 position 的 View？"                  │
│       │ 没有                                                         │
│       ▼                                                              │
│  ④ RecycledViewPool (按 viewType 分桶，每桶默认 5 个)                 │
│     问："有没有同类型（viewType 相同）的空壳 ViewHolder？"             │
│     命中 → 需要重新 bind（因为数据已经不对了）                         │
│     场景：正常滚动时的主要复用来源                                     │
│       │ 没有                                                         │
│       ▼                                                              │
│  ⑤ onCreateViewHolder() → inflate 新布局 → onBindViewHolder()        │
│     最昂贵的路径：创建 + 绑定                                         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.3 逐级深入：每级缓存的设计考量

#### 第一级：Scrap —— "layout 过程中的暂存区"

**是什么：** RecyclerView 每次重新 layout 时，会先把屏幕上所有的 ViewHolder "暂存"到 Scrap 列表中，layout 完成后再从中取回来。

**为什么需要：** 这是一个经常被误解的缓存。Scrap 不是为了"滚动复用"，而是为了**layout 过程的正确性**。想象这个场景：你调用了 `notifyItemChanged(3)`，RecyclerView 需要重新 layout。它不能一边遍历子 View 一边修改子 View 列表（ConcurrentModification），所以它先把所有子 View "detach" 到 Scrap 中，然后重新 layout 时再从 Scrap 中取回来。

**两个列表的区别：**
- `mAttachedScrap`：数据没变的 ViewHolder → 取回来直接用
- `mChangedScrap`：数据变了的 ViewHolder（如 `notifyItemChanged`）→ 取回来需要重新 bind

```java
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null; // 懒初始化，大部分情况用不到

    // layout 开始时，把屏幕上的 ViewHolder 都暂存起来
    void scrapView(View view) {
        final ViewHolder holder = getChildViewHolderInt(view);
        if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
                || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
            // 数据没变 → 放入 mAttachedScrap
            holder.setScrapContainer(this, false);
            mAttachedScrap.add(holder);
        } else {
            // 数据变了 → 放入 mChangedScrap
            if (mChangedScrap == null) {
                mChangedScrap = new ArrayList<>();
            }
            holder.setScrapContainer(this, true);
            mChangedScrap.add(holder);
        }
    }
}
```

**面试关键点：** Scrap 的生命周期仅限于一次 layout 过程。layout 完成后，Scrap 列表会被清空。它本质上不是"缓存"，而是"暂存区"。

#### 第二级：Cache (mCachedViews) —— "最近离开屏幕的 VIP 席位"

**是什么：** 一个固定大小（默认 2）的列表，存储刚刚滑出屏幕的 ViewHolder。

**为什么默认只有 2 个？** 因为用户最常见的操作是"往下滑一点又滑回来"。2 个刚好覆盖上下各 1 个 item 的场景，内存开销可控。

**核心特点：按 position + viewType 精确匹配。** 这意味着如果一个 ViewHolder 被缓存在 position=5，只有当你再次需要 position=5 的 ViewHolder 时才能命中。命中后 **不需要重新 bind**，因为数据完全一致。

```java
// 从 CachedViews 查找
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        // 精确匹配 position，并且 ViewHolder 没有被标记为无效
        if (!holder.wasReturnedFromScrap()
                && holder.getLayoutPosition() == position
                && !holder.isInvalid()) {
            return holder;
        }
    }
    return null;
}
```

**Cache 满了怎么办？** 最早进入的 ViewHolder 会被"降级"到 RecycledViewPool：

```java
void recycleCachedViewAt(int cachedViewIndex) {
    ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);
    // 降级到 RecycledViewPool
    addViewHolderToRecycledViewPool(viewHolder, true);
    mCachedViews.remove(cachedViewIndex);
}
```

**实战价值：** 如果你的列表用户经常来回滑动（如聊天列表），可以通过 `setItemViewCacheSize(n)` 增大 Cache，让更多刚滑出的 item 免于重新 bind。

#### 第三级：ViewCacheExtension —— "开发者的后门"

这是一个很少使用的扩展点。设计意图是让开发者在 RecycledViewPool 之前插入自己的缓存逻辑。

```java
public abstract static class ViewCacheExtension {
    @Nullable
    public abstract View getViewForPositionAndType(
            @NonNull Recycler recycler, int position, int type);
}
```

**实际使用场景极少，** 因为大多数需求通过调整 Cache 大小和 Pool 大小就能解决。可能的场景包括：
- 广告位缓存（广告 View 创建成本极高，不希望被回收）
- 固定位置的 Header 缓存

#### 第四级：RecycledViewPool —— "类型化的对象池"

**是什么：** 按 viewType 分桶存储的 ViewHolder 池。每个 viewType 默认最多存 5 个。

**核心设计思想：到了这一级，position 信息已经丢失了。** Pool 只关心 viewType——它提供一个"空壳" ViewHolder，你需要重新绑定数据。

```java
public static class RecycledViewPool {
    private static final int DEFAULT_MAX_SCRAP = 5;

    static class ScrapData {
        final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
        long mCreateRunningAverageNs = 0;  // 创建耗时的滑动平均值
        long mBindRunningAverageNs = 0;    // 绑定耗时的滑动平均值
    }

    // 按 viewType 分桶
    SparseArray<ScrapData> mScrap = new SparseArray<>();

    @Nullable
    public ViewHolder getRecycledView(int viewType) {
        final ScrapData scrapData = mScrap.get(viewType);
        if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
            final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
            // LIFO：从末尾取出（热数据优先）
            for (int i = scrapHeap.size() - 1; i >= 0; i--) {
                if (!scrapHeap.get(i).isAttachedToTransitionOverlay()) {
                    return scrapHeap.remove(i);
                }
            }
        }
        return null;
    }

    public void putRecycledView(ViewHolder scrap) {
        final int viewType = scrap.getItemViewType();
        final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
        if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
            return; // 满了就丢弃，不会无限膨胀
        }
        scrap.resetInternal(); // 清除所有状态
        scrapHeap.add(scrap);
    }
}
```

**两个容易忽略的设计细节：**

1. **`mCreateRunningAverageNs` 和 `mBindRunningAverageNs`**：Pool 会统计每种 viewType 的创建和绑定耗时。这个数据被 GapWorker（预取机制）用来判断"在下一帧到来之前，我还有没有时间预取一个 ViewHolder"。

2. **跨 RecyclerView 共享**：这是 Pool 独有的能力。在 ViewPager + 多个 RecyclerView 的场景下，共享 Pool 可以显著减少 ViewHolder 的创建次数。

### 2.4 串联理解：一次滑动的完整生命旅程

假设用户向上滑动，item-0 滑出屏幕顶部，item-10 需要从底部出现：

```
时刻 1：item-0 滑出屏幕
    → item-0 的 ViewHolder 被放入 mCachedViews (现在 Cache 里有 item-0)
    → 如果 Cache 已满（比如里面有 item-(-1) 和 item-(-2)）
      → item-(-2) 被"降级"到 RecycledViewPool
      → item-0 进入 Cache

时刻 2：LayoutManager 需要 item-10 的 ViewHolder
    → 查 Scrap？没有（不在 layout 过程中）
    → 查 Cache？没有（Cache 里是 item-0、item-(-1)，position 不匹配）
    → 查 ViewCacheExtension？没设置
    → 查 Pool？找同 viewType 的空壳 ViewHolder
      → 找到了！调用 onBindViewHolder(holder, 10) 绑定新数据
      → 或没找到 → onCreateViewHolder() + onBindViewHolder()

时刻 3：用户又往下滑回去，需要 item-0
    → 查 Cache？命中！item-0 还在 Cache 里
    → 直接用，不需要 onBindViewHolder()，数据和位置完全匹配
    → 用户体验：瞬间显示，没有任何闪烁
```

**这就是为什么来回滑动时 RecyclerView 如此流畅的原因。**

### 2.5 缓存机制的核心源码：tryGetViewHolderForPositionByDeadline

这是整个缓存机制的入口方法，理解它就理解了缓存的全部：

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {

    ViewHolder holder = null;
    boolean fromScrapOrHiddenOrCache = false;

    // === 步骤 1：预布局阶段 → 从 mChangedScrap 获取 ===
    // 预布局是为了计算动画的起始位置
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }

    // === 步骤 2：从 Scrap 和 Cache 获取（按 position 匹配）===
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            // 验证这个 ViewHolder 是否还能用（position 对不对、是否被标记为无效）
            if (!validateViewHolderForOffsetPosition(holder)) {
                // 不能用 → 回收到 Pool，继续找
                if (!dryRun) {
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }

    if (holder == null) {
        final int type = mAdapter.getItemViewType(offsetPosition);

        // === 步骤 3：通过 stableId 再找一次 Scrap 和 Cache ===
        // stableId 是"语义匹配"：即使 position 变了，只要 id 一样就能复用
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(
                    mAdapter.getItemId(offsetPosition), type, dryRun);
            if (holder != null) {
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }

        // === 步骤 4：ViewCacheExtension ===
        if (holder == null && mViewCacheExtension != null) {
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
            }
        }

        // === 步骤 5：RecycledViewPool（按 viewType 匹配）===
        if (holder == null) {
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal(); // 清除旧状态，准备重新绑定
            }
        }

        // === 步骤 6：都没命中 → 创建新的 ===
        if (holder == null) {
            long start = getNanoTime();
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            long end = getNanoTime();
            // 记录创建耗时，供预取机制使用
            mRecyclerPool.factorInCreateTime(type, end - start);
        }
    }

    // === 绑定数据 ===
    // 只有"需要绑定"的才会调用 onBindViewHolder
    // Scrap 和 Cache 命中的不需要，Pool 和新创建的需要
    if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }

    return holder;
}
```

**注意 `deadlineNs` 参数**：这是预取机制的关键。GapWorker 调用这个方法时会传入一个截止时间，如果创建或绑定来不及在下一帧前完成，就会提前终止，避免造成卡顿。

---

## 三、布局流程：LayoutManager 如何工作

### 3.1 整体流程

RecyclerView 的布局流程遵循 Android 标准的 measure → layout → draw 三步：

```
RecyclerView.onMeasure()
    → LayoutManager.onMeasure()

RecyclerView.onLayout()
    → dispatchLayout()
        → dispatchLayoutStep1()  // 预布局：记录动画前状态
        → dispatchLayoutStep2()  // 实际布局：LayoutManager.onLayoutChildren()
        → dispatchLayoutStep3()  // 触发动画

RecyclerView.onDraw()
    → ItemDecoration.onDraw()        // 在 item 下方绘制
    → super.onDraw() (绘制子 View)
    → ItemDecoration.onDrawOver()    // 在 item 上方绘制
```

### 3.2 三步布局的设计原因

**为什么需要"预布局"（Step1）？** 这是为了实现**可预测的 item 动画**。

假设你删除了 position=2 的 item。RecyclerView 需要知道：
1. 删除前，每个 item 在什么位置？（预布局记录）
2. 删除后，每个 item 在什么位置？（实际布局计算）
3. 有了前后位置，才能播放从 A 到 B 的平移动画

如果没有预布局，RecyclerView 无法知道动画的起始状态，就只能做简单的淡入淡出。

```java
void dispatchLayout() {
    // Step 1：预布局，在 mChangedScrap 中保存即将变化的 ViewHolder
    // 让 LayoutManager 额外布局一些"即将消失"的 item，记录它们的位置
    dispatchLayoutStep1();

    // Step 2：真正的布局
    // LayoutManager.onLayoutChildren() 在这里被调用
    dispatchLayoutStep2();

    // Step 3：比较 Step1 和 Step2 的结果，触发 ItemAnimator
    dispatchLayoutStep3();
}
```

### 3.3 LinearLayoutManager 的 fill 过程

LinearLayoutManager 的核心布局逻辑可以简化为：**确定一个锚点，然后向两个方向填充，直到填满屏幕。**

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // 1. 确定锚点（从哪个位置开始布局）
    //    - 通常是当前屏幕上第一个可见 item 的位置
    //    - 或者是 scrollToPosition() 指定的位置
    updateAnchorInfoForLayout(recycler, state, mAnchorInfo);

    // 2. 把屏幕上所有子 View 暂存到 Scrap
    detachAndScrapAttachedViews(recycler);

    // 3. 从锚点向两个方向填充
    if (mAnchorInfo.mLayoutFromEnd) {
        // 锚点在底部：先向上填充，再向下填充
        fill(recycler, mLayoutState, state, false); // 向上
        fill(recycler, mLayoutState, state, false); // 向下
    } else {
        // 锚点在顶部：先向下填充，再向上填充
        fill(recycler, mLayoutState, state, false); // 向下
        fill(recycler, mLayoutState, state, false); // 向上
    }
}
```

**fill 方法** 是一个循环：不断从 Recycler 取 ViewHolder → 测量 → 布局，直到没有剩余空间：

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {

    int remainingSpace = layoutState.mAvailable;

    while (remainingSpace > 0 && layoutState.hasMore(state)) {
        // 取出一个 ViewHolder 并布局
        layoutChunk(recycler, state, layoutState, layoutChunkResult);

        // 减去这个 item 消耗的空间
        remainingSpace -= layoutChunkResult.mConsumed;
    }
    return consumedSpace;
}
```

**layoutChunk** 处理单个 item：

```java
void layoutChunk(RecyclerView.Recycler recycler, ...) {
    // 1. 从 Recycler 获取 View（触发四级缓存查找）
    View view = layoutState.next(recycler);

    // 2. addView 到 RecyclerView
    addView(view);

    // 3. 测量（考虑 ItemDecoration 的偏移）
    measureChildWithMargins(view, 0, 0);

    // 4. 布局到正确位置
    layoutDecoratedWithMargins(view, left, top, right, bottom);
}
```

### 3.4 滚动时的回收与填充

当用户滑动时，LayoutManager 的 `scrollVerticallyBy()` 被调用：

```
用户手指向上滑动 (dy > 0)
    │
    ├── 1. 回收顶部滑出屏幕的 item
    │      → removeAndRecycleView(topView, recycler)
    │      → ViewHolder 进入 mCachedViews 或 Pool
    │
    ├── 2. 在底部填充新 item
    │      → fill(recycler, ...) 向下填充
    │      → 从 Recycler 获取 ViewHolder（触发缓存查找）
    │
    └── 3. 偏移所有子 View
           → offsetChildrenVertical(-scrolled)
```

**这里有一个重要的性能细节：** RecyclerView 不会每次滑动 1px 就做一次完整的 layout。它只在 item 滑出/滑入时才做回收和填充，平时只是调用 `offsetChildrenVertical()` 做简单的位移，这是一个非常轻量的操作。

---

## 四、数据更新机制

### 4.1 notifyDataSetChanged vs 精确通知

```
notifyDataSetChanged()
    → 标记所有 ViewHolder 为 FLAG_INVALID
    → 所有 ViewHolder 降级到 RecycledViewPool（Cache 清空）
    → 所有 item 都需要重新 bind
    → 无法触发动画（因为不知道哪些变了）
    → 性能最差

notifyItemChanged(position)
    → 只标记指定 ViewHolder 为 FLAG_UPDATE
    → 其他 ViewHolder 不受影响
    → 只重新 bind 变化的 item
    → 可以触发变化动画

notifyItemInserted / Removed / Moved
    → 精确的增删移操作
    → 触发对应的动画
    → 其他 item 只需要调整位置
```

### 4.2 DiffUtil：自动计算最小更新

**问题：** 手动调用 `notifyItemChanged/Inserted/Removed` 很麻烦，尤其是当新旧列表差异很大时。

**DiffUtil** 基于 **Myers 差分算法** 自动计算两个列表的最小编辑操作：

```kotlin
// 核心回调：告诉 DiffUtil 如何比较两个 item
class UserDiffCallback : DiffUtil.ItemCallback<User>() {

    // 第一层比较：是不是"同一个东西"（通常比较 id）
    override fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
        return oldItem.id == newItem.id
    }

    // 第二层比较：内容有没有变化（只在 areItemsTheSame=true 时调用）
    override fun areContentsTheSame(oldItem: User, newItem: User): Boolean {
        return oldItem == newItem
    }

    // 第三层（可选）：具体哪些字段变了（用于 payload 局部更新）
    override fun getChangePayload(oldItem: User, newItem: User): Any? {
        val diff = mutableMapOf<String, Any>()
        if (oldItem.name != newItem.name) diff["name"] = newItem.name
        if (oldItem.avatar != newItem.avatar) diff["avatar"] = newItem.avatar
        return if (diff.isEmpty()) null else diff
    }
}
```

**三层比较的设计思想：**
1. `areItemsTheSame`：快速过滤 —— id 不同直接当作"删除旧的 + 插入新的"
2. `areContentsTheSame`：精确判断 —— id 相同但内容不同才需要更新
3. `getChangePayload`：极致优化 —— 只告诉 Adapter 变了哪些字段，而不是整个 item 重新 bind

**时间复杂度：** O(N + D²)，N 是列表长度，D 是差异数量。最坏情况 O(N²)。

### 4.3 AsyncListDiffer 和 ListAdapter

DiffUtil.calculateDiff() 可能耗时（大列表），所以不能在主线程调用。AsyncListDiffer 封装了线程切换：

```kotlin
// 推荐用法：继承 ListAdapter（内部使用 AsyncListDiffer）
class UserAdapter : ListAdapter<User, UserAdapter.ViewHolder>(UserDiffCallback()) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.context), parent, false)
        return ViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(getItem(position))
    }

    // payload 局部更新：只刷新变化的部分
    override fun onBindViewHolder(holder: ViewHolder, position: Int, payloads: List<Any>) {
        if (payloads.isEmpty()) {
            onBindViewHolder(holder, position)
        } else {
            @Suppress("UNCHECKED_CAST")
            val changes = payloads[0] as? Map<String, Any> ?: return
            holder.bindPartial(changes)
        }
    }

    class ViewHolder(private val binding: ItemUserBinding) :
            RecyclerView.ViewHolder(binding.root) {

        fun bind(user: User) {
            binding.tvName.text = user.name
            binding.ivAvatar.load(user.avatar)
        }

        fun bindPartial(changes: Map<String, Any>) {
            changes["name"]?.let { binding.tvName.text = it as String }
            changes["avatar"]?.let { binding.ivAvatar.load(it as String) }
        }
    }
}

// 使用：只需要 submitList，其余全自动
adapter.submitList(newUserList)
```

**AsyncListDiffer 的并发安全：** 通过 `mMaxScheduledGeneration` 计数器实现。每次 `submitList` 递增计数器，后台计算完成后检查计数器是否匹配——如果不匹配说明有更新的 submitList 提交了，当前结果作废。

---

## 五、预取机制（Prefetch）：利用空闲时间

### 5.1 问题背景

每一帧有 16.6ms（60fps）的时间预算。RecyclerView 的 layout 工作（包括 create + bind ViewHolder）通常在帧的前半段完成，后半段可能是空闲的。

同时，下一帧要显示的 item 是可以**预测**的——如果用户在向下滑，那下一帧大概率需要下方的 item。

### 5.2 GapWorker 的工作原理

```
一帧的时间线 (16.6ms @ 60fps)
├── VSYNC 信号到来
├── Input 事件处理
├── Animation 处理
├── Traversal (measure/layout/draw)
│   └── RecyclerView layout 完成
│       └── 计算下一帧可能需要的 item
├── ← 帧的工作完成，剩余时间
│   └── GapWorker 开始预取
│       ├── 调用 tryGetViewHolderForPositionByDeadline()
│       │   传入 deadlineNs = 下一帧 VSYNC 的时间
│       ├── 如果时间够 → create + bind ViewHolder
│       └── 如果时间不够 → 放弃，避免影响下一帧
├── 下一帧 VSYNC 到来
│   └── 预取的 ViewHolder 已经在 Cache 中，直接使用
```

```java
// GapWorker 核心逻辑
final class GapWorker implements Runnable {

    @Override
    public void run() {
        long nextFrameNs = TimeAnimator.getFrameTime() + frameIntervalNs;
        // 用下一帧的时间作为截止时间
        prefetch(nextFrameNs);
    }

    void prefetch(long deadlineNs) {
        // 根据滑动速度预测需要的 item
        // 调用 tryGetViewHolderForPositionByDeadline 提前准备
        // 准备好的 ViewHolder 会被放入 mCachedViews
    }
}
```

**配合嵌套 RecyclerView 的优化：**

```kotlin
// 外层 RecyclerView 的每个 item 是一个横向 RecyclerView
// 告诉 LayoutManager 内层 RecyclerView 首次展示需要几个 item
// 这样预取时会同时预取内层的 item
(innerRecyclerView.layoutManager as LinearLayoutManager)
    .initialPrefetchItemCount = 4
```

---

## 六、性能优化实战

### 6.1 流畅度优化

#### 6.1.1 问题定位：使用 Systrace / Perfetto 分析卡顿

**第一步：抓取 trace**

```bash
# 使用 Perfetto（推荐，Android 10+）
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace \
<<EOF
buffers: {
    size_kb: 63488
}
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/sched_switch"
            ftrace_events: "power/suspend_resume"
            atrace_categories: "view"
            atrace_categories: "am"
            atrace_categories: "input"
            atrace_apps: "com.your.app"
        }
    }
}
duration_ms: 10000
EOF

# 或使用 systrace（兼容旧版本）
python systrace.py -a com.your.app -t 5 sched gfx view input
```

**第二步：在 trace 中定位 RecyclerView 问题**

打开 trace 文件后，重点关注以下 tag：

```
关键 trace 标签：
┌─────────────────────────────────────────────────────────────┐
│ RV Scroll                          RecyclerView 滚动        │
│ RV OnLayout                        RecyclerView 布局        │
│ RV FullInvalidate                  全量刷新（性能警告！）     │
│ RV Prefetch                        预取                     │
│ RV CreateView                      onCreateViewHolder       │
│ RV BindView                        onBindViewHolder         │
│ Choreographer#doFrame              帧回调                   │
│ measure / layout / draw            标准渲染流程              │
└─────────────────────────────────────────────────────────────┘

看什么：
1. 每帧耗时是否超过 16.6ms？
   → 超过就会掉帧，在 trace 中表现为两个 VSYNC 之间没有完成渲染

2. RV CreateView 耗时是否过长？
   → 超过 5ms 要警惕，可能是布局层级太深

3. RV BindView 耗时是否过长？
   → 超过 3ms 要警惕，可能在 bind 中做了耗时操作

4. 是否频繁出现 RV CreateView？
   → 说明缓存不够用，Pool 命中率低
```

**第三步：使用自定义监控代码**

```kotlin
/**
 * RecyclerView 性能监控器
 * 统计 create/bind 耗时和缓存命中率
 */
class RVPerformanceMonitor(private val tag: String) {

    private var createCount = 0
    private var bindCount = 0
    private var totalCreateTimeNs = 0L
    private var totalBindTimeNs = 0L
    private val frameDrops = mutableListOf<Long>()

    /**
     * 在 Adapter 中埋点
     */
    fun onCreateStart() = System.nanoTime()

    fun onCreateEnd(startNs: Long) {
        val duration = System.nanoTime() - startNs
        createCount++
        totalCreateTimeNs += duration
        if (duration > 5_000_000) { // 超过 5ms 警告
            Log.w(tag, "Slow onCreateViewHolder: ${duration / 1_000_000}ms")
        }
    }

    fun onBindStart() = System.nanoTime()

    fun onBindEnd(startNs: Long) {
        val duration = System.nanoTime() - startNs
        bindCount++
        totalBindTimeNs += duration
        if (duration > 3_000_000) { // 超过 3ms 警告
            Log.w(tag, "Slow onBindViewHolder: ${duration / 1_000_000}ms")
        }
    }

    /**
     * 帧率监控
     */
    fun attachFrameMonitor(recyclerView: RecyclerView) {
        var lastFrameTimeNs = 0L
        recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
            override fun onScrolled(recyclerView: RecyclerView, dx: Int, dy: Int) {
                val now = System.nanoTime()
                if (lastFrameTimeNs != 0L) {
                    val frameDuration = now - lastFrameTimeNs
                    if (frameDuration > 17_000_000) { // 超过 17ms = 掉帧
                        frameDrops.add(frameDuration)
                        Log.w(tag, "Frame drop: ${frameDuration / 1_000_000}ms")
                    }
                }
                lastFrameTimeNs = now
            }
        })
    }

    fun dumpStats() {
        Log.i(tag, """
            === RecyclerView Performance Report ===
            Create: $createCount times, avg ${if (createCount > 0) totalCreateTimeNs / createCount / 1_000_000 else 0}ms
            Bind:   $bindCount times, avg ${if (bindCount > 0) totalBindTimeNs / bindCount / 1_000_000 else 0}ms
            Frame drops: ${frameDrops.size} times
            ========================================
        """.trimIndent())
    }
}
```

#### 6.1.2 常见卡顿原因与解决方案

**原因 1：onCreateViewHolder 耗时（> 5ms）**

```
根因：item 布局层级过深，inflate 耗时
诊断：trace 中 RV CreateView 耗时过长，或 inflate 占用大量时间
```

```kotlin
// 方案 1：减少布局层级
// Bad：嵌套 LinearLayout
<LinearLayout>          // 第1层
  <LinearLayout>        // 第2层
    <LinearLayout>      // 第3层
      <TextView/>
      <ImageView/>
    </LinearLayout>
  </LinearLayout>
</LinearLayout>

// Good：使用 ConstraintLayout 打平层级
<ConstraintLayout>      // 只有1层
  <TextView/>
  <ImageView/>
</ConstraintLayout>

// 方案 2：异步 inflate（适用于复杂布局）
class MyAdapter : RecyclerView.Adapter<ViewHolder>() {
    // 预先在后台线程 inflate
    private val viewCache = ConcurrentLinkedQueue<View>()

    init {
        // 在空闲时预创建几个 View
        repeat(5) {
            AsyncLayoutInflater(context).inflate(R.layout.item_complex, parent) { view, _, _ ->
                viewCache.offer(view)
            }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = viewCache.poll()
            ?: LayoutInflater.from(parent.context).inflate(R.layout.item_complex, parent, false)
        return ViewHolder(view)
    }
}
```

**原因 2：onBindViewHolder 耗时（> 3ms）**

```
根因：在 bind 中做了耗时操作（图片解码、数据格式化、创建对象）
诊断：trace 中 RV BindView 耗时过长
```

```kotlin
// Bad：在 onBindViewHolder 中做耗时操作
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    val item = items[position]

    // 1. 每次 bind 都创建新对象 → GC 压力
    holder.itemView.setOnClickListener { onClick(position) }

    // 2. 在主线程做日期格式化 → 耗时
    holder.dateView.text = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault()).format(item.date)

    // 3. 同步解码图片 → 严重卡顿
    val bitmap = BitmapFactory.decodeFile(item.imagePath)
    holder.imageView.setImageBitmap(bitmap)
}

// Good：优化后的 bind
class ViewHolder(view: View) : RecyclerView.ViewHolder(view) {
    // 1. 点击监听器在 ViewHolder 创建时设置一次，不在 bind 中重复创建
    init {
        itemView.setOnClickListener {
            onItemClick?.invoke(bindingAdapterPosition, getItem(bindingAdapterPosition))
        }
    }
    var onItemClick: ((Int, Item) -> Unit)? = null

    fun bind(item: Item) {
        // 2. 日期预先格式化好，或使用缓存的 formatter
        dateView.text = item.formattedDate // 在数据层预处理

        // 3. 使用图片库异步加载
        Glide.with(imageView)
            .load(item.imagePath)
            .override(200, 200) // 指定尺寸，避免加载原图
            .into(imageView)
    }
}
```

**原因 3：频繁 create（缓存命中率低）**

```
根因：viewType 过多、缓存池太小
诊断：trace 中频繁出现 RV CreateView
```

```kotlin
// 方案：合理配置缓存
fun optimizeRecyclerView(rv: RecyclerView) {
    // 1. 增大 Cache（适用于频繁来回滑动）
    rv.setItemViewCacheSize(4) // 默认 2，按需增大

    // 2. 增大 Pool 容量（多 viewType 场景）
    val pool = rv.recycledViewPool
    pool.setMaxRecycledViews(TYPE_TEXT, 15)
    pool.setMaxRecycledViews(TYPE_IMAGE, 10)
    pool.setMaxRecycledViews(TYPE_VIDEO, 5) // 视频卡片较重，少缓存

    // 3. 如果 item 大小固定（大部分场景都是）
    rv.setHasFixedSize(true) // 避免每次数据变化都触发 RecyclerView 自身的 requestLayout

    // 4. 嵌套 RecyclerView 场景：共享 Pool
    val sharedPool = RecyclerView.RecycledViewPool().apply {
        setMaxRecycledViews(INNER_ITEM_TYPE, 20)
    }
    // 所有内层 RecyclerView 共享这个 Pool
}
```

**原因 4：notifyDataSetChanged 导致全量刷新**

```
根因：使用了 notifyDataSetChanged() 而不是精确通知
诊断：trace 中出现 RV FullInvalidate，所有 item 都重新 bind
```

```kotlin
// 终极方案：使用 ListAdapter + DiffUtil
// 已在第四章详细介绍，这里强调一个关键点：

// Bad
fun updateData(newList: List<Item>) {
    items = newList
    notifyDataSetChanged() // 全量刷新，所有缓存失效
}

// Good
adapter.submitList(newList) // DiffUtil 自动计算差异，精确刷新
```

#### 6.1.3 滑动流畅度优化清单

```kotlin
/**
 * RecyclerView 流畅度优化完整配置
 */
fun setupHighPerformanceRecyclerView(
    recyclerView: RecyclerView,
    adapter: RecyclerView.Adapter<*>,
    layoutManager: LinearLayoutManager
) {
    recyclerView.apply {
        // 1. 基础配置
        this.layoutManager = layoutManager
        this.adapter = adapter
        setHasFixedSize(true)
        setItemViewCacheSize(4)

        // 2. 关闭默认动画（如果不需要增删动画）
        // 默认动画在快速滑动时可能造成性能问题
        itemAnimator = null
        // 或者只是缩短动画时间
        // (itemAnimator as? SimpleItemAnimator)?.supportsChangeAnimations = false

        // 3. 预取优化
        layoutManager.initialPrefetchItemCount = 4

        // 4. 图片加载策略：快速滑动时暂停加载
        addOnScrollListener(object : RecyclerView.OnScrollListener() {
            override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
                when (newState) {
                    RecyclerView.SCROLL_STATE_IDLE -> {
                        Glide.with(recyclerView.context).resumeRequests()
                    }
                    RecyclerView.SCROLL_STATE_SETTLING -> {
                        // fling 状态，暂停加载
                        Glide.with(recyclerView.context).pauseRequests()
                    }
                    // DRAGGING 状态可以不暂停，因为用户在慢慢看
                }
            }
        })
    }
}
```

### 6.2 内存优化

#### 6.2.1 内存问题定位

**使用 Android Profiler 分析内存：**

```
1. 打开 Android Studio → Profiler → Memory
2. 操作 RecyclerView（快速滑动、来回滑动）
3. 观察以下指标：

   锯齿状内存曲线 → 频繁 GC → onBindViewHolder 中创建了大量临时对象
   持续上升的曲线 → 内存泄漏 → ViewHolder 持有了不该持有的引用
   突然的大幅增长 → 图片/Bitmap 没有正确管理
```

**使用 Dump Heap 检查 ViewHolder 数量：**

```
1. 在 Profiler 中点击 "Dump Java Heap"
2. 搜索你的 ViewHolder 类名
3. 检查：
   - 实例数量是否合理？（不应该远超屏幕可见数 + 缓存数）
   - 有没有被 Activity/Fragment 持有导致无法回收？
```

#### 6.2.2 常见内存问题与解决方案

**问题 1：ViewHolder 中的图片没有及时释放**

```kotlin
class MyAdapter : RecyclerView.Adapter<ViewHolder>() {

    // 关键回调：ViewHolder 被回收时释放资源
    override fun onViewRecycled(holder: ViewHolder) {
        super.onViewRecycled(holder)
        // 释放图片引用，让 Glide/Picasso 可以回收内存
        Glide.with(holder.imageView).clear(holder.imageView)
        holder.imageView.setImageDrawable(null)
    }

    // 当 RecyclerView 从窗口移除时
    override fun onDetachedFromRecyclerView(recyclerView: RecyclerView) {
        super.onDetachedFromRecyclerView(recyclerView)
        // 清理所有资源
    }
}
```

**问题 2：ViewHolder 持有 Activity 引用导致泄漏**

```kotlin
// Bad：直接传 Activity
class MyAdapter(private val activity: Activity) : RecyclerView.Adapter<VH>() {
    // activity 被 Adapter 持有，如果 Adapter 被其他地方引用，Activity 就泄漏了
}

// Good：使用 WeakReference 或者只传需要的接口
class MyAdapter(
    private val onItemClick: (Item) -> Unit  // 只传 lambda，不持有 Activity
) : RecyclerView.Adapter<VH>()

// 更好：在 ViewHolder 层面处理点击
class ViewHolder(view: View, private val onItemClick: (Int) -> Unit) :
        RecyclerView.ViewHolder(view) {
    init {
        itemView.setOnClickListener { onItemClick(bindingAdapterPosition) }
    }
}
```

**问题 3：缓存过大导致内存浪费**

```kotlin
// 根据 item 的复杂度决定缓存大小
fun configureCacheForItemComplexity(rv: RecyclerView) {
    val pool = rv.recycledViewPool

    // 简单文本 item：布局小，可以多缓存
    pool.setMaxRecycledViews(TYPE_SIMPLE_TEXT, 15)

    // 图片卡片 item：内存占用大，少缓存
    pool.setMaxRecycledViews(TYPE_IMAGE_CARD, 5)

    // 视频播放器 item：非常重，最少缓存
    pool.setMaxRecycledViews(TYPE_VIDEO_PLAYER, 2)
}
```

**问题 4：onBindViewHolder 创建大量临时对象**

```kotlin
// Bad：每次 bind 都创建新对象
override fun onBindViewHolder(holder: VH, position: Int) {
    // 每次 bind 创建新的 SpannableString → GC 压力
    val spannable = SpannableStringBuilder(item.title)
    spannable.setSpan(ForegroundColorSpan(Color.RED), 0, 3, 0)
    holder.titleView.text = spannable

    // 每次 bind 创建新的 Rect → GC 压力
    val rect = Rect(0, 0, 100, 100)
}

// Good：预处理或复用对象
class ViewHolder(view: View) : RecyclerView.ViewHolder(view) {
    // 复用 SpannableStringBuilder
    private val spannableBuilder = SpannableStringBuilder()

    fun bind(item: Item) {
        spannableBuilder.clear()
        spannableBuilder.append(item.title)
        // 复用 builder，不创建新对象
        titleView.text = spannableBuilder
    }
}
```

### 6.3 嵌套 RecyclerView 优化（常见场景）

嵌套 RecyclerView（如首页 Feed 流中嵌套横向列表）是性能问题的重灾区：

```kotlin
/**
 * 嵌套 RecyclerView 的完整优化方案
 */
class OuterAdapter : RecyclerView.Adapter<OuterViewHolder>() {

    // 优化 1：所有内层 RecyclerView 共享 Pool
    private val innerPool = RecyclerView.RecycledViewPool().apply {
        setMaxRecycledViews(INNER_VIEW_TYPE, 20) // 内层 item 类型相同，多缓存
    }

    // 优化 2：保存内层 RecyclerView 的滚动位置
    private val scrollStates = SparseArray<Parcelable?>()

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): OuterViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_outer, parent, false)
        val holder = OuterViewHolder(view)

        // 优化 3：内层 RecyclerView 配置只做一次（在 create 而非 bind 中）
        holder.innerRecyclerView.apply {
            setRecycledViewPool(innerPool)
            setHasFixedSize(true)
            setItemViewCacheSize(4)
            (layoutManager as? LinearLayoutManager)?.initialPrefetchItemCount = 4
        }

        return holder
    }

    override fun onBindViewHolder(holder: OuterViewHolder, position: Int) {
        // 优化 4：bind 时恢复滚动位置
        val innerAdapter = InnerAdapter()
        innerAdapter.submitList(items[position].innerItems)
        holder.innerRecyclerView.adapter = innerAdapter

        // 恢复上次滚动位置
        val state = scrollStates.get(position)
        if (state != null) {
            holder.innerRecyclerView.layoutManager?.onRestoreInstanceState(state)
        } else {
            holder.innerRecyclerView.scrollToPosition(0)
        }
    }

    override fun onViewRecycled(holder: OuterViewHolder) {
        super.onViewRecycled(holder)
        // 优化 5：回收时保存滚动位置
        val position = holder.bindingAdapterPosition
        if (position != RecyclerView.NO_POSITION) {
            scrollStates.put(position,
                holder.innerRecyclerView.layoutManager?.onSaveInstanceState())
        }
    }
}
```

**另一个方案：用 ConcatAdapter 替代嵌套**

如果内层列表不需要独立滚动（只是展示不同类型的内容），可以用 ConcatAdapter 把多个 Adapter 合并成一个扁平列表，完全避免嵌套：

```kotlin
val headerAdapter = HeaderAdapter()
val horizontalSectionAdapter = HorizontalSectionAdapter() // 自己实现横向布局
val feedAdapter = FeedAdapter()
val footerAdapter = FooterAdapter()

recyclerView.adapter = ConcatAdapter(
    ConcatAdapter.Config.Builder()
        .setIsolateViewTypes(false) // 不隔离 viewType → 相同类型可以跨 Adapter 复用
        .build(),
    headerAdapter,
    horizontalSectionAdapter,
    feedAdapter,
    footerAdapter
)
```

---

## 七、ItemDecoration 与 ItemAnimator

### 7.1 ItemDecoration：理解绘制时机

ItemDecoration 的三个方法对应三个不同的时机：

```
RecyclerView 绘制流程：
1. ItemDecoration.onDraw()        ← 在子 View 下方绘制（会被子 View 遮挡）
2. 绘制所有子 View (dispatchDraw)
3. ItemDecoration.onDrawOver()    ← 在子 View 上方绘制（会遮挡子 View）

getItemOffsets() 在测量阶段调用 → 为装饰预留空间
```

**实战案例：吸顶 Header**

吸顶效果用 `onDrawOver` 实现，因为 Header 需要绘制在所有 item 上方：

```kotlin
class StickyHeaderDecoration(
    private val callback: StickyHeaderCallback
) : RecyclerView.ItemDecoration() {

    interface StickyHeaderCallback {
        fun isHeader(position: Int): Boolean
        fun getHeaderView(position: Int, parent: RecyclerView): View
    }

    override fun onDrawOver(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        val topChild = parent.getChildAt(0) ?: return
        val topPosition = parent.getChildAdapterPosition(topChild)
        if (topPosition == RecyclerView.NO_POSITION) return

        // 找到当前顶部应该显示的 Header
        val headerPosition = findHeaderPositionBefore(topPosition)
        if (headerPosition == -1) return

        val headerView = callback.getHeaderView(headerPosition, parent)

        // 测量和布局 Header
        val widthSpec = View.MeasureSpec.makeMeasureSpec(parent.width, View.MeasureSpec.EXACTLY)
        val heightSpec = View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED)
        headerView.measure(widthSpec, heightSpec)
        headerView.layout(0, 0, headerView.measuredWidth, headerView.measuredHeight)

        // 计算推动效果：下一个 Header 推动当前 Header
        val nextHeaderPosition = findNextHeader(topPosition + 1)
        var translateY = 0f
        if (nextHeaderPosition != -1) {
            val nextHeaderView = parent.findViewHolderForAdapterPosition(nextHeaderPosition)?.itemView
            if (nextHeaderView != null && nextHeaderView.top < headerView.measuredHeight) {
                translateY = (nextHeaderView.top - headerView.measuredHeight).toFloat()
            }
        }

        c.save()
        c.translate(0f, translateY)
        headerView.draw(c)
        c.restore()
    }

    private fun findHeaderPositionBefore(position: Int): Int {
        for (i in position downTo 0) {
            if (callback.isHeader(i)) return i
        }
        return -1
    }

    private fun findNextHeader(startPosition: Int): Int {
        // 只在可见范围内查找
        for (i in startPosition until startPosition + 20) {
            if (callback.isHeader(i)) return i
        }
        return -1
    }
}
```

### 7.2 ItemAnimator：预布局与动画的协作

ItemAnimator 的工作依赖于 RecyclerView 的三步布局：

```
Step 1 (预布局)：
  → recordPreLayoutInformation() 记录每个 ViewHolder 的位置
  → LayoutManager 会"多布局"一些即将消失的 item（这样才知道它们从哪里消失的）

Step 2 (实际布局)：
  → recordPostLayoutInformation() 记录最终位置

Step 3 (动画)：
  → 比较前后位置，决定动画类型：
    - 前有后无 → animateDisappearance（消失动画）
    - 前无后有 → animateAppearance（出现动画）
    - 前后都有但位置变了 → animatePersistence（移动动画）
    - 同一位置但内容变了 → animateChange（变化动画）
  → runPendingAnimations() 执行所有动画

动画执行顺序：Remove → Move/Change → Add
```

**性能提示：** 如果你不需要增删动画，设置 `recyclerView.itemAnimator = null` 可以跳过整个预布局阶段，减少一次完整的 layout 过程。

---

## 八、嵌套滑动机制

### 8.1 设计思想

传统的事件分发（dispatchTouchEvent → onInterceptTouchEvent → onTouchEvent）是从外向内的单向传递。但 CoordinatorLayout + AppBarLayout + RecyclerView 这种场景需要内外协商——RecyclerView 滑动前要先问外层"你要不要先消费一部分？"

嵌套滑动通过 `NestedScrollingChild` 和 `NestedScrollingParent` 接口实现了**内向外的协商机制**：

```
用户滑动 RecyclerView (dy = 100px)
    │
    ├── 1. RecyclerView.dispatchNestedPreScroll(dy=100)
    │   └── 父 View (如 CoordinatorLayout)：
    │       "我先消费 60px 来收起 AppBarLayout"
    │       consumed = [0, 60]
    │
    ├── 2. RecyclerView 消费剩余 40px
    │   └── 列表滚动 40px
    │   └── 但实际上只能滚动 30px（到底了）
    │       unconsumed = 10px
    │
    └── 3. RecyclerView.dispatchNestedScroll(
    │       dyConsumed=30, dyUnconsumed=10)
        └── 父 View：
            "还有 10px 我也不要了" 或 "我再来个 overScroll 效果"
```

---

## 九、面试实战

### 9.1 基础概念题

**Q：RecyclerView 的四级缓存是什么？各自在什么场景下命中？**

> 面试官考察：对缓存机制的理解深度

**回答思路：**

先说设计目标，再逐级展开：

"RecyclerView 缓存的设计目标是**尽量减少 create 和 bind 的调用次数**。四级缓存按照离屏幕的距离递增，复用成本递增：

1. **Scrap**：layout 过程中的暂存区。layout 开始时把屏幕上的 ViewHolder 都 detach 到 Scrap，layout 完成时取回来。命中后 0 成本。

2. **Cache (mCachedViews)**：存放刚滑出屏幕的 ViewHolder，默认 2 个。按 position 精确匹配，命中后不需要 bind。典型场景是用户来回小幅滑动。

3. **ViewCacheExtension**：开发者自定义缓存，实际很少使用。

4. **RecycledViewPool**：按 viewType 分桶，每桶默认 5 个。命中后需要重新 bind，因为 position 信息已丢失。可以跨 RecyclerView 共享。"

**追问：Cache 为什么不需要 bind，Pool 为什么需要？**

"Cache 是按 position 匹配的——一个 ViewHolder 缓存在 position=5 的位置，只有当你再次需要 position=5 时才能命中，所以数据一定是对的，不需要重新 bind。Pool 只按 viewType 匹配，取出来的 ViewHolder 原来可能是 position=3 的数据，现在要用在 position=15，当然需要重新 bind。"

**追问：为什么 Cache 默认只有 2 个？**

"2 个覆盖了最常见的场景——往一个方向滑动时，上下各 1 个刚滑出的 item 被缓存。增大 Cache 可以提高来回滑动时的命中率，但代价是内存占用增加（Cache 中的 ViewHolder 不会清除数据）。需要根据业务场景权衡。"

---

### 9.2 源码分析题

**Q：描述一下 RecyclerView 从接收到 notifyItemChanged(5) 到界面更新的完整流程。**

> 面试官考察：对数据更新到 UI 刷新的链路理解

```
notifyItemChanged(5)
    │
    ├── AdapterHelper 记录一个 UPDATE 操作
    │   → mPendingUpdates.add(UpdateOp(UPDATE, 5, 1))
    │
    ├── 请求重新布局
    │   → requestLayout()
    │
    ├── [等待下一帧 VSYNC]
    │
    ├── dispatchLayoutStep1() —— 预布局
    │   ├── 消费 mPendingUpdates
    │   ├── 遍历所有子 View，找到 position=5 的 ViewHolder
    │   ├── 标记 FLAG_UPDATE
    │   ├── recordPreLayoutInformation() 记录动画前状态
    │   └── 把 position=5 的 ViewHolder 放入 mChangedScrap
    │
    ├── dispatchLayoutStep2() —— 实际布局
    │   ├── LayoutManager.onLayoutChildren()
    │   │   ├── detachAndScrapAttachedViews() → 其余 ViewHolder 进 mAttachedScrap
    │   │   └── fill()
    │   │       ├── position=0~4: 从 mAttachedScrap 取回，直接用
    │   │       ├── position=5:
    │   │       │   ├── mAttachedScrap 没有 (在 mChangedScrap 中)
    │   │       │   ├── Cache 没有
    │   │       │   ├── Pool 取出一个同 viewType 的 ViewHolder
    │   │       │   │   (或者如果 supportsChangeAnimations=false，
    │   │       │   │    则从 mChangedScrap 取回)
    │   │       │   └── onBindViewHolder(holder, 5) 重新绑定数据
    │   │       └── position=6~N: 从 mAttachedScrap 取回
    │   └── recordPostLayoutInformation()
    │
    └── dispatchLayoutStep3() —— 触发动画
        ├── 比较 position=5 的 pre/post 信息
        ├── animateChange() → 交叉淡入淡出（默认效果）
        │   旧 ViewHolder 淡出，新 ViewHolder 淡入
        └── 旧 ViewHolder 动画结束后回收到 Pool
```

---

### 9.3 性能优化场景题

**Q：线上监控发现 RecyclerView 列表滑动帧率只有 45fps，如何定位和优化？**

> 面试官考察：实际问题解决能力

**回答思路（按优先级排序）：**

"我会按以下步骤排查：

**第一步：抓 Trace 定位瓶颈。** 用 Perfetto 或 Systrace 抓取滑动过程的 trace，看每帧的耗时分布：
- 如果 `RV CreateView` 占比高 → 缓存不够或布局太复杂
- 如果 `RV BindView` 占比高 → bind 中有耗时操作
- 如果 `measure/layout` 占比高 → 布局层级太深或有嵌套测量
- 如果 `draw` 占比高 → 过度绘制或自定义绘制耗时

**第二步：针对性优化。**

如果是 create 耗时：
- 检查 item 布局层级，用 ConstraintLayout 优化
- 增大 RecycledViewPool 容量减少 create 次数
- 考虑异步 inflate 预创建 ViewHolder

如果是 bind 耗时：
- 检查有没有在 bind 中做图片解码、日期格式化等操作
- 使用 payload 局部更新，避免整个 item 重新 bind
- 把耗时计算移到数据层预处理

如果是全量刷新导致：
- 把 `notifyDataSetChanged()` 替换为 DiffUtil
- 实现 `getChangePayload` 支持极致的局部更新

**第三步：验证效果。** 优化后再抓一次 trace，对比帧耗时分布，确认帧率恢复到 60fps。同时在线上加入帧率监控（通过 Choreographer.FrameCallback 统计掉帧率），持续跟踪。"

---

### 9.4 设计思考题

**Q：如果让你设计一个 RecyclerView 的缓存机制，你会怎么设计？为什么 Google 设计成现在这样？**

> 面试官考察：架构思维和设计理解

**回答思路：**

"我会从**缓存命中率 vs 内存开销**的平衡角度来设计。

最简单的方案是一个大的 HashMap<position, ViewHolder>，但问题是：
1. 内存不可控——10000 条数据就缓存 10000 个 ViewHolder
2. 数据变化时 position 会变，维护成本高

Google 的四级缓存本质是按**数据新鲜度**分层：

- **Scrap**：解决 layout 过程中的临时存储问题（技术需要，不是性能优化）
- **Cache**：针对"刚刚离开屏幕"的 ViewHolder，position 精确匹配 → 零成本复用。空间小（默认 2），命中率高（因为用户最常做的就是来回滑动一小段）
- **Pool**：针对"已经离开屏幕较远"的 ViewHolder，只保留 View 结构，丢弃数据 → 省一次 create。按 viewType 分桶，控制内存上限

这个设计的精妙之处在于：
1. 渐进式降级——越远的缓存，复用成本越高，但比重新创建便宜
2. 空间可控——每级都有上限，不会无限膨胀
3. 可共享——Pool 可以跨 RecyclerView 共享，在 ViewPager 场景下效果显著
4. 可定制——每级的大小都可以调整，开发者可以根据业务特点优化"

---

### 9.5 高频追问速答

| 追问 | 要点 |
|------|------|
| setHasFixedSize(true) 做了什么？ | 当数据变化时，直接重新 layout children 而不是触发 RecyclerView 自身的 requestLayout()。避免了 RecyclerView 重新测量自身大小。 |
| setHasStableIds(true) 的作用？ | 启用 stableId 匹配：即使 position 变了（如插入/删除），只要 id 一致就能从 Cache/Scrap 中找到对应的 ViewHolder，提高缓存命中率和动画准确性。 |
| RecyclerView 和 ListView 缓存的本质区别？ | ListView 两级缓存（ActiveViews + ScrapViews）只按 position 匹配；RecyclerView 四级缓存支持 position 匹配、viewType 匹配、stableId 匹配，粒度更细，复用率更高。 |
| notifyItemChanged 和 notifyDataSetChanged 对缓存的影响？ | `notifyItemChanged` 只标记指定 ViewHolder 为 UPDATE，其他不受影响；`notifyDataSetChanged` 标记所有 ViewHolder 为 INVALID，Cache 全部降级到 Pool，性能最差。 |
| RecyclerView 的预取是在哪个线程执行的？ | **主线程**，但是在主线程的空闲时间（当前帧渲染完成后、下一帧 VSYNC 到来前）。通过 Choreographer 调度，有严格的 deadline 控制。 |
| 什么情况下应该关闭默认动画？ | 快速刷新大量数据、对帧率要求极高、不需要增删动画效果的场景。关闭后跳过预布局阶段，减少一次完整 layout。 |
| ConcatAdapter 的 isolateViewTypes 参数？ | `true`（默认）：每个子 Adapter 的 viewType 互相隔离，不共享 ViewHolder；`false`：共享 viewType，相同类型可以跨 Adapter 复用，提高缓存利用率。 |

---

## 十、知识串联总图

```
                       RecyclerView 核心知识图谱

  ┌──────────────────────────────────────────────────────────────┐
  │                     设计哲学                                  │
  │         职责分离 + 组合模式 + 强制 ViewHolder                  │
  └──────────────────────┬───────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │ 缓存机制  │    │ 布局流程  │    │  数据更新     │
  │          │    │          │    │              │
  │ Scrap    │◄──►│ 三步布局  │◄──►│ DiffUtil     │
  │ Cache    │    │ fill 填充 │    │ AsyncListDiffer│
  │ Extension│    │ 锚点机制  │    │ Payload      │
  │ Pool     │    │          │    │              │
  └────┬─────┘    └────┬─────┘    └──────┬───────┘
       │               │                 │
       └───────────────┼─────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │   性能优化      │
              ├────────────────┤
              │ 流畅度          │
              │ · Trace 定位    │
              │ · 布局优化      │
              │ · bind 优化     │
              │ · 预取优化      │
              ├────────────────┤
              │ 内存            │
              │ · 缓存大小控制  │
              │ · 资源释放      │
              │ · 泄漏防护      │
              ├────────────────┤
              │ 嵌套场景        │
              │ · 共享 Pool     │
              │ · ConcatAdapter │
              │ · 状态保存      │
              └────────────────┘
```

---

> 本文持续更新中。核心理解一句话总结：**RecyclerView 的一切设计都围绕"最小化 create + bind 的调用次数"展开，四级缓存是手段，职责分离是基础，DiffUtil 是数据层的配合。**
