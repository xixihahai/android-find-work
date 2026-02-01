# Fragment 详解

## 1. 概述

Fragment 是 Android 中可重用的 UI 组件，代表 Activity 中的一部分用户界面或行为。Fragment 有自己的生命周期，可以在 Activity 运行时动态添加、移除或替换。

**Fragment 的核心特点：**
- 模块化 UI 设计
- 有独立的生命周期，但依附于 Activity
- 可以在多个 Activity 中复用
- 支持回退栈管理
- 适配不同屏幕尺寸（平板、手机）

**Fragment 的使用场景：**
- ViewPager/ViewPager2 页面
- 底部导航切换
- 对话框（DialogFragment）
- 平板多面板布局
- Navigation 组件导航

## 2. 核心原理

### 2.1 Fragment 生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                    Fragment 添加到 Activity                      │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onAttach(context)                             │
│  - Fragment 与 Activity 关联                                     │
│  - 可以获取 Activity 引用                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onCreate(savedInstanceState)                  │
│  - Fragment 创建                                                 │
│  - 初始化非 UI 相关数据                                          │
│  - 恢复保存的状态                                                │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onCreateView(inflater, container, state)      │
│  - 创建 Fragment 的视图层级                                      │
│  - 返回 Fragment 的根 View                                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onViewCreated(view, savedInstanceState)       │
│  - View 创建完成                                                 │
│  - 初始化 View 相关操作                                          │
│  - 设置监听器、绑定数据                                          │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onActivityCreated() [已废弃]                  │
│  - Activity.onCreate() 完成                                      │
│  - 推荐使用 onViewCreated()                                      │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onStart()                                     │
│  - Fragment 可见                                                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onResume()                                    │
│  - Fragment 可交互                                               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Fragment Running                              │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onPause()                                     │
│  - Fragment 失去焦点                                             │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onStop()                                      │
│  - Fragment 不可见                                               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onDestroyView()                               │
│  - View 被销毁                                                   │
│  - 清理 View 相关资源                                            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onDestroy()                                   │
│  - Fragment 被销毁                                               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    onDetach()                                    │
│  - Fragment 与 Activity 解除关联                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Fragment 与 Activity 生命周期对应关系

| Activity 生命周期 | Fragment 生命周期 |
|------------------|------------------|
| onCreate() | onAttach() → onCreate() → onCreateView() → onViewCreated() |
| onStart() | onStart() |
| onResume() | onResume() |
| onPause() | onPause() |
| onStop() | onStop() |
| onDestroy() | onDestroyView() → onDestroy() → onDetach() |

### 2.3 FragmentManager 与事务

```
┌─────────────────────────────────────────────────────────────────┐
│                      FragmentManager                             │
│  - 管理 Fragment 的添加、移除、替换                               │
│  - 管理回退栈                                                    │
│  - 处理 Fragment 事务                                            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   FragmentTransaction                            │
│  - add()：添加 Fragment                                          │
│  - remove()：移除 Fragment                                       │
│  - replace()：替换 Fragment                                      │
│  - hide()/show()：隐藏/显示 Fragment                             │
│  - attach()/detach()：附加/分离 Fragment                         │
│  - addToBackStack()：添加到回退栈                                │
│  - commit()/commitNow()/commitAllowingStateLoss()               │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 关键源码解析

### 3.1 FragmentManager 执行事务

```java
// androidx/fragment/app/FragmentManager.java
public abstract class FragmentManager {
    
