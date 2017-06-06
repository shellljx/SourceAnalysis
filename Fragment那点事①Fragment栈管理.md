# Fragment 那点事之① 栈管理

> 文章里所有分析都是根据Android Sdk 25.3.1

在分析栈管理之前先来了解几个基本的概念和 FragmentManager 中主要属性代表什么意思。

### FragmentManagerImpl

`FragmentManagerImpl` 是 `FragmentManager` 的实现类，具体实现了管理 Fragment 的各种操作。

### BackStackRecord

BackStackRecord 实现了 FragmentTransaction 是 FragmentManager 执行操作的事务单元，一次 commit 就会产生一个 BackStackRecord 记录，而在一个 BackStackRecord 记录中有一个 OP 列表。`popBackStack()` 方法也会产生一个相应的 BackStackRecord 记录单元。

### OP

`OP` 是 BackStackRecord 的一个子类，表示一个事务中的一个对 Fragment 具体的操作。里面包含了操作类型、操作对象 Fragment、入栈进入退出动画、出栈进入退出动画。

```java
static final class Op {
  int cmd;
  Fragment fragment;
  int enterAnim;
  int exitAnim;
  int popEnterAnim;
  int popExitAnim;
}
```

具体有下面几种对 Fragment 的操作:

```java
static final int OP_NULL = 0;
static final int OP_ADD = 1;
static final int OP_REPLACE = 2;
static final int OP_REMOVE = 3;
static final int OP_HIDE = 4;
static final int OP_SHOW = 5;
static final int OP_DETACH = 6;
static final int OP_ATTACH = 7;
```

### FragmentManager 变量

- mActive 不仅有活着的 fragments 还有在返回栈中等待被复原的 fragments。

- mAdded 是 `mActive` 的子集表示活着的 fragments

- `ArrayList<OpGenerator>` mPendingActions 待执行动作

- `ArrayList<BackStackRecord>` mTmpRecords
  用于优化执行的临时变量

- `ArrayList<Boolean>` mTmpIsPop
  用于优化执行的临时变量

- `ArrayList<Integer>` mAvailIndices

  mActive 中可用的索引列表

- `ArrayList<BackStackRecord>` mBackStack

  **返回栈** BackStackRecord的返回栈

- `ArrayList <BackStackRecord>` mBackStackIndices
  **返回栈索引列表**，用来给 BackStackRecord 分配 `mIndex` 需要。

- `ArrayList<Integer>` mAvailBackStackIndices
  **返回栈可用索引列表**，存储着 `mBackStackIndices` 列表中值为 null 的下标即可以插入 `BackStackRecord` 。

### 流程图

