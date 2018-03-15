---
title: 基础知识之Runtime
date: 2018-03-15 16:39:50
tags: iOS
categories: 技术
---

# Runtime

# what is Runtime

Objective-C Runtime是一个将C语言转化为面向对象语言的扩展。C++和Objective-C都是在C的基础上加入面向对象的特性扩充而成的程序设计语言，但是C++是基于静态类型而Objective-C是基于动态运行时类型的。用C++编写的程序通过编译器直接把函数地址硬编码进入可执行文件，而Objective-C无法通过编译器直接把函数地址硬编码进入可执行文件，而是在函数运行的时候，利用Rumtime根据条件判断做出决定。函数标识与函数过程的真正内容之间的关联可以动态修改。

## Objective-C相关基础

### id & Class

objc.h文件中

```c++
#if !OBJC_TYPES_DEFINED
typedef struct objc_class *Class;//Class是指向objc_class结构体的指针

struct objc_object{
  Class isa OBJC_ISA_AVAILABILITY;//isa是Class，即是指向objc_class结构体的指针
}

typedef struct objc_object *id;//id是指向objc_object结构体的指针
#endif
```

runtime.h文件中

```C++
typedef struct objc_class *Class;
struct objc_class{
  Class isa					OBJC_ISA_AVAILABILITY;//metaClass
  #if !__OBJC2__
  Class supe_class			OBJC2_UNAVAILABLE;//父类
  const char *name			OBJC2_UNAVAILABLE;//类名
  long verson			OBJC2_UNAVAILABLE;//类的版本信息，默认为0，可以通过runtime函数class_setVersion或者class_getVersion进行修改、读取
  long info			OBJC2_UNAVAILABLE;//类信息，供运行时使用的一些位标识，如CLS_CLASS表示该类为普通class，其中包含示例方法和变量;CLS_META表示该类为metaClass，其中包含类方法
  long instance_size			OBJC2_UNAVAILABLE;//该类的示例变量大小，其中包括从父类继承下来的实例变量
  struct objc_ivar_list *ivars			OBJC2_UNAVAILABLE;//该类的成员变量地址列表
  struct objc_method_list **methodLists			OBJC2_UNAVAILABLE;//方法地址列表，与info的标志位有关，如果是CLS_CLASS，则存储实例方法，如果是CLS_META，则存储类方法
  struct objc_cache *cache			OBJC2_UNAVAILABLE;//缓存最近使用的方法地址，用于提升效率
  struct objc_protocol_list *protocols			OBJC2_UNAVAILABLE;//存储该类声明遵守的协议的列表
  #endif
}
```

类和对象的区别就是类比对象多了很多特征成员。类也可以当做一个objc_object来对待，即类和对象都是对象，分别称作类对象(class object)和实例对象(instance object)

- isa

  objc_object(实例对象)中isa指针指向的类结构称为class(也就是该对象所属的类)，其中存放着普通成员变量与动态方法("-"开头的方法)，而此处isa指针指向的类结构称为metaclass，其中存放着static类型的成员变量与static类型的方法("+"开头的方法)。


- super_class

  指向该类的父类的指针，如果该类是根类(NSObject\NSProxy)，则super_class为nil

**所有的metaclass中的isa指针都是指向根metaclass的，而根metaclass则指向自身。metaclass通过继承根类产生，与根class结构体成员一致，不同的是根metaclass的isa指针指向自己**

### SEL

SEL是selector在Objective-C中的表示类型，selector可以理解为区别方法的ID

```c++
typedef struct objc_selector *SEL;

struct objc_selector{
  char *name; OBJC2_UNAVAILABLE;//名称
  char *types; OBJC2_UNAVAILABLE;//类型
}
```

### IMP

objc.h中

```c++
typedef id (*IMP)(id, SEL, ...);
```

IMP是implementation的缩写，是由编译器生成的一个函数指针，当发起一个消息后，该函数指针决定了最终指向哪段代码。

### Method