    public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
    }
    
    void execPendingActions(boolean allowStateLoss) {
        ensureExecReady(allowStateLoss);
        
        boolean didSomething = false;
        // 执行所有待处理的事务
        while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
            mExecutingActions = true;
            try {
                // 优化事务
                removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
                cleanupExec();
            } finally {
                mExecutingActions = false;
            }
            didSomething = true;
        }
        
        // 更新 Fragment 状态
        updateOnBackPressedCallbackEnabled();
        doPendingDeferredStart();
        mFragmentStore.burpActive();
    }
}
```

### 3.2 FragmentTransaction 实现

```java
// androidx/fragment/app/BackStackRecord.java
final class BackStackRecord extends FragmentTransaction implements
        FragmentManager.BackStackEntry, FragmentManager.OpGenerator {
    
    final FragmentManager mManager;
    ArrayList<Op> mOps = new ArrayList<>();
    
    @Override
    public FragmentTransaction add(int containerViewId, Fragment fragment, String tag) {
        doAddOp(containerViewId, fragment, tag, OP_ADD);
        return this;
    }
    
    @Override
    public FragmentTransaction replace(int containerViewId, Fragment fragment, String tag) {
        doAddOp(containerViewId, fragment, tag, OP_REPLACE);
        return this;
    }
    
    void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
        final Class<?> fragmentClass = fragment.getClass();
        final int modifiers = fragmentClass.getModifiers();
        
        // 检查 Fragment 类是否合法
        if (fragmentClass.isAnonymousClass() || !Modifier.isPublic(modifiers)
                || (fragmentClass.isMemberClass() && !Modifier.isStatic(modifiers))) {
            throw new IllegalStateException("Fragment " + fragmentClass.getCanonicalName()
                    + " must be a public static class to be properly recreated from instance state.");
        }
        
        fragment.mFragmentManager = mManager;
        
        if (tag != null) {
            fragment.mTag = tag;
        }
        
        if (containerViewId != 0) {
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }
        
        // 添加操作到列表
        addOp(new Op(opcmd, fragment));
    }
    
    @Override
    public int commit() {
        return commitInternal(false);
    }
    
    @Override
    public int commitAllowingStateLoss() {
        return commitInternal(true);
    }
    
    int commitInternal(boolean allowStateLoss) {
        if (mCommitted) throw new IllegalStateException("commit already called");
        mCommitted = true;
        
        if (mAddToBackStack) {
            mIndex = mManager.allocBackStackIndex();
        } else {
            mIndex = -1;
        }
        
        // 加入待执行队列
        mManager.enqueueAction(this, allowStateLoss);
        return mIndex;
    }
}
```

### 3.3 Fragment 状态管理

```java
// androidx/fragment/app/FragmentStateManager.java
class FragmentStateManager {
    
    void moveToExpectedState() {
        // 根据目标状态移动 Fragment
        if (mFragment.mState < mFragmentStore.getTargetState()) {
            // 向上移动状态
            switch (mFragment.mState) {
                case Fragment.INITIALIZING:
                    // 执行 attach
                    attach();
                    // fall through
                case Fragment.ATTACHED:
                    // 执行 create
                    create();
                    // fall through
                case Fragment.CREATED:
                    // 创建 View
                    createView();
                    // fall through
                case Fragment.VIEW_CREATED:
                    // Activity 创建完成
                    activityCreated();
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    // 执行 start
                    start();
                    // fall through
                case Fragment.STARTED:
                    // 执行 resume
                    resume();
                    break;
            }
        } else if (mFragment.mState > mFragmentStore.getTargetState()) {
            // 向下移动状态
            switch (mFragment.mState) {
                case Fragment.RESUMED:
                    pause();
                    // fall through
                case Fragment.STARTED:
                    stop();
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    // fall through
                case Fragment.VIEW_CREATED:
                    destroyView();
                    // fall through
                case Fragment.CREATED:
                    destroy();
                    // fall through
                case Fragment.ATTACHED:
                    detach();
                    break;
            }
        }
    }
}
```

## 4. 实战应用

### 4.1 基本使用

```kotlin
// Fragment 定义
class HomeFragment : Fragment() {
    
    private var _binding: FragmentHomeBinding? = null
    private val binding get() = _binding!!
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentHomeBinding.inflate(inflater, container, false)
        return binding.root
    }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 初始化 View
        binding.button.setOnClickListener {
            // 处理点击
        }
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null  // 避免内存泄漏
    }
    
    companion object {
        fun newInstance(param: String): HomeFragment {
            return HomeFragment().apply {
                arguments = Bundle().apply {
                    putString("param", param)
                }
            }
        }
    }
}

