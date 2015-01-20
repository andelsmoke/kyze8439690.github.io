---
layout: post
title: "修复android 2.3 不能以RGBA8888格式解析图片的问题"
date: 2013-09-03 22:58
comments: true
categories: android
---

在android 2.3,解析bitmap的时候，不能用RGBA8888格式解析图片成bitmap，只能解析成RGB565,就算加了Config也一样。
在android google code project上的解释是这样的：[https://code.google.com/p/android/issues/detail?id=13038](https://code.google.com/p/android/issues/detail?id=13038)   
大致意思就是，原本记录了这个issue，然后被脚本自动标记为close了，导致这个bug没有修复。好吧，看来只能自己解决这个问题了。
在网上搜了一番，偶然发现一个曲线救国的方法。  
{% gist 7408999 convertBitmap.java %}  
这个函数是最近在研究zxing解析本地图片时候发现的，突然发现这个函数貌似能用与解决不能解析为RGBA8888的bug，在测试之后，果然生效了。使用方法如下：

	Bitmap bitmap = convert(bitmap, Bitmap.Config.ARGB_8888);

这个函数用的也是很曲线救国的方法了，用canvas将原来的bitmap画到属性为RGBA8888的空bitmap上，成功绕过了android 2.3不能直接解析为RGBA8888的bug。  
（吐槽：做android开发越久越觉得android坑啊……）