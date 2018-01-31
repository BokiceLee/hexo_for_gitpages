---
title: RunTime
date: 2018-01-31 17:52:31
tags: iOS
Categories: 技术
---

Objc与Runtime系统进行交互

	1.Objective-C源代码，消息的执行会使用到一些编译器为实现动态语言特性而创建的数据结构和函数，Objc中的类/方法/协议等在runtime中都由一些数据结构来定义。
	2.Foundation框架的NSObject类定义的方法，可以在运行时获得类的信息，并检查一些特性。
	3.对runtime函数的直接调用，Runtime系统是由一系列函数和数据结构组成，具有公共接口的动态共享库。
Runtime基础数据结构    objc_msgSend:
	1.SEL，是selector在Objc中的表示类型，selector是方法选择器，可以理解为区分方法的ID，而这个ID的数据结构是SEL。是一个映射到方法的C字符串，可以用Objc编译器命令@selector()或者Runtime系统的sel_registerName函数来获得一个SEL类型的方法选择器。不同类中相同名字的方法对应的方法选择器是相同的，即使方法名字相同而变量类型不同也会导致他们具有相同的方法选择器，因此Objc中方法命名有时会带上参数类型
	2.id，指向类实例的指针，objc_object结构体包含isa指针，isa指针不总是指向实例对象所属的类，不能依靠它来确定类型，而是应该用class方法来确定实例对象的类。
	3.Class 指向objc_class结构的指针，objc_class继承于objc_object
		3.1.cache_t 优化方法调用的性能
			_buckets 存储IMP->bucket_t存储了指针与IMP的键值对
			_mask，_occpuied对应vtable
		3.2.class_data_bits_t
	
		3.3.class_ro_t 在class_rw_t中含对应的指针
			method_list_t
			ivar_list_t
			property_list_t
			protocol_list_t
		3.4.class_rw_t 在class_data_bits_t中含对应的指针，提供了运行时对类扩展的能力，而class_ro_t存储的大多数是类在编译时就已经确定的信息，二者都存有类的方法/属性/协议等信息，不过存储他们的列表实现方式不同。
			method_array_t
			property_array_t
			protocol_array_t
			class_rw_t的内容可以在运行时被动态修改，可以说运行时对类的扩展大都是存储在这里的。
		3.5.realizeClass
			在某个类初始化之前，objc_class->class()返回的指针指向的其实是个class_ro_t结构体，等到realizeClass京台方法在类第一次初始化时被调用，它会开辟class_rw_t的空间，并将class_ro_t指针赋值给class_rw_t->ro，realizeClass函数处理的类才是真正的类，调用它时不能对类做写操作。
	4.Category 为现有类提供扩展性，category_t结构体的指针
		存储了类别中可以扩展的实例方法/类方法/协议/实例属性和类属性，App启动镜像文件时，会在_read_images函数间接调用到attachCategories函数，完成向类中添加Category的工作，原理:向class_rw_t中的method_array_t,property_array_t,protocol_array_t数组中分别添加method_list_t,property_list_t,protocol_list_t指针，调用attachCategories函数之前，会先使用unattachedCategoriesForClass函数获取类中还未添加的类别列表。
	
	5.Method 是一种代表类中的某个方法的类型。存储了方法名，方法类型和方法实现。
		SEL，方法名类型，相同名字的方法即使在不同类中定义，方法选择器也相同。
		types，方法类型，char指针，存储着方法的参数类型和返回值类型
		imp 指向了方法的实现，本质上是函数指针
	6.Ivar 代表类中实例变量的类型，是一个指向ivar_t结构体的指针
	7.objc_property_t，指向objc_property结构体的指针
		可以通过class_copyPropertyList和protocol_copyPropertyList方法来获取类和协议中的属性，返回类型为指向指针的指针
	8.protocol_t
	9.IMP 函数指针，由编译器生成，发起一个objC之后，最终执行的代码，由该函数指针指定