// Activity 中使用
class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        if (savedInstanceState == null) {
            supportFragmentManager.beginTransaction()
                .add(R.id.fragment_container, HomeFragment.newInstance("data"))
                .commit()
        }
    }
    
    fun replaceFragment(fragment: Fragment) {
        supportFragmentManager.beginTransaction()
            .replace(R.id.fragment_container, fragment)
            .addToBackStack(null)
            .commit()
    }
}
```

### 4.2 Fragment 通信

#### 方式1：ViewModel 共享（推荐）

```kotlin
// 共享 ViewModel
class SharedViewModel : ViewModel() {
    private val _selectedItem = MutableLiveData<String>()
    val selectedItem: LiveData<String> = _selectedItem
    
    fun selectItem(item: String) {
        _selectedItem.value = item
    }
}

// Fragment A - 发送数据
class ListFragment : Fragment() {
    
    private val sharedViewModel: SharedViewModel by activityViewModels()
    
    fun onItemClick(item: String) {
        sharedViewModel.selectItem(item)
    }
}

// Fragment B - 接收数据
class DetailFragment : Fragment() {
    
    private val sharedViewModel: SharedViewModel by activityViewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        sharedViewModel.selectedItem.observe(viewLifecycleOwner) { item ->
            // 更新 UI
            binding.textView.text = item
        }
    }
}
```

#### 方式2：Fragment Result API（AndroidX Fragment 1.3.0+）

```kotlin
// Fragment A - 设置结果监听
class FragmentA : Fragment() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 设置结果监听
        setFragmentResultListener("requestKey") { key, bundle ->
            val result = bundle.getString("resultKey")
            // 处理结果
        }
    }
    
    fun navigateToB() {
        parentFragmentManager.beginTransaction()
            .replace(R.id.container, FragmentB())
            .addToBackStack(null)
            .commit()
    }
}

// Fragment B - 返回结果
class FragmentB : Fragment() {
    
    fun returnResult() {
        setFragmentResult("requestKey", bundleOf("resultKey" to "result data"))
        parentFragmentManager.popBackStack()
    }
}

// 父子 Fragment 通信
class ParentFragment : Fragment() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 监听子 Fragment 结果
        childFragmentManager.setFragmentResultListener("childRequest", this) { key, bundle ->
            // 处理子 Fragment 返回的结果
        }
    }
}

class ChildFragment : Fragment() {
    
    fun sendResultToParent() {
        // 向父 Fragment 发送结果
        parentFragmentManager.setFragmentResult("childRequest", bundleOf("data" to "value"))
    }
}
```

#### 方式3：接口回调（传统方式）

```kotlin
// 定义接口
interface OnFragmentInteractionListener {
    fun onItemSelected(item: String)
}

// Fragment
class ListFragment : Fragment() {
    
    private var listener: OnFragmentInteractionListener? = null
    
    override fun onAttach(context: Context) {
        super.onAttach(context)
        listener = context as? OnFragmentInteractionListener
            ?: throw RuntimeException("$context must implement OnFragmentInteractionListener")
    }
    
    override fun onDetach() {
        super.onDetach()
        listener = null
    }
    
    fun onItemClick(item: String) {
        listener?.onItemSelected(item)
    }
}

// Activity 实现接口
class MainActivity : AppCompatActivity(), OnFragmentInteractionListener {
    
    override fun onItemSelected(item: String) {
        // 处理 Fragment 回调
    }
}
```

### 4.3 ViewPager2 + Fragment 懒加载

```kotlin
// ViewPager2 适配器
class ViewPagerAdapter(
    fragmentActivity: FragmentActivity
) : FragmentStateAdapter(fragmentActivity) {
    
    private val fragments = listOf(
        HomeFragment::class.java,
        SearchFragment::class.java,
        ProfileFragment::class.java
    )
    
    override fun getItemCount(): Int = fragments.size
    
    override fun createFragment(position: Int): Fragment {
        return fragments[position].newInstance()
    }
}

// 懒加载 Fragment 基类
abstract class LazyFragment : Fragment() {
    
    private var isLoaded = false
    
