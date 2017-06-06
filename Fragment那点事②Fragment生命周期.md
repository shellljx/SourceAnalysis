# Fragment那点事②Fragment生命周期

> 文章里所有分析都是根据Android Sdk 25.3.1 v4包

### 几种状态

Fragment 的状态 `mState` 总共有如下几种:

```java
static final int INITIALIZING = 0;     // Not yet created.
static final int CREATED = 1;          // Created.
static final int ACTIVITY_CREATED = 2; // The activity has finished its creation.
static final int STOPPED = 3;          // Fully created, not started.
static final int STARTED = 4;          // Created and started, not resumed.
static final int RESUMED = 5;          // Created started and resumed.
```

`mState` 的初始状态默认为 `INITIALIZING` ，其实这几个状态的名字和生命周期函数不是一一对应的，而是在某个状态内触发一个或多个生命周期函数。那这几种状态都表示什么意思呢？每种状态的过程中又执行了什么生命周期函数呢？由于代码过多就不贴代码了，源码在 `FragmentManager.moveToState(Fragment f, int newState, int transit, int transitionStyle,boolean keepActive)` 方法中，请自行查看。该方法中的参数 `f` 表示要操作的 Fragment，`newState` 表示要转换到的目标 state。

**INITIALIZING**

从后面的注释可知该状态的 Fragment 还没有创建，当 Fragment 处于这种状态的时候 `mState = 0` 代表要去初始化 Framgent 的一个过程，由源码可知主要做了如下几件事情：

* 当 `f.mSavedFragmentState != null` 的时候表示这个 Fragment 被 onSaveInstanceState 保存了状态，这个状态包括但不限于：内部 View 的状态 `mSavedViewState`、target Fragment 的状态、`mUserVisibleHint` 的状态，然后取出各状态的值赋值给相应属性。当 `f.mUserVisibleHint = false` 的时候表示该 Fragment 是不被用户可见的，所以要让 Fragment 的状态不能超过 `STOPPED` (已经创建但是没有 started)。

* 对 `mHost`、`mParentFragment`、`mFragmentManager` 等属性进行赋值。

* 触发 **Fragment.onAttach(Context context)** 周期函数和 **onAttachFragment(Fragment f)** 周期函数。

* 对 `f.mRetaining` 进行判断，该属性表示意外重启时保存了 Fragment 实例，`Fragment.mRetainInstance` 表示重启时是否要保存实例，这在后面的 *Fragment那点事③保存与恢复* 中会有详细的介绍。当为 true 的时候直接回复 ChildFragment 的SavedState，并把 Fragment 的状态修改为 `CREATED` (已经创建)。反之触发 **onCreate(savedInstanceState)** 周期函数并把 Fragment 的状态修改为 `CREATED` (已经创建)。

  ```java
  if (!f.mRetaining) {
    f.performCreate(f.mSavedFragmentState);
    dispatchOnFragmentCreated(f, f.mSavedFragmentState, false);
  } else {
    f.restoreChildFragmentState(f.mSavedFragmentState);
    f.mState = Fragment.CREATED;
  }
  ```

* 对属性 `f.mFromLayout` 进行判断，该属性表示 Fragment 是否是在 xml 中直接通过 `<fragment/>` 声明的，如果是就直接触发 **onCreateView** 周期函数和 **onViewCreated** 周期函数。(大部分情况下都是在代码中动态添加 Fragment 的)。

**CREATED**

该状态表示 Fragment 已经创建 `mState = 1`，当 `newState > Fragment.CREATED` 的时候执行如下逻辑：

* 判断 `f.mFromLayout` 属性，如果不是从 xml 中直接声明的而是在代码中动态添加的(通常都是添加到一个  `<FrameLayout/>` 载体中)，判断 `f.mContainerId` 是否等于 0 ，如果不等于说明该载体 ViewGroup 有效并通过 `onFindViewById` 得到载体 ViewGroup。
* 触发 **onCreateView** 周期函数返回并赋值给 `f.mView` 由此可以看出该属性表示 Fragment 的根 View。
* 如果 `f.mView != null` 就赋值给 `f.mInnerView` 并根据 `f.mHidden` 属性判断是否要隐藏根 View，并触发**onViewCreated** 周期函数。
* 触发 **onActivityCreated** 周期函数，如果根 View 不等于 null 就调用 `f.restoreViewState(f.mSavedFragmentState)` 恢复内部 View 的 SavedState。然后把 `f.mSavedFragmentState` 置空。

**ACTIVITY_CREATED**

该状态表示 Activity 已经完成了它的创建过程，此时 `mState = 2` 。当 `newState > ACTIVITY_CREATED` 的时候把 Fragment 的状态改变为 `STOPPED` (已经完全创建完毕，但是还没有 started)。

**STOPPED**

该状态表示 Fragment 已经完成了创建过程，但是还没有 started，此时 `mState = 3`。当 `newState > Fragment.STOPPED` 的时候触发 **onStart** 周期函数并将 Fragment 的状态改成 `STARTED` (已经创建并且 started 但是还没有 resumed)。

**STARTED**

