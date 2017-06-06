# Fragment那点事③保存与恢复

> 文章里所有分析都是根据Android Sdk 25.3.1 v4包

上一篇分析了最典型的情况下添加 Fragment 的生命周期，现在再来分析一下特殊情况下 Fragment 的生命周期有什么不同。

### onSaveInstanceState

当 Activity 被系统内存杀死后会调用 `onSaveInstanceState` 自动保存一些状态，其中就包括托管在该 Activity 内的 Fragment 的状态。

```java
@Override
protected void onSaveInstanceState(Bundle outState) {
  super.onSaveInstanceState(outState);
  Parcelable p = mFragments.saveAllState();
  if (p != null) {
    outState.putParcelable(FRAGMENTS_TAG, p);
  }
  if (mPendingFragmentActivityResults.size() > 0) {
    outState.putInt(NEXT_CANDIDATE_REQUEST_INDEX_TAG, mNextCandidateRequestIndex);
    int[] requestCodes = new int[mPendingFragmentActivityResults.size()];
    String[] fragmentWhos = new String[mPendingFragmentActivityResults.size()];
    for (int i = 0; i < mPendingFragmentActivityResults.size(); i++) {
      requestCodes[i] = mPendingFragmentActivityResults.keyAt(i);
      fragmentWhos[i] = mPendingFragmentActivityResults.valueAt(i);
    }
    outState.putIntArray(ALLOCATED_REQUEST_INDICIES_TAG, requestCodes);
    outState.putStringArray(REQUEST_FRAGMENT_WHO_TAG, fragmentWhos);
  }
}
```

在 FragmentActivity 的 onSaveInstanceState 方法中主要做了如下几件事情：

1. 调用 `mFragments.saveAllState()` 通过 FragmentController 执行 `FragmentManager.saveAllState()` 方法，初始化一个 FragmentManagerState 实例，并且得到 `mActive` 的 FragmentState 数组、 `mAdded` 的 Fragment 的 下标(mIndex)数组和返回栈的 BackStackState 数组。从这可以看出 saveInstanceState 保存的不是 Fragment 的实例，而是 Fragment 和 和返回栈的状态。
2. 当有没有传递回来的 startActivityForResult 事件的时候就把相应 requestCode 和源 `fragment.mWho` 保存。

### FragmentManager.saveAllState()

在 saveAllState() 方法中是保存并返回 FragmentManager 状态的逻辑，主要分为 3 部分:

```java
// First collect all active fragments.
int N = mActive.size();
FragmentState[] active = new FragmentState[N];
boolean haveFragments = false;
for (int i=0; i<N; i++) {
  Fragment f = mActive.get(i);
  if (f != null) {
    if (f.mIndex < 0) {
      throwException(new IllegalStateException(
        "Failure saving state: active " + f
        + " has cleared index: " + f.mIndex));
    }
    haveFragments = true;

    FragmentState fs = new FragmentState(f);
    active[i] = fs;

    if (f.mState > Fragment.INITIALIZING && fs.mSavedFragmentState == null) {
      fs.mSavedFragmentState = saveFragmentBasicState(f);

      if (f.mTarget != null) {
        if (f.mTarget.mIndex < 0) {
          throwException(new IllegalStateException(
            "Failure saving state: " + f
            + " has target not in fragment manager: " + f.mTarget));
        }
        if (fs.mSavedFragmentState == null) {
          fs.mSavedFragmentState = new Bundle();
        }
        putFragment(fs.mSavedFragmentState,
                    FragmentManagerImpl.TARGET_STATE_TAG, f.mTarget);
        if (f.mTargetRequestCode != 0) {
          fs.mSavedFragmentState.putInt(
            FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG,
            f.mTargetRequestCode);
        }
      }
    } else {
      fs.mSavedFragmentState = f.mSavedFragmentState;
    }
    
    if (DEBUG) Log.v(TAG, "Saved state of " + f + ": "
                     + fs.mSavedFragmentState);
  }
}
```

