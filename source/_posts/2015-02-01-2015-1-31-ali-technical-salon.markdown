---
layout: post
title: "2015-1-31 阿里技术沙龙 - Android应用性能优化实践"
date: 2015-02-01 11:42:04 +0800
comments: true
categories: android
---

周末跑去深圳参加了一场阿里主持的技术沙龙，主题是《如何构建高可用的APP》，沙龙中相关的ppt和视频可以在他们的[微博](http://weibo.com/taobaodeveloperclub)中找到。沙龙中收获比较大的是有关UC的何杰分享的Android应用性能优化实践，和手Q web业务优化的解析。  

<!--more-->

Android应用性能优化实践中，提及了比较多性能优化的干货。一般我们对于性能的调试多是依赖开发者工具，而uc对性能优化做了额外有趣而且有创意的事情。  
uc对应用有六个硬性的指标，分别是以下几点：  

 - 启动时间
 - 性能
 - 稳定性
 - 内存占用
 - 电量消耗
 - apk大小

应该是这六点，希望我没有记错。其中性能是一个比较重要而又比较难的难点。性能问题造成的因素很多，产品功能因素，代码原因，设备配置，设备运行软件数量，是否安装安全软件，rom适配，等等。有很多问题，在开发测试甚至灰度测试的时候难以发现，依赖用户反馈时，对问题描述量化等也比较难。  
uc在性能优化的思路是，先确保主路径的流畅，再追求整体的卡顿优化；线下分析结合线上监控查询定位性能问题；先解决卡，再解决顿。  
工具上，先是老生常谈的开发者工具，TraceView，Overdraw，Systrace（这个比较高级），StrictMode，Hierarchy Viewer等是必定要用的调试工具。接着是关键点的打点统计，对启动时间，相应速度的监控（输出时间差）。接着就是用户反馈的分析，ANR（Application not response）日志分析，StrictMode下的ANR分析，以及我第一次了解到的Looper Hook调试方法。我个人认为的重点如下： 
 
 - 用户反馈分类：按照使用功能，发生频率，用户类型分类  
 - StrictMode ANR更加严格，会暴露更多可能的潜在问题
 - Looper Hook：在UI线程以外开启一条线程，定时向UI线程post一个runnable，并记下post时间。runnable的内容是将执行时间同步回发送线程，如果UI线程被阻塞，那么post过去的runnable不能被准时执行，那么同步回去的执行时间会与post时间有这较大的差值，设定几个阀值（1000ms或5000ms），用来评测不同情况下卡UI线程的情况，并可以通过log Message.what的值和Message.callback的类型来判断发生场景。对于这里，建议大家看下Looper，Message，Handler，MessageQueue的源码实现。

而在优化的细节上，uc也总结了很多方案（200+）。有一些关键点有：

 - 分段加载
 - 延迟加载：耗时操作延后执行，执行时机通过post runnable到UIThread触发，表明UIThread已经idle，可以尝试进行耗时操作
 - 缓存复用：不只是数据，View也可以复用，在Activity rotate的时候，在View被destory前保存View实例，在Activity recreate之后将View重新add到layout中，只需要重新measure layout draw，省略new View的过程
 - 运行时线程管理防止抢占资源，造成UIThread运行卡顿
 - 引入异步dns及cache防止获取网络代理卡顿
 - 解决start第三方Activity外部crash导致app卡死：采用外壳Activity方案解决，即start第三方Activity时采用：Activity -> ShieldActivity -> Third party Activity 的调用方法。
 - 尝试开启GPU加速（难点，代价比较大）
 - SharedPreference：commit()是UIThread线程执行，如果保存数据过大，可能卡UIThread，换用apply()。
 - 安全软件拦截：沟通反馈

在举出案例之后，也给出了一些总结：

 - 培养异步化的思维，不只是开发，产品也需要多考虑这一方面
 - 不要尝试假定用户可能的使用场景在一个小的范围内，实际情况可能很极限（举了一个用户下载列表8k多条目的例子）
 - 预加载 + 闲时加载 + 按需加载
 - 线程限制管理（控制并发） + 任务队列
 - 压力测试
 - 防御式编程（[Wiki](http://zh.wikipedia.org/wiki/%E9%98%B2%E5%BE%A1%E6%80%A7%E7%BC%96%E7%A8%8B)）
 - 禁止UIThread做如下操作：文件IO读写，耗CPU操作，IPC同步

洋洋洒洒居然写了这么多，由此可以看出这个分享的干货之多。真心十分感谢阿里的这次技术分享，真心学到东西了。沙龙的ppt和视频应该近期在他们的微博就会放出，大家可以关注一下。