```C++
typedef struct objc_method *Method;

struct objc_method{
  SEL method_name OBJC_UNAVAILABLE;//方法名
  char *method_types OBJC_UNAVAILABLE;//方法类型，存储了方法的参数类型和返回值类型
  IMP method_imp OBJC_UNAVAILABLE;//方法实现
}
```

### Ivar

类中示例变量的类型

```
typedef struct objc_ivar *Ivar;
struct objc_ivar{
  char *ivar_name OBJC2_UNAVAILABLE;//变量名
  char *ivar_type OBJC2_UNAVAILABLE;//变量类型
  int ivar_offset OBJC2_UNAVAILABLE;//基地址偏移字节
  #ifdef __LP64__
  int space OBJC2_UNAVAILABLE;//占用空间
  #endif
}
```

### objc_property_t

objc_property_t 是属性

```C++
typedef struct objc_property *objc_property_t;
```

objc_property是内置的类型，与之关联的是objc_property_attribute_t，是对属性的名称、编码类型、原子类型等的描述

```c++
typedef struct{
  const char *name;
  const char *value;
} objc_property_attribute_t;
```

### Cache

```c++
typedef struct objc_cache *Cache;
struct objc_cache{
  unsigned int mask OBJC2_UNAVAILABLE;
  unsigned int occupied OBJC2_UNAVAILABLE;
  Method buckets[1] OBJC2_UNAVAILABLE;
}
```

mask:指定分配cache buckets的总数

occupied:实际占用cache buckets的总数

buckets:指定Method数据结构指针的数组，包含不超过mask+1个元素。

objc_msgSend每次调用一次方法后，就会把该方法缓存到cache列表中，下次直接优先从cache列表中寻找，若未找到再到methodLists中查找方法。

### Category

动态为已存在的类添加方法

```C++
typedef struct objc_category *Category;
struct objc_category{
  char *category_name;//类别名
  char *class_name;//类名
  struct objc_method_list *instance_methods;//实例方法名
  struct objc_method_list *class_method;//类方法名
  struct objc_protocol_list *protocols;//协议列表
}
```

## Objective-C的消息传递

### 基本消息传递

Runtime会根据类型自动转换成以下函数之一

- objc_msgSend:普通消息
- objc_msgSend_stret:消息中有数据结构作为返回值
- objc_msgSendSuper:将消息发送给父类
- objc_msgSendSuper_stret

objc_msgSend函数的调用过程

1. 检测selector是否要忽略
2. 检测target是否nil对象
3. 调用实例方法时，先在自身isa指向的类的methodLists中查找该方法，如果找不到则通过class的super_class指针找到父类的类对象结构体，在父类的methodLists中查找该方法，直到根class;
4. 若调用的是类方法，则在其metaclass中执行步骤3.
5. 动态方法解析

### 消息动态解析

1. resolveInstanceMethod

   决定是否动态添加方法，是则通过class_addMethod动态添加方法，否则继续

2. forwardingTargetForSelector

   指定备选对象(不能是self)响应这个selector，如果返回nil，则继续

3. methodSignatureForSelector

   通过该方法进行签名，如果返回nil，则消息无法处理，如果返回methodSignature，则继续

4. forwardInvocation

   通过anInvocation对象做很多处理，如修改实现方法，修改响应对象等，如果方法调用成功则结束，如果失败则进入doesNotRecognizeSelector方法

# ———————分割线——————————

## 成员变量与属性

### 类型编码

   编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的selector关联到一起，这种编码防范在其他情况下也是非常有用的，可以使用@encode编译器指令来获取，当给定一个类型时，@encode返回该类型的字符串编码。这些类型可以是注入int、指针这样的基本类型，也可以是结构体、类等类型。任何可以作为sizeof()操作参数的类型都可以用于@encode()。

   类型编码大多与用于存档和分发的编码类型是相同的，但有一些不能在存档时使用。

   另外还有一些编码类型，@encode虽然不会直接返回他们，但他们可以作为协议中声明的方法的类型限定符。

   对于属性而言，还有一些特殊的类型编码，以表明属性是只读、拷贝、retain等。