1. **得到 mActive 的 FragmentState 数组**

   遍历 `mActive` 中的 Fragment(还记得前面分析出栈的时候，是吧 mActive 中相应下标处设置成 null，所以得到的 FragmentState 数组相应下标处也为 null)

   - 如果 `f.mIndex<0` (Fragment 在 mActive 中的下标)说明该 Fragment 不在 FragmentManager 的管理之内，无法被保存状态。
   - 然后以 Fragment 为参实例化一个 FragmentState 并把一些基础属性赋值。
   - 当 Fragment 已经初始化并且状态对象为空就调用 `saveFragmentBasicState()` 方法，在该方法中首先触发 `onSaveInstanceState` 可以让开发者保存一些自定义的属性到 Bundle 中，然后调用 `saveHierarchyState` 方法保存 Fragment 内 View 的状态(只有部分 View 比如 TextView 实现了 View 的 onSaveInstanceState 方法)到 Bundle 中。保存 Fragment 的 `mUserVisibleHint` 属性到 Bundle 中，把该 Bundle 赋值给 FragmentState.mSavedFragmentState。如果有 Target fragment 就把 `mIndex` 和 `mTargetRequestCode` 保存到 mSavedFragmentState 属性中。

2. **得到 mAdded 中 Fragment 的 mIndex 列表**

   `Fragment.mIndex` 是在 `mActive` 中的下标，在前面分析入栈的时候，在 `makeActive()` 方法时设置的。 

3. **得到 mBackStackState 返回栈状态信息列表**

### restoreAllState()

在 FragmentActivity.onCreate() 方法中，如果`savedInstanceState!=null` 就调用 `mFragments.restoreAllState` 方法，恢复保存的 Fragment 。

```java
NonConfigurationInstances nc =
  (NonConfigurationInstances) getLastNonConfigurationInstance();
if (nc != null) {
  mFragments.restoreLoaderNonConfig(nc.loaders);
}
if (savedInstanceState != null) {
  Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
  mFragments.restoreAllState(p, nc != null ? nc.fragments : null);
}
```

在上面的代码片段中还有一句 `getLastNonConfigurationInstance()` 这个是 FragmentActivity 用来保存 `mRetainInstance = true` 的 Fragment 实例对象和 Loader 实例的，这是和 saveInstanceState 不同的另一种保存机制，等下面具体分析。现在就先默认 `nc=null`，只分析 saveInstanceState 。

### FragmentManager.restoreAllState()

最终恢复 Fragment 的具体操作是在 restoreAllState() 方法中完成的，下面分段分析一下主要逻辑分别作了什么:

1. 如果 saved state 为空或者 `mActive` 为空就终止往下执行。

   ```java
   // If there is no saved state at all, then there can not be
   // any nonConfig fragments either, so that is that.
   if (state == null) return;
   FragmentManagerState fms = (FragmentManagerState)state;
   if (fms.mActive == null) return;

   List<FragmentManagerNonConfig> childNonConfigs = null;
   ```

2. 调用 `FragmentState.instantiate()` 方法恢复 `mActive` 列表，可以点进去该方法查看到底恢复了 Fragment 的哪些属性(恢复了 mHidden，mIndex 等等)。`mActive` 包括所有被 FragmentManager 管理的 Fragment。并且恢复 `mAvailIndices` 列表，这里面存放着在 `mActive` 中值为 null 的下标，即可以被插入 Fragment 的位置。

   ```java
   // Build the full list of active fragments, instantiating them from
   // their saved state.
   mActive = new ArrayList<>(fms.mActive.length);
   if (mAvailIndices != null) {
     mAvailIndices.clear();
    }
   for (int i=0; i<fms.mActive.length; i++) {
     FragmentState fs = fms.mActive[i];
     if (fs != null) {
       FragmentManagerNonConfig childNonConfig = null;
       if (childNonConfigs != null && i < childNonConfigs.size()) {
         childNonConfig = childNonConfigs.get(i);
       }
       Fragment f = fs.instantiate(mHost, mParent, childNonConfig);
       if (DEBUG) Log.v(TAG, "restoreAllState: active #" + i + ": " + f);
       mActive.add(f);
       // Now that the fragment is instantiated (or came from being
       // retained above), clear mInstance in case we end up re-restoring
       // from this FragmentState again.
       fs.mInstance = null;
     } else {
       mActive.add(null);
       if (mAvailIndices == null) {
         mAvailIndices = new ArrayList<Integer>();
       }
       if (DEBUG) Log.v(TAG, "restoreAllState: avail #" + i);
       mAvailIndices.add(i);
     }
   }
   ```

