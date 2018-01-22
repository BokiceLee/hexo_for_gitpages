---
title: 深入理解RunLoop
date: 2018-01-22 16:17:48
tags: iOS
categories: 技术
---

*拜读ibireme大神的博客[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)  ，此为手记，希望后续有更深的理解*

### RunLoop概念

​	Event Loop模型:让线程能随时处理时间但并不退出，基本代码逻辑

```
function loop(){
  initialize();
  do{
    var message=get_next_message();
    process_message(message);
  }while(message!=quit);
}
```

实现关键点:如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用，在有消息到来时立刻被唤醒。

因此RunLoop实际上是一个对象，管理了需要处理的事件和消息，并提供了一个入口函数来执行上面Event Loop的逻辑。

OSX/IOS系统中，提供了两个这样的对象，NSRunLoop和CFRunLoopRef

- CFRunLoopRef 基于CoreFoundation框架内，提供了纯C函数的API，所有这些API都是线程安全的。
- NSRunLoop 基于CFRunLoopRef封装，提供了面向对象的API，API非线程安全

### RunLoop与线程的关系

IOS开发中的线程对象:pthread_t和NSThread

- pthread_t:
  - Pthread_main_thread_np()
  - pthread_self()
- NSThread:
  - [NSThread mainThread]
  - [NSThread currentThread]

CFRunLoop是基于pthread来管理的，苹果不允许 直接创建RunLoop，而提供自动获取的函数，CFRunLoopGetMain()和CFRunLoopGetCurrent()，从其内部实现可以看出，线程创建时不会有RunLoop，RunLoop的创建发生在第一次获取时，而RunLoop的销毁发生在线程结束时，因此只能在线程的内部获取其RunLoop(因为只能调用CFRunLoopGetCurrent接口，主线程除外)

### RunLoop对外的接口

CoreFoundation关于RunLoop的类有:

- CFRunLoopRef
- CFRunLoopModeRef:对外不暴露，只是通过CFRunLoopRef的接口进行了封装
- CFRunLoopSourceRef
- CFRunLoopTimerRef
- CFRunLoopObserverRef

一个RunLoop包含了若干个Mode，每个Mode包含若干个Source/Timer/Observer，每次调用RunLoop的主函数时，只能制定其中一个Mode，这个Mode被称作CurrentMode，如果需要切换Mode，只能退出Loop，再重新制定一个Mode进入，从而分隔开不同组的Source/Timer/Observer，让其互不影响。

#### CFRunLoopSourceRef

事件产生的地方，Source有两个版本，Source0和Source1 。

- Source0只包含一个回调函数指针，并不能主动触发事件。使用时需要线调用CFRunLoopSourceSignal(source)，将source标记为待处理，然后手动调用CFRunLoopWakeUp(runloop)来唤醒RunLoop，让其处理这个事件。
- Source1包含了一个mach_port和一个回调函数指针，被用于通过内核和其他线程相互发送消息，这种Source能主动唤醒RunLoop的线程。

#### CFRunLoopTimerRef

基于时间的触发器，和NSTimer是toll-free bridged的，可以混用，包含一个时间长度和一个回调函数指针，当加入到RunLoop时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒来执行这个回调。

#### CFRunLoopObserverRef

观察者，每个Observer都包含一个回调函数指针，当RunLoop的状态发生变化时，观察者能通过回调接受到这个变化。可以观察的时间点有

- kCFRunLoopEntry
- kCFRunLoopBeforeTimers
- kCFRunLoopBeforeSources
- kCFRunLoopBeforeWaiting
- kCFRunLoopAfterWaiting
- kCFRunLoopExit

上述的Source/Timer/Observer统称为mode item，一个item可以被同时加入多个mode，但是一个item被重复加入同一个mode时不会有效果的，如果一个mode中一个item都没有，则RunLoop会直接退出，不进入循环。

### RunLoop的Mode

CFRunLoopMode:

```
struct __CFRunLoopMode{
  CFStringRef _name;
  CFMutableSetRef _sources0;
  CFMutableSetRef _sources1;
  CFMutableArrayRef _observers;
  CFMutableArrayRef _timers;
};
```

CFRunLoop:

```
struct __CFRunLoop{
  CFMutableSetRef _commonModes;
  CFMutableSetRef _commonModeItems;
  CFRunLoopModeRef _currentMode;
  CFMutableSetRef _modes;
}
```

