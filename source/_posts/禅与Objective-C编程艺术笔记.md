---
title: 禅与Objective-C编程艺术笔记
date: 2018-03-15 16:38:35
tags: iOS
categories: 技术
---

# 禅与Objective-c编程艺术

## 条件语句

- 使用大括号

- 不用尤达表达式

- nil和bool的检查使用！而不是==

- 避免将方法重要的部分嵌套到分支上

- 复杂表达式赋值给变量

- 三元运算符:object?:[self createObject];

- 错误处理:先对方法返回值进行判断而不是处理错误变量，因为成功情况下会对error参数写入垃圾值，此时如果检查error将导致错误。

  ```
  NSError *error=nil;
  if(![self trySomethingWithError:&error]){
    //handle Error
  }
  ```

## Case语句

- 多行语句时需要加大括号
- 使用enum时使用NS_ENUM

## 命名

- 推荐使用长的，描述性的方法和变量名
- Constants常量应该使用驼峰命名法，并用相关类名作为前缀，推荐使用 static const 声明
- 方法
  - 方法类型后应有一个空格
  - 方法段之间应有一个空格
  - 参数名称之前应该有一个描述性关键词
  - and不应用作阐明有多个参数
- 字面值:NSString,NSDictionary,NSArray和NSNumber字面值应该用在任何创建不可变的实力对象，nil不能放进NSArray和NSDictionary，否则将导致Crash。

## 类

- 类名: 类名应该加上三个大写字母作为前缀(两个字母的为Apple的类保留)，以减少Objective-C没有命名空间带来的问题

- 创建子类时应该把说明性部分放在前缀和父类名中间。

- Initializer和dealloc初始化

  - 将dealloc方法放在实现问价你的最前面(@syntheize和@dynamic之后)

  - init应该放在dealloc之后

  - 如果有多个初始化方法，designated initializer应该放在第一个，secondary initializer紧接其后

  - init方法如下:

    ```objective-c
    - (instancetype)init{
      self=[super init];
      if(self){
        //custom initialization
      }
      return self;
    }
    ```

  - 永远不在designated中调用secondary初始方法

- 单例

- 属性

  - 尽可能描述性命名，避免缩写，小字母开头，驼峰

  - 对象声明应为NSString *text;

  - 除了init和dealloc方法，否则总是使用setter和getter访问属性

    setter

    - 使用setter会遵守内存管理语义
    - KVO通知会被自动执行
    - 更容易debug
    - 允许一个单独的地方为设置值添加额外的逻辑

    getter

    - 对未来的变化有扩展能力
    - 允许子类化
    - 更简单的debug
    - 让意图更加清晰明确
    - 自动产生KVO通知
    - 消息发送添加的开销微不足道

  - init和dealloc

    永远不能在init等初始化函数和dealloc中使用getter和setter方法，而需要直接访问示例变量。

  - 尽量使用点符号来访问和设置属性。这有助于区分属性访问和方法调用

- 属性定义

  - 属性参数顺序应为原子性，读写，内存管理，这样做属性的修改更容易修改正确，并且更好阅读。
  - 必须使用nonatomic，除非特别需要的情况。因为iOS中，atomic带来的锁特别影响性能，并且不能保证线程安全。
  - block必须使用copy(到堆上)，以保证其存活到结束
  - 为完成一个公有的getter和私有的setter，应声明公开的属性为readonly并在类扩展中重新定义通用的属性为readwrite
  - BOOL类型的描述性属性建议通过getter=修改访问方法

- 可变对象

  - 任何可以用一个可变对象设置的属性(如NSString，NSArray，NSURLRequest)的内存管理类型必须是copy的，这可以用来确保包装，并且在对象不知道的情况下避免改变值。
  - 避免在公开的接口中暴露可变的对象，可以提供只读属性来返回对象的不可变副本。

- 懒加载(争议)

  - getter方法不应该有副作用
  - 第一次访问的时候改变了初始化的小号，产生了副作用
  - 不是KVO友好的

- 参数断言

  若要求参数满足特定条件，最好使用NSParameterAssert()来断言条件

- 私有方法

  永远不要在私有方法前加上\_前缀，\_前缀为Apple保留

- 相等性

## Categories

应该在category方法前加上自己的小写前缀和下划线

## Protocols



## NSNotification

定义自己的NSNotification时应把通知的名字定义为字符串常量。并在公开的接口文件将其声明为extern，并且在对应的实现文件里面定义。

## 代码美化

- 空格
  - 设置tab为四个空格
- 冒号对齐
- line breaks换行
- 控制语句使用Egyptian风格括号，类的实现、方法的实现使用非Egyptian风格括号

## 代码组织

- 利用代码块

  GCC和Clang特性:代码块如果在闭合的圆括号内的话，返回最后语句的值

  该方法非常适合组织小块的代码，能让读者聚焦于关键的变量和函数。

  所有的变量都在代码块中，可以减少对其他作用域的命名污染。

  ```
  NSURL *url = ({
      NSString *urlString = [NSString stringWithFormat:@"%@/%@", baseURLString, endpoint];
      [NSURL URLWithString:urlString];
  });
  ```

- pragma

  - 用于分离

    - 不同功能组的方法
    - protocols的实现
    - 对父类方法的重写

  - 忽略未使用变量的编译警告

    ```
    - (void)giveMeFive
    {
        NSString *foo;
        #pragma unused (foo)

        return 5;
    }
    ```

- 明确编译器警告和错误:#error,#warning

- 字符串文档(除非函数非公开，短或者显而易见，否则都需要)，用于描述函数的调用符号和语义

  - 单行注释用//

  - 多行用

    ```
    /**
    一行总结.
    空行
    详细介绍
    */
    ```

## 对象之间通讯

- Block
  - 是在栈上创建的
  - 可以复制堆上
  - 有自己的私有的栈变量的常量复制
  - 可变的站上的变量和指针必须用__block关键字声明
  - strongSelf?
- 委托和数据源
- 继承
- 多重委托

## AOP