### 关联对象

   针对分类不能添加新的成员变量，Objective-C提供的解决方案，即Associated Object。

   可以将关联对象想象成一个Objctive-C对象，这个对象通过给定的key连接到类的一个实例上，由于使用的是C接口，所以key是一个void指针。并通过指定一个内存管理策略，告诉Runtime如何管理这个对象的内存，内存管理策略可以由以下值指定:

```objective-c
   OBJC_ASSOCIATION_ASSIGN
   OBJC_ASSOCIATION_RETAIN_NONATOMIC
   OBJC_ASSOCIATION_COPY_NONATOMIC
   OBJC_ASSOCIATION_RETAIN
   OBJC_ASSOCIATION_COPY
```

当宿主对象被释放时，会根据指定的内存管理策略来处理关联对象，如果指定的策略时assign，当宿主释放时，关联对象不会被释放，如果指定的是retain或copy，则宿主释放时，关联对象会被释放。

将一个对象连接到其他对象所需要做的就是以下两行代码:

```objective-c
static char myKey;
objc_setAssociatedObject(self, &myKey, anObject, OBJC_ASSOCIATION_RETAIN);
```

当使用同一个key来关联另一个对象时，会自动释放之前关联的对象。

```objective-c
id anObject = objc_getAssociatedObject(self, &myKey);
```

可以使用objc_removeAssociatedObjects函数来移除一个关联对象，或者使用objc_setAssociatedObject函数将key指定的关联对象设置为nil。

### 操作方法

#### 成员变量

```objective-c
//获取成员变量名
const char * ivar_getName(Ivar v);
//获取成员变量类型编码
const char * ivar_getTypeEncoding(Ivar v);
//获取成员变量的偏移量，对于类型id或其他对象类型的实例变量，可以调用object_getIvar和object_setIvar来直接访问成员变量，而不是偏移量
ptrdiff_t ivar_getOffset(Ivar v);
```

#### 关联对象

```objective-c
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
void objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object);
```

#### 属性

```objective-c
//获取属性名
const char * property_getName(objc_property_t property);
//获取属性特性描述字符串
const char * property_getAttributes(objc_property_t property);
//获取属性中指定的特定
char * property_copyAttributeValue(objc_property_t property,const char *attributeName);
//获取属性的特性列表，返回的char*在使用完后要调用free()释放掉
objc_property_attribute_t * property_copyAttributeList(objc_property_t property, unsigned int *outCount);
```

## 方法与消息

方法的selector用于表示运行时方法的名字，Objective-C在编译时会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(int类型的地址)，这个标识就是SEL。

任何两个类，只要方法名相同，那么方法的SEL就是一样的。所以在Objective-C同一个类中，不能存在两个同名方法，参数类型不同也不行。从而导致了Objective-C在处理相同方法名且参数个数相同但类型不同的方法方面的能力很差。

不同的类可以拥有相同的selector，因为不同类的实例对象执行相同的selector时，会在格子的方法列表中取根据selector去寻找自己对应的IMP。

在方法集合中查找方法时通过查找相应的SEL即可，而SEL实际上是根据方法名hash化了的字符串，而对于字符串的比较仅仅比较其地址即可。然而数量增多会导致性能下降，而SEL仅仅使用函数名可以使总量减少。

可以在运行时添加新的selector，也可以在运行时获取已存在的selector，可以通过以下三种方法来获取SEL

1. sel_registerName函数
2. Objective-C编译器提供的@selector()
3. NSSelectorFromString()方法

### IMP

实际上是一个函数指针，指向方法实现的首地址。

这个函数是用当前CPU架构实现的标准的C调用约定。第一个参数指向self的指针(如果是实例方法，则指向类实例的内存地址，如果是类方法，则指向元类内存地址)，第二个参数是方法选择器(selector)，接下来是方法的实际参数列表。

SEL是为了查找方法的最终实现IMP。每个方法的SEL唯一，因此可以通过SEL方便快速准确地获取其对应的IMP，即获得了执行这个方法代码的入口点。

通过取得IMP，可以跳过Runtime的消息传递机制，直接执行IMP指向的函数实现，省去了Runtime消息传递过程中所做的一系列查找操作，会比直接向对象发送消息高效一些。

