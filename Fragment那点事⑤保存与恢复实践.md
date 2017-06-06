# Fragment那点事③保存与恢复实践

> 文章里所有分析都是根据Android Sdk 25.3.1 v4包

**经过前面的分析之后现在有这么一个需求:**

1. APP 总共 3 个 Tab，首页(HomePageFragment)，分类(CategoryFragment)，我的(MineFragment)这 3 个 Tab 为第一级。
2. TopicListFragment 根据穿件来不同的 tab 字段可以分别加载首页、分类(热门，精华，分享，招聘)页面的帖子列表。
3. 要实现 Activity 被重建的时候所有帖子列表页面维持不变，不重新加载新的数据、帖子列表懒加载。
4. Activity 重建用屏幕旋转来模拟。

**效果图**

![](http://7vzpfd.com1.z0.glb.clouddn.com/BD3D9172-D100-49F0-8DC0-D4A6E8FF398A.png)

通过分析可以发现，最外层分别为 首页，分类，我的 3 个 Fragment。在首页嵌套一个 TopicListFragment 到 HomePageFragment 里去，在分类页面嵌套一个 ViewPager，ViewPager 的每一页都是 TopicListFragment。

### MainActivity

根据之前分析的 onSaveInstanceState 在该方法中保存 `currPosition` 当前可见的 Tab id。

```java
@Override
public void onSaveInstanceState(Bundle outState) {
  super.onSaveInstanceState(outState);
  outState.putInt("currPosition", mCurrPosition);
}
```

MainActivity.onCreate() 方法中做如下处理，判断 `savedInstanceState == null` 。

* 如果为真，说明没有发生重启，直接初始化 3 个Fragment 实例并 add 进 Activity 中。
* 如果为假，说明 FragmentActivity 已经帮我们做好了 Fragment 的保存与恢复工作，不用再重复的去添加新的实例到 FragmentManager，直接去 FragmentManager 中去获取这 3 个 Fragment 的实例(在外部获取实例在切换 tab 的时候需要)。

```java
@Override
public void initView(Bundle savedInstanceState) {
  
  if (savedInstanceState == null) {
    mFragments = new BaseFragment[3];
    mFragments[0] = HomePageFragment.newInstance();
    mFragments[1] = CategoryFragment.newInstance();
    mFragments[2] = MineFragment.newInstance();
    FragmentUtils.addMultiple(getSupportFragmentManager(), R.id.content, mCurrPosition, mFragments);
  } else {
    mCurrPosition = savedInstanceState.getInt("currPosition");
    mFragments[0] = findFragment(HomePageFragment.class);
    mFragments[1] = findFragment(CategoryFragment.class);
    mFragments[2] = findFragment(MineFragment.class);

    if (mCurrPosition != 0) {
      updateNavigationBarState(mCurrPosition);
    }
  }
}
```

下面是 `FragmentUtils.addMultiPle()` 方法，添加多个 Fragment 到 content 并设置显示和隐藏。并调用 `Fragment.setUserVisibleHint` 更新 mUserVisibleHint 属性，在 Fragment 中默认该属性为 true。

```java
public static void addMultiple(FragmentManager manager, int containerId, int showPosition, BaseFragment... fragments) {
  FragmentTransaction transaction = manager.beginTransaction();
  for (int i = 0; i < fragments.length; i++) {
    String tag = fragments[i].getClass().getName();
    transaction.add(containerId, fragments[i], tag);
    if (showPosition != i) {
      transaction.hide(fragments[i]);
    }
    fragments[i].setUserVisibleHint(!fragments[i].isHidden());
  }
  transaction.commit();
}
```

下面是 `FragmentUtils.FindFragment()` 方法，根据添加 Fragment 的时候设置的 Tag，可以获取 Fragment。

```java
public static <T extends BaseFragment> T findFragment(FragmentManager manager, Class<T> tClass) {
  if (manager.getFragments() == null) {
    return null;
  }
  return (T) manager.findFragmentByTag(tClass.getName());
}
```

切换 tab 的时候 show 要显示的 Fragment，其余 Fragment 被 hide，因为 show 和 hide 并不会改变 `mUserVisibleHint` 所以要手动调用 setUserVisibleHint() 。

```java
public static void showHideFragment(FragmentManager manager, Fragment show, Fragment hide, boolean animation, boolean backStack) {
  FragmentTransaction transaction = manager.beginTransaction();
  if (animation) {
    transaction.setCustomAnimations(
      R.anim.fragment_translate_in, R.anim.fragment_translate_out
      ,R.anim.fragment_pop_in,R.anim.fragment_pop_out);
  }
  transaction.show(show);
  if (hide == null) {
    List<Fragment> fragments = manager.getFragments();
    if (fragments != null) {
      for (Fragment fragment : fragments) {
        if (fragment != show) {
          transaction.hide(fragment);
        }
      }
    }
  } else {
    transaction.hide(hide);
  }
  if (backStack) {
    transaction.addToBackStack("showHideFragment");
  }
  transaction.commit();
}
```

### HomePageFragment

在 HomePageFragment.onViewCreated() 方法中的处理和 MainActivity.onCreate() 方法中相似。这个时候 ChildFragmentManager 已经恢复完毕，调用 `findFragmentByTag` 查看是否包含 TopicListFragment，如果不包含就重新实例化添加。

```java
@Override
public void initView(View root) {
  if (getChildFragmentManager().findFragmentByTag(TopicListFragment.class.getName()) == null) {
    TopicListFragment fragment = TopicListFragment.instance(null);
    fragment.setUserVisibleHint(true);
    FragmentUtils.replace(getChildFragmentManager(), R.id.home_page_content, fragment, false, TopicListFragment.class.getName());
  }
}
```

### TopicListFragment

在 onCreate() 方法中设置 `setRetainInstance(true)` 由之前的分析可知，当 Activity 发生重启后，该 Fragment 实例会被保存下来，已经获取的帖子数据也不用重新请求，直接通过 RecyclerView 的自我恢复重新填充。

```java
@Override
public void onCreate(@Nullable Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  setRetainInstance(true);
}
```

### CategoryFragment

在 CategoryFragment 中的布局是一个 ViewPager，在 ViewPager 中实现了 `View.onSaveInstanceState()` 这个方法，在之前介绍 Fragment 保存与恢复中(Fragment 保存内部 View 状态的流程)讲过，部分 View 实现了该方法，可以自行保存状态。那就看看 ViewPager 保存了什么东西

```java
@Override
public Parcelable onSaveInstanceState() {
  Parcelable superState = super.onSaveInstanceState();
  SavedState ss = new SavedState(superState);
  ss.position = mCurItem;
  if (mAdapter != null) {
    ss.adapterState = mAdapter.saveState();
  }
  return ss;
}
```

通过分析可知，保存了当前页面的 position 并调用 `mAdapter.saveState()` 方法保存 `mAdapter` 的状态(FragmentPagerAdapter 并没有保存任何东西返回 null)。

### FragmentPagerAdapter

```java
@Override
public Object instantiateItem(ViewGroup container, int position) {
  if (mCurTransaction == null) {
    mCurTransaction = mFragmentManager.beginTransaction();
  }
  final long itemId = getItemId(position);
  
  // Do we already have this fragment?
  String name = makeFragmentName(container.getId(), itemId);
  Fragment fragment = mFragmentManager.findFragmentByTag(name);
  if (fragment != null) {
    if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
    mCurTransaction.attach(fragment);
  } else {
    fragment = getItem(position);
    if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
    mCurTransaction.add(container.getId(), fragment,
                        makeFragmentName(container.getId(), itemId));
  }
  if (fragment != mCurrentPrimaryItem) {
    fragment.setMenuVisibility(false);
    fragment.setUserVisibleHint(false);
  }
  return fragment;
}
```

由上面 `instantiateItem` 方法可知，PagerAdapter 是靠宿主的 FragmentManager 来管理 Fragment 的，即所有的 Fragment 都被托管在 CategoryFragment 的 ChildFragmentManager 中，当 `findFragmentByTag` 不为空时就直接使用该 fragment 实例，反之调用 `getItem(position)` 实例化一个新的 Fragment。

本文中使用的案例[源码地址](https://github.com/shellljx/CNode-android)