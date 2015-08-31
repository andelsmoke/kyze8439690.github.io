---
layout: post
title: "tools 命名空间的使用与 support library annotations 介绍"
date: 2015-08-31 23:33:09 +0800
comments: true
categories: 
---

[知识来源PPT](https://speakerdeck.com/rock3r/tools-of-the-trade-droidcon-nyc-2015)

# Tools 命名空间

tools 命名空间是在 Android Studio 中引入的 编辑预览特性，可以生成一些只在 IDE 预览界面生效的特性。

<!-- more -->

### 属性预览

```
<TextView
	tools:text="test title"
	/>
```

这个属性应该很多人都用过，设置只在 IDE 预览界面生效的属性。可以方便地对照设计稿调 UI 而避免将预览属性编译进 apk 中（是否会编译进 apk 我也没具体验证过，不过 android 在遇到不支持的 xml 属性会直接忽略，所以无伤大碍）。而且这些属性在被include的时候能够保持，所以推荐在 headerView，footerView，adapterItem，activity，fragment 等xml中使用，在 CustemView 中配合 ```isInEditMode()``` 函数使用更佳！不过没有代码提示功能有点遗憾。

### Lint提示忽略

对于有洁癖的人来说，XML Editor 有很多小黄点实在无法忍受。比如布局的 RTL 支持，ImageView 的 description 定义等。这些属性虽然官方建议添加，但是在实际开发环境做支持实在有点困难。这时候就可以用 ```tools:ignore``` 属性将小黄点去处，我常用的 ignore 属性如下：

```
tools:ignore="ContentDescription, RtlHardcoded"
```

有时候在 Style 中需要对原生属性和 appcompat 自定义属性同时设置，但是原生属性在低版本上不支持，当然就如我前面说的，android 在遇到不支持的属性会直接忽略，但是如果强迫发作，可以用下面的方法忽略：

```
<item name="android:actionModeShareDrawable" tools:ignore="NewApi">@drawable/ic_abc_share</item>
```

### 风格套用

这个是我比较少用的属性，举例如下，大家可以试试：

```
<merge
	tools:context=".MainActivity"
	tools:showIn="@layout/activity_main"
	tools:menu="map"
	tools:actionBarNavStyle="tabs"
	/>
```

```
<fragment
	tools:layout="@layout/fragment_main"
	/>
```

### ListView 预览

正常情况下，大家的 ListView 预览都是千篇一律的固定样式。其实可以套用 HeaderView/FooterView/ItemView 到预览界面中，需要注意的是，itemView 预览的 tools:attribute 只在第一个 item 中生效，但是我想应该不碍事吧 :)

```
<ListView
	tools:listheader="@layout/list_header"
	tools:listfooter="@layout/list_footer"
	tools:listitem="@layout/list_item"
	/>
```

总结：以上设置可能大家觉得很繁琐，很麻烦，是否有必要花费时间做这些事情呢？我个人觉得，任何一个有追求的 Android 开发者都应该在学习在 xml 中设置 tools attributes，**他可以让你还没运行就能发现更多UI上的问题，在多人协作的时候，也能让协作者更加方便地理解你的代码，避免部分属性的二次渲染(xml渲染一次，java修改渲染一次)**，希望 google 能添加更多 tools attributes 特性，例如定义 merge 节点外层预览等。

# Support Library Annotations

Support Library Annotations 相信大家都用过，相比 tools attributes，他所带来的好处更加明显，能够根据代码逻辑需要添加限制，并在编写/编译时对错误进行提示。在Library项目，多人协作中起到很好的辅导作用。

### @Nullable 与 @NonNull

顾名思义，声明返回值/全局变量/参数可能为 null 或者不允许为 null，从而对潜在的 NullPointerException 进行警告, 例如访问一个@Nullable 的成员变量/函数，将 @Nullable 变量设置为 @NonNull 的参数等。我一般会结合 Gson 使用，最新的 ButterKnife 也用这个 annotation 替换了老版本内置的 @Optional。由于过于常用，这里就不贴 sample 了。

### 资源ID限制

限制参数只能为资源 ID 而非普通 int 数值，可用 annotation 类型如下：

| Name | Description          |
| ------------ | -------------|
| @AnimatorRes | R.animator.xxx |
| @AnimRes | R.anim.xxx |
| @AnyRes | 任意类型的资源 ID |
| @ArrayRes | R.array.xxx |
| @AttrRes | R.attr.xxx |
| @BoolRes | R.bool.xxx |
| @ColorRes | R.color.xxx |
| @DimenRes | R.dimen.xxx |
| @DrawableRes | R.drawable.xxx |
| @FractionRes | 3.14159 或者 10%p 之类的值，具体请查看 ```getResources().getFraction()``` 函数 |
| @IdRes | R.id.xxx |
| @IntegerRes | R.integer.xxx |
| @InterpolatorRes | R.interpolator.xxx |
| @LayoutRes | R.layout.xxx |
| @MenuRes | R.menu.xxx |
| @PluralsRes | 复数资源，具体案例下面会解释 |
| @RawRes | R.raw.xxx |
| @StringRes | R.string.xxx |
| @StyleableRes | R.styleable.xxx |
| @StyleRes | R.style.xxx |
| @XmlRes | R.xml.xxx |

大部分都很好理解，其中可能比较陌生的就是 @FractionRes 与 @ PluralsRes 了。

#### @FractionRes

FractionRes 类型的资源在 Animation Xml 中比较常见，比如100%p，p代表parent，也就是占 parent 的 100%，例如在 TranslateAnimation 中就很常见（除非你能将 parent 大小 hardcode）

#### @PluralsRes

PluralsRes 没用过也是正常，英语的名字复数一般在后面加 s 或者 es，单数不用，所以在 String 格式化的时候需要用一种叫做 Plurals 的资源类型，sample 如下，简单易懂：

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <plurals name="tutorials">
        <item quantity="zero">no Tutorial </item>
        <item quantity="one">one Tutorial </item>
        <item quantity="other">%d  Tutorials</item>
    </plurals>
</resources>
```

顺便附上 java 调用

```
String quantityString = getResources().getQuantityString(R.plurals.tutorials,
    number);
```

资源 Annotation 强烈建议在 CustomView 中使用。

### 参数范围限制

- @FloatRange
- @IntRange

Sample：

```
public void setAlpha(@IntRange(from=0, to=255) int alpha) { ... }
```

### 容器长度限制

@Size

常见容器类都可使用，例如：

```
public void setLocation(@Size(2) int[] location) { ... }
```

### 枚举限制

- @IntDef
- @StringDef

由于 Android 不推荐使用枚举，所以一般会用 int 或者 String 代替枚举，但是这样就缺少了取值限制，所以需要结合这两个 annotation 来使用。

Sample：

```
@Retention(RetentionPolicy.SOURCE)
@IntDef({ExpertHelpAdapter.TYPE_EMPTY, ExpertHelpAdapter.TYPE_LISTVIEW, ExpertHelpAdapter.TYPE_WEBVIEW})
public @interface AdapterType {}

@AdapterType
private int mType = TYPE_WEBVIEW;

@AdapterType
public int getType() {
    return mType;
}

public void setType(@AdapterType int type) {
	mType = type;
	notifyDataSetChanged();
}
```

注意添加 ```@Retention(RetentionPolicy.SOURCE)``` ，指定此 annotation 只在代码中生效（非运行时），建议在所有替代枚举的地方使用。

### 线程限制 

| Name | Description          |
| -------------- | ------------- |
| @MainThread | 只能在主线程线程运行 |
| @UiThread | 只能在UI线程运行 |
| @BinderThread | 只能在Binder线程运行 |
| @WorkerThread | 只能在自定义线程中运行 |


这几个 annotation 我也没用过，查询了相关文档，作出解释如下：

- MainThread 与 UiThread 在大部分情况下可混用，可能是Application.onCreate 与 Activity.onCreate 的区别？只能等待大神解答了。
- BinderThread： ContentProvider 做增删改查的线程。
- WorkerThread： 自开线程，一般就是非UI线程。

以上 Annotation 建议在多线程模块中使用，特别涉及 UI 回调与非 UI 回调。

### 架构注解？

- @CallSuper：一般添加在函数声明处，要求覆盖时必须调用super实现。例如

```
@CallSuper
protected void onCreate(@Nullable Bundle savedInstanceState) { ... }
```

- @CheckResult：确保函数返回值被使用，没有被忽略
- @VisibleForTesting：暴露测试接口用

### 权限限制

@RequiresPermission：检查相关权限是否已申请，建议在 library 项目中使用。

### 混淆限制

@Keep：混淆时保持变量或方法不被混淆，**但该特性目前还未被支持**。

### 调试用注解

@ViewDebug.ExportedProperty：添加在getXXX方法前，可让CustomView的内部参数在 Hierarchy Viewer 中查询，使用方法如下：

```
@ExportedProperty(category = "layout")
int x = 1;

@ExportedProperty(category = "layout")
public boolean isFocused() {
    return true;
}
```

其中 layout 为 Hierarchy Viewer 中的属性目录。

但是我觉得这个注解实际意义不大，首先Hierarchy Viewer使用限制很多，需要 debug 版本的 ROM 才能正常开启，虽然有 ViewServer 这个 Library 可以尝试开启，但是成功率不高。其次是 Hierarchy Viewer dump 数据实在过慢，UI界面卡顿明显，稍微复杂一点的视图定位会耗费很多时间，而且很容易手机端 app crash，电脑 dump 数据丢失。所以，还是老老实实 log 或者断点调试吧……