    override fun onResume() {
        super.onResume()
        if (!isLoaded && !isHidden) {
            lazyLoad()
            isLoaded = true
        }
    }
    
    abstract fun lazyLoad()
}

// 使用 Lifecycle 实现懒加载（推荐）
class HomeFragment : Fragment() {
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // ViewPager2 默认使用 RESUMED 状态，Fragment 可见时才会 onResume
        // 因此可以直接在 onViewCreated 或 onResume 中加载数据
        loadData()
    }
    
    private fun loadData() {
        // 加载数据
    }
}

// Activity 设置
class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        val viewPager = findViewById<ViewPager2>(R.id.viewPager)
        viewPager.adapter = ViewPagerAdapter(this)
        
        // 设置预加载页面数（默认为 1）
        viewPager.offscreenPageLimit = 1
        
        // 配合 TabLayout
        TabLayoutMediator(tabLayout, viewPager) { tab, position ->
            tab.text = when (position) {
                0 -> "首页"
                1 -> "搜索"
                2 -> "我的"
                else -> ""
            }
        }.attach()
    }
}
```

### 4.4 Navigation 组件

```kotlin
// nav_graph.xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment"
        android:label="Home">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left"
            app:popEnterAnim="@anim/slide_in_left"
            app:popExitAnim="@anim/slide_out_right" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment"
        android:label="Detail">
        <argument
            android:name="itemId"
            app:argType="string" />
    </fragment>
</navigation>
```

```kotlin
// Activity
class MainActivity : AppCompatActivity() {
    
    private lateinit var navController: NavController
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        val navHostFragment = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
        navController = navHostFragment.navController
        
        // 配合 BottomNavigationView
        val bottomNav = findViewById<BottomNavigationView>(R.id.bottom_nav)
        bottomNav.setupWithNavController(navController)
    }
    
    override fun onSupportNavigateUp(): Boolean {
        return navController.navigateUp() || super.onSupportNavigateUp()
    }
}

// Fragment 导航
class HomeFragment : Fragment() {
    
    fun navigateToDetail(itemId: String) {
        // 方式1：使用 Safe Args（推荐）
        val action = HomeFragmentDirections.actionHomeToDetail(itemId)
        findNavController().navigate(action)
        
        // 方式2：使用 Bundle
        val bundle = bundleOf("itemId" to itemId)
        findNavController().navigate(R.id.action_home_to_detail, bundle)
    }
}

// 接收参数
class DetailFragment : Fragment() {
    
    private val args: DetailFragmentArgs by navArgs()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        val itemId = args.itemId
        // 使用参数
    }
}
```

### 4.5 DialogFragment

```kotlin
class ConfirmDialogFragment : DialogFragment() {
    
    private var onConfirm: (() -> Unit)? = null
    private var onCancel: (() -> Unit)? = null
    
    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        val title = arguments?.getString("title") ?: "确认"
        val message = arguments?.getString("message") ?: ""
        
        return MaterialAlertDialogBuilder(requireContext())
            .setTitle(title)
            .setMessage(message)
            .setPositiveButton("确定") { _, _ ->
                onConfirm?.invoke()
            }
            .setNegativeButton("取消") { _, _ ->
                onCancel?.invoke()
            }
            .create()
    }
    
    fun setOnConfirmListener(listener: () -> Unit): ConfirmDialogFragment {
        onConfirm = listener
        return this
    }
    
    fun setOnCancelListener(listener: () -> Unit): ConfirmDialogFragment {
        onCancel = listener
        return this
    }
    
    companion object {
        fun newInstance(title: String, message: String): ConfirmDialogFragment {
            return ConfirmDialogFragment().apply {
                arguments = bundleOf(
                    "title" to title,
                    "message" to message
                )
            }
        }
    }
}

// 使用
ConfirmDialogFragment.newInstance("删除", "确定要删除吗？")
    .setOnConfirmListener { 
        // 执行删除
    }
    .show(supportFragmentManager, "confirm_dialog")