![流程图](http://7vzpfd.com1.z0.glb.clouddn.com/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.png)

下面是最基本的 add Fragment 的代码片段:

```java
getSupportFragmentManager().beginTransaction()
  .add(R.id.content, fragment)
  //.remove(fragment)
  //.replace(R.id.content,fragment2)
  .commit();
```

通过 `beginTransaction()` 获取一个 FragmentTransaction 然后调用 `add()` 方法，其实不管是 `add()` 还是 `remove()`，`replace()` 等方法，最后都是调用的 `doAddOp()` 方法添加一个 OP 对象到 `BackStackRecord.mOps` 列表中。然后再调用 `commit()` 提交该事务。

### BackStackRecord.commitInternal()

`commit()` 方法调用 `commitInternal()` 方法，在该方法中用来给每个 BackStackRecord 事务分配 `mIndex` 标识。如果该事务不需要加入返回栈就分配 -1 否则就调用 `mManager.allocBackStackIndex(this)` 返回标识。

```java
int commitInternal(boolean allowStateLoss) {
  if (mCommitted) throw new IllegalStateException("commit already called");
  mCommitted = true;
  if (mAddToBackStack) {
    mIndex = mManager.allocBackStackIndex(this);
  } else {
    mIndex = -1;
  }
  mManager.enqueueAction(this, allowStateLoss);
  return mIndex;
}
```

### BackStackRecord.mIndex 的管理

**FragmentManager.allocBackStackIndex()**

该方法用于给加入到返回栈的 BackStackRecord 分配 `mIndex` 下标，分配方法不是持续的递增而是使用 `mBackStackIndices` 和 `mAvailBackStackIndices` 配合避免频繁为索引列表开辟新的内存空间。从代码片段中可知，当 `mAvailBackStackIndices` 为空时说明没有可用的下标，就直接返回 `mBackStackIndices.size()` 作为下标并把该 BackStackRecord 添加到 `mBackStackIndices` 中，反之返回 `mAvailBackStackIndices` 末尾值作为下标并将 BackStackRecord 添加到该位置。

```java
public int allocBackStackIndex(BackStackRecord bse) {
  synchronized (this) {
    if (mAvailBackStackIndices == null || mAvailBackStackIndices.size() <= 0) {
      if (mBackStackIndices == null) {
        mBackStackIndices = new ArrayList<BackStackRecord>();
      }
      int index = mBackStackIndices.size();
      if (DEBUG) Log.v(TAG, "Setting back stack index " + index + " to " + bse);
      mBackStackIndices.add(bse);
      return index;
    } else {
      int index = mAvailBackStackIndices.remove(mAvailBackStackIndices.size()-1);
      if (DEBUG) Log.v(TAG, "Adding back stack index " + index + " with " + bse);
      mBackStackIndices.set(index, bse);
      return index;
    }
  }
}
```

**FragmentManager.freeBackStackIndex()**

该方法是出栈的时候调用的，当 BackStackRecord 出栈后来更新返回栈可用索引列表的。由下面的代码可知，是吧相应位置设置成 null，然后把该下标加入可用返回栈列表的列尾。

```java
public void freeBackStackIndex(int index) {
  synchronized (this) {
    mBackStackIndices.set(index, null);
    if (mAvailBackStackIndices == null) {
      mAvailBackStackIndices = new ArrayList<Integer>();
    }
    if (DEBUG) Log.v(TAG, "Freeing back stack index " + index);
    mAvailBackStackIndices.add(index);
  }
}
```

**FragmentManager.setBackStackIndex()**

该方法是在 `restoreAllState()` 方法中调用的，用来处理当 Activity 意外内存重启后恢复 Fragment 返回栈索引列表(mBackStackIndices)和可用索引列表(mAvailBackStackIndices)的。当传进来的 index 大于 N 时，遍历从 N 开始到 index 的位置 add(null) 并把下标添加到 `mAvailBackStackIndices` 中，反之直接把 BackStackRecord 填充到 index 位置。

```java
public void setBackStackIndex(int index, BackStackRecord bse) {
  synchronized (this) {
    if (mBackStackIndices == null) {
      mBackStackIndices = new ArrayList<BackStackRecord>();
    }
    int N = mBackStackIndices.size();
    if (index < N) {
      if (DEBUG) Log.v(TAG, "Setting back stack index " + index + " to " + bse);
      mBackStackIndices.set(index, bse);
    } else {
      while (N < index) {
        mBackStackIndices.add(null);
        if (mAvailBackStackIndices == null) {
          mAvailBackStackIndices = new ArrayList<Integer>();
        }
        if (DEBUG) Log.v(TAG, "Adding available back stack index " + N);
        mAvailBackStackIndices.add(N);
        N++;
      }
      if (DEBUG) Log.v(TAG, "Adding back stack index " + index + " with " + bse);
      mBackStackIndices.add(bse);
    }
  }
}
```

### FragmentManager.execPendingActions()

在 `commitInternal()` 中分配完 `mIndex` 标识后调用 `mManager.enqueueAction(this, allowStateLoss)` 把 BackStackRecord 添加到 `mPendingActions` 待执行动作列表中，然后通过 Handler 发送 Runnable 消息到主线程消息队列中等待被执行。

```java
Runnable mExecCommit = new Runnable() {
  @Override
  public void run() {
    execPendingActions();
  }
};
```

在 Runnable 中执行 `execPendingActions()` 方法，该方法代码片段如下。分为几点来分析:

1. `mTmpRecords`，` mTmpIsPop` 前者用来临时存储所有待执行的动作(mPendingActions)生成的 BackStackRecord，后者用来存储 BackStackRecord 是否为出栈。
2. `generateOpsForPendingActions` 方法遍历 `mPendingActions` 调用 `OpGenerator.generateOps()` 方法生成 BackStackRecord 添加到 `mTmpRecords` 并把是否为出栈添加到 `mTmpIsPop` 中，然后调用 `mPendingActions.clear()` 把待执行动作列表清空，因为已经被转化为 BackStackRecord 列表了。
3. 调用 `optimizeAndExecuteOps(mTmpRecords, mTmpIsPop)` 方法优化并执行 BackStackRecord 列表。
4. 最后调用 `cleanupExec()` 清空列表。

```java
/**
* Only call from main thread!
*/
public boolean execPendingActions() {
  ensureExecReady(true);

  boolean didSomething = false;
  while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
    mExecutingActions = true;
    try {
      optimizeAndExecuteOps(mTmpRecords, mTmpIsPop);
    } finally {
      cleanupExec();
    }
    didSomething = true;
  }
  doPendingDeferredStart();
  return didSomething;
}
```

 ### OpGenerator.generateOps() 

关于该接口的实现有两个地方，即 `BackStackRecord.generateOps()` 和 `PopBackStackState.generateOps()` 两处。前者为 非 pop 出栈，对应 `commit()` 操作提交的各种类型 BackStackRecord，后者为 pop 出栈，对应 `popBackStack()` 等方法提交的 PopBackStackState。由下面 BackStackRecord.generateOps() 的实现可知，如果设置了 `addToBackStack` 是在这里把 BackStackRecord 添加到返回栈 `mBackStack` 中去的。

```java
//BackStackRecord.generateOps()
@Override
public boolean generateOps(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop) {
  if (FragmentManagerImpl.DEBUG) {
    Log.v(TAG, "Run: " + this);
  }

  records.add(this);
  isRecordPop.add(false);
  if (mAddToBackStack) {
    mManager.addBackStackState(this);
  }
  return true;
}
```

 PopBackStackState.generateOps() 源码先不分析，等到出栈的时候再详细分析。

### FragmentManager.optimizeAndExecuteOps()

该方法合并相邻的允许被优化(`mAllowOptimization = true`)的 BackStackRecord 。比如有一个 Transaction 添加到了返回栈，然后另一个 Transaction 是要把前面的 Transaction pop 出栈，这两个 Transaction 将会被优化。

比如下面的代码片段，不允许优化时 a1Fragment 被 add 进 FragmentManager，走一遍 Fragment 的生命周期，然后 popBackStack() 再把 Fragment remove 掉。而优化后不会再执行 Fragment 的生命周期就被直接 remove 掉。

```java
getSupportFragmentManager().beginTransaction()
  .add(R.id.content, a1Fragment)
  .setAllowOptimization(true)
  .addToBackStack("a1")
  .commit();
getSupportFragmentManager().popBackStack();
```

### FragmentManager.executeOpsTogether()

在该方法中会执行 `expandReplaceOps` 方法把 replace 替换(目标 fragment 已经被 add )成相应的 remove 和 add 两个操作，或者(目标 fragment 没有被 add )只替换成 add 操作。

```java
if (!isPop) {
   record.expandReplaceOps(mTmpAddedFragments);
} else {
   record.trackAddedFragmentsInPop(mTmpAddedFragments);
}
```

然后调用 `executeOps` 方法，根据不同 Op.cmd 调用不同的方法对 fragment 进行操作。我们就先来分析 add 操作的流程。会调用 `mManager.addFragment(f, false)` 方法。在 addFragment 方法中先执行 `makeActive` 方法，把 add 进来的 fragment 添加到 `mActive` 列表中，看这个分配 index 的步骤是不是很熟悉？没错和给 BackStackRecord 分配 index 的步骤非常像，即移除 fragment 的时候是把相应下标处置 null，然后把该下标保存在 `mAvailIndices` 列表中，添加 fragment 的时候就会去这个列表中取出最后一个下标。在 SDK23 版本时因为就因为这种分配方法导致了一个 bug ：①添加 A1,B1,C1 三个Fragment。② remove 掉这三个 fragment 并且 add 进去 a2,b2 两个 fragment，期望是要显示 b2 但是实际情况确实显示的 a2。在 Sdk >= 24 中这个问题已经被修复。

```java
void makeActive(Fragment f) {
    if (f.mIndex >= 0) {
        return;
    }

    if (mAvailIndices == null || mAvailIndices.size() <= 0) {
        if (mActive == null) {
            mActive = new ArrayList<Fragment>();
        }
        f.setIndex(mActive.size(), mParent);
        mActive.add(f);

    } else {
        f.setIndex(mAvailIndices.remove(mAvailIndices.size()-1), mParent);
        mActive.set(f.mIndex, f);
    }
    if (DEBUG) Log.v(TAG, "Allocated fragment index " + f);
}
```

然后当 fragment 没有 Detached 的时候执行下面逻辑，由此可见 `mAdded` 是 `mActive` 的一个子集，没有detached 的 fragment 才会被加入到 `mAdded` 。

```
if (!fragment.mDetached) {
    if (mAdded.contains(fragment)) {
        throw new IllegalStateException("Fragment already added: " + fragment);
    }
    mAdded.add(fragment);
    ...        
}
```

然后执行 moveToState 方法改变 fragment 的状态(添加 fragment 的不同情况会有所不同，但最后都是要 moveToState 改变 fragment 的状态的，进而触发 fragment 的生命周期函数) 。

```java
moveToState(Fragment f, int newState, int transit, int transitionStyle,
            boolean keepActive)
```

在moveToState() 方法中有下面这段代码，我查看了 SDK 23 的 moveToState() 方法的源码是遍历的 mActive列表里的 fragments，而现在是遍历的 mAdded，因为由上面提到的bug可知 mActive fragments 有可能已经乱序了所以在 SDK 25里是不会出现乱序的这个 bug 了。

```java
// Must add them in the proper order. mActive fragments may be out of order
if (mAdded != null) {
    final int numAdded = mAdded.size();
    for (int i = 0; i < numAdded; i++) {
        Fragment f = mAdded.get(i);
        moveFragmentToExpectedState(f);
        if (f.mLoaderManager != null) {
            loadersRunning |= f.mLoaderManager.hasRunningLoaders();
        }
    }
}
```

## BackStackRecord 出栈

BackStackRecord 出栈的方法有如下几个：

- popBackStack()
- popBackStackImmediate()
- popBackStack(int id/String name, int flags)
- popBackStackImmediate(int id/String name, int flags)

#### popBackStack()

PopBackStackState 实现了 OpGenerator 接口,具体实现如下：

- 参数 records 用来存放出栈的 BackStackRecord 
- 参数 isRecordPop 用来存放相应 BackStackRecord 是否为出栈(显然为 true)
- 参数 name 表示出栈到相应 name 的 BackStackRecord
- 参数 id 表示出栈到相应 id 的 BackStackRecord
- 参数 flags (0 或者 POP_BACK_STACK_INCLUSIVE)
- POP_BACK_STACK_INCLUSIVE 如果参数 `flags ==POP_BACK_STACK_INCLUSIVE` 并且设置了 name 或者 id 那么，所有符合该 name 或者 id 的 BackStackRecord 都将被匹配，直到遇到一个不匹配的或者到达了栈底，然后出栈所有 BackStackRecord 直到最终匹配到的下标位置。否则只匹配第一次 name 或者 id 相符的 BackStackRecord，然后出栈所有 BackStackRecord 直到但不包括匹配到的下标位置。

```java
    boolean popBackStackState(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop,
            String name, int id, int flags) {
        if (mBackStack == null) {
            return false;
        }
        if (name == null && id < 0 && (flags & POP_BACK_STACK_INCLUSIVE) == 0) {
            int last = mBackStack.size() - 1;
            if (last < 0) {
                return false;
            }
            records.add(mBackStack.remove(last));
            isRecordPop.add(true);
        } else {
            int index = -1;
            if (name != null || id >= 0) {
                // If a name or ID is specified, look for that place in
                // the stack.
                index = mBackStack.size()-1;
                while (index >= 0) {
                    BackStackRecord bss = mBackStack.get(index);
                    if (name != null && name.equals(bss.getName())) {
                        break;
                    }
                    if (id >= 0 && id == bss.mIndex) {
                        break;
                    }
                    index--;
                }
                if (index < 0) {
                    return false;
                }
                // flags&POP_BACK_STACK_INCLUSIVE 位与运算 
                if ((flags&POP_BACK_STACK_INCLUSIVE) != 0) {
                    index--;
                    // Consume all following entries that match.
                    while (index >= 0) {
                        BackStackRecord bss = mBackStack.get(index);
                        if ((name != null && name.equals(bss.getName()))
                                || (id >= 0 && id == bss.mIndex)) {
                            index--;
                            continue;
                        }
                        break;
                    }
                }
            }
            if (index == mBackStack.size()-1) {
                return false;
            }
            for (int i = mBackStack.size() - 1; i > index; i--) {
                records.add(mBackStack.remove(i));
                isRecordPop.add(true);
            }
        }
        return true;
    }
```

如果 返回栈 mBackStack 为空就终止出栈操作并返回 false，当` name == null && id < 0 && (flags & POP_BACK_STACK_INCLUSIVE) == 0` (调用的是popBackStack()方法)时，把返回栈最后一个 BackStackRecord出栈。当 name 或者 id 被指定的时候，倒序遍历 `mBackStack` ，如果遇到 name 或者 id 相符就退出循环，此时 index 为第一次匹配到的下标，如果`flags==POP_BACK_STACK_INCLUSIVE` 继续遍历返回栈，直至栈底或者遇到不匹配的跳出循环。最后出栈所有 BackStackRecord。

popBackStack() 方法调用 enqueueAction 方法，添加出栈动作操作到队列中，这样又返回到 BackStackRecord 入栈流程那里的 generateOpsForPendingActions() 方法，分别调用 BackStackRecord.generateOps()，PopBackStackState.generateOps() 实现。