3. 恢复 `mAdded` 列表，根据 `fms.mAdded` 中保存的下标位置从 `mActive` 中得到已经被 add 的 Fragments。

   ```java
   // Build the list of currently added fragments.
   if (fms.mAdded != null) {
     mAdded = new ArrayList<Fragment>(fms.mAdded.length);
     for (int i=0; i<fms.mAdded.length; i++) {
       Fragment f = mActive.get(fms.mAdded[i]);
       if (f == null) {
         throwException(new IllegalStateException(
           "No instantiated fragment for index #" + fms.mAdded[i]));
       }
       f.mAdded = true;
       if (DEBUG) Log.v(TAG, "restoreAllState: added #" + i + ": " + f);
       if (mAdded.contains(f)) {
         throw new IllegalStateException("Already added!");
       }
       mAdded.add(f);
     }
   } else {
     mAdded = null;
   }
   ```

4. 恢复返回栈 `mBackStack` 调用 `setBackStackIndex()` 方法恢复 `mBackStackIndices` 和 `mAvailBackStackIndices` 这两个列表是用来分配 `BackStackRecord.mIndex` 的，相应流程在前面入栈的时候已经分析过。

   ```java
   // Build the back stack.
   if (fms.mBackStack != null) {
     mBackStack = new ArrayList<BackStackRecord>(fms.mBackStack.length);
     for (int i=0; i<fms.mBackStack.length; i++) {
       BackStackRecord bse = fms.mBackStack[i].instantiate(this);
       if (DEBUG) {
         Log.v(TAG, "restoreAllState: back stack #" + i
               + " (index " + bse.mIndex + "): " + bse);
         LogWriter logw = new LogWriter(TAG);
         PrintWriter pw = new PrintWriter(logw);
         bse.dump("  ", pw, false);
         pw.close();
       }
       mBackStack.add(bse);
       if (bse.mIndex >= 0) {
         setBackStackIndex(bse.mIndex, bse);
       }
     }
   } else {
     mBackStack = null;
   }
   ```

在 FragmentActivity.onCreate() 中恢复玩 FragmentManager 里的所有 Fragment ，接着调用 `mFragments.dispatchCreate()` 开始执行这些 Fragment 的生命周期。到此第一中保存恢复 Fragment 的方式分析完了，这种方式保存的是 FragmentManager 中所有 Fragment 的状态，而不是保存的实例，然后在 FragmentActivity.onCreate() 方法中再去根据这些状态去恢复 Fragment，在子 Activity 中相应的作出一些判断当内存重启的时候就可以避免重复的去添加 Fragment。

### RetainInstance

与之前保存 SaveState 不同，这种保存方式是当 Activity 被重启的时候通过 `onRetainNonConfigurationInstance()` 方法保存 Object。

**onRetainNonConfigurationInstance()**

这是一个 final 类型的方法，开发者无法 Override，FragmentActivity 里有默认的实现，返回的是 NonConfigurationInstances 对象。该对象主要包含 3 部分:

1. 调用 `onRetainCustomNonConfigurationInstance()` 所以开发者要想保存一个 Object 可以重写该方法。
2. 获取 FragmentManagerNonConfig 实例，主要逻辑是在 `FragmentManager.retainNonConfig()` 方法中实现的。遍历 `mActive` 中所有的 Fragment。
   * 添加 `mRetainInstance = true` 的 Fragment 到 FragmentManagerNonConfig.mFragments 列表，并设置 `Fragment.mRetaining = true` 来标识该 Fragment 已经被保留了实例。 
   * 调用 `Fragment.mChildFragmentManager.retainNonConfig()` 获取ChildFragmentManager 的 FragmentManagerNonConfig，并添加到FragmentManagerNonConfig.mFragments.mChildNonConfigs 列表中(有的 Fragment 里并没有直接嵌套 Fragment，所以该 ChildFragmentManager 的 FragmentManagerNonConfig 为 null)。
