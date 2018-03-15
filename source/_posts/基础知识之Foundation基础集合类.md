---
title: 基础知识之Foundation基础集合类
date: 2018-03-15 16:37:10
tags: iOS
categories: 技术
---

# Foundation之基础集合类

## NSArray

### 性能特征

#### 访问

基于存储对象的多少，使用多种内部变种，对于个别对象的访问并不保证O(1)的访问时间;

对于NSArray中值的访问时间，保证在任何一种实现下最坏情况是O(lgN)，通常是O(1)，

#### 线性搜索

最坏情况下的复杂度为O(N*lgN)

#### 插入删除

耗时通常和数据中的值的数量成线性关系，某些实现的最坏情况下为O(N*lgN)

#### 其他

没有对于性能上特别有优势的数据位置

### 相关方法

##### isEqual:

大多数方法使用该方法来判断对象间的关系如containsObject

#### indexOfObjectIdentialTo:

检查指针是否相等

#### firstObject/lastObject(iOS7)

空数组放回nil，而常规访问方法抛NSRangeException

#### 通过数组构造可变数组:

```
NSMutableArray *mutableObjects=[array mutableCopy];
if(!mutableObjects){
  mutableObjects=[NSMutableArray array];
}
```

或

```
NSMutableArray *mutableObjects=[NSMutableArray arrayWithArray:array];
```

#### 逆序数组:

```
array.reverseObjectEnumerator.allObjects返回一个新的数组，每个NSEnumerator都实现了allObjects
```

#### 排序(没有太大的性能差别):

##### NSString:

```
[array sortedArrayUsingSelector:@selector(localizedCaseInsensitiveCompare:)];
```

##### NSNumber:

```
[array sortedArrayUsingSelector:@selector(compare:)];
```

##### 基于函数指针的排序方法:

```
- (NSData *)sortedArrayHint;
- (NSArray *)sortedArrayUsingFunction:(NSInteger (*)(id, id, void *))comparator
                          context:(void *)context;
- (NSArray *)sortedArrayUsingFunction:(NSInteger (*)(id, id, void *))comparator
                          context:(void *)context hint:(NSData *)hint;
```

##### 基于block的排序方法

```
- (NSArray *)sortedArrayUsingComparator:(NSComparator)cmptr;
- (NSArray *)sortedArrayWithOptions:(NSSortOptions)opts
                usingComparator:(NSComparator)cmptr;
```

#### 枚举:

##### indexesOfObjectsWithOptions:passingTest:

​	使用并发的情况下最快，数量少的情况下开并发可能由于线程管理导致更慢

##### filteredArrayUsingPredicate:

​	最慢

##### enumerateObjectsUsingBlock:

##### for(id obj in array):

​	最快

##### 使用NSEnumerator



#### NSFastEnumeration(?):

NSEnumeration每次迭代都有运行时开销

而NSFastEnumeration通过CountByEnumeratingWithState:objects:count:返回一个数据块，数据块被解析成id类型的C数组

#### arrayWithCapacity:

效率无差别，无明显作用

## NSDictionary

```
1.[NSDictionary dictionaryWithObjectsAndKeys:object,key,nil]
2.@{key:value,...}
key-value顺序不同
```

key是被拷贝的并且需要是不变的，如果一个key被用于在字典中放入一个值后被改变的话，那么该值将无法获得。

NSDictionary中的key是copy的，但是(使用CFDcitionarySetValue()时)CFDictionary中的key是retain的。原因是CoreFoundation没有通用的拷贝对象的方法。(使用setObject:forKey时)CFDictionary会增加额外的处理逻辑使key被拷贝。

### 性能特征:

#### 访问:

任何一种实现下最坏的情况是O(N)，通常是O(1)

#### 插入删除

一般来说是O(1)，某些实现中的最坏情况是O(N*N)，通过键来访问将比直接访问值要快，但是需要花费多很多的内存空间。

#### 枚举:

##### keysOfEntiresWithOptions:passingTest:

开并发最快，不开并发也很快

##### enumerateKeysAndObjectsUsingBlock:

可以高效地提前获取value

##### for(id key in dict)

快速枚举只能枚举key，必须每次自己获取value

##### 使用NSEnumerator

##### 基于C数组，通过getObjects:andKeys:枚举

最快，在数组元素很多时会崩溃

#### dictionaryWithCapacity:

同array，capacity参数无意义

### 排序

只能对键数组排序

```
- (NSArray *)keysSortedByValueUsingSelector:(SEL)comparator;
- (NSArray *)keysSortedByValueUsingComparator:(NSComparator)cmptr;
- (NSArray *)keysSortedByValueWithOptions:(NSSortOptions)opts
                      usingComparator:(NSComparator)cmptr;
```

### 共享键(?)

## NSSet

检查对象存在通常是O(1)操作，比NSArray快很多。

只在被使用的哈希方法平衡的情况下能高效工作。

NSSet会retain其中的对象，并且对象应该是不可变的，否则会出现问题。

##### allObjects

​	方法能将对象转化为NSArray，

##### anyObject

​	返回任一对象

### 相关操作

##### intersectSet:

##### minusSet

##### unionSet

### setWithCapacity

没有明显差距

### 性能特征:

因为NSSet在每个被添加的对象上执行hash和isEqual方法并管理一系列哈希值，因此在添加元素时耗费了很多时间。

## NSOrderedSet

## NSHashTable

## NSMapTable

## NSPointerArray

## NSCache

线程安全，适合缓存那些创建起来代价高昂的对象，能自动对内存警告做出反应并基于可设置的“成本”清理自己，其键是retain的(NSDictionary是copy的).

### 性能

加入的线程安全带来耗时成本，元素的添加慢，因为要多维护一个决定何时回收对象的成本系数。

## NSIndexSet