消息
	1.objc_msgSend函数
		消息发送步骤
			1.检测selector是否要忽略，如retain，release
			2.检测target是否nil对象，保证不crash
			3.开始查找类的IMP，从cache里找，找到了跳转到对应的函数执行
			4.找方法分发表
			5.找超类分发表，直到找到NSObject类为止
			6.进入动态方法分析
			PS：分发表指的是Class中的方法列表，将方法选择器和方法实现地址联系起来
			编译器会根据情况在objc_msgSend，objc_msgSend_stret，objc_msgSendSuper或objc_msgSendSuper_stret四个方法中选择一个来调用，如果消息是传递给超类的，那么会调用名字带有super的函数，如果消息返回时数据结构而不是简单值，那么会调用名字带有stret(struct return)的函数
	2.方法中的隐藏参数
		self的内容是在方法运行时被偷偷动态传入的。当objc_msgSend找到方法对应的实现时，它将直接调用该方法实现，并将消息中所又的参数都传递给方法实现，同时，它还将传递两个隐藏的参数，接收消息的对象和方法选择器。源代码方法的定义中并没有声明这两个参数，它们时在代码被编译时被插入实现中的，self是方法实现中访问消息接收者对象的实例变量的途径。当方法中的super关键字接收到消息时，编译器会创建一个objc_super结构体，知名消息应该被传递给特定超类的定义，但是recevier仍然时self本身，当想通过[self class]获取超类时，编译器只是将self的id指针和class的SEL传递给objc_msgSendSUper函数，因为只有在NSObjct类才能找到class方法，然后class方法调用object_getClass，接着调用objc_msgSend(objc_super->receiver,@selector(class))，传入的第一个参数是指向self的id指针，与调用[self class]相同，所以我们得到的永远都是self的类型
	3.获取方法地址
		NSObject类中有个methodForSelector实例方法，可以用它来获取某个方法选择器对应的IMP，将方法当作函数调用，需要明确给出两个隐藏参数。methodForSelector方法又Cocoa的Runtime系统提供，不是Objc自身的特性。
动态方法解析
	可以动态提供一个方法的实现，例如可以用@dynamic关键字在类的实现文件中修饰一个属性，表明会为这个属性动态提供存取方法，也就是说编译器不会再默认生成setPropertyName和protertyName方法，而需要我们动态提供，可以通过分别重载resolveInstanceMethod和resolveClassMethod方法来添加实例方法实现和类方法实现，因为当Runtime系统在Cache和方法分发表中找不到要执行的方法时，Runtime会调用resolveInstanceMethod或resolveClassMethod来给程序员一次动态添加方法实现的机会，需要用class_addMethod函数完成向特定类添加特定方法实现的操作。动态方法解析会在消息转发机制浸入前执行。
	当self为实例对象时，[self class]与object_getClass(self)等价，因为前者会调用后者，object_getClass([self class])得到元类
	当self为类对象时，[self class]返回值为自身，即self，object_getClass(self)与object_getClass([self class])等价。
消息转发
	1.重定向
		在消息转发机制执行前，Runtime系统会在给我们一次偷梁换柱的机会，即通过重载forwardingTargetForSelector:方法替换消息的接受者为其他对象
	2.转发
		当动态方法解析不作处理返回NO时，消息转发机制会被处罚，这时forwardInvocation方法会被执行，可以重写这个方法来定义转发逻辑，消息的唯一参数是NSInvocatoin类型的对象，该对象封装了原始的消息和消息的参数，可以实现forwardInvocation方法来对不能处理的消息做一些默认的处理，也可以将消息转发给其他对象来处理。在forwardInvocation消息发送前，runtime系统会向对象发送methodSignatureForSelector消息，并取到返回的方法签名用于生成NSInvocation对象，所以在重写forwardInvocation的同时也要重写methodSignatureForSelector方法，否则会抛异常。
		forwardInvocation方法像一个不能识别的消息的分发中心，将这些消息转发给不同接收对象，也可以像运输站将所有的消息都发送给同一个接收对象，可以将一个消息翻译成另外一个消息。forwardInvocation方法也可以对不同的消息提供同样的响应，这取决于方法的具体实现，该方法所提供是将不同对象连接到消息链的能力。
		forwardInvocation只有在消息接收对象中无法正常响应消息时才会被调用，如果我们希望一个对象将negotiate消息转发到其他对象，则这个对象不能有negotiate方法，否则forwardInvocation将不可能被调用。
	3.转发和多继承
		消息转发弥补了Objc不支持多继承的性质，也避免因为多继承导致单个类变得臃肿复杂
	4.替代者对象
		转发不仅能模拟多继承，也能使轻量级对象代表重量级对象。
	5.转发与继承
		NSObject不会将两者混淆，像respondsToSelector和isKindOfClass这类方法只会考虑继承体系，而不考虑转发链。
健壮的实例变量
	当一个类被编译时，实例变量的布局也就形成了，它表明访问类的实例变量的位置。从对象头部开始，实例变量依次根据自己所占空间而产生位移。在健壮的实例变量下编译器生成的实例变量布局跟以前一样，但是当runtime系统检测到与超类有部分重叠时会调整新添加的实例变量的位移，那样在子类中新添加成员就被保护起来了。
	健壮的实例变量下，不要使用sizeof(SomeClass)，而是用class_getInstanceSize([SomeClass class])代替，也不要使用offsetof(SomeClass,SomeIvar),而要用ivar_getOffset(class_getInstanceVariable([SomeClass class],"SomeIvar"))来代替。
Objective-C Associated Objects
	OSX 10.6 之后，Runtime系统让Objc支持向对象动态添加变量，涉及到的函数有以下三个，objc_setAssociatedObject，objc_getAssociatedObject，objc_removeAssociatedObjects，这些方法以键值对的形式动态地向对象添加/获取/删除关联值，关联策略由枚举变量确定。
Method Swizzling
​	