CommonModes:一个Mode将自己标记为"Common"属性(具体实现是将其ModeName添加到RunLoop的"commonModes"中)。每当RunLoop的内容发生变化时，RunLoop都会自动将_commonModeItems里的Source/Observer/Timer同步到具有"Common"标记的所有Mode里。

若想要一个Timer在两个Mode中都能得到回调，一种方法是将Timer分别加入到这两个Mode中，另一种方法是将Timer加入到顶层RunLoop的"commonModeItems"中，"commonModeItems"北RunLoop自动更新到所有具有"Common"属性的Mode里去。

CFRunLoop对外暴露的管理Mode接口只有两个，

```
CFRunLoopAddCommonMode(CFRunLoopRef runloop,CFStringRef modeName);
CFRunLoopRunInMode(CFStringRef modeName,...);
```

Mode暴露的管理mode item的接口有

```
CFRunLoopAddSource(CFRunLoopRef rl,CFRunLoopSourceRef source,CFStringRef modeName);
CFRunLoopAddObserver(CFRunLoopRef rl,CFRunLoopObserverRef observer,CFStringRef modeName);
CFRunLoopAddTimer(CFRunLoopRef rl,CFRunLoopTimerRef timer,CFStringRef mode);
CFRunLoopRemoveSource(CFRunLoopRef rl,CFRunLoopSourceRef source,CFStringRef modeName);
CFRunLoopRemoveObserver(CFRunLoopRef rl,CFRunLoopObserverRef observer,CFStringRef modeName);
CFRunLoopRemoveTimer(CFRunLoopRef rl,CFRunLoopTimerRef timer,CFStringRef mode);
```

只能通过mode name来操作内部的mode，当传入一个新的mode name但RunLoop内部没有对应的mode时，RunLoop会自动创建对应的CFRunLoopModeRef，对于一个RunLoop来说，其内部的mode只能增加不能删除。

苹果公开提供的Mode有两个，

- kCFRunLoopDefaultMode(NSDefaultRunLoopMode)
- UITrackingRunLoopMode

可以用这两个Mode Name来操作对应的Mode。

同时苹果还提供了一个操作COmmon标记的字符串，kCFRunLoopCommonModes(NSRunLoopCommonModes)，可以用这个字符串来操作Common Items或标记一个Mode为"Common",需要注意区分"Common"和其他mode name

### RunLoop的内部逻辑

### RunLoop的底层实现

RunLoop的核心是基于mach port的，其进入休眠时调用的函数是mach_msg(),虾面介绍OSX/IOS的系统架构。

苹果官方将整个系统大致划分为4个层次，

- 应用层 包括用户能接触到的图形应用
- 应用框架层 开发人员接触到的Cocoa等框架
- 核心框架层包括各种核心框架/OpenGL等内容
- Darwin即操作系统的核心，包括系统内核/驱动/shell等内容，此层为开源。

Darwin在硬件层上的组成部分为Mach/BSD/IOKit，共同组成XNU内核。

- XNU内核的内环被称作Mach，其作为微内核，仅提供了诸如处理器调度，IPC等非常少量的基础业务。
- BSD层可以看作是围绕Mach层的一个外环，其提供了诸如进程管理/文件系统和网络等功能。
- IOKit层是为设备驱动提供了一个面向对象的一个框架。

Mach中，所有东西都是通过自己的对象实现的，进程/线程/虚拟内存都被称为对象，Mach的对象间不能直接调用，只能通过消息传递的方式实现对象间的通信，"消息"是Mach中最基础的概念，消息在两个端口之间传递，就是Mach的IPC的核心。

Mach消息实际上是一个二进制数据包(BLOB)，其头部定义了当前端口local_port和目标端口remote_port，发送和接收消息是通过同一个API进行的，其option标记了消息传递的方向，

```
mach_msg_return_t mach_msg(
	mach_msg_header_t *msg,
	mach_msg_option_t option,
	mach_msg_size_t send_size,
	mach_msg_size_t rcv_size,
	mach_port_name_t rcv_name,
	mach_msg_timeout_t timeout,
	mach_port_name_t notify
);
```

为了实现消息的发送和接收，mach_msg()函数实际上调用了一个Mach陷阱，即函数mach_msg_trap()，在用户态调用mach_msg_trap()时会触发陷阱机制，切换到内核态，内核态中内核实现的mach_msg()函数会完成实际的工作。