3. 保存 Loader 实例。

```java
@Override
public final Object onRetainNonConfigurationInstance() {
  if (mStopped) {
    doReallyStop(true);
  }
  
  Object custom = onRetainCustomNonConfigurationInstance();

  FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();
  SimpleArrayMap<String, LoaderManager> loaders = mFragments.retainLoaderNonConfig();

  if (fragments == null && loaders == null && custom == null) {
    return null;
  }
  
  NonConfigurationInstances nci = new NonConfigurationInstances();
  nci.custom = custom;
  nci.fragments = fragments;
  nci.loaders = loaders;
  return nci;
}
```

**getLastNonConfigurationInstance()**

在 FragmentActivity.onCreate() 中调用该方法得到 Activity 重建前被保存的 NonConfigurationInstances 实例，然后调用把 savedInstanceState 和 FragmentManagerNonConfig 传入 `FragmentManager.restoreAllState` 方法中，之前只分析了 SaveInstance 这一种情况，现在把这两种情况一起分析一下。

```java
// First re-attach any non-config instances we are retaining back
// to their saved state, so we don't try to instantiate them again.
if (nonConfig != null) {
  List<Fragment> nonConfigFragments = nonConfig.getFragments();
  childNonConfigs = nonConfig.getChildNonConfigs();
  final int count = nonConfigFragments != null ? nonConfigFragments.size() : 0;
  for (int i = 0; i < count; i++) {
    Fragment f = nonConfigFragments.get(i);
    if (DEBUG) Log.v(TAG, "restoreAllState: re-attaching retained " + f);
    FragmentState fs = fms.mActive[f.mIndex];
    fs.mInstance = f;
    f.mSavedViewState = null;
    f.mBackStackNesting = 0;
    f.mInLayout = false;
    f.mAdded = false;
    f.mTarget = null;
    if (fs.mSavedFragmentState != null) {
      fs.mSavedFragmentState.setClassLoader(mHost.getContext().getClassLoader());
      f.mSavedViewState = fs.mSavedFragmentState.getSparseParcelableArray(
      FragmentManagerImpl.VIEW_STATE_TAG);
      f.mSavedFragmentState = fs.mSavedFragmentState;
    }
  }
}
```

在上面代码片段中可以得知系统把保存的 Fragment 实例设置给了它的 `FragmentState.mInstance` 并把 `mAdded`、`mInLayout` 等属性设置成 false，而 Fragment 内部 View 的状态用的是 FragmentState 来更新的(分析 `FragmentManager.saveAllState()` 的时候讲过)。

在 `FragmentManager.saveAllState()` 方法中恢复 `mActive` 列表时调用下面的方法，点进去发现`Fragment.mInstance != null` 的时候直接就返回了 `mInstance` 这不正是之前保存的 Fragment 对象嘛，而不是由 SavedState 来初始化的。 

```java
Fragment f = fs.instantiate(mHost, mParent, childNonConfig);
```

那再来看一下被保留下来的 Fragment 即 `Fragment.mRetaining = true` 实例的生命周期

```java
if (!f.mRetaining) {
  f.performCreate(f.mSavedFragmentState);
  dispatchOnFragmentCreated(f, f.mSavedFragmentState, false);
} else {
  f.restoreChildFragmentState(f.mSavedFragmentState);
  f.mState = Fragment.CREATED;
}
f.mRetaining = false;
```

```java
if (!f.mRetaining) {
  f.performDestroy();
  dispatchOnFragmentDestroyed(f, false);
} else {
  f.mState = Fragment.INITIALIZING;
}
```

发现，当被保存下来的 Fragment 被 add 进 Activity 中的时候，并不走 `onCreate` 周期函数，而是直接调用 `restoreChildFragmentState` 去恢复 ChildFragmentManager。所以可以在 `onCreate()` 方法中初始化对象避免重复初始化。当 Activity 因重启被销毁的时候 Fragment 也不会走 `onDestroy()` 方法。

*Fragment 保存于恢复分析完毕*