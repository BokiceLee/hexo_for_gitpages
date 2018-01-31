---
title: CocoaAsyncSocket
date: 2018-01-23 17:20:10
tags: iOS
categories: 技术
---

### CocoaAsyncSocket简介

CocoaAsyncSocket是谷歌的开发者基于BSD-Socket写的IM框架，为Mac和iOS提供了易于使用的强大的异步套接字库，对于tcp和udp协议，CocoaAsyncSocket分别封装了AsyncSocket和AsyncUdpSocket两个类，向上封装简单易用OC接口，省去了用户(程序员)面向socket(tcp)以及数据流(udp)编程的麻烦。

### GCD vs RunLoop

#### GCD版本

当前CocoaAsyncSocket框架基于GCD主要封装了GCDAsyncSocket和GCDAsyncUdpSocket两个类。

#### RunLoop版本

在7.5.0版本之前CocoaAsyncSocket有一个RunLoop版本，封装了AsyncSocket和AsyncUdpSocket两个类，从2016年的7.5.0版本起，考虑到功能相似等其他原因，为了投入更多精力在GCD版本上，开发者将RunLoop从该版本中移除，并不再提供维护([github上相关issue](https://github.com/robbiehanson/CocoaAsyncSocket/issues/454))，项目中相关的[wiki](https://github.com/robbiehanson/CocoaAsyncSocket/wiki/Reference_AsyncSocket)也已经被清空。<u>**从这一点来看，我们的项目进行版本的迁移非常有必要。**</u>

#### 对比

GCDAsyncSocket版本完全依赖于GCD，从而保证了:

- 线程安全

  委托回调方法均异步dispatch到初始化时指定的队列中，从而保证了socket和委托/处理的可并发性。

  socket运行在内部的socket queue中，因此是线程安全的，socket queue可在初始化时指定，若未指定socket queue，则自动创建。

```
/**
 * GCDAsyncSocket uses the standard delegate paradigm,
 * but executes all delegate callbacks on a given delegate dispatch queue.
 * This allows for maximum concurrency, while at the same time providing easy thread safety.
 * 
 * You MUST set a delegate AND delegate dispatch queue before attempting to
 * use the socket, or you will get an error.
 * 
 * The socket queue is optional.
 * If you pass NULL, GCDAsyncSocket will automatically create it's own socket queue.
 * If you choose to provide a socket queue, the socket queue must not be a concurrent queue.
 * If you choose to provide a socket queue, and the socket queue has a configured target queue,
 * then please see the discussion for the method markSocketQueueTargetQueue.
 * 
 * The delegate queue and socket queue can optionally be the same.
**/
- (instancetype)init;
- (instancetype)initWithSocketQueue:(nullable dispatch_queue_t)sq;
- (instancetype)initWithDelegate:(nullable id<GCDAsyncSocketDelegate>)aDelegate delegateQueue:(nullable dispatch_queue_t)dq;
- (instancetype)initWithDelegate:(nullable id<GCDAsyncSocketDelegate>)aDelegate delegateQueue:(nullable dispatch_queue_t)dq socketQueue:(nullable dispatch_queue_t)sq;

```



- 性能的提高
  内部利用了大量技术如kqueue(macOS上高效的IO复用技术)来减少系统调用并优化了缓冲区分配，从而实现了更高的性能。

AsyncSocket RunLoop版本则完全基于RunLoop

- 线程安全

  框架提供了一个函数用于RunLoop的设置，当创建一个socket的时候，默认将socket添加到当前RunLoop中，同时框架允许将socket移到其他自定义的RunLoop和设置RunLoopModes中

  ```
  /**
   * When you create an AsyncSocket, it is added to the runloop of the current thread.
   * So for manually created sockets, it is easiest to simply create the socket on the thread you intend to use it.
   * 
   * If a new socket is accepted, the delegate method onSocket:wantsRunLoopForNewSocket: is called to
   * allow you to place the socket on a separate thread. This works best in conjunction with a thread pool design.
   * 
   * If, however, you need to move the socket to a separate thread at a later time, this
   * method may be used to accomplish the task.
   * 
   * This method must be called from the thread/runloop the socket is currently running on.
   * 
   * Note: After calling this method, all further method calls to this object should be done from the given runloop.
   * Also, all delegate calls will be sent on the given runloop.
  **/
  - (BOOL)moveToRunLoop:(NSRunLoop *)runLoop;

  ```

  框架提供了以下函数用于检查线程安全

  ```
  - (void)checkForThreadSafety
  {
  	if (theRunLoop && (theRunLoop != CFRunLoopGetCurrent()))
  	{
  		// AsyncSocket is RunLoop based.
  		// It is designed to be run and accessed from a particular thread/runloop.
  		// As such, it is faster as it does not have the overhead of locks/synchronization.
  		// 
  		// However, this places a minimal requirement on the developer to maintain thread-safety.
  		// If you are seeing errors or crashes in AsyncSocket,
  		// it is very likely that thread-safety has been broken.
  		// This method may be enabled via the DEBUG_THREAD_SAFETY macro,
  		// and will allow you to discover the place in your code where thread-safety is being broken.
  		// 
  		// Note:
  		// 
  		// If you find you constantly need to access your socket from various threads,
  		// you may prefer to use GCDAsyncSocket which is thread-safe.
  		
  		[NSException raise:AsyncSocketException
  		            format:@"Attempting to access AsyncSocket instance from incorrect thread."];
  	}
  }
  ```

  注释指出了RunLoop版本是为单个线程而设计的，因此不做锁/同步机制，以此保证了高性能，<u>线程的安全需要由用户(程序员)来保证</u>。

  ​

#### Which one?

[groups.google.com](https://groups.google.com/forum/#!topic/cocoaasyncsocket/l4JWV3R0_00)上作者Robbie的对于两个版本的观点如下，

- GCDAsyncSocket要求Mac OSX 10.6+ or iOS 4.0+，GCDAsyncSocket的性能大部分情况下远远优于AsyncSocket
- 只有当在iOS上使用SSL/TLS时两者性能才比较接近。此时使用GCDAsyncSocket并不能带来很大的性能提升。

**"[When I switched to GCDAsyncSocket] I ended up creating a dedicated queue to process the incoming data. ... Which ended up moving all [processing] off the main thread. ... My UI is now much smoother during the update process!"**

### 可行性

#### iOS版本

iOS 4发布时间为2010年3月，根据[苹果官方设备版本统计](https://developer.apple.com/support/app-store/)如下图，目前在有93%的移动设备上iOS为iOS10以上。

![苹果官方设备系统版本统计](/Users/gengshuchen/Desktop/statistics.png)

考虑到极少数地区设备较为落后，调研目前可能使用低于iOS4版本的设备如下

| 设备       | iPhone          | iPhone 3G      | iPhone 3GS    |
| -------- | --------------- | -------------- | ------------- |
| 初始版本操作系统 | iPhone OS 1.0   | iPhone OS. 2.0 | iPhone OS 3.0 |
| 最新版本操作系统 | iPhone OS 3.1.3 | iOS 4.2.1      | iOS 6.1.6     |

| 设备        | iPod Touch(第一代) | iPod Touch(第二代) | iPod Touch(第三代) | iPad(第一代)     |
| --------- | --------------- | --------------- | --------------- | ------------- |
| 初始版本操作系统  | iPhone OS 1.1   | iPhone OS 2.1.1 | iPhone OS 3.1.1 | iPhone OS 3.2 |
| 最新版本的操作系统 | iPhone OS 3.1.3 | iOS 4.2.1       | iOS 5.1.1       | iOS 5.1.1     |

*系统版本信息参考[维基百科](https://en.wikipedia.org/wiki/List_of_iOS_devices)，版本高于iPhone OS3的设备未列出(iOS4版本之前称为iphone OS)*

**<u>综上，在采用GCDAsyncSocket时需要考虑上述设备的兼容。</u>**

### 总结

RunLoop版本目前已经不再维护，同时其线程安全需要由程序员保证，大部分情况下性能远低于GCD版本，因此对CocoaAsyncSocket的迁移是非常有必要的。