RunLoop的核心是一个mach_msg()，RunLoop调用这个函数去接收消息，如果没有别人发送port消息过来，内核会将线程置于等待状态。

### 苹果用RunLoop实现的功能

App 启动后，系统默认注册了5个Mode，

- kCFRunLoopDefaultMode:App的默认Mode，通常主线程是在这个Mode下运行的
- UITrackingRunLoopMode:界面跟踪Mode，用于ScrollView跟踪触摸华东，保证界面滑动时不受其他Mode影响。
- UIInitializationRunLoopMode:在刚启动App时进入的第一个Mode，启动完成后不再使用。
- GSEventReceiveRunLoopMode:接收系统时间的内部Mode，通常用不到
- kCFRunLoopCommonModes，占位Mode，没有实际作用

当RunLoop进行回调时，一般都是通过一个很长的函数调用出去的，

#### AutoreleasePool

​	todo

#### 事件响应

​	苹果注册了一个基于mach port的Source1来接收系统事件，当一个硬件事件(触摸/锁屏/摇晃)发生后，首先由IOKit.framework生成一个IOHIDEvent事件并由SpringBoard接收，SpringBoard只接收按键/触摸/加速/接近传感器这几种Event，随后用mach port转发给需要的App进程，随后苹果注册的这个Source1就会触发回调，并调用响应的函数进行内部的分发。

#### 手势识别

​	todo

#### 界面更新

​	当在操作UI时，如果改变了Frame，更新了UIView/CALayer的层次，或者手动调用了相关方法，UIView/CALayer就被标记为待处理，并被提交到一个全局的容器去，苹果注册了一个Observer监听BeforeWaiting和Exit事件，回调执行一个函数，在函数中遍历所有待处理的UIView/CALayer以执行实际的绘制和调整，并更新UI界面。

#### 定时器

​	NSTimer(与CFRunLoopTimerRef时toll-free bridged的)注册到RunLoop后，RunLoop会为其重复的时间点注册好时间，RunLoop会在Tolerance之内回调Timer，若超时则等待下一个时间点。CADisplayLink与此类似，实现原理不同。

#### PerfromSelector

​	调用该方法实际上内部创建了一个NSTimer添加到当前线程的RunLoop中，如果当前线程没有RunLoop，则方法失效。(存疑)

#### 关于GCD

​	GCD提供的某些接口用到了RunLoop，例如dispatch_async()到主线程时，libsDispatch会唤醒主线程的RunLoop，并在回调里执行对象的block，dispatch到其他线程由libDispatch处理，与上述机制不同。

#### 关于网络请求

​	iOS中网络请求的接口层次如下:

- CFSocket 最底层的接口，只负责socket通信
- CFNetwork 基于CFSocket等接口的上层封装，ASIHttpRequest工作与这一层
- NSURLConnection 基于CFNetwork的更高层的封装，提供面向对象的接口，AFNetworking工作与这一层
- NSURLSession iOS7新增的接口，底层用到了NSURLConnection的部分功能。AFNetworking2和Alamofire工作于这一层

##### NSURLConnection的工作过程

使用NSURLConnection时，通常会传入一个Delegate，当调用了[connection start]后，这个Delegate会不停收到时间回调，实际上，start这个函数内部会获取CurrentRunLoop，然后在其中的DefaultMoe添加4个Source0，CFMultiplexerSource时负责各种Delegate回调的，CFHTTPCookieStorage时处理各种cookie的。

当开始网络传输时，NSURLConnection创建了两个新线程，NSURLConnectionLoader和CFSocket，其中CFSocket线程用于处理底层socket连接，NSURLCOnnectionLoader线程内部使用RunLoop来接收底层socket的事件，并通过之前添加的Source0通知上层的Delegate。

NSURLConnectionLoader中的RunLoop通过一些基于mach port的Source接收来自底层CFSocket的通知，当收到通知后，其会在何时的时机向CFMultiplexerSource等Source0发送通知，同时唤醒Delegate线程的RunLoop来让其处理这些通知，CFMultiplexerSource会在Delegate线程的RunLoop对Delegate执行实际的回调。

### RunLoop的实际应用举例

#### AFNetworking

​	todo

#### AsyncDisplayKit

​	todo

