---
title: 基础知识之Block
date: 2018-03-15 16:41:56
tags: iOS
categories: 技术
---

# Block

## block结构

```objective-c
int main(int argc,const char* argv[]){
  @autoreleasepool{
    ^{};
  }
  return 0;
}
```

### block编译转换结构

对上述代码执行clang -rewrite-objc编译转换成C++实现如下，

```c++
struct __block_impl{
  void *isa;//指针，指向所属类，即block的类型
  int Flags;//标志变量，在实现block的内部操作时用到
  int Reserved;//保留变量
  void *FuncPtr;//block执行时调用的函数指针
}
struct __main_block_impl_0{//block的C++实现，其中的0表示main中的第几个block
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp,struct __main_block_desc_0 *desc,int flags=0){//显示构造函数
    impl.isa=&_NSConcreteStackBlock;
    impl.Flags=flags;
    impl.FuncPtr=fp;
    Desc=desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself){
}
static struct __main_block_desc_0{
  size_t reserved;//保留字段
  size_t Block_size;//块大小
} __main_block_desc_0_DATA={0,sizeof(struct __main_block_impl_0)};//定义结构体的同时创建了__main_block_desc_0_DATA，并赋值，以供下面main函数中对__main_block_impl_0的初始化
int main(int argc,const char * argv[]){
  /*@autoreleasepool*/{__AtAutoreleasePool __autoreleasepool;
  	(void (*))&__main_block_impl_0((void *)__main_block_func_0,&__main_block_desc_0_DATA);
  }
}
```

### block实际结构

Block_private.h文件中对block相关结构体的真实定义如下:

```c++
struct Block_descriptor{
  unsigned long int reserved;
  unsigned long int size;
  void (*copy)(void *dst,void *src);//辅助拷贝函数
  void (*dispose)(void *);//辅助销毁函数，此两函数在处理block范围外的变量时使用
}

struct Block_layout{
  void *isa;
  int flags;
  int reserved;
  void (*invoke)(void *,...);
  struct Block_descriptor *descriptor);
}
```

综上，block就是一个里面存储了<u>指向函数体中包含block时?的代码块的函数指针</u>，以及<u>block外部上下文变量</u>等信息的结构体。

## block类型

### NSConcreteGlobalBlock和NSConcreteStackBlock

Block.h中声明

在全局区域创建，编译后isa指向&_NSConcreteGlobalBlock，变量存储在全局数据存储区。

在函数体中创建，编译后isa指向&_NSConcreteStackBlock，变量存储在栈区。

### NSConcreteMallocBlock

Block_private.h中声明，无法直接创建该类型block，由NSConcreteStackBlock拷贝得到。block的拷贝通过调用一下函数实现:

```c++
static void *_Block_copy_internal(const void *arg,const int flags){
  struct Block_layout *aBlock;
  ...
  aBlock=(struct Block_layout *)arg;
  if(!isGC){
    //申请block堆内存
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if(!result)return (void*)0;
    //拷贝栈中block到刚申请的堆内存中
    memmove(result,aBlock,aBlock->descriptor->size);//bitcopy first
    //reset refcount 重置引用数
    result->flags &= ~(BLOCK_REFCOUNT_MASK);
    result->flags |= BLOCK_NEEDS_FREE | 1;
    //改变isa指向_NSConcreteMallocBlock，即堆类型
    result->isa = _NSConcreteMallocBlock;
    if (result->flags & BLOCK_HAS_COPY_DISPOSE){
      (*aBlock->descriptor->copy)(result,aBlock);//调用拷贝辅助函数
    }
    return result;
  }else{
    ...
  }
}
```

## 捕捉变量对block结构的影响

### 局部变量

```objective-c
- (void)test{
  int a;
  ^{a;};
}
```

编译转换得到:

```c++
struct __Person__test_block_impl_0{
  strut __block_impl impl;
  struct __Person__test_block_desc_0* Desc;
  int a;//添加了成员变量，因为是值传递，所以赋值无意义，若传递指针，则可行，但如果block的调用在局部变量的作用域外则会出现野指针。
  __Person__test_block_impl_0(void *fp,struct __Person__test_block_desc_0 *desc,int _a,int flags=0) : a(_a){
    impl.isa=&_NSConcreteStackBlock;
    impl.Flags=flags;
    impl.FuncPtr=fp;
    Desc=desc;
  }
};
sstatic void __Person__test_block_func_0(struct __Person__test_block_impl_0 *__cself){
  int a = __cself->a;
}
static struct __Person__test_block_desc_0{
  size_t reserved;
  size_t Block_size;
} __Person__test_block_desc_0_DATA = {0,sizeof(struct __Person__test_block_impl_0)};
static void _I_Person_test(Person *self,SEL _cmd){
  int a;
  (void (*)())&__Person__test_block_impl_0((void *)__Person__test_block_func_0, &__Person__test_block_desc_0_DATA,a);
}
```

### 全局变量

```objective-c
static int a;
int b;
- (void)test{
  ^{
    a=10;
    b=10;
  };
}
```

编译转换:

```c++
static int a;
int b;
struct __Person__test_block_impl_0{
  struct __block_impl impl;
  struct __Person__test_block_desc_0 *Desc;
  __Person__test_block_impl_0(void *fp,struct __Person_test_block_desc_0 *desc, int flags=0){
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = = fp;
    Desc = desc;
  }
};

static void __Person__test_block_func_0(struct __Person__test_block_impl_0 *__cself){
  a = 10;
  b = 10;
}

static struct __Person__test_block_desc_0{
  size_t reserved;
  size_t Block_size;
} __Person_test__block_desc_0_DATA = { 0, sizeof(struct __Person__test_block_impl_0)};

static void _I_Person_test(Person *self,SEL _cmd){
  (void (*)())&__Person__test_block_impl_0((void *)__Person__test_block_func_0, &__Person__test_block_desc_0_DATA);
}
```

因为全局变量存储在静态数据存储区，程序结束前不会销毁，因此直接访问，不用预留位置。

### 局部静态变量

```objective-c
- (void)test{
  static int a;
  ^{
    a=10;
  };
}
```

编译转换:

```c++
struct __Person__test_block_impl_0{
  struct __block_impl impl;
  struct __Person__test_block_desc_0 *Desc;
  int *a;
  __Person__test_block_impl_0(void *fp, struct __Person_test_block_desc_0 *desc, int *_a, int flags=0): a(_a) {
    impl.isa = &_NSConcreteStatckBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __Person__test_block_func_0(struct __Person__test_block_impl_0 *__cself){
  int *a = __cself->a;
  (*a) = 10;//通过地址进行修改
}
static struct __Person__test_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __Person__test_block_desc_0_DATA = { 0, sizeof(struct __Person__test_block_impl_0)};

static void _I_Persion_test(Person *self, SEL cmd){
  static int a;
  (void (*)())&__Person__test_block_impl_0((void *)__Person__test_block_func_0, &__Person__test_block_desc_0_DATA, &a);//传入地址
}
```

虽然静态局部变量存储在静态数据存储区，生命周期与程序一样，但是其作用范围局限于定义它的函数中，所以block只能通过其地址进行访问。

### \_\_block修饰的变量

```objective-c
- (void)test{
  __block int a;
  ^{
    a = 10;
  };
}
```

编译转换:

```c++
struct __Block_byref_a_0{//用于包装局部变量a结构体
  void *__isa;//是个对象
  __Block_byref_a_0 *__forwarding;//用来指向它在堆中的拷贝，保证操作的值始终是堆中的拷贝
  int __flags;
  int __size;
  int a;
};
struct __Person_test_block_impl_0{
  struct __block_impl impl;
  struct __Person__test_block_desc_0 *Desc;
  __Block_byref_a_0 *a;//block内部会保存一个byref的指针，指向传进来的byref的forwarding
  __Person__test_block_impl_0(void *fp, struct __Person__test_block_desc_0 *desc, __Block_byref_a_0 *_a, int flags) : a(_a->__forwarding){
   impl.isa = &_NSConcreteStackBlock;
   impl.Flags = flags;
   impl.FuncPtr = fp;
   Desc = desc;
  }
};
static void __Person__test_block_func_0(struct __Person__test_impl_0 *__cself){
  __Block_byref_a_0 *a = __cself->a;
  (a->__forwarding->a) = 10;//修改的是forwarding指向的值
}
static void __Person__test_block_copy_0(struct __Person__test_block_impl_0 *dst,struct __Person__test_block_impl_0 *src){
  _Block_object_assign_copy((void*)&dst->a,(void*)&src->a,8);
}//当block被copy到堆中时，该函数会将__Block_byref_a_0拷贝到堆中，所以即使局部变量所在的栈被销毁，block依然能对堆中的局部变量进行操作
static void __Person__test_block_dispose_0(struct __Person_test_block_impl_0 *src){
  _Block_object_dispose((void *)src->a);
}
static struct __Person__test_block_desc_0{
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __Person__test_block_impl_0*,struct __Person__test_block_impl_0*);
  void (*dispose)(struct __Person_test_block_impl_0*);
}__Person__test_block_desc_0_DATA = {0, sizeof(struct __Person__test_block_impl_0),__Person_test_block_copy_0,__Person_test_block_dispose_0};
static void _I_Person_test(Person * self, SEL _cmd){
  __attribute__((__blocks__(byref))) __Block_byref_a_0 a= {(void *)0, (__Block_byref_a_0 *)&a, 0, sizeof(__Block_byref_a_0)};//栈上的byref的forwarding指向自己
  (void (*)())&__Person__test_block_impl_0((void *)__Person__test_block_func_0, &__Person__test_block_desc_0_DATA, (__Block_byref_a_0 *)&a,570425344);
}
```

```
static void _Block_byref_assign_copy(void *dst,const void *arg, const int flags){
  struct Block_byref **destp = (struct Block_byref **)dst;
  struct Block_byref *src = (struct Block_byref *)arg;
  copy->forwarding = copy;//拷贝后堆中的forwarding还是指向自己
  src->forwarding = copy;//拷贝后栈的的forwarding指向堆中的拷贝
}
```

### self隐式循环引用

## 不同类型block的复制

block的复制代码在\_Block\_copy\_internal函数中

### 栈block

```C++
static void *_Block_copy_internal(const void *arg,const int flags){
  struct Block_layout *aBlock;
  ...
  aBlock=(struct Block_layout *)arg;
  if(!isGC){
    //申请block堆内存
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if(!result)return (void*)0;
    //拷贝栈中block到刚申请的堆内存中
    memmove(result,aBlock,aBlock->descriptor->size);//bitcopy first
    //reset refcount 重置引用数
    result->flags &= ~(BLOCK_REFCOUNT_MASK);
    result->flags |= BLOCK_NEEDS_FREE | 1;
    //改变isa指向_NSConcreteMallocBlock，即堆类型
    result->isa = _NSConcreteMallocBlock;
    if (result->flags & BLOCK_HAS_COPY_DISPOSE){
      (*aBlock->descriptor->copy)(result,aBlock);//调用拷贝辅助函数
    }
    return result;
  }else{
    ...
  }
}
```

参考前面的代码可知，栈block的赋值不仅仅复制了内容，还添加了

- 往flags中并入了BLOCK_NEEDS_FREE(表明了block需要释放，在release以及再次拷贝时会用到)
- 如果有辅助copy函数，则调用(用于拷贝block捕获的变量)

### 堆block

```C++
if(aBlock->flags & BLOCK_NEEDS_FREE){
  latching_incr_int(&aBlock->flags);
  return aBlock;
}
```

堆中block的拷贝只是单纯地改变了引用计数

### 全局block

```C++
...
else if(aBlock->flags & BLOCK_IS_GLOBAL){
  return aBlock;
}
...
```

直接返回

## block辅助函数

在捕获\_\_block修饰的变量时，block才会有copy和dispose两个辅助函数

捕捉变量拷贝函数为\_\_Block\_object\_assign，在调用复制block的函数\_Block\_copy\_internal时，会根据block有无辅助函数来对捕捉变量拷贝函数进行调用。而在该函数中，也会通过判断捕捉变量包装而成的对象是否有辅助函数来进行调用。

### __block修饰的基本类型的辅助函数



### 对象的辅助函数

## ARC中block的工作

### block实验

### block作为参数传递



### block作为返回值

非ARC情况下，如果返回值是block，一般按以下进行:

```
return [[block copy] autorelease];
```

### block属性

ARC会自动帮strong类型且捕获外部变量的block进行copy，所以在定义block类型的属性时，也可以使用strong，不一定使用copy。

若block不捕获外部变量，则block变为全局类型，也脱离了栈声明周期的约束?