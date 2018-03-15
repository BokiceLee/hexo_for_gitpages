---
title: 基础知识之Foundation知识点
date: 2018-03-15 16:33:27
tags: iOS
categories: 技术
---

# Foundation基础

## NSObject

- 几乎是所有类的基类或协议
- isKindOfClass(是否是某一类)/isMemberOfClass(是否是某一类或其子类)
- respondsToSelector
- description
- encodeWithCoder，initWithCoder NSCoding仅有成员
- conformsToProtocol
- 类对象中的isa指向的类结构被称为metaclass，与[object class]有区别，比如KVO时
- __weak如何实现对象值自动设置为nil

## NSString & NSMutableString

- 属性用copy/strong

## NSArray & NSMutableArray

- 遍历方式，倒序遍历方式(reverseObjectEnumerator)
- 初始化方法
- 浅拷贝:使用copy、arrayWithArray，copyWithZone均为浅拷贝，数组元素均为就对象
- 深拷贝[[NSArray alloc] initWithArray:someArray copyItems:YES];且item必须实现NSCopying协议
- 拷贝
  - 对非集合对象拷贝
    - [immutableObject copy] 浅拷贝
    - [immutableObject mutableCopy] 深拷贝
    - [mutableObject copy] 深拷贝
    - [mutableObject mutableCopy] 深拷贝
  - 对集合对象拷贝
    - [immutableObject copy] 浅拷贝
    - [immutableObject mutableCopy] 单层深拷贝
    - [mutableObject copy] 单层深拷贝
    - [mutableObject mutableCopy] 单层深拷贝

## NSDictionary & NSMutableDictionary

取值时最好判断object的类型

## NSNumber & NSInteger & NSRange

NSNumber专门用来封装基础类型，包括整型，单精度，双精度和字符类型

## NSNull

用于处理Model中可能出现的空值。不同于nil

## NSData

字节缓冲区:datawithContentsOfURL

## NSUserDefaults

用于APP setting

## NSDate & NSDateFormatter & NSCalendar

判断获取时间:components:fromDate:toDate:options:

获取时间戳

## NSCoding & NSCoder

仅有的两个方法，进行数据的序列化和反序列化

encodeWithCoder:(NSCoder *)aCoder;

initWithCoder:(NSCoder *)aDecoder;

## NSCopying & NSZone

alllocWithZone:深拷贝，NSCopying唯一required方法

## NSAutoreleasePool

降低内存峰值，在Runloop休眠前释放自动释放池中的对象

## NSFileManager

删除文件之前先判断是否存在

## NSTimer

并不精确，若错过则等下一趟

## NSLog

## NSClassFromString & NSStringFromClass

## NSIndexPath

用于tableview

## NSError

## NSException

## NSStringEncoding

## NSProgressIndicator

## NSBundle

项目资源包

## NSNetServiceBrowser

## NSValue

可以包装任意对象，可以用NSValue将struct存到NSArray和NSDictionary中

## NSURLConnection

## NSURLSession & NSURLSessionTask

## NSURLRequest

## NSInputStream & NSOutputStream

## NSPredicate

## NSLayoutConstraint

## NSLock & NSRecursiveLock & NSCondition

## NSMethodSignature

## NSInvocation

## NSSet

## NSUrl

## AVPlayer

## NSNotificationCenter