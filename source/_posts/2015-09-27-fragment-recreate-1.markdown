---
layout: post
title: "Fragment recreate(1)"
date: 2015-09-27 11:14:15 +0800
comments: true
categories: android
---

- 说明

    一般都是用v4的Fragment实现，可以有`getChildFragmentManager()`的支持，这里以 v4 版本为例。

- 命名

    FragmentActivity 源码中的变量命名其实很乱，比如一个 FragmentManagerImpl 的实例，叫做 mFragments，后面需要注意。

<!-- more -->

- 保存与恢复

    FragmentActivity.onCreate() 中有这样一个函数：`getLastNonConfigurationInstance()`
与之相对应的有`onRetainNonConfigurationInstance()`。
    
    - `onRetainNonConfigurationInstance()`：将要保存的实例保存起来，在 recreate 之后恢复。需要注意的是，函数的返回类型是 Object，说明这里保存的是任何实例（Activity，Fragment，etc），而不是 SaveState。与利用 SaveState 新建实例恢复状态不同的方式不同。这个方法在 api level 13 中取消使用。FragmentActivity 中覆盖了Activity 的这个方法，加入了一些额外的 Object，比如一个 Fragment 组和一个 Loader 组。现在要用原来`onRetainNonConfigurationInstance()`方法来保存 Object 的话，请使用`onRetainCustomNonConfigurationInstance()`方法。
    - `getLastNonConfigurationInstance()`：这个方法返回了一个 NonConfigurationInstances 实例，里面包含了上面所说保存的 Fragment 实例组和 Loader 实例组。

    FragmentActivity 利用上面两个函数来还原部分 Fragment 和 Loader。为什么是部分 Fragment 呢？因为这个是有选择的筛选，判断原则就是 fragment 是不是`mRetainInstance`等于 true。还记得有一个 hold 住 fragment 实例的例子吗？一个包含 AsyncTask 的 fragment，在屏幕旋转 Activity recreate 之后，fragment 中的 AsyncTask 并没有重新跑，而是继续执行。那个例子的关键，就是 fragment 初始化之后，需要调用`setRetainIntance(true)`方法，将 `mRetainInstance` 赋值为 true。所以 FragmentActivity 会将所有这种类型的 Fragment 都用 `onRetainCustomNonConfigurationInstance()` 方法保存起来，以便 AsyncTask 继续执行。

    同理，Loader 本来的作用就是，在 Activity recreate 之后，避免重新 load 的过程。所以 FragmentActivity 也会利用这个方法将**所有** loader 保存起来，recreate 之后复用。

    具体代码如下：

    ```java
    public final Object onRetainNonConfigurationInstance() {
        //...
        Object custom = onRetainCustomNonConfigurationInstance();    //覆盖这个函数来保存自己的Object
        ArrayList<Fragment> fragments = mFragments.retainNonConfig();
        //...
        if (fragments == null && !retainLoaders && custom == null) {
            return null;
        }
        
        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.activity = null;
        nci.custom = custom;
        nci.children = null;
        nci.fragments = fragments;
        nci.loaders = mAllLoaderManagers;
        return nci;
    }
    ```

    `retainNonConfig()`的实现如下（这函数的命名我也没懂）：

    ```java
ArrayList<Fragment> retainNonConfig() {
        ArrayList<Fragment> fragments = null;
        if (mActive != null) {
            for (int i=0; i<mActive.size(); i++) {
                Fragment f = mActive.get(i);
                if (f != null && f.mRetainInstance) {
                    if (fragments == null) {
                        fragments = new ArrayList<Fragment>();
                    }
                    fragments.add(f);
                    f.mRetaining = true;
                    f.mTargetIndex = f.mTarget != null ? f.mTarget.mIndex : -1;
                    if (DEBUG) Log.v(TAG, "retainNonConfig: keeping retained " + f);
                }
            }
        }
        return fragments;
    }
    ```

    那剩下部分的 fragment 如何恢复呢？这就要靠常见的 create ＋ restore state 模式了。
    
    android 源码中很多地方都用到这种 create ＋ restore 的模式，比如 Activity，View/ViewGroup，Fragment 等。如果 fragment.mRetainInstance 等于 false，则采用下面的代码来保存（`FragmentActivity.onSaveInstanceState()`）:

    ```java
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        Parcelable p = mFragments.saveAllState();
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
    }
    ```

    这里的 mFragments 其实是一个 FragmentManagerImpl 实例，前面说过了。他的 onSaveInstanceState 方法中用了一个类，叫做 `FragmentState`，用来保存 Fragment 的状态。其中包含的关键元素有：

    - ClassName：fragment 的具体类名
    - Index：fragment 在 Activity 中的下标，一个 Activity 含有多个 Fragment
    - FragmentId：如果 Fragment 是在 xml 中初始化的话，则为 xml 中的ID，否则和下面的 containerId 一致
    - ContainerId：`add(R.id.container, mFragment)`中的container id
    - Tag：add 的时候用的 tag
    - Detached：fragment 状态
    - Arguments：`fragment.setArguments(args)`的 Bundle 类型参数
    
    观察上面的元素，可以发现足以重建一个完整的 fragment，类型，参数，状态，id，tag 都有了。而 fragment 内部 View 的状态则用 View 自带的状态恢复机制去恢复。当然这里要求你用标准的模式去初始化一个fragment，new ＋ setArguments()，如果你用new ＋ 自定义setXXX方法的话，那这里就不能正常恢复了。

    两种保存机制对应的恢复代码在 `FragmentManager.restoreAllState()`中，内容太长这里就不贴了。大致逻辑为：恢复 retain fragments -> 恢复 state fragments -> 指定 target fragment -> 重建 added fragment 组 -> 重建 backStack

    正因为 fragment 自带这种简单的 fragment 恢复机制，我们经常看见很多源码的 activity 的 onCreate 中，如果 savedInstanceState 为 null，才做 fragment traction 操作，避免重复 add。
    
