---
layout: post
title: "Why is ListView's HeaderView/FooterView setVisibility(View.GONE) not working?"
date: 2015-06-19 18:14:54 +0800
comments: true
categories: 
---

使用 `ListView` 的时候，根据需求需要动态将HeaderView/FooterView隐藏掉，这时你会发现 `setVisibility(View.GONE)` 根本没有效果，两个折衷的方案是:

- 动态将HeaderView/FooterView remove掉，要显示的时候再add回去。
- 在HeaderView/FooterView外面包一个Container ViewGroup(例如 `FrameLayout`)，再把这个Container作为HeaderView/FooterView add 到`ListView` 中。

以上两个方案都能实现隐藏 HeaderView/FooterView 的效果。下面我从源码介绍以下为何 `View.GONE` 不生效，以及为何以上 workaround 能够生效的原因。

<!--more-->

总所周知，一个 `View` 能在屏幕上显示出来，需要经历 **measure / layout / draw** 三个步骤，measure 步骤负责测量View的大小，layout 步骤负责布局，draw 步骤负责绘制。一个 `View` 占屏幕多大位置，一般是由 measure 步骤决定。

对于 `ListView` 来说，不论是 Header 还是 Footer 还是普通的 ItemView，对他来说，都是普通的子 View。Header/Footer 和 ItemView 的区别，在 `HeaderViewListAdapter` 中体现。

`ListView` HeaderView/FooterView 设置隐藏不生效，表现为仍然占据原有的位置空间。所以我们先从 `ListView` 的 `onMeasure()` 函数入手。

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // Sets up mListPadding
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    //...

    int childWidth = 0;
    int childHeight = 0;
    int childState = 0;

    mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
    if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED ||
            heightMode == MeasureSpec.UNSPECIFIED)) {
        final View child = obtainView(0, mIsScrap);

		 //测量 itemView
        measureScrapChild(child, 0, widthMeasureSpec);

        childWidth = child.getMeasuredWidth();
        childHeight = child.getMeasuredHeight();
        //...
    }

    //...

    if (heightMode == MeasureSpec.UNSPECIFIED) {
        heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                getVerticalFadingEdgeLength() * 2;
    }

    if (heightMode == MeasureSpec.AT_MOST) {
        // TODO: after first layout we should maybe start at the first visible position, not 0
        heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
    }

    //设置 ListView dimension
    setMeasuredDimension(widthSize , heightSize);
    mWidthMeasureSpec = widthMeasureSpec;        
}
```

从这里可以看出每一个 itemView 的测量，都是由 `measureScrapChild()` 这个函数完成的，所以我们再来看看这个函数的源码。

```java
private void measureScrapChild(View child, int position, int widthMeasureSpec) {
    LayoutParams p = (LayoutParams) child.getLayoutParams();
    if (p == null) {
        p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
        child.setLayoutParams(p);
    }
    p.viewType = mAdapter.getItemViewType(position);
    p.forceAdd = true;

    int childWidthSpec = ViewGroup.getChildMeasureSpec(widthMeasureSpec,
            mListPadding.left + mListPadding.right, p.width);
    int lpHeight = p.height;
    int childHeightSpec;
    if (lpHeight > 0) {
        childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
    } else {
        childHeightSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
    }
    child.measure(childWidthSpec, childHeightSpec);
}
```

从这段源码可以看出，每个 itemView 的测量，要先判断 view 是否存在 `AbsListView.LayoutParams`，如果不存在则new一个新的。`generateDefaultLayoutParams()` 源码如下。

```java
@Override
protected ViewGroup.LayoutParams generateDefaultLayoutParams() {
    return new AbsListView.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.WRAP_CONTENT, 0);
}
```

`WRAP_CONTENT` 值为 -2，`MATCH_PARENT` 为 -1，所以默认情况下 itemView 的大小由他自身决定（`MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED)`）。

在以上的源码中可以总结出以下几点：

- `ListView` 的源码中并没有针对 Visibility 做特殊处理，一般 `ViewGroup` 都会跳过 Visibility 为 `View.GONE` 的 childView, 让他们大小为0。所以设置 HeaderView/FooterView 为 `View.GONE` 是无效的。

- `ListView` 中对 itemView 的测量取决于 `LayoutParams`，想通过设置 `LayoutParams` 来隐藏 `ListView` 的某一项是行不通的，因为 0，-1，-2都会变成 ItemView 自己来决定自己的大小。你可以这样做一个测试：给 HeaderView/FooterView 设置一个 height 为 1 的 `AbsListView.LayoutParams`，你会发现 HeaderView/FooterView终于能够自我收缩了！

那为何给 HeaderView/FooterView 包一层 Container 就可以实现隐藏的效果呢？分析如下：

- 默认结构是这样的： **ListView -> HeaderView/FooterView(View.GONE)**  
由于 ListView 没有对 Visibility 做处理，所以 HeaderView/FooterView 会被当成 `View.VISIBLE` 一样去 measure，隐藏失败。

- Container Workaround 是这样的：**ListView -> Container(WRAP_CONTENT) -> HeaderView/FooterView(View.GONE)**  
ListView还是让 ItemView 自己决定自己的大小，Container 是 `WRAP_CONTENT`，继续看下一层，HeaderView/FooterView 是 `View.GONE`，从而导致 Container measure 出来的 measureHeight 是 0，所以 HeaderView/FooterView 被隐藏。

当然还有第三种 Workarround：就是覆盖 HeaderView/FooterView的 `getMeasuredHeight()` 函数，让它有选择地按照实际情况返回 0 或者 `super.getMeasuredHeight()`。