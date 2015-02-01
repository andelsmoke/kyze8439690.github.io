---
layout: post
title: "(译)在ViewGroup中处理Touch Events"
date: 2013-10-17 00:18
comments: true
categories: Android, Translate
---
  
在一个[```ViewGroup```][1]中处理touch events需要格外注意。因为在[```ViewGroup```][1]里面有着各种要处理不同touch events的子view，这是很常见的。为了确保每个view能正确地获取到属于他的touch events，我们必须覆盖[```onInterceptTouchEvent()```][2]函数。

<!--more-->

------------
####ViewGroup中的Touch Events
当在一个[```ViewGroup```][1]表面，或者里面的子view表面检测到一个touch event的时候，会调用[```ViewGroup```][1]的[```onInterceptTouchEvent()```][2]函数。如果[```onInterceptTouchEvent()```][2]函数返回true时，这个MotionEvent将会被截断，也就意味着，不会传递给子view，而是交给父层([```ViewGroup```][1])的[```onTouchEvent```][4]函数处理。

[```onInterceptTouchEvent()```][2]方法提供了一个让父层在所以子view之前先获取到touch event的途径。如果你在[```onInterceptTouchEvent()```][2]中返回了true的话，先前获取到touch events的子view会接收到一个[```ACTION_CANCEL```][3]，而先前的events会传到父层的[```onTouchEvent```][4]做处理。[```onInterceptTouchEvent()```][2]也能直接返回false去简单地观察touch events按view hierarchy传递给他们原先的目标（子view），用子view的[```onTouchEvent```][4]去处理这些events。

<!-- more -->

在下面这些代码段中，MyViewGroup继承于[```ViewGroup```][1]，并包含了多个子view。如果你在子view上横向拖动，子view不再会获取到touch events，而MyViewGroup会处理这些touch events来滚动他的内容。但是，如果你按下子view中的button，或者纵向滚动子view，父层不会截获这些touch events，因为子view才是传递的目标。在这种情况中，[```onInterceptTouchEvent()```][2]应该返回false，这样MyViewGroup的[```onTouchEvent```][4]才不会被调用。

    public class MyViewGroup extends ViewGroup {

	    private int mTouchSlop;
	
	    ...
	
	    ViewConfiguration vc = ViewConfiguration.get(view.getContext());
	    mTouchSlop = vc.getScaledTouchSlop();
	
	    ...
	
	    @Override
	    public boolean onInterceptTouchEvent(MotionEvent ev) {
	        /*
	         * 这个方法只是决定我们是否要截获这个手势.
	         * 如果我们返回true，那么onTouchEvent 就会被调用，然后我们就开始进行滚动的操作
	         */
	
	
	        final int action = MotionEventCompat.getActionMasked(ev);
	
	        // 先处理触摸手势完成的情况
	        if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
	            // 释放滚动.
	            mIsScrolling = false;
	            return false; // 不会截获touch event，让子view处理他
	        }
	
	        switch (action) {
	            case MotionEvent.ACTION_MOVE: {
	                if (mIsScrolling) {
	                    // 处于正在滚动的状态，因此要截获touch event！
	                    return true;
	                }
	
	                // 如果用户手指横向移动量超过了阀值，则开始滚动
	
	                // 作为读者的练习:)
	                final int xDiff = calculateDistanceX(ev); 
	
	                // 触摸阀值应该用ViewConfiguration常量来计算
	                if (xDiff > mTouchSlop) { 
	                    // 开始滚动!
	                    mIsScrolling = true;
	                    return true;
	                }
	                break;
	            }
	            ...
	        }
	
	        // 通常情况下，我们不会想要截获touch event，它们应该让子view来处理
	        return false;
	    }
	
	    @Override
	    public boolean onTouchEvent(MotionEvent ev) {
	        // 这里我们开始处理touch event (e.g. 如果action是ACTION_MOVE,滚动container）。
	        // 这个方法只有在touch event在onInterceptTouchEvent中被截获时调用
	        ...
	    }
	}

需要注意的是[`ViewGroup`][1]也提供了一个[`requestDisallowInterceptTouchEvent()`][5]函数。ViewGroup可以通过调用这个函数，让子view阻止所有父层利用[```onInterceptTouchEvent()```][2]来截取touch event。

-----------
####使用ViewConfiguration常量

上面的代码中使用了[`ViewConfiguration`][6]去初始化一个叫mTouchSlop的变量。你可以使用[`ViewConfiguration`][6]类去获取Android系统内置的距离，速度，次数等。

"Touch slop" 可以解释为一个用户手势被判断为滑动的距离。Touch slop通常用来防止用户的其他触摸行为（如触摸屏幕上的元素）被判断为滑动。

另外两个最常使用的[`ViewConfiguration`][6]方法是[`getScaledMinimumFlingVelocity()`][7]和[`getScaledMaximumFlingVelocity()`][8]。这两个方法会返回初始化滑动的最大速度值和最小速度值，以像素/秒为单位。例如：

    ViewConfiguration vc = ViewConfiguration.get(view.getContext());
	private int mSlop = vc.getScaledTouchSlop();
	private int mMinFlingVelocity = vc.getScaledMinimumFlingVelocity();
	private int mMaxFlingVelocity = vc.getScaledMaximumFlingVelocity();
	
	...
	
	case MotionEvent.ACTION_MOVE: {
	    ...
	    float deltaX = motionEvent.getRawX() - mDownX;
	    if (Math.abs(deltaX) > mSlop) {
	        // 发生了滑动
	    }
	
	...
	
	case MotionEvent.ACTION_UP: {
	    ...
	    } if (mMinFlingVelocity <= velocityX && velocityX <= mMaxFlingVelocity
	            && velocityY < velocityX) {
	        // 条件被满足
	    }
	}