```

## 5. 常见面试题

### 问题1：Fragment 的生命周期有哪些？与 Activity 的对应关系？

**答案要点：**
- **生命周期**：onAttach → onCreate → onCreateView → onViewCreated → onStart → onResume → onPause → onStop → onDestroyView → onDestroy → onDetach
- **与 Activity 对应**：
  - Activity.onCreate() 时：Fragment 执行到 onViewCreated
  - Activity.onStart() 时：Fragment.onStart()
  - Activity.onResume() 时：Fragment.onResume()
  - Activity.onDestroy() 时：Fragment 执行 onDestroyView → onDestroy → onDetach
- **注意**：onActivityCreated() 已废弃，推荐使用 onViewCreated()

### 问题2：add、replace、hide、show、attach、detach 的区别？

**答案要点：**
| 方法 | 作用 | 生命周期影响 |
|-----|------|-------------|
| add | 添加 Fragment | 完整生命周期 |
| remove | 移除 Fragment | 执行到 onDetach |
| replace | 替换 Fragment | 移除旧的，添加新的 |
| hide | 隐藏 Fragment | 不影响生命周期，只是 View 不可见 |
| show | 显示 Fragment | 不影响生命周期 |
| attach | 重新附加 | 从 onCreateView 开始 |
| detach | 分离 | 执行到 onDestroyView，保留实例 |

### 问题3：commit、commitNow、commitAllowingStateLoss 的区别？

**答案要点：**
- **commit()**：异步执行，加入主线程队列
- **commitNow()**：同步执行，立即生效，不能添加到回退栈
- **commitAllowingStateLoss()**：允许状态丢失，避免 onSaveInstanceState 后提交异常
- **使用建议**：
  - 正常情况使用 commit()
  - 需要立即生效使用 commitNow()
  - 在 onSaveInstanceState 后使用 commitAllowingStateLoss()

### 问题4：Fragment 之间如何通信？

**答案要点：**
1. **ViewModel 共享**（推荐）：通过 activityViewModels() 共享 ViewModel
2. **Fragment Result API**：setFragmentResult/setFragmentResultListener
3. **接口回调**：通过 Activity 中转
4. **EventBus/LiveData**：事件总线
- 推荐使用 ViewModel 或 Result API，解耦且生命周期安全

### 问题5：ViewPager2 + Fragment 如何实现懒加载？

**答案要点：**
- **ViewPager2 默认支持懒加载**：
  - 使用 RESUMED 状态控制 Fragment 生命周期
  - Fragment 可见时才会 onResume
- **实现方式**：
  - 在 onResume 或 onViewCreated 中加载数据
  - 使用 setMaxLifecycle() 控制最大生命周期状态
- **offscreenPageLimit**：控制预加载页面数，默认为 1

### 问题6：为什么 Fragment 需要无参构造函数？

**答案要点：**
- 系统在配置变更或内存回收后需要**重建 Fragment**
- 重建时通过**反射调用无参构造函数**创建实例
- 参数通过 **arguments Bundle** 传递，会被自动保存和恢复
- **正确做法**：
  ```kotlin
  companion object {
      fun newInstance(param: String) = MyFragment().apply {
          arguments = bundleOf("param" to param)
      }
  }
  ```

### 问题7：Fragment 中使用 ViewBinding 需要注意什么？

**答案要点：**
- **必须在 onDestroyView 中置空 binding**：
  ```kotlin
  private var _binding: FragmentBinding? = null
  private val binding get() = _binding!!
  
  override fun onDestroyView() {
      super.onDestroyView()
      _binding = null
  }
  ```
- **原因**：Fragment 的 View 可能被销毁但 Fragment 实例保留（如 detach），binding 持有 View 引用会导致内存泄漏

### 问题8：Navigation 组件的优势是什么？

**答案要点：**
1. **可视化导航图**：直观展示页面关系
2. **Safe Args**：类型安全的参数传递
3. **自动处理回退栈**：简化回退逻辑
4. **支持深层链接**：DeepLink 支持
5. **与其他组件集成**：BottomNavigationView、Toolbar 等
6. **动画支持**：统一的转场动画配置
