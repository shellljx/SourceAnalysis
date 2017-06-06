# Fragment那点事④mAdded&mActive

> 文章里所有分析都是根据Android Sdk 25.3.1 v4包
>
> 资料链接 [StackOverFlow](https://stackoverflow.com/questions/25695960/difference-between-madded-mactive-in-source-code-of-support-fragmentmanager)、[mAdded and mActive Fragment Lists](https://github.com/kgleong/software-engineering/wiki/Fragment-Manager#madded-and-mactive-fragment-lists)

经过前面几篇对 FragmentManager 源码的分析，对 `mAdded` 和 `mActive` 这两个属性有了一定的理解，下面是在 StackOverFlow 上的一个问题，详细的总结了这两个属性的具体差异。

**mAdded:**

1. 包含了所有已经 added 并且没有被从 Activity 中 removed 和 detached 的 Fragments。
2. 这些 Fragment 在视图交互上有如下特性:
   * 对下面事件作出反应:
     * 低内存事件
     * configuration changes (比如屏幕旋转)
   * 展示自定义 menu 并对 menu 的点击作出回应 `onMenuItemSelected()`。
   * Fragment 的生命周期响应它宿主 Activity 的生命周期
3. 不在 `mAdded` 中的 Fragments 不能对事件作出相应和展示自定义的 menu。

**mActive:**

`mAdded` 的一个超集，是绑定到一个 Activity 上的所有 Fragment。包括返回栈中所有的通过任何 `FragmentTransaction` 添加的 Fragments。这是非常重要的因为如下原因:

* 当一个 Activity 要保存它的 State 时，它必须保存它所有 Fragment 的状态，因为 `mActive` 保存了所有 Fragment，所以系统只要存储这个列表里的 Fragment 的状态就好了。而`mAdded` 只是被序列化成一个整形数组，每个元素指向 Fragment 在 `mActive` 中的下标位置(这块在前面 Fragment 的存储与恢复中分析到了)。
* 在恢复 Activity 的状态时，FragmentManager 的状态也会被恢复，`mActive` 列表就可以被用来恢复 `mAdded` 列表，因为保存状态的时候`mAdded` 被简单的保存为整形数组。
* 当一个 Activity 经历它的各生命周期时，它必须引起所有绑定的 Fragment 经历各自的生命周期。
  * 该 Activity 的 FragmentManager 有义务去引导所有 Fragemnt 转换到正确的状态，这其中包括屏幕上可见的 Fragment 的 View 层级的初始化，并且调用正确的生命周期函数。
  * 为了确保完整，FragmentManager 将遍历`mActive` 中所有的 Fragment，而不仅仅是 `mAdded`。
* 它持有所有 BackStack 返回栈引用的对象。
  * 这确保了返回栈中对 Fragment 操作的回滚能够实现。

**什么情况下这两个 Fragment 的列表会变动呢？**

* **mAdded**

  * 如果一个 Fragment 被添加到 Activity 中，那么这个 Fragment 会被 **added** 到该列表。
  * Fragment 被从该列表中移除:
    * Fragment 被从 Activity 中 removed。
    * Fragment 从 Activity 中 detached。

* **mActive:**

  * 如果一个 Fragment 被添加到 Activity 中，那么这个 Fragment 会被 **added** 到该列表。
  * **只有**在下面这两种情况 Fragment 才会被从该列表中移除:
    * Fragment 被从 Activity 中移除，并且没有在返回栈中。
    * 一个 transaction 从返回栈中被 pop 出来、 Fragment 的 add 或者 replace 操作被逆向，即返回栈不再持有该 Fragment。

* **结论**

  `mAdded` 是 **alive** in a sense (在感官上存活的)，而 `mActive` 是所有绑定在 Activity 上的 Framgent 列表。`mActive` 包括 **alive** Fragments (`mAdded`) 和 **freeze dried** (冷冻储存) Fragments (仍然在返回栈中等待被恢复)。

  继续对比活着的和冷冻储存的实体的区别：当 activities 触发回调函数比如 `onConfigurationChanged()` 活着 `onLowMemory()`，真正关系这些回调事件的只是活着的 Fragments。

  所以你会发现在 `FragmentManagerImpl` 回调只会遍历 `mAdded` 即活着的 Fragments。

  `fragmentManager.dispatchLowMemory()` 在 `activity.onLowMemory()` 中被调用。

  ```java
  public void dispatchLowMemory() {
      if (mAdded != null) {
          for (int i=0; i<mAdded.size(); i++) {
              Fragment f = mAdded.get(i);
              if (f != null) {
                  f.performLowMemory();
              }
          }
      }
  }
  ```

  ​