### 方法相关操作函数

#### 方法

```objective-c
//调用指定方法的实现
id method_invoke(id receiver, Method m, ...);
//调用返回一个数据结构的方法的实现
void method_invoke_stret(id receiver, Method m, ...);
//获取方法名
SEL method_getName(Method m);
//返回方法的实现
IMP method_getImplementation(Method m);
//获取描述方法参数和返回值类型的字符串
const char * method_getTypeEncoding(Method m);
//获取方法的返回值类型的字符串
char * method_copyReturnType(Method m);
//获取方法的指定位置的类型字符串
char * method_copyArgumentType(Method m, unsigned int index);
//通过引用返回方法的返回值类型字符串
void method_getReturnType(Method m, char *dst, size_t dst_len);
//返回方法的参数的个数
unsigned int method_getNumberOfArguments(Method m);
//通过引用返回方法指定位置参数的类型字符串
void Method_getArgumentType(Method m, Unsigned int index, char *dst, size_t dst_len);
//返回指定方法的方法描述结构体
struct objc_method_description * method_getDescription(Method m);
//设置方法的实现，返回值是old实现
IMP method_setImplementation(Method m, IMP imp);
//交换两个方法的实现
void method_exchangeImplementations(Method m1, Method m2);
```

#### 方法选择器

```objective-c
//返回给定选择器指定的方法的名称
const char * sel_getName(SEL sel);
//注册一个方法，将方法名映射到一个选择器，并返回这个选择器。在将一个方法添加到类定义时，必须在objective-C Runtime系统中注册方法名以获取方法的选择器
SEL sel_registerName(const char *str);
//注册一个方法
SEL sel_getUid(const char *str);
//比较两个选择器
BOOL sel_isEqual(SEL lhs, SEL rhs);
```

### 方法调用流程

#### 隐藏参数

objc_msgSend有两个隐藏参数

1. 消息接收对象
2. 方法的selector

在定义方法的源代码中没有声明，是在编译期间被插入实现代码的。

虽然这两个参数没有显式声明，但是在代码中仍然可以引用。可以通过self来引用接收者对象，用_cmd来引用选择器。

#### 获取方法地址

Runtime中方法的动态绑定让我们写代码时更具灵活性，例如将消息转发给我们想要的对象，或者随意交换一个方法的实现。然而方法实现的查找带来性能的损耗，尽管方法的缓存能在一定程度上降低这种损耗。

如果要避开动态绑定方式，可以获取方法实现的地址，然后像调用函数一样来直接调用它，特别是当需要在一个循环内频繁地调用一个特定的方法时，通过这种方式可以提高程序的性能。

NSObjecti类提供了methodForSelector方法让我们可以获取方法的指针，然后通过这个指针来调用实现代码，使用时需要将methodForSelector返回的指针转换为合适的函数类型，函数参数和返回值都需要匹配。

示例:

```objective-c
void (*setter)(id, SEL, BOOL);
int i;
setter = (void (*)(id,SEL,BOOL))[target methodForSelector:@selector(setFilled:)];
for(i=0;i<1000;i++){
  setter(targetList[i],@selector(setFilled:),YES);
}
```

函数的前两个参数必须是id和SEL

这种方式只适用于在类似于for循环这种情况下频繁调用同一方法来提高性能的情况。

methodForSelector:是由Cocoa运行时提供的，不是Objective-C语言的特性

### 消息转发

默认情况下，如果以[object message]的方式调用方法，如果object无法响应message消息时，编译器会报错。如果以perform...的形式来调用，则需要等到运行时才能确定object是否能结束message。

当一个对象无法接受某一消息时，会启动消息转发机制。

消息转发机制基本上分三个步骤:

1. 动态方法解析
2. 备用接收者
3. 完整转发

#### 动态方法解析

对象在接收到未知消息时，首先会调用所属类的类方法+resolveInstanceMethod或者 +resolveClassMethod。在这个方法中将有机会为该未知消息新增一个处理方法，不过使用该方法的前提是提前实现该方法。这需要在运行时通过class_addMethod函数动态添加到类里面。

