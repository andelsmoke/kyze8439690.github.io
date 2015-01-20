---
layout: post
title: "Android developer tools（开发人员工具）"
date: 2015-01-20 15:01:18 +0800
comments: true
categories: android
---
在做Android开发的过程中，不可避免地需要使用到自带的android developer tools（开发人员工具），这是一个强大的开发辅助工具，随着android版本的更新，developer tools也集成了越来越多十分方便的调试功能，这里以android 4.4.4版本为例子，说说其中一部分我常用工具的使用（恕我才疏学浅没能全部懂用）。  

<!--more-->

### **显示布局边界**  
这个工具用于显示普通view布局的size，margin等属性，实际使用场景为：查看view的实际位置，检查界面是由普通view拼装而成或用surfaceView（WebView）实现。  
{% img /images/show_border.jpg 400 640 显示布局边界 %}  

### **强制使用从右到左的布局**
也就是RTL布局（Right-to-Left），由于一些国家地区的语言习惯，书写阅读是从右到左（类似中国古代的书写习惯）。有些开发者可能很奇怪，在2.3或以上的android版本，padding，margin，gravity等，是left，right，top，bottom组合，到了3.0或以后，就多了个**start**和**end**，这个也是为了RTL布局而添加的。一般开发者都会无视这个选项，沿用常见的左右布局，但是如果你做的是国际化应用的话，就需要考虑RTL布局了。而这个开发者选项，就是让开发者可以在**不切换语言区域的情况下，调试RTL布局**。  
{% img /images/show_rtl.jpg 400 640 强制使用从右到左的布局 %} 

### **显示GPU视图更新**
随着android版本的更新，越来越多的绘制操作能使用GPU来完成，详见[http://developer.android.com/guide/topics/graphics/hardware-accel.html](http://developer.android.com/guide/topics/graphics/hardware-accel.html)，而这个工具打开之后，使用GPU绘制的区域会用红色来标注，而没有红色标注的区域，则是使用CPU绘制的。这个选项也可以用来查看redraw的区域大小。  
{% img /images/show_gpu_redraw.jpg 400 640 显示GPU视图更新 %} 

### **调试GPU过度绘制**
这个就是经典了Overdraw问题(过度绘制)了。我们知道，动画是通过一帧一帧不断重绘，连续播放实现的。而在绘制完成的时候，每一个像素点可能已经**不可避免地**被绘制了一次以上，由于每个像素点最终显示出来的只有一种颜色（假设透明度100%），所以先前绘制的操作都是无用的，这就是overdraw了。Overdraw会造成什么问题呢？为了达到60FPS的绘制速率，每一次绘制，都需要在16ms内完成（1000ms / 60），如果绘制的时间过长，就会导致绘制占用的时间过长，用户在使用时就会有卡顿的感觉。需要注意的一点是：**Overdraw是无法避免的，我们只能尽量通过优化减少他。**打开调试GPU过度绘制选项（显示过度绘制区域），屏幕会显示一些从浅绿到深红的色块，这些色块指示出了overdraw的程度，颜色越往深红靠，说明overdraw越严重。由于overdraw无法避免，要整个界面都显示白色几乎不可能，我们只能尽量让红色的区域减少。如果你发现你的app几乎整个都是深红色的，那说明你需要好好优化一下布局了。  
{% img /images/show_overdraw.jpg 400 640 调试GPU过度绘制 %} 

### **GPU呈现模式分析**
这个选项用于显示绘制速率。打开时屏幕下部会显示绘制速率条形图和一条横线。该横线表示在它以下的绘制时间少于16ms，绘制是流畅的。如果条形超出了横线，说明当前发生了绘制的卡滞，需要根据当前操作优化程序，提供一致的流畅体验。  
{% img /images/show_fps.jpg 400 640 GPU呈现模式分析 %} 

### **不保留活动**
这个选项用于调试activity被销毁的情况。当系统可用内存不足，新程序要求分配内存的时候，除了gc以外，系统可能会把后台的activity destroy掉以回收内存，为了能恢复被destory activity的状态，系统在destory时会调用[onSaveInstanceState(Bundle)](http://developer.android.com/reference/android/app/Activity.html#onSaveInstanceState(android.os.Bundle\))方法，将activity状态保存在bundle中；在activity重启的onCreate(Bundle)方法中，将保存的状态恢复。  
需要注意的是saveInstance是一个遍历操作，从高层到底层，换句话说，从activity到fragment，fragment到viewgroup，viewgroup到view，一层一层地调用saveInstance方法。反之，恢复的时候也是这样。Fragment/ViewGroup/View的状态保存需要一个条件：必须拥有id或者tag。这是保存状态时使用的标识。而什么状态需要保存呢？聚个栗子🌰，activity/view/viewGroup/fragment中的自定义变量，没有用持久化保存（sqlite，sharedPreference等）的数据，都需要用bundle保存起来。  

回到主题，不保留活动选项，正是用来调试这个情况的。由于你无法预知系统什么时候会回收activity释放内存空间，我们需要手动触发他。打开这个选项之后，每一个不是visible的activity，都会直接被调用onSavedInstance并destory掉。所以，你只需要打开不保留活动，打开你要测试的activity，按home键回到桌面，再返回这个activity并观察现象尽可。如果状态恢复有问题，这时候应该会发生一个NPE（NullPointerException），或者视图上的数据显示有问题。这时再根据情况修改代码即可。
  
最后附上cyrilmottier大大有关android状态恢复的[ppt](/attachments/deepdiveintoandroidrestorationbycyrilmottier-140924114735-phpapp02.pdf)，网络情况的好的直接看[原链接](http://www.slideshare.net/parisandroidug/deep-dive-into-android-restoration-droidcon-paris-2014)。