# AWTimeProfiler

iOS平台因为UIKit本身的特性,需要将所有的UI操作都放在主线程执行,所以也造成不少程序员都习惯将一些线程安全性不确定的逻辑,以及其它线程结束后的汇总工作等等放到了主线,所以主线程中包含的这些大量计算、IO、绘制都有可能造成卡顿。
在Xcode中已经集成了非常方便的调试工具Instruments,它可以帮助我们在开发测试阶段分析软件运行的性能消耗,但一款软件经过测试流程和实验室分析肯定是不够的,在正式环境中由大量用户在使用过程中监控、分析到的数据更能解决一些隐藏的问题。

### 1.寻找卡顿切入点
监控卡顿，说白了就是找到主线程都在干些啥。 我们知道一个线程的消息事件处理都是依赖于NSRunLoop来驱动,所以要知道线程正在调用什么方法，就需要从NSRunLoop来入手。
RunLoop的执行代码大致如下:
```objc
{
/// 1. 通知Observers，即将进入RunLoop
/// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
do {

/// 2. 通知 Observers: 即将触发 Timer 回调。
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
/// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

/// 4. 触发 Source0 (非基于port的) 回调。
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);

/// 6. 通知Observers，即将进入休眠
/// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);

/// 7. sleep to wait msg.
mach_msg() -> mach_msg_trap();


/// 8. 通知Observers，线程被唤醒
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);

/// 9. 如果是被Timer唤醒的，回调Timer
__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);

/// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);

/// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);


} while (...);

/// 10. 通知Observers，即将退出RunLoop
/// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}

```

从上可以看出RunLoop处理事件的时间主要出在两个阶段:
+ kCFRunLoopBeforeSources和kCFRunLoopBeforeWaiting之间
+ kCFRunLoopAfterWaiting之后


### 2.RunLoop 函数
我们可以使用CFRunLoopObserverRef来监控NSRunLoop的状态,通过它可以实时获得这些状态值的变化。
1. 设置Runloop observer的运行环境
CFRunLoopObserverContext context = {0, (__bridge void *)self, NULL, NULL};


2. 创建Runloop observer对象
第一个参数：用于分配observer对象的内存
第二个参数：用以设置observer所要关注的事件，详见回调函数myRunLoopObserver中注释
第三个参数：用于标识该observer是在第一次进入runloop时执行还是每次进入runloop处理时均执行
第四个参数：用于设置该observer的优先级
第五个参数：用于设置该observer的回调函数
第六个参数：用于设置该observer的运行环境
CFRunLoopObserverCreate(<#CFAllocatorRef allocator#>, <#CFOptionFlags activities#>, <#Boolean repeats#>, <#CFIndex order#>, <#CFRunLoopObserverCallBack callout#>, <#CFRunLoopObserverContext *context#>)


3. 将新建的observer加入到当前thread的runloop
CFRunLoopAddObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);


4. 将observer从当前thread的runloop中移除
CFRunLoopRemoveObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);


5. 释放 observer
CFRelease(_observer); _observer = NULL;

### 3.信号量
```objc
//创建信号量，参数：信号量的初值，如果小于0则会返回NULL
dispatch_semaphore_create（信号量值）

//等待降低信号量
dispatch_semaphore_wait（信号量，等待时间）

//提高信号量
dispatch_semaphore_signal(信号量)

```
注意：正常的使用顺序是先降低然后再提高，这两个函数通常成对使用。

### 4.量化卡顿的程度
原理：
利用观察Runloop各种状态变化的持续时间来检测计算是否发生卡顿
一次有效卡顿采用了“N次卡顿超过阈值T”的判定策略，即一个时间段内卡顿的次数累计大于N时才触发采集和上报：举例，卡顿阈值T=500ms、卡顿次数N=1，可以判定为单次耗时较长的一次有效卡顿；而卡顿阈值T=50ms、卡顿次数N=5，可以判定为频次较快的一次有效卡顿。
实践：
我们需要开启一个子线程,实时计算两个状态区域之间的耗时是否到达某个阀值。另外卡顿需要覆盖到多次连续小卡顿和单次长时间卡顿两种情景。

### 5.测试用例
人为设置卡顿(休眠)，来测试我们实时监控困顿的代码是否有效。

### 6.记录卡顿数据
当检测到卡顿时,抓取堆栈信息,然后在客户端做一些过滤处理,(Debug)可以保存在本地，(Release)可以上传服务器，通过收集一定量的卡顿数据后，经过分析便能准确定位需要优化的地方。
获取堆栈信息后，可以使用Demo中AWCallStack类来解析。