示例:

```objective-c
void functionForMethod1(id self, SEL _cmd){
  NSLog(@"%@,%p", self, _cmd);
}
+(BOOL)resolveInstanceMethod:(SEL)sel{
  NSString *selectorString = NSStringFromSelector(sel);
  if([selectorString isEqualToString:@"method1"]){
    class_addMethod(self.class,@selector(method1),(IMP)functionForMethod1,"@:");
  }
  return [super resolveInstanceMethod:sel];
}
```

这种方案更多是为了实现@dynamic属性

#### 备用接收者

如果上一步无法处理消息，Runtime会继续调用以下方法:

```objective-c
-(id)forwardingTargetForSelector:(SEL)aSelector
```

如果一个对象实现了这个方法，并返回一个非nil的结果，则这个对象会作为消息的新接收者，且消息会被分发到这个对象。指定self将导致无线循环。

使用这个方法通常是在对象内部，可能还有一些列其他对象能处理该消息，便可以这些对象来处理消息并返回。

示例:

```objective-c
@interface SUTRuntimeMethodHelper : NSObject

-(void)method2;

@end

@implementation SUTRuntimeMethodHelper

-(void) method2 {
  NSLog(@"%@,%p",self,_cmd);
}

@end
  
#pragma mark -
  
@interface SUTRuntimeMethod(){
 SUTRuntimeMethodHelper *_helper;
}
@end

@implementation SUTRuntimeMethod

+(instancetype)object{
  return [[self alloc] init];
}
-(instancetype)init{
  self=[super init];
  if(self!=nil){
    _helper=[[SUTRuntimeMethodHelper alloc] init];
  }
  return self;
}
-(void)test{
  [self performSelector:@selector(method2)];
}
-(void)forwardingTargetForSelector:(SEL)aSelector{
  NSLog(@"forwardingTargetForSelector");
  NSString *selectorString = NSStringFroSelector(aSelector);
  if([selectorString isEqualToString:@"method2"]){
    return _helper;
  }
  return [super forwardingTargetForSelector:aSelecotr];
}
@end
```

适合于指向将消息转发到另一个能处理该消息的对象上，但这一步无法对消息进行处理，如操作消息的参数和返回值。

#### 完整消息转发

如果上一步无法处理未知消息，唯一能做的就是启用完整的消息转发机制，此时调用一下方法:

```objective-c
-(void)forwardInvocation:(NSInvocation *)anInvocation
```

运行时系统会在这一步给消息接收者最后一次机会将消息转发给其他对象。对象会创建一个表示消息的NSInvocation对象，把尚未处理的消息相关的消息有改观的全部细节都封装在anInvocation中，包括selector，目标target和参数，我们可以在forwardInvocation方法中选择将消息转发给其他对象。

forwardInvocation方法的实现有两个任务:

1. 定位可以相应封装在anInvocation中的消息的独享，这个对象不需要能处理所有未知消息
2. 使用anInvocation作为参数，将消息发送到选中的对象。anInvocation将会保留调用结果，运行时系统会提取这一结果并将其发送到消息的原始发送者。

在这个方法中可以实现一些更复杂的功能，可以对消息的内容进行修改。

此外，必须重写以下方法:

```objective-c
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
```

消息转发机制使用从这个方法中获取的信息来创建NSInvocation对象，因此需要重写这个方法为给定的selector提供一个合适的方法签名。

```objective-c
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
  NSMethodSignature *signature=[super methodSignatureForSelector:aSelector];
  if(!signature){
    if([SUTRuntimeMethodHelper instancesRespondToSelector:aSelector]){
      
    }signature=[SUTRuntimeMethodHelper instanceMethodSignatureForSelector:aSelector];
  }
  return signature;
}
-(void)forwardInvocation:(NSInvocation *)anInvocation{
  if([SUTRuntimeMethodHelper instancesResponedToSelector:anInvocation.selector]){
    [anInvocation invokeWithTarget:_helper];
  }
}
```

