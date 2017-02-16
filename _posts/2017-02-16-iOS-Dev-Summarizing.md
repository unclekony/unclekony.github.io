---
layout: post
title: "iOS开发知识点总结"
date: 2016-02-16
excerpt: "对iOS开发中runtime、多线程等知识点的归纳总结"
---

## 底层原理

#### weak的实现
>
runtime为每个类对象都维护了一张全局的weak\_table，里面的key是赋值对象的内存地址，value是weak修饰符的属性对象内存地址。当某个对象delloc的时候，系统会检查该类对象是否存在弱引用，如果存在则检查于之对应的weak\_table，将所有能匹配到这个对象的value重新指向nil。

[weak的生命周期：具体实现方法](!http://www.cocoachina.com/ios/20150605/11990.html)

#### kvc的实现
>
kvc主要是根据类对象的isa指针和方法名，从method\_list里面找到对应方法的函数指针，然后结合参数去执行。

#### kvo的实现
>
kvo主要运用了isa-swizzling的方法，在调用addObserve方法的时候，讲对象的isa指针指向了一个新的继承于当前类的子类对象，并添加了一个相同名字的setter方法在方法列表里面，在setter方法里面加上了willChangeValueForKey/didChangeValueForKey的触发。当我们调用了setter方法的时候，会先触发will然后再调用原类对象的setter方法再调用did。当调用removeObserve方法的时候，该对象的isa指针又重新指向了原类对象了。

[isa-swizzling?什么鬼？](http://www.nscookies.com/isa-swizzling/)

[从自己实现isa-swizzling到说一些Runtime的内容](http://www.pluto-y.com/isa-swizzling-and-runtime/)

#### GCD的实现
>
主要由libdispatch、Libc和XNU内核实现了GCD，通过C语言层面的管理线程的容器和FIFO队列维护这个过程。

#### block实现原理
![Objective-C下的Block](http://okxyl92j3.bkt.clouddn.com/Objective-C%E7%9A%84block_1.jpg)

![C++下的Block](http://okxyl92j3.bkt.clouddn.com/Objective-C%E7%9A%84block_2.jpg)

>
- 从画线和画圈处得知block本质上是一个函数指针，而且这个函数指针是包含在一个结构体里面，该结构体还会包含block参数的字段。
- 箭头处可以发现为何说block会引用self。

## iOS内存管理机制

#### 基础部分
>
iOS的内存管理是通过引用数来管理对象的。在MRC的时期，需要自己手动地调用retain/release方法，在ARC时期，在静态编译阶段，编译器会分析代码在适当的地方添加retain/release/autorelease方法。

#### autorelease

>
每一个线程在每一个runloop开始的时候都会创建一个autoreleasepool，系统会将autoreleased的对象放到这个pool里面。当这个runloop要结束的时候，系统会将pool里面的所有autoreleased对象进行释放。我们可以通过@autoreleasepool关键字去创建一个局部的autoreleasepool，当超过了这个作用域的时候，里面的autoreleased对象都会全部释放。@autoreleasepool适合解决创建大量临时对象。

#### 问题
>
- 用什么方法在创建之后会放到autoreleasepool中？


## block

#### 概念
>
具备保存上下文变量内存块的函数指针。OC的闭包。

#### __block实现原理
![Objective-C下的__block](http://okxyl92j3.bkt.clouddn.com/Objective-C%E4%B8%8B%E7%9A%84__block_1.jpg)

![C++下的__block](http://okxyl92j3.bkt.clouddn.com/Objective-C%E4%B8%8B%E7%9A%84__block_2.jpg)

> 
- __block其实是由一个传值过程变为传地址的过程。

## 多线程

#### NSThread
>
GCD和NSOperation会把对象添加到autoreleasepool中，但NSThread需要自己手动创建并触发释放。

### GCD

#### 串行、并行、同步、异步
>
- 串行、并行是针对线程队列，同步、异步是针对线程的任务。 
- 系统对串行队列的调度是单条线程把任务一个一个地执行。
- 系统对并行队列的调度是由系统管理多个线程去完成任务，任务之前无需等待。
- 同步(sync)指当前线程会阻塞，等待同步线程完成当前任务后再去执行。
- 异步(async)指当前线程不需要阻塞，可以直接执行当前任务。
- dispatch\_get\_main\_queue是串行队列。
- dispatch\_get\_global\_queue是并行队列。

#### 锁种类
>
[iOS 中的各种锁](http://www.cocoachina.com/ios/20161129/18216.html)

#### GCD实现同步
>
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
	dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
	dispatch_aync(queue, ^{
		dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW,10ull *  NSEC_PER_SEC);
		dispatch_after(time, queue, ^{
			NSLog("111111");
			dispatch_semaphore_signal(semaphore);
		}
	})
	dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
	NSLog("2222222");

#### GCD实现单例
>
	+ (instancetype)shareInstance
	{
    	static dispatch_once_t onceToken = 0;
    	static id instance;
    	dispatch_once(&onceToken, ^{
        	instance = [[self alloc] init];
    	});
    	return instance;
	}

## runtime

#### 概念
>
runtime是运行于操作系统与应该用程序的一个中间层，为OC消息转发机制、运行时动态生成类、对象以和方法，消息转发机制，以及Method Swizzling提供基础，并且暴露一套C接口给应用者调用。

#### Object、Class、MetaClass的关系
NSObject的实现
![NSObject](http://okxyl92j3.bkt.clouddn.com/NSObject%E5%AE%9E%E7%8E%B0%E6%88%AA%E5%9B%BE.jpg)

Class的实现
![Class](http://okxyl92j3.bkt.clouddn.com/Class%E5%AE%9E%E7%8E%B0%E6%88%AA%E5%9B%BE.jpg)

objc_class的实现
![objc_class](http://okxyl92j3.bkt.clouddn.com/objc_class%E5%AE%9E%E7%8E%B0%E6%88%AA%E5%9B%BE.jpg)

isa是对象指向的对应Class结构体的指针
![isa](http://okxyl92j3.bkt.clouddn.com/isa%E6%8C%87%E9%92%88%E6%88%AA%E5%9B%BE.jpg)

>
在OC下面everything都是对象，instance是对象，class也是对象。

#### Method、IMP、SEL的关系
>
objc\_method\_list指向的列表存储的正式Method。

(下图非官方公开代码)
![Method](http://okxyl92j3.bkt.clouddn.com/objc_method%E5%AE%9E%E7%8E%B0%E6%88%AA%E5%9B%BE.jpg)

#### “调用方法”的根本是“消息派发”

通过*clang -rewrite-objc* 把OC代码转成C++得到下图。
![消息派发](http://okxyl92j3.bkt.clouddn.com/%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8%E7%9A%84%E5%AE%9E%E9%99%85%E8%BF%87%E7%A8%8B.jpg)

>
runtime会在当前isa指向的objc\_class结构体里面的objc\_cache搜索对应的Method,如果没有则会在objc\_method\_list继续搜索，如果还没有会在super\_class逐层往上找。

#### 消息转发机制
>
当runtime找不到某个方法的时候，它会给开发者机会去处理，这个处理的机制就是消息转发机制，而我们常说的消息转发机制其实分两个过程，动态决议和消息转发。

![消息转发机制](http://okxyl92j3.bkt.clouddn.com/%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E8%BF%87%E7%A8%8B.jpg)

>
- 动态决议的resolveInstanceMethod和resolveClassMethod主要是针对当前对象。
- 消息转发forwardingTargetForSelector和forwardInvocation是针对其他对象。

#### Method swizzling
[Method Swizzling 和 AOP 实践](http://tech.glowing.com/cn/method-swizzling-aop/)


## runloop

#### 概念
>
- runloop其实就是程序的事件循环机制，我们程序能不断地响应键盘、触碰等事件全靠它。
- runloop管理着所有我们需要处理的事件和消息，这些事件和消息都会经过“接受”->“等待”->“处理”这些状态。

#### runloop与线程的关系
>
系统有一个全局Map管理线程对象和runloop对象的一对一关系。当第一次去获取runloop对象的时候系统才会去初始化这个map，才会把主线程的runloop放到map管理。

#### runloop跟NSTimer的关系
[关于NSRunLoop和NSTimer的深入理解](http://www.superqq.com/blog/2016/05/05/ios-nsrunllop-nstimer/)

#### 原理实现
这一块后面单独写一篇详细的，给出链接。