该状态表示 Fragment 已经创建并且 started 但是还没有 resumed，此时 `mState = 4`。当 `newState > Fragment.STARTED` 触发 **onResume** 周期函数，并将 Fragment 的状态改变为 `RESUMED` (已经创建并且 started和 resumed)。

**RESUMED**

该状态表示 Fragment 已经创建并且 started和 resumed，此时 `mState = 5` 。

**逆过程**

1. 当 Fragment 的状态为 RESUMED 并且 `newState < Fragment.RESUMED` 触发 **onPause** 周期函数，并将 Fragment 的状态改变为 `STARTED` 。
2. 当 Fragment 的状态为 STARTED 并且 `newState < Fragment.STARTED` 触发 **onStop** 周期函数，并将 Fragment 的状态改为 `STOPPED`。
3. 当 Fragment 的状态为 STOPPED 并且 `newState < Fragment.STOPPED` 将 Fragment 状态改为`ACTIVITY_CREATED` 。
4. 当 Fragment 的状态为 ACTIVITY_CREATED 并且 `newState < Fragment.ACTIVITY_CREATED` 的时候
   * 首先判断根 Veiw 是否为空，如果不为空时调用 `mHost.onShouldSaveFragmentState(f)` 方法判断是否应该保持 Fragment 的状态(当 Activity 内存重启时需要保持状态)。如果需要保持就调用 `saveFragmentViewState` 方法进行保存。
   * 触发 **onDestroyView** 周期函数，并将 Fragment 的状态改为 `CREATED`。
   * 执行退出动画。
   * 从 `f.mContainer` (载体 ViewGrouop) 中 remove 掉根 View。
5. 当 Fragment 的状态为 CREATED 并且 `newState < Fragment.CREATED` 当退出动画执行完毕后，判断属性 `f.mRetaining` 当该属性为 false 的时候说明不需要保存 Fragment 实例，触发 **onDestroy** 周期函数，并将 Fragment 的状态改为 `INITIALIZING` 反之说明要保存 Fragment 的实例，直接将 Fragment 的状态改为 `INITIALIZING`。然后触发 **onDetach** 周期函数。如果 `keepActive == false` 并且 `f.mRetaining == false` 调用 `makeInactive(f)` 方法把该 Fragment 从 `mActive` 列表中移除。

### 具体示例分析

分析原理从一件具体的事情上去着手去一步步的 debug 能够产生事半功倍的效果，在 Activity 中添加 Fragment 最典型的步骤就是在 Activity 的 onCreate() 方法中添加如下代码:

```java
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_ordering_bug);
        FragmentManager.enableDebugLogging(true);
        getSupportFragmentManager().beginTransaction()
                .add(R.id.content, a1Fragment)
                .commit();
    }
```

Fragment 是被添加到 FragmentActivity 中的，Activity 的生命周期决定了被添加的 Fragment 的生命周期。所以从 `getSupportFragmentManager()` 点进去进入到了 FragmentActivity 类，那就从这里一步步的分析整个流程。