NSObject的forwardingInvocation方法实现知识简单调用了doesNotRecognizeSelector:方法，不会转发任何消息，所以将导致异常。

#### 消息转发与多重继承

消息转发虽然类似于继承，但NSObject的一些方法还是能区分两者，如respondsToSelector:和isKindOfClass:只能用于继承体系而不能用于转发链。

## Method Swizzling

Method swizzling是改变一个selector的实际实现的技术。通过这一技术，可以在运行时修改类的分发表中selector对应的函数，来修改方法的实现。

### swizzling应该总是在+load中执行

Objective-C运行时会自动调用每个类的两个方法，+load会在类初始加载时调用，+initialize会在第一次调用类的类方法或实例方法之前被调用。这两个方法都是可选的 ，且只有在实现了他们时才会被调用。由于method swizzling会影响到类的全局状态，因此尽量避免在并发处理中出现竞争的情况。+load能保证在类的初始化过程中被加载，并保证这种改变应用级别的行为的一致性。相比之下initialize在其执行时不提供这种保证。

### swizzling应该总是在dispatch_once中执行

确保代码之执行一次

### 注意事项

1. 总是调用方法的原始实现。
2. 避免冲突

## 协议与分类

### Category

见上

### Protocol

```objective-c
typedef struct objc_object Protocol;
```

即Protocol其实是一个对象结构体

### 操作函数

#### category

可以通过针对objc_class的操作函数来获取分类的信息。

#### protocol

```objective-c
//返回指定协议，如果仅声明了协议而未在任何类中实现该协议，则函数返回nil
Protocol * objc_getProtocol(const char *name);
//获取运行时所知道的所有协议的数组，获取到的数组需要使用free来释放
Protocol ** objc_copyProtocolList(unsigned int *outCount);
//创建新的协议实例，如果同名的协议已存在，则返回nil
Protocol * objc_allocateProtocol(const char *name);
//在运行时中注册新创建的协议，创建一个新的协议后，必须调用该函数以在运行时中注册新的协议，协议注册后便可以使用，但不能再做修改，即无法再调用protocol_addMethodDescription/protocol_addProtocol/protocol_addProperty方法
void objc_registerProtocol(Protocol *proto);
//为协议添加方法
void protocol_addMethodDescription(Protocol *proto, SEL name, const char *type, BOOL isRequiredMethod, BOOL isInstanceMethod);
//添加一个已注册的协议到协议中
void protocol_addProtocol(Protocol *proto, Protocol *addition);
//为协议添加属性
void protocol_addProperty(Protocol *proto, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount, BOOL isRequiredProperty, BOOL isInstanceProperty);
//返回协议名
const char * protocol_getName(Protocol *p);
//测试两个协议是否相等
BOOL protocol_isEqul(Protocol *proto, Protocol *other);
//获取协议中指定条件的方法的方法描述数组
struct objc_method_description * protocol_copyMethodDescriptionList(Protocol *p, BOOL isRequireMethod, BOOL isInstanceMethod, unsigned int *outCount);
//获取协议中指定方法的方法描述
struct objc_method_description protocol_getMethodDescriptiion(Protocol *p, SEL aSel, BOOL isRequiredMethod, BOOL isInstanceMethod);
//获取协议中的属性列表
objc_proprety_t * protocol_copyPropertyList (Protocol *proto,unsigned int *outCount);
//获取协议的指定属性
objc_property_t protocol_getProperty(Protocol *proto, const char *name, BOOL isRequiredProperty, BOOL isInstanceProperty);
//获取协议采用的协议
Protocol ** protocol_copyProtocolList(Protocol *proto,unsigned int *outCount);
//查看协议是否采用了另一个协议
BOOL protocol_conformsToProtocol(Protocol *proto, Protocol *other);
```

## 其他

### super

self是类的隐藏参数，super不是类的隐藏参数而是编译器标示符，负责告诉编译器去调用父类的方法，而消息的接收者仍是self。即发送消息时，调用的是objc_msgSendSuper函数。

参考

[http://southpeak.github.io/](http://southpeak.github.io/)