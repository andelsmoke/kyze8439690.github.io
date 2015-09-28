---
layout: post
title: "Fragment recreate(2)"
date: 2015-09-28 19:44:54 +0800
comments: true
categories: 
---

除了常见的 Activity - Fragment 模式，还有 Activity - ViewPager - Fragment 模式，这种情况又略有不同。

有时候调试 activity recreate 的时候，会发现 ViewPager 变成一片空白，没有 Fragment 显示出来。该有的 Fragment recreate 哪里去了？

<!-- more -->

查看 FragmentAdapter 的源码中的`instantiateItem()`函数，代码如下：

```java
public Object instantiateItem(ViewGroup container, int position) {
   if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }
    final long itemId = getItemId(position);
    String name = makeFragmentName(container.getId(), itemId);
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        mCurTransaction.attach(fragment);
    } else {
        fragment = getItem(position);
        mCurTransaction.add(container.getId(), fragment,
                makeFragmentName(container.getId(), itemId));
    }
	//...
    return fragment;
}
```

可以看出，这里的逻辑为，如果 FragmentManager 中有可用的 Fragment 实例，就直接使用该实例，避免重复创建。否则才通过`getItem()`函数创建新的 fragment 实例。这里有一个细节，就是 Fragment 的 tag 是通过 makeFragmentName() 函数获取。具体代码如下：

```java
private static String makeFragmentName(int viewId, long id) {
    return "android:switcher:" + viewId + ":" + id;
}
```

这里的 viewId 就是 containerId，也就是 ViewPager 的 id；id 就是 getItemId()，也就是 position。所以以后可以通过构造这个 tag 直接将 ViewPager 的 Fragment find 出来（你可以在你的代码中这样做，注意类型检测与 NullPointer）。这里也是第一个关键点：**ViewPager 必须拥有 id**，否则 ViewPager 中的 Fragment 可能无法恢复。

回到刚刚的`instantiateItem()`函数，两条路径都只是 beginTransaction 却没有 commit，commit 操作在`finishUpdate()`中完成。代码如下：

```java
public void finishUpdate(ViewGroup container) {
	if (mCurTransaction != null) {
        mCurTransaction.commitAllowingStateLoss();
        mCurTransaction = null;    //为了gc？
        mFragmentManager.executePendingTransactions();
    }
}
```

可以看到，这里用的是 commitAllowingStateLoss()，而不是 commit，而且紧随一个 executePendingTransactions 使之生效。我个人的猜测是为了防止 activity 在后台的时候 commit 导致 crash（系统不允许 activity 在`onSaveInstanceState()`之后做 fragment transaction操作，以防止状态丢失）	。有失必有得，避免了crash，就可能导致 state 丢失。所以这里就引来了第二个关键点：**避免 activity 在后台的时候更换当前 fragment**，你可以检测 activity 状态，有必要的时候将操作封装在 Runnable 中，activity resume 之后执行这个 Runnable。只要在 resume 之后做 commit，那么 fragment state 就能被正确地保存。

在上一篇 blog 中，我提到过，Activity，View/ViewGroup，Fragment都能自动保存 state 并事后恢复，这里可不包括PagerAdapter。实际测试发现，recreate之后，viewPager.getAdapter() 为 null。既然没有 adapter，那么也就不会有 instantiateItem 啦。所以第三个要点是：**recreate 之后必须调用 setAdapter**，并且`getItemId()`必须返回和先前一样的值，以便 adapter 将 fragment item恢复出来。

还有一个细节就是，如果你的结构是 Activity : Fragment : ViewPager : Fragment 的话，那么在第一个 Fragment 中 new adapter 的时候，必须使用`getChildFragmentManager()`，而不是`getFragmentManager()`。childFramentManager 会给每一个 fragment 赋予正确的 parentFragment，以便后面恢复 fragment的时候恢复正确的 mWho 值（内部标识）。

总结以上，要 ViewPager 中的 Fragment 能正确回复，需要注意：

- ViewPager 要有 id
- 避免 Activity 在后台的更换当前 Fragment（commit操作）
- Activity recreate 之后必须调用 setAdapter 并保障 itemId 一致。