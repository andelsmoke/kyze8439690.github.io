---
layout: post
title: "(译)Data Binding 指南"
date: 2016-02-17 18:32:59 +0800
comments: true
categories: 
---

**翻译自 [http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html)**

这个文档用于解释如何使用 Data Binding Library 编写声明式的布局，减少应用中逻辑以及布局所需要的“胶水代码”。

Data Binding Library 提供了灵活性与通用性 － 它是一个 support library，可以在 Android 2.1(API level 7+)以上的平台使用。

为了使用 data binding，gradle plugin的版本必须是 1.5.0-alpha1以上。

<!--more-->

##编译环境

为了使用 Data Binding，首先在 Android SDK manager 中下载最新的 Support repository。

然后在 build.gradle 中添加 dataBinding 段。

使用以下代码段配置 data binding：

	android {
	    ....
	    dataBinding {
	        enabled = true
	    }
	}
	
如果你的 app module 依赖了一个使用 data binding 的库，那么你的 app module 的 build.gradle 也必须配置 data binding。

同时，确定使用了支持该特性的 Android Studio。在 Android Studio 1.3 以及之后的版本提供了 data binding 的支持，详见 [Android Studio Support for Data Binding](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html#studio_support)。

##Data Binding 布局文件

###编写你的第一个 data binding 表达式

Data binding 布局文件与普通布局文件有一点不同。它以一个 `layout` 标签作为根节点，里面是 `data` 标签与 `view` 标签。view 标签的内容就是不使用 data binding 时的普通布局文件内容。以下是一个例子：

	<?xml version="1.0" encoding="utf-8"?>
	<layout xmlns:android="http://schemas.android.com/apk/res/android">
	   <data>
	       <variable name="user" type="com.example.User"/>
	   </data>
	   <LinearLayout
	       android:orientation="vertical"
	       android:layout_width="match_parent"
	       android:layout_height="match_parent">
	       <TextView android:layout_width="wrap_content"
	           android:layout_height="wrap_content"
	           android:text="@{user.firstName}"/>
	       <TextView android:layout_width="wrap_content"
	           android:layout_height="wrap_content"
	           android:text="@{user.lastName}"/>
	   </LinearLayout>
	</layout>

在 `data` 标签中的 user 变量 描述了一个布局中会用到的属性。

	<variable name="user" type="com.example.User"/>
	
布局文件中的表达式使用 “@{}” 的语法。在这里，TextView 的文本被设置为 user中的 firstName 属性。

	<TextView android:layout_width="wrap_content"
	          android:layout_height="wrap_content"
	          android:text="@{user.firstName}"/>
	          
###数据对象

假设你有一个 plain-old Java object(POJO) 的 User 对象。

	public class User {
	   public final String firstName;
	   public final String lastName;
	   public User(String firstName, String lastName) {
	       this.firstName = firstName;
	       this.lastName = lastName;
	   }
	}
	
这种类型的对象拥有不可改变的数据。在应用中，读一次且不变动数据的对象非常常见。也可以使用 JavaBeans 对象：

	public class User {
	   private final String firstName;
	   private final String lastName;
	   public User(String firstName, String lastName) {
	       this.firstName = firstName;
	       this.lastName = lastName;
	   }
	   public String getFirstName() {
	       return this.firstName;
	   }
	   public String getLastName() {
	       return this.lastName;
	   }
	}

从 data binding 的角度看，这两个类是一样的。用于 TextView 的 `android:text` 属性的表达式`@{user.firstName}`，会读取 POJO 对象的 `firstName` 域以及 JavaBeans 对象的 `getFirstName()` 方法。此外，如果 `firstName()` 方法存在的话也同样可用。

###绑定数据

在默认情况下，会基于布局文件生成一个 Binding 类，将它转换成帕斯卡命名并在名字后面接上"Binding"。上面的那个布局文件叫 `main_activity.xml`，所以会生成一个 `MainActivityBinding` 类。这个类包含了布局文件中所有的绑定关系（`user`变量），会根据绑定表达式给布局文件赋值。在 inflate 的时候创建 binding 的方法如下：

	@Override
	protected void onCreate(Bundle savedInstanceState) {
	   super.onCreate(savedInstanceState);
	   MainActivityBinding binding = DataBindingUtil.setContentView(this, R.layout.main_activity);
	   User user = new User("Test", "User");
	   binding.setUser(user);
	}

就这么简单！运行应用，你会发现测试用户已经显示在界面中了。你也可以通过以下这种方式：

	MainActivityBinding binding = MainActivityBinding.inflate(getLayoutInflater());
	
如果你在 ListView 或者 RecyclerView 的 adapter 中使用 data binding，你可以这样写：

	ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
	//or
	ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
	
###绑定事件

事件可以直接与 handler 函数绑定，类似于 `android:onClick` 可以指定 Activity 中的一个函数一样。事件属性的命名由 listener 的函数命名决定。举个例子，`View.OnLongClickListener` 中有一个 `onLongClick()` 函数，所以这个事件的对应属性就是 `android:onLongClick`。

为了将事件分配给 handler，只需要使用一个 binding 表达式，值为要调用的函数名。举个例子，如果你的数据对象有两个函数：

	public class MyHandlers {
	    public void onClickFriend(View view) { ... }
	    public void onClickEnemy(View view) { ... }
	}

分配点击事件的 binding 表达式如下：

	<?xml version="1.0" encoding="utf-8"?>
	<layout xmlns:android="http://schemas.android.com/apk/res/android">
	   <data>
	       <variable name="handlers" type="com.example.Handlers"/>
	       <variable name="user" type="com.example.User"/>
	   </data>
	   <LinearLayout
	       android:orientation="vertical"
	       android:layout_width="match_parent"
	       android:layout_height="match_parent">
	       <TextView android:layout_width="wrap_content"
	           android:layout_height="wrap_content"
	           android:text="@{user.firstName}"
	           android:onClick="@{user.isFriend ? handlers.onClickFriend : handlers.onClickEnemy}"/>
	       <TextView android:layout_width="wrap_content"
	           android:layout_height="wrap_content"
	           android:text="@{user.lastName}"
	           android:onClick="@{user.isFriend ? handlers.onClickFriend : handlers.onClickEnemy}"/>
	   </LinearLayout>
	</layout>
	
也有一个特殊的点击事件 handler，他们有一些不同于 `android:onClick` 的属性来避免冲突。下面是一些用来避免冲突的属性：

|Class   |Listener Setter   |Attribute   |
|---|---|---|
|[SearchView](http://developer.android.com/reference/android/widget/SearchView.html)   |[setOnSearchClickListener(View.OnClickListener)](http://developer.android.com/reference/android/widget/SearchView.html#setOnSearchClickListener(android.view.View.OnClickListener))   |  `android:onSearchClick` |
|[ZoomControls](http://developer.android.com/reference/android/widget/ZoomControls.html)   |[setOnZoomInClickListener(View.OnClickListener)](http://developer.android.com/reference/android/widget/ZoomControls.html#setOnZoomInClickListener(android.view.View.OnClickListener))   |`android:onZoomIn`   |
|[ZoomControls](http://developer.android.com/reference/android/widget/ZoomControls.html)   |[setOnZoomOutClickListener(View.OnClickListener)](http://developer.android.com/reference/android/widget/ZoomControls.html#setOnZoomOutClickListener(android.view.View.OnClickListener))   |`android:onZoomOut`   |

##布局细节

###导入

`data`标签内可以有多个 `import` 标签。你可以在布局文件中像使用 Java 一样导入引用。

	<data>
	    <import type="android.view.View"/>
	</data>

现在 View 可以被这样引用：

	<TextView
	   android:text="@{user.lastName}"
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"
	   android:visibility="@{user.isAdult ? View.VISIBLE : View.GONE}"/>
	   
当类名发生冲突时，可以使用 alias：

	<import type="android.view.View"/>
	<import type="com.example.real.estate.View"
	        alias="Vista"/>

现在，`Vista` 可以用来引用 `com.example.real.estate.View` ，与 `View` 在布局文件中同时使用。导入的类型也可以用于变量的类型引用和表达式中：

	<data>
	    <import type="com.example.User"/>
	    <import type="java.util.List"/>
	    <variable name="user" type="User"/>
	    <variable name="userList" type="List&lt;User>"/>
	</data>
	
> 注意：Android Studio 还没有对导入提供自动补全的支持。你的应用还是可以被正常编译，要解决这个问题，你可以在变量定义中使用完整的包名。

	<TextView
	   android:text="@{((User)(user.connection)).lastName}"
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	   
导入也可以用于在表达式中使用静态域/方法:

	<data>
	    <import type="com.example.MyStringUtils"/>
	    <variable name="user" type="com.example.User"/>
	</data>
	…
	<TextView
	   android:text="@{MyStringUtils.capitalize(user.lastName)}"
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	   
和 Java 一样，`java.lang.*` 会被自动导入。

###变量

`data` 标签中可以有任意数量的 `variable` 标签。每个 `variable` 标签描述了会在 binding 表达式中使用的属性。

	<data>
	    <import type="android.graphics.drawable.Drawable"/>
	    <variable name="user"  type="com.example.User"/>
	    <variable name="image" type="Drawable"/>
	    <variable name="note"  type="String"/>
	</data>
	
变量类型会在编译时被检查，所以如果变量声明了 `Observable` 接口或者是一个[可观察容器类](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html#observable_collections)，那它会被反射使用。如果变量是一个没有声明 Observable* 接口的基类或借口，变量的变动则不会引起 UI 的变化！

当针对不同配置编写不同的布局文件时（比如横屏竖屏的布局），变量会被合并。所以这些不同配置的布局文件之间不能存在冲突。

自动生成的 binding 类会为每一个变量生产 getter/setter 函数。这些变量会使用 Java 的默认赋值，直到 setter 函数被调用。默认赋值有 `null`，`0`(`int`)，`false`(`boolean`)等。

binding 类也会生一个一个命名为 `context` 的特殊变量，这个变量被用于表达式中。`context` 变量其实就是 rootView 的 [`getContext()`](http://developer.android.com/reference/android/view/View.html#getContext()) 的返回值。`context` 变量会被同名的显式变量覆盖。

###自定义 Binding 类名

默认情况下，binding 类的名称取决于布局文件的命名，以大写字母开头，移除下划线，后续字母大写并追加 “Binding” 结尾。这个类会被放置在 databinding 包中。举个例子，布局文件 `contact_item.xml` 会生成 `ContactItemBinding` 类。如果 module 包名为 `com.example.my.app`，binding 类会被放在 `com.example.my.app.databinding` 中。

通过修改 `data`标签中的`class` 属性，可以修改 Binding 类的命名与位置。举个例子：

	<data class="ContactItem">
	    ...
	</data>

以上会在 databinding 包中生成名为 `ContactItem` 的binding 类。如果需要放置在不同的包下，可以在前面加 "."：

	<data class=".ContactItem">
	    ...
	</data>
	
这样的话，`ContactItem` 会直接生成在 module 包下。如果提供完整的包名，binding 类可以放置在任何包名中：

	<data class="com.example.ContactItem">
	    ...
	</data>
	
### Includes

在使用应用命名空间的布局中，变量可以传递到任何 include 布局中。

	<?xml version="1.0" encoding="utf-8"?>
	<layout xmlns:android="http://schemas.android.com/apk/res/android"
	        xmlns:bind="http://schemas.android.com/apk/res-auto">
	   <data>
	       <variable name="user" type="com.example.User"/>
	   </data>
	   <LinearLayout
	       android:orientation="vertical"
	       android:layout_width="match_parent"
	       android:layout_height="match_parent">
	       <include layout="@layout/name"
	           bind:user="@{user}"/>
	       <include layout="@layout/contact"
	           bind:user="@{user}"/>
	   </LinearLayout>
	</layout>
	
需要注意，`name.xml` 与 `contact.xml` 中都需要声明 `user` 变量。

Data binding 不支持直接包含 merge 节点。举个例子，**以下的代码就不能正常运行**：

	<?xml version="1.0" encoding="utf-8"?>
	<layout xmlns:android="http://schemas.android.com/apk/res/android"
	        xmlns:bind="http://schemas.android.com/apk/res-auto">
	   <data>
	       <variable name="user" type="com.example.User"/>
	   </data>
	   <merge>
	       <include layout="@layout/name"
	           bind:user="@{user}"/>
	       <include layout="@layout/contact"
	           bind:user="@{user}"/>
	   </merge>
	</layout>
	
##表达式语言

###通用特性

表达式语言与 Java 表达式有很多相似之处。下面是相同之处：

- 数学计算 `+ - / * %`
- 字符串连接 `+`
- 逻辑 `&& ||`
- 二进制 `& | ^`
- 一元 	`+ - ! ~`	
- 位移 `>> >>> <<`
- 比较 `== > < >= <=`
- `instanceof`
- 组 `()`
- 字面量 - 字符，字符串，数字，`null`
- 类型转换
- 函数调用
- 域存取
- 数组存取 `[]`
- 三元运算符 `?:`

例子：

	android:text="@{String.valueOf(index + 1)}"
	android:visibility="@{age &lt; 13 ? View.GONE : View.VISIBLE}"
	android:transitionName='@{"image_" + id}'
	
###缺失的操作符

一些 Java 中的操作符在表达式语法中不能使用。

- `this`
- `super`
- `new`
- 显式泛型调用 `<T>`

###Null合并运算符

Null合并运算符(`??`)会在非 null 的时候选择左边的操作，反之选择右边。

	android:text="@{user.displayName ?? user.lastName}"
	
等同于

	android:text="@{user.displayName != null ? user.displayName : user.lastName}"

###属性引用

首先是先前[编写你的第一个 data binding 表达式](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html#writing_your_first_data_binding_expressions)中所提到的：JavaBean 引用。当表达式引用了一个类内的属性时，他会尝试直接调用域，getter，还有 ObservableFields。

	android:text="@{user.lastName}"
	
###避免NullPointerException

自动生成的 data binding 代码会自动检查和避免 null pointer exceptions。举个例子，在表达式 `@{user.name}` 中，如果 `user` 是 null，`user.name` 会赋予默认值 `null`。如果你引用了 `user.age`，因为 age 是 `int` 类型，所以默认赋值为 0。

###容器类

通用的容器类：数组，lists，sparse lists，和 map，可以用 `[]` 操作符来存取

	<data>
	    <import type="android.util.SparseArray"/>
	    <import type="java.util.Map"/>
	    <import type="java.util.List"/>
	    <variable name="list" type="List&lt;String>"/>
	    <variable name="sparse" type="SparseArray&lt;String>"/>
	    <variable name="map" type="Map&lt;String, String>"/>
	    <variable name="index" type="int"/>
	    <variable name="key" type="String"/>
	</data>
	…
	android:text="@{list[index]}"
	…
	android:text="@{sparse[index]}"
	…
	android:text="@{map[key]}"

###字符串字面量

使用单引号把属性包起来，就可以很简单地在表达式中使用双引号：

	android:text='@{map["firstName"]}'
	
也可以用双引号将属性包起来。这样的话，字符串字面量就可以用`&quot;`或者反引号(`) 来调用

	android:text="@{map[`firstName`}"
	android:text="@{map[&quot;firstName&quot;]}"

###资源

也可以在表达式中使用普通的语法来引用资源：

	android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
	
字符串格式化和复数形式可以这样实现：

	android:text="@{@string/nameFormat(firstName, lastName)}"
	android:text="@{@plurals/banana(bananaCount)}"
	
当复数形式有多个参数时，应该这样写：

	
	  Have an orange
	  Have %d oranges
	
	android:text="@{@plurals/orange(orangeCount, orangeCount)}"
	
一些资源需要显示类型调用。

|Class   |Listener Setter   |Attribute   |
|---|---|---|
|String[]   |@array   |@stringArray   |
|int[]   |@array   |@intArray   |
|TypedArray   |@array   |@typedArray   |
|Animator   |@animator   |@animator   |
|StateListAnimator   |@animator   |@stateListAnimator   |
|color `int`   |@color   |@color   |
|ColorStateList   |@color   |@colorStateList   |

##数据对象

任何 POJO 都能用在 data binding 中，但是更改 POJO 并不会同步更新 UI。data binding 的强大之处就在于它可以让你的数据拥有更新通知的能力。这里有三种不同的数据变动通知机制，[Observable 对象](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html#observable_objects)，[observable 域](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html#observablefields)，与 [observable 容器类](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html#observable_collections)。

当以上的 observable 对象绑定在 UI 上，数据发生变化时，UI 就会同步更新。

###Observable 对象

当一个类声明了 [`Observable`](http://developer.android.com/reference/android/databinding/Observable.html) 接口时，data binding 会设置一个 listener 在绑定的对象上，以便监听对象域的变动。

[`Observable`](http://developer.android.com/reference/android/databinding/Observable.html) 接口有一个添加/移除 listener 的机制，但通知取决于开发者。为了简化开发，我们创建了一个基类 [`BaseObservable`](http://developer.android.com/reference/android/databinding/BaseObservable.html)，来实现 listener 注册机制。这个类也实现了域变动的通知，你只需要在 getter 上使用 [`Bindable`](http://developer.android.com/reference/android/databinding/Bindable.html) 注解，并在 setter 中实现通知。

	private static class User extends BaseObservable {
	   private String firstName;
	   private String lastName;
	   @Bindable
	   public String getFirstName() {
	       return this.firstName;
	   }
	   @Bindable
	   public String getLastName() {
	       return this.lastName;
	   }
	   public void setFirstName(String firstName) {
	       this.firstName = firstName;
	       notifyPropertyChanged(BR.firstName);
	   }
	   public void setLastName(String lastName) {
	       this.lastName = lastName;
	       notifyPropertyChanged(BR.lastName);
	   }
	}

`Bindable` 注解会在编译时在 BR 类内生成一个元素。而 BR 类会生成在 module 的 package 下。如果数据基类不可修改，[`Observable`](http://developer.android.com/reference/android/databinding/Observable.html) 接口的存储和 listener 通知可以用 [`PropertyChangeRegistry`](http://developer.android.com/reference/android/databinding/PropertyChangeRegistry.html) 来实现。

###Observable域

创建 [`Observable`](http://developer.android.com/reference/android/databinding/Observable.html) 类还是需要花费一点时间的，如果开发者想要省时，或者数据类的域很少的话，可以使用 [`ObservableField`](http://developer.android.com/reference/android/databinding/ObservableField.html) 以及它的派生 [`ObservableBoolean`](http://developer.android.com/reference/android/databinding/ObservableBoolean.html)，[`ObservableByte`](http://developer.android.com/reference/android/databinding/ObservableByte.html)，[`ObservableChar`](http://developer.android.com/reference/android/databinding/ObservableChar.html)，[`ObservableShort`](http://developer.android.com/reference/android/databinding/ObservableShort.html)，[`ObservableInt`](http://developer.android.com/reference/android/databinding/ObservableInt.html)，[`ObservableLong`](http://developer.android.com/reference/android/databinding/ObservableLong.html)，[`ObservableFloat`](http://developer.android.com/reference/android/databinding/ObservableFloat.html)，[`ObservableDouble`](http://developer.android.com/reference/android/databinding/ObservableDouble.html)，[`ObservableParcelable`](http://developer.android.com/reference/android/databinding/ObservableParcelable.html)。`ObservableFields` 是单一域的自包含 observable 对象。原始版本避免了在存取过程中做打包/解包操作。要使用它，在数据类中创建一个 public final 域：

	private static class User {
	   public final ObservableField<String> firstName =
	       new ObservableField<>();
	   public final ObservableField<String> lastName =
	       new ObservableField<>();
	   public final ObservableInt age = new ObservableInt();
	}
	
就这么简单！要存取数据，只需要使用 get set 方法：

	user.firstName.set("Google");
	int age = user.age.get();
	
###Observable 容器类

一些应用会使用更加灵活的结构来保持数据。Observable 容器类允许使用 key 来获取这类数据。当 key 是类似 String 的一类引用类型时，使用 [`ObservableArrayMap`](http://developer.android.com/reference/android/databinding/ObservableArrayMap.html) 会非常方便。

	ObservableArrayMap<String, Object> user = new ObservableArrayMap<>();
	user.put("firstName", "Google");
	user.put("lastName", "Inc.");
	user.put("age", 17);
	
在布局中，可以用 String key 来获取 map 中的数据：

	<data>
	    <import type="android.databinding.ObservableMap"/>
	    <variable name="user" type="ObservableMap&lt;String, Object>"/>
	</data>
	…
	<TextView
	   android:text='@{user["lastName"]}'
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	<TextView
	   android:text='@{String.valueOf(1 + (Integer)user["age"])}'
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	   
当 key 是整数类型时，可以使用 [`ObservableArrayList`](http://developer.android.com/reference/android/databinding/ObservableArrayList.html)：

	ObservableArrayList<Object> user = new ObservableArrayList<>();
	user.add("Google");
	user.add("Inc.");
	user.add(17);
	
在布局文件中，使用下标获取列表数据：

	<data>
	    <import type="android.databinding.ObservableList"/>
	    <import type="com.example.my.app.Fields"/>
	    <variable name="user" type="ObservableList&lt;Object>"/>
	</data>
	…
	<TextView
	   android:text='@{user[Fields.LAST_NAME]}'
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	<TextView
	   android:text='@{String.valueOf(1 + (Integer)user[Fields.AGE])}'
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	   
##生成Binding

生成的 binding 类将布局中的 View 与变量绑定在一起。就像先前提到过的，类名和包名可以[自定义](http://developer.android.com/intl/zh-cn/tools/data-binding/guide.html#custom_binding_class_names)。生成的 binding 类会继承 [`ViewDataBinding`](http://developer.android.com/reference/android/databinding/ViewDataBinding.html)。

###创建

binding 应该在 inflate 之后创建，确保 View 的层次结构不会在绑定前被干扰。绑定布局有好几种方式。最常见的是使用 binding 类中的静态方法。inflate 函数会 inflate View 并将 View 绑定到 binding 类上。此外有更加简单的函数，只需要一个 LayoutInflater 或一个 ViewGroup：

	MyLayoutBinding binding = MyLayoutBinding.inflate(layoutInflater);
	MyLayoutBinding binding = MyLayoutBinding.inflate(layoutInflater, viewGroup, false);
	
如果布局使用不同的机制来 inflate，则可以独立做绑定操作：

	MyLayoutBinding binding = MyLayoutBinding.bind(viewRoot);
	
有时绑定关系是不能提前确定的。这种情况下，可以使用 [`DataBindingUtil`](http://developer.android.com/reference/android/databinding/DataBindingUtil.html) ：

	ViewDataBinding binding = DataBindingUtil.inflate(LayoutInflater, layoutId,
	    parent, attachToParent);
	ViewDataBinding binding = DataBindingUtil.bindTo(viewRoot, layoutId);

###带有 ID 的 View

布局中每一个带有 ID 的 View，都会生成一个 public final 域。binding过程会做一个简单的赋值，在 binding 类中保存对应 ID 的 View。这种机制相比调用 findViewById 效率更高。举个例子：

	<layout xmlns:android="http://schemas.android.com/apk/res/android">
	   <data>
	       <variable name="user" type="com.example.User"/>
	   </data>
	   <LinearLayout
	       android:orientation="vertical"
	       android:layout_width="match_parent"
	       android:layout_height="match_parent">
	       <TextView android:layout_width="wrap_content"
	           android:layout_height="wrap_content"
	           android:text="@{user.firstName}"
	   android:id="@+id/firstName"/>
	       <TextView android:layout_width="wrap_content"
	           android:layout_height="wrap_content"
	           android:text="@{user.lastName}"
	  android:id="@+id/lastName"/>
	   </LinearLayout>
	</layout>

将会在 binding 类内生成：

	public final TextView firstName;
	public final TextView lastName;
	
ID 在 data binding 中并不是必需的，但是在某些情况下还有有必要对 View 进行操作。

###变量

每一个变量会有相应的存取函数：

	<data>
	    <import type="android.graphics.drawable.Drawable"/>
	    <variable name="user"  type="com.example.User"/>
	    <variable name="image" type="Drawable"/>
	    <variable name="note"  type="String"/>
	</data>
	
并在 binding 类中生成对应的 getter setter：

	public abstract com.example.User getUser();
	public abstract void setUser(com.example.User user);
	public abstract Drawable getImage();
	public abstract void setImage(Drawable image);
	public abstract String getNote();
	public abstract void setNote(String note);
	
###ViewStub

[`ViewStub`](http://developer.android.com/reference/android/view/ViewStub.html) 相比普通 View 有一些不同。ViewStub 一开始是不可见的，当它们被设置为可见，或者调用 inflate 方法时，ViewStub 会被替换成另外一个布局。

因为 `ViewStub` 实际上不存在于 View 结构中，binding 类中的类也得移除掉，以便系统回收。因为 binding 类中的 View 都是 final 的，所以我们使用了一个叫 [`ViewStubProxy`](http://developer.android.com/reference/android/databinding/ViewStubProxy.html) 的类来代替 `ViewStub`。开发者可以使用它来操作 ViewStub，获取 `ViewStub` inflate 时得到的视图。

但 inflate 一个新的布局时，必须为新的布局创建一个 binding。因此，`ViewStubProxy` 必须监听 `ViewStub` 的 [`ViewStub.OnInflateListener`](http://developer.android.com/reference/android/view/ViewStub.OnInflateListener.html)，并及时建立 binding。由于 ViewStub 只能有一个 OnInflateListener，你可以将你自己的 listener 设置在 `ViewStubProxy` 上，在 binding 建立之后， listener 就会被触发。

###高级 binding

####动态变量

有时候，有一些不可知的 binding 类。例如，[`RecyclerView.Adapter`](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html) 可以用来处理不同布局，这样的话它就不知道应该使用哪一个 binding 类。而在 [`onBindViewHolder(VH, int)`](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html#onBindViewHolder(VH,%20int)) 的时候，binding 类必须被赋值。

在这种情况下，RecyclerView 的布局内置了一个 item 变量。`BindingHolder` 有一个 `getBinding` 方法，返回一个 [`ViewDataBinding`](http://developer.android.com/reference/android/databinding/ViewDataBinding.html) 基类。

	public void onBindViewHolder(BindingHolder holder, int position) {
	   final T item = mItems.get(position);
	   holder.getBinding().setVariable(BR.item, item);
	   holder.getBinding().executePendingBindings();
	}

####直接 binding

当变量或者 observable 发生变动时，会在下一帧触发 binding。有时候 binding 需要马上执行，这时候可以使用 [`executePendingBindings()`](http://developer.android.com/reference/android/databinding/ViewDataBinding.html#executePendingBindings())。

####后台线程

只要数据不是容器类，你可以直接在后台线程做数据变动。Data binding 会将变量/域转为局部量，避免同步问题。

##属性 Setter

当绑定数据发生变动时，生成的 binding 类必须根据 binding 表达式调用 View 的 setter 函数。Data binding 框架内置了几种自定义赋值的方法。

###自动 Setter

对一个 attribute 来说，data binding 会尝试寻找对应的 setAttribute 函数。属性的命名空间不会对这个过程产生影响，只有属性的命名才是决定因素。

举个例子，针对一个与 TextView 的 `android:text` 绑定的表达式，data binding会自动寻找 `setText(String)` 函数。如果表达式返回值为 int 类型， data binding则会寻找 `setText(int)` 函数。所以需要小心处理函数的返回值类型，必要的时候使用强制类型转换。需要注意的是，data binding 在对应名称的属性不存在的时候也能继续工作。你可以轻而易举地使用 data binding 为任何 setter “创建” 属性。举个例子，support 库中的 DrawerLayout 并没有任何属性，但是有很多 setter，所以你可以使用自动 setter 的特性来调用这些函数。

	<android.support.v4.widget.DrawerLayout
	    android:layout_width="wrap_content"
	    android:layout_height="wrap_content"
	    app:scrimColor="@{@color/scrim}"
	    app:drawerListener="@{fragment.drawerListener}"/>
	    
###重命名 Setter

一些属性的命名与 setter 不对应。针对这些函数，可以用 [`BindingMethods`](http://developer.android.com/reference/android/databinding/BindingMethods.html) 注解来将属性与 setter 绑定在一起。举个例子，`android:tint` 属性可以这样与 [`setImageTintList(ColorStateList)`](http://developer.android.com/reference/android/widget/ImageView.html#setImageTintList(android.content.res.ColorStateList)) 绑定，而不是 `setTint`:

	@BindingMethods({
	       @BindingMethod(type = "android.widget.ImageView",
	                      attribute = "android:tint",
	                      method = "setImageTintList"),
	})
	
Android 框架中的 setter 重命名已经在库中实现了，开发者只需要专注于自己的 setter。

###自定义 Setter

一些属性需要自定义 setter 逻辑。例如，目前没有与 `android:paddingLeft` 对应的 setter，只有一个 `setPadding(left, top, right, bottom)` 函数。结合静态 binding adapter 函数与 [`BindingAdapter`](http://developer.android.com/reference/android/databinding/BindingAdapter.html) 注解可以让开发者自定义属性 setter。

Android 属性已经内置一些 BindingAdapter。例如，这是一个 `paddingLeft` 的自定义 setter：

	@BindingAdapter("android:paddingLeft")
	public static void setPaddingLeft(View view, int padding) {
	   view.setPadding(padding,
	                   view.getPaddingTop(),
	                   view.getPaddingRight(),
	                   view.getPaddingBottom());
	}
	
Binding adapter 在其他自定义类型上也很好用。举个例子，一个 loader 可以在非主线程加载图片。

当存在冲突时，开发者创建的 binding adapter 会覆盖 data binding 的默认 adapter。

你也可以创建多个参数的 adapter：

```
@BindingAdapter({"bind:imageUrl", "bind:error"})
public static void loadImage(ImageView view, String url, Drawable error) {
   Picasso.with(view.getContext()).load(url).error(error).into(view);
}
```

```  
<ImageView app:imageUrl=“@{venue.imageUrl}”
app:error=“@{@drawable/venueError}”/>
```

当 **imageUrl** 与 **error** 存在时这个 adapter 会被调用。imageUrl 是一个 String，error 是一个 Drawable。

- 在匹配时自定义命名空间会被忽略
- 你可以为 android 命名空间编写 adapter

Binding adapter 方法可以获取旧的赋值。只需要将旧值放置在前，新值放置在后：

	@BindingAdapter("android:paddingLeft")
	public static void setPaddingLeft(View view, int oldPadding, int newPadding) {
	   if (oldPadding != newPadding) {
	       view.setPadding(newPadding,
	                       view.getPaddingTop(),
	                       view.getPaddingRight(),
	                       view.getPaddingBottom());
	   }
	}
	
事件 handler 仅可用于只拥有一个抽象方法的接口或者抽象类。例如：

	@BindingAdapter("android:onLayoutChange")
	public static void setOnLayoutChangeListener(View view, View.OnLayoutChangeListener oldValue,
	       View.OnLayoutChangeListener newValue) {
	    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
	        if (oldValue != null) {
	            view.removeOnLayoutChangeListener(oldValue);
	        }
	        if (newValue != null) {
	            view.addOnLayoutChangeListener(newValue);
	        }
	    }
	}
	
当 listener 内置多个函数时，必须分割成多个 listener。例如，[`View.OnAttachStateChangeListener`](http://developer.android.com/reference/android/view/View.OnAttachStateChangeListener.html) 内置两个函数：[`onViewAttachedToWindow()`](http://developer.android.com/reference/android/view/View.OnAttachStateChangeListener.html#onViewAttachedToWindow(android.view.View)) 与 [`onViewDetachedFromWindow()`](http://developer.android.com/reference/android/view/View.OnAttachStateChangeListener.html#onViewDetachedFromWindow(android.view.View))。在这里必须为两个不同的属性创建不同的接口。

	@TargetApi(VERSION_CODES.HONEYCOMB_MR1)
	public interface OnViewDetachedFromWindow {
	    void onViewDetachedFromWindow(View v);
	}
	
	@TargetApi(VERSION_CODES.HONEYCOMB_MR1)
	public interface OnViewAttachedToWindow {
	    void onViewAttachedToWindow(View v);
	}
	
因为改变一个 listener 会影响到另外一个，我们必须编写三个不同的 adapter，包括修改一个属性的，和修改两个属性的。

	@BindingAdapter("android:onViewAttachedToWindow")
	public static void setListener(View view, OnViewAttachedToWindow attached) {
	    setListener(view, null, attached);
	}
	
	@BindingAdapter("android:onViewDetachedFromWindow")
	public static void setListener(View view, OnViewDetachedFromWindow detached) {
	    setListener(view, detached, null);
	}
	
	@BindingAdapter({"android:onViewDetachedFromWindow", "android:onViewAttachedToWindow"})
	public static void setListener(View view, final OnViewDetachedFromWindow detach,
	        final OnViewAttachedToWindow attach) {
	    if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB_MR1) {
	        final OnAttachStateChangeListener newListener;
	        if (detach == null && attach == null) {
	            newListener = null;
	        } else {
	            newListener = new OnAttachStateChangeListener() {
	                @Override
	                public void onViewAttachedToWindow(View v) {
	                    if (attach != null) {
	                        attach.onViewAttachedToWindow(v);
	                    }
	                }
	
	                @Override
	                public void onViewDetachedFromWindow(View v) {
	                    if (detach != null) {
	                        detach.onViewDetachedFromWindow(v);
	                    }
	                }
	            };
	        }
	        final OnAttachStateChangeListener oldListener = ListenerUtil.trackListener(view,
	                newListener, R.id.onAttachStateChangeListener);
	        if (oldListener != null) {
	            view.removeOnAttachStateChangeListener(oldListener);
	        }
	        if (newListener != null) {
	            view.addOnAttachStateChangeListener(newListener);
	        }
	    }
	}
	
上面的例子比普通情况下复杂，因为 View 是 add/remove [`View.OnAttachStateChangeListener`](http://developer.android.com/reference/android/view/View.OnAttachStateChangeListener.html) 而不是 set。`android.databinding.adapters.ListenerUtil` 可以用来辅助跟踪旧的 listener 并移除它。

对应 [`addOnAttachStateChangeListener(View.OnAttachStateChangeListener)`](http://developer.android.com/reference/android/view/View.html#addOnAttachStateChangeListener(android.view.View.OnAttachStateChangeListener)) 支持的 api 版本，通过向 `OnViewDetachedFromWindow` 和 `OnViewAttachedToWindow` 添加 `@TargetApi(VERSION_CODES.HONEYCHOMB_MR1)` 注解，data binding 代码生成器会知道这些 listener 只会在 Honeycomb MR1 或更新的设备上使用。

##转换器

###对象转换

当 binding 表达式返回对象时，会选择一个 setter（自动 Setter，重命名 Setter，自定义 Setter），将返回对象强制转换成 setter 需要的类型。

下面是一个使用 ObservableMap 保存数据的例子：

	<TextView
	   android:text='@{userMap["lastName"]}'
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>

在这里，`userMap` 会返回 Object 类型的值，而返回值会被自动转换成 `setText(CharSequence)` 所需要的类型。当对参数类型存在疑惑时，开发者需要手动做类型转换。

###自定义转换

有时候会自动在特定类型直接做类型转换。例如，当设置背景的时候：

	<View
	   android:background="@{isError ? @color/red : @color/white}"
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	   
在这里，背景需要的是 `Drawable`，但是 color 是一个整数。当需要 `Drawable` 却返回了一个整数时，`int` 会自动转换成 `ColorDrawable`。这个转换是在一个 BindingConversation 注解的静态函数中实现：

@BindingConversion
public static ColorDrawable convertColorToDrawable(int color) {
   return new ColorDrawable(color);
}

需要注意的是，这个转换只能在 setter 阶段生效，所以 **不允许** 混合类型：

	<View
	   android:background="@{isError ? @drawable/error : @color/white}"
	   android:layout_width="wrap_content"
	   android:layout_height="wrap_content"/>
	   
###Android Studio 对 Data binding 的支持

Android Studio 支持 data binding 表达式的高亮，并会在编辑器中标出表达式中的语法错误。

在预览窗口显示的是 data binding 表达式的默认值。下面是一个设置默认值的例子，`TextView` 的 text 默认值为 `PLACEHOLDER`。

	<TextView android:layout_width="wrap_content"
	   android:layout_height="wrap_content"
	   android:text="@{user.firstName, default=PLACEHOLDER}"/>
	   
如果你需要在设计阶段显示默认值，你可以使用 tools 属性代替默认值表达式，详见[设计阶段布局属性](http://tools.android.com/tips/layout-designtime-attributes)。