#### OnCreate()

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
    if (savedInstanceState != null) {
      Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
      mFragments.restoreAllState(p, nc != null ? nc.fragments : null);
      // Check if there are any pending onActivityResult calls to descendent Fragments.
      if (savedInstanceState.containsKey(NEXT_CANDIDATE_REQUEST_INDEX_TAG)) {
        mNextCandidateRequestIndex =
          savedInstanceState.getInt(NEXT_CANDIDATE_REQUEST_INDEX_TAG);
        int[] requestCodes =
          savedInstanceState.getIntArray(ALLOCATED_REQUEST_INDICIES_TAG);
        String[] fragmentWhos =
          savedInstanceState.getStringArray(REQUEST_FRAGMENT_WHO_TAG);
        if (requestCodes == null || fragmentWhos == null ||
            requestCodes.length != fragmentWhos.length) {
                    Log.w(TAG, "Invalid requestCode mapping in savedInstanceState.");
        } else {
          mPendingFragmentActivityResults = new SparseArrayCompat<>(requestCodes.length);
          for (int i = 0; i < requestCodes.length; i++) {
            mPendingFragmentActivityResults.put(requestCodes[i], fragmentWhos[i]);
          }
        }
      }
    }
  mFragments.dispatchCreate();
}
```

 上面是 `onCreate` 中关于 Fragment 生命周期的主要代码，总共干了 3 件事情:

1. 当 `savedInstanceState != null` 的时候说明 Activity 被重建，调用 `restoreAllState` 去执行恢复 Fragment 状态的流程。
2. 当 `savedInstanceState != null` 并且 `savedInstanceState.containsKey(NEXT_CANDIDATE_REQUEST_INDEX_TAG)` 说明 Activity 在被系统销毁的时候还有没传递回来的 `startActivityForResult` 回调，把`requestCode` 和 发起 fragment 的标识保存在 `mPendingFragmentActivityResults` 中。
3. 调用 `dispatchCreate()` 分发 create() 流程。

### mFragments.dispatchCreate()

首先按最简单的情况分析 Activity 没有被重建而是第一次启动，直接调用 `mFragments.dispatchCreate()` 点开 FragmentController 源码发现最终是调用 `FragmentManager.dispatchCreate()` 方法，在 FragmentManager 中调用 `moveToState(Fragment.CREATED, false);` 把该 FragmentManager 实例的状态 `mCurState` 转换到 `Fragment.CREATED` 但是现在还没有执行添加 Fragment 代码所以 `FragmentManager.mActive` 为空。然后再执行子 Activity onCreate() 方法里的添加 Fragment 的代码片段，即发送一个事务消息(添加 Fragment 的代码片段)到主线程消息队列中等待执行(而不是真正意义上执行 add Fragment 的流程)。

### onStart()

当调试到下面添加 Fragment 的流程时发现 `mManager.mCurState=2` 即 FragmentManager 的状态现在处于 `ACTIVITY_CREATED` 即 FragmentActivity 已经执行到 `onStart()` 方法了。因为 `commit()` 是发送一个消息到主线程等待被执行，所以当队列里的消息被执行的时候 FragmentActivity 已经执行到了 `onStart()` 方法，如果使用 `commitNow()` 才是马上执行添加 Fragment 的方法。

```java
// BackStackRecord.executeOps() 部分代码
if (!mAllowOptimization) {
  // Added fragments are added at the end to comply with prior behavior.
  // 此时 mManager.mCurState = 2
  mManager.moveToState(mManager.mCurState, true);
}
```

那我们就来分析一下 onStart() 方法验证前面的猜想。下面是源码：

```java
@Override
protected void onStart() {
  super.onStart();
  
  mStopped = false;
  mReallyStopped = false;
  mHandler.removeMessages(MSG_REALLY_STOPPED);
  
  if (!mCreated) {
    mCreated = true;
    mFragments.dispatchActivityCreated();
  }
  
  mFragments.noteStateNotSaved();
  mFragments.execPendingActions();

  mFragments.doLoaderStart();

  // NOTE: HC onStart goes here.

  mFragments.dispatchStart();
  mFragments.reportLoaderStart();
}
```

可以看出在 onStart() 方法中主要做了下面这几件事情:

1. 调用 `mFragments.dispatchActivityCreated()` 分发 ActivityCreated 的事件，通过 FragmentController 调用 FragmentManager 的 `dispatchActivityCreated()` 方法，在 FragmentManager 中 调用 `moveToState(Fragment.ACTIVITY_CREATED, false)()` 把 FragmentManager 的状态转换到 `ACTIVITY_CREATED` ，所以现在 `FragmentManager.mCurState = 2`。但是现在消息队列中的添加 Fragment 的代码片段还没有被执行。

2. 调用 `mFragments.execPendingActions()` 通过 FragmentController 调用 FragmentManager 的 `execPendingActions()` 方法，开始执行添加 Fragment 的代码片段。(关于这里的具体分析请看[Fragment 那点事之① 栈管理)在 `FragmentManager.moveToState(Fragment f, int newState, int transit, int transitionStyle,boolean keepActive)` 方法中分别触发 `Fragment.onAttach`、

   `Fragment.onCreate`、`Fragment.onCreateView`、`Fragment.onViewCreated`、`Fragment.onActivityCreated` 方法，Fragment 的状态现在为 `ACTIVITY_CREATED` (Activity 已经完成了创建过程) 。

3. 调用 `mFragments.dispatchStart()` 方法，进而执行 `FragmentManager.dispatchStart()` 方法，在 FragmentManager 中执行 `moveToState(Fragment.STARTED, false)` 。把 FragmentManager 的状态转换到 `STARTED` ，现在 `FragmentManager.mCurState = 4` 。该 FragmentManager 管理的 Fragment 的状态会变成 `STOPPED` (即已经完全 created 但是还没有 started 的状态) ，然后紧接着就触发 `Fragment.onStart()` 方法，Fragment 的状态变成 `STARTED` (已经创建并且 started，但是还没有 resumed)。

### onPostResume

```java
@Override
protected void onPostResume() {
  super.onPostResume();
  mHandler.removeMessages(MSG_RESUME_PENDING);
  onResumeFragments();
  mFragments.execPendingActions();
}
```

在这个方法里调用 `onResumeFragments()` 通过 FragmentController 向 Fragments 分发 onResume 事件，调用 `FragmentManager.dispatchResume()` 方法，执行 `moveToState(Fragment.RESUMED, false)` 将 FragmentManager 的状态转换成 `RESUMED` 即 `FragmentManager.mCurState = 5` 。该 FragmentManager 管理的 Fragment 的状态也被转换成了 `RESUMED` ，并触发 `Fragment.onResume` 方法。至此 Fragment 已经 started 并且resumed。

### 其余生命周期函数

其余的生命周期和前面分析的这几个原理基本一样，比如 onPause() 时 Fragment 的状态为 已经完全创建(created)并且 started 但是没有 resumed 所以通过 FragmentController 分发把 FragmentManager 管理下的 Fragments 的状态都转化为 `STARTED` 并且触发 `Fragment.OnPause` 方法。