------------
####扩展一个子view的点击热区

Android提供了一个[`TouchDelegate`][9]类去让父类扩展它的子view的触摸区域。当子view很小但需要大的触摸区域的时候，这个类大有用处。如果你想要的话，你也能用这个类去缩小子view的触摸区域。

在下面的例子中，有一个作为例子的[`ImageButton`][10](也就是说父类会扩展这个子view的触摸区域)

    <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	     android:id="@+id/parent_layout"
	     android:layout_width="match_parent"
	     android:layout_height="match_parent"
	     tools:context=".MainActivity" >
	 
	     <ImageButton android:id="@+id/button"
	          android:layout_width="wrap_content"
	          android:layout_height="wrap_content"
	          android:background="@null"
	          android:src="@drawable/icon" />
	</RelativeLayout>

下面的代码段会完成下面的事情：

- 获取父view并post一个[`Runnable`](http://developer.android.com/reference/java/lang/Runnable.html "Runnable")到UI线程。这会确保父类在调用[`getHitRect()`](http://developer.android.com/reference/android/view/View.html#getHitRect(android.graphics.Rect))方法前先勾画出他的子类。[`getHitRect()`](http://developer.android.com/reference/android/view/View.html#getHitRect(android.graphics.Rect))方法的作用是在父类的坐标系中获取子view的hit rectangle（触摸区域）。
- 找到[`ImageButton`][10]子view并调用[`getHitRect()`](http://developer.android.com/reference/android/view/View.html#getHitRect(android.graphics.Rect))去获取子类触摸区域范围。
- 扩展[`ImageButton`][10]的hit rectangle范围。
- 初始化[`TouchDelegate`][9]对象，参数是扩展后的hit rectangle和[`ImageButton`][10]子view。
- 在父view中设置[`TouchDelegate`][9]，这样在这个触摸范围内的touch event都会传给[`ImageButton`][10]

在[`ImageButton`][10]子view的触摸范围容量内，父view会接收所有的touch events，如果touch event发生在子类的hit rectangle内，父类会将touch event传给子类做处理。

	public class MainActivity extends Activity {
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        // 获取父view
	        View parentView = findViewById(R.id.parent_layout);
	        
	        parentView.post(new Runnable() {
	            // post到父类的消息队列中，确保在调用getHitRect()前勾画出子类
	            @Override
	            public void run() {
	                // 实例view的区域范围（ImageButton）
	                Rect delegateArea = new Rect();
	                ImageButton myButton = (ImageButton) findViewById(R.id.button);
	                myButton.setEnabled(true);
	                myButton.setOnClickListener(new View.OnClickListener() {
	                    @Override
	                    public void onClick(View view) {
	                        Toast.makeText(MainActivity.this, 
	                                "Touch occurred within ImageButton touch region.", 
	                                Toast.LENGTH_SHORT).show();
	                    }
	                });
	     
	                // ImageButton的hit rectangle
	                myButton.getHitRect(delegateArea);
	            
	                // 在ImageButton边框的右边和底边扩展触摸区域
	                delegateArea.right += 100;
	                delegateArea.bottom += 100;
	            
	                // 初始化TouchDelegate.
	                // "delegateArea" is the bounds in local coordinates of 
	                // the containing view to be mapped to the delegate view.
	                // "myButton" is the child view that should receive motion
	                // events.
	                TouchDelegate touchDelegate = new TouchDelegate(delegateArea, 
	                        myButton);
	     
	                // Sets the TouchDelegate on the parent view, such that touches 
	                // within the touch delegate bounds are routed to the child.
	                if (View.class.isInstance(myButton.getParent())) {
	                    ((View) myButton.getParent()).setTouchDelegate(touchDelegate);
	                }
	            }
	        });
	    }
	}

  [1]: http://developer.android.com/reference/android/view/ViewGroup.html
  [2]: http://developer.android.com/reference/android/view/ViewGroup.html#onInterceptTouchEvent(android.view.MotionEvent)
  [3]: http://developer.android.com/reference/android/view/MotionEvent.html#ACTION_CANCEL
  [4]: http://developer.android.com/reference/android/view/View.html#onTouchEvent(android.view.MotionEvent)
  [5]: http://developer.android.com/reference/android/view/ViewGroup.html#requestDisallowInterceptTouchEvent(boolean)
  [6]: http://developer.android.com/reference/android/view/ViewConfiguration.html
  [7]: http://developer.android.com/reference/android/view/ViewConfiguration.html#getScaledMinimumFlingVelocity()
  [8]: http://developer.android.com/reference/android/view/ViewConfiguration.html#getScaledMaximumFlingVelocity()
  [9]: http://developer.android.com/reference/android/view/TouchDelegate.html
  [10]: http://developer.android.com/reference/android/widget/ImageButton.html
  