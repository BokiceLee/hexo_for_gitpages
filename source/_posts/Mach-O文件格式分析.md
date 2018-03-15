---
title: Mach-O文件格式分析
date: 2018-03-15 16:41:06
tags: iOS
categories: 技术
---

### Mach-O文件格式分析

Mach-O是OS X和iOS的可执行文件、库文件、Dsym文件、动态库、动态连接器式。其组成结构为Header、Load commands、Data。使用MachOView开源工具可以查看gcc编译生成的.out文件的具体格式

#### Header的数据结构

##### 32位

```
struct mach_header{
  uint32_t magic;
  cpu_type_t cputype;
  cpu_subtype_t cpusubtype;
  uint32_t filetype;
  uint32_t ncmds;
  uint32_t sizeofcmds;
  uint32_t flags;
}
```

##### 64位

```
struct mach_header_64{
  uint32_t magic;
  cpu_type_t cputype;
  cpu_subtype_t cpusubtype;
  uint32_t filetype;
  uint32_t ncmds;
  uint32_t sizeofcmds;
  uint32_t flags;
  uint32_t reserved;
}
```

##### 字段含义

magic:魔数，用于快速确认该文件用于64位还是32位

cputype:CPU类型

cpusubtype:对应的具体类型

filetype:文件类型，比如可执行文件、库文件、Dsym文件等。

```
MH_OBJECT 0x1 relocatable object file //可重定位的对象文件
MH_EXECUTE 0x2 demand paged executable file //按需分页的可执行文件
MH_FVMLIB 0x3 fixed VM shared library file
MH_CORE 0x4 core file
MH_PRELOAD 0x5 preloaded executable file
MH_DYLIB 0x6 dynamically bound shared library
MH_DYLINKER 0x7 dynamic link editor
MH_BUNDLE 0x8 dynamically bound bundle file
MH_DYLIB_STUB 0x9 shared library stub for static
MH_DSYM 0xa companion file with only debug
MH_KEXT_BUNDLE 0xb x86_64 kexts
```

ncmds:加载命令条数

sizeofcmds:所有加载命令的大小

reserved:保留字段

flags:标志位

```
MH_NOUNDEFS 0x1 目前没有未定义的符号，不存在链接依赖
MH_DYLDLINK 0x4 该文件是dyld(动态链接器，当内核执行LC_DYLINKER时，连接器会启动，查找进程所依赖的动态库，并加载到内存中)的输入文件，无法被再次静态链接
MH_PIE 0x200000 加载程序在随机的地址空间，只在MH_EXECUTE中使用
MH_TWOLEVEL 0x80 两级名称空间(dyld的独有特性，说是符号空间中还包括所在库的信息，使得两个不同的库导出相同的符号，对应平坦名称空间)
```

#### Load commands

Load commands加载指令清晰地告诉加载器如何处理二进制数据，有些命令由内核处理，有些由动态链接器处理。

示例:

```
将文件的32位或64位的段映射到进程地址空间，加载的主要命令，负责指导内核来设置进程的内存空间
LC_SEGMENT 0x1
LC_SEGMENT_64 0x19

唯一的UUID，标识二进制文件
LC_UUID 0x1b

启动动态加载链接器
LC_LOAD_DYLINKER 0xe

代码签名和加密
LC_CODE_SIGNATURE 0x1d
LC_ENCRYPTION_INFO 0x21
```

##### 数据结构

```
struct load_command{
  uint32_t cmd;//type of load command
  uint32_t cmdsize;//total size of command in bytes
}
```

##### 示例:LC_SEGMENT_64

LC_SEGMENT和LC_SEGMENT_64是加载的主要命令，负责指导内核来设置进程的内存空间。

```
struct segment_command_64{// for 64-bit architectures
  uint32_t cmd;//LC_SEGMENT_64
  uint32_t cmdsize;//includes sizeof section_64 structs，load command的大小
  char segname[16];//segment name，段的名称
  uint64_t vmaddr;//memory address of this segment，段的虚拟内存地址
  uint64_t vmsize;//memory size of this segment，段的虚拟内存大小
  uint64_t fileoff;//file offset of this segment，段在文件中的偏移量
  uint64_t filesize;//amount to map from file，段在文件中的大小
  vm_prot_t maxprot;//maximum VM protection
  vm_prot_t initprot;//initial VM protection
  uint32_t nsects;//number of sections in segment，标示了segment有多少个section
  uint32_t flags;//flags
}
```

将该段对应的文件内容加载到内存中:从offset处加载file size大小到虚拟内存vmaddr处

#### Segments

端数据Segments包含多个segment，每个segment定义了一些Mach-O文件的数据、地址和内存保护属性，这些数据在动态链接器加载程序时被映射到虚拟内存中。每个段都有不同的功能，一般包括:

- _PAGEZEOR:空指针陷阱段，映射到虚拟内存空间的第一页，用于捕捉对NULL指针的引用。
- __TEXT:包含了执行代码以及其他只读数据。为了让内核将它直接从可执行文件映射到共享内存，静态链接器设置该段的虚拟内存权限为不允许写，当该段被映射到内存后，可以被所有进程共享。(主要用在framworks，bundles和共享库等程序中，也可以为同一个可执行文件的多个进程拷贝使用)
- __DATA:包含了程序数据，该段可写
- __OBJC:Objective-C运行时支持库
- __LINKEDIT:含有为动态链接库使用的原始数据，比如符号，字符串，重定位表条目等等。

一般段会按不同的功能划分为几个区(section)。

##### Section的数据结构

```
struct section{
  char sectname[16];//name of this section，如_text,stubs
  char segname[16];//segment this section goes in，该section所属的segment，如__TEXT
  uint32_t addr;//memory address of this section，该section在内存的起始位置
  uint32_t size;//size in bytes of this section，该section的大小
  uint32_t offset;//file offset of this section，该section的文件位移
  uint32_t align;//section alignment，字节大小对齐
  uint32_t reloff;//file offset of relocation entries，重定位入口的文件偏移
  uint32_t nreloc;//number of relocation entries，需要重定位的入口数量
  uint32_t flags;//section 的 type and attributes
  uint32_t reserved1;//reserved (for offset or index)
  uint32_t reserved2;//reserved (for count or size)
}
```

段中可能包含的section:

__TEXT段:

\_\_text(实际上的代码部分),\_\_cstring,\_\_picsymbol\_stub,\_\_symbol\_stub,\_\_const,\_\_litera14,\_\_litera18

__DATA段:

\_\_data(实际的初始数据),\_\_la\_symbol\_ptr,\_\_nl\_symbol\_ptr,\_\_dyld,\_\_const,\_\_mod\_init\_func,\_\_mod\_term\_func,\_\_bss,\_\_common

\_\_IMPORT段:

\_\_jump\_table,\_\_pointers

### 加载过程

在linux中通过fork来创建子进程，然后执行exec来替换为另一个可执行程序。

原因:fork创建子进程时，拷贝文本、数据和bss段、堆以及用户栈，但是新的线程只会复制调用了fork的线程，其他线程不复制。在多线程中，该线程可能并不拥有锁，最终将导致死锁问题和内存泄漏。在线程中调用exec函数，可以废弃掉锁，从而避免死锁。

我们在用户态会通过exec\*系列函数来加载一个可执行文件，同时exec\*都只是对系统调用execve的封装，那我们加载Mach-O的流程，就从execve说起。Mach-O有多种文件类型，比如MH_DYLIB文件、MH_BUNDLE文件，MH_EXECUTE文件(这些需要dyld动态加载)，MH_OBJECT(内核加载)等，所以一个进程往往不是只需要内核加载器就可以完成加载的，还需要dyld来进行动态加载配合。考虑内核加载和dyld加载两种情况，有如下流程:

execve()->\_mac\_execve()->exec\_active\_image()->exec\_mach\_imgact->

1.load\_machfile()->parse\_machfile()->load\_dylinker()->dyld->main()

2.dyld->main()

##### execve()

只是直接调用\_\_mac\_execve()

##### _\_mac\_execve()

主要为加载镜像进行数据的初始化，以及资源相关的操作，在其内部会执行exec_activate_image()，镜像加载的工作都是由它完成的。

##### exec_activate_image

主要拷贝可执行文件到内存中，并根据不同的可执行文件类型选择不同的加载函数，所有的镜像的加载要么终止在一个错误上，要么最终完成加载镜像。OS X中专门处理可执行文件格式的程序叫execsw镜像加载器。

OS X有三种可执行文件

mach-o由exec_mach_imgact处理

fat binary由exec_fat_imgact处理

interpreter由exec_shell_imgact处理

##### exec_mach_imgact

主要用来对Mach-O做检测，会检测Mach-O头部，解析其架构，检查imgp等内容，并拒绝Dylib和Bundle这样的文件(由dyld负责加载)，然后把Mach-O映射到内存中去，调用load_machfile()。

主要完成以下几个过程:

1.为vfork生成新的线程

2.把Mach-O映射进内存

3.签名、uid等权限处理，dyld相关的处理工作

4.释放资源

##### load_machfile

加载Mach-O中的各种load command命令，在其内部会禁止数据段执行，防止溢出漏洞攻击，同时设置地址空间布局随机化(ASLR)，还有一些映射的调整。

##### parse_machfile:真正负责对加载命令解析

parse_machfile会根据load_command的种类选择不同的函数来加载。

对命令的加载会进行多次扫描，当扫描三次之后，并且存在dylinker_command命令时，会执行load_dylinker()，启动动态链接器(dyld)

#### 动态链接过程

动态链接也有区分，一种是加载主程序，是由load commands指定的dylib，以静态的方式存放在二进制文件里。一种是由DYLD_INSERT_LIBRARIES动态指定。



### 名词解析

Mach:微内核，主要实现基本的进程，虚拟内存管理，任务调度，进程通信和消息机制。

Mach-O:苹果系操作系统的可执行文件格式

BSD:类Unix操作系统中的一个分支的总称，iOS中的BSD是指对Mach层的封装和扩展，它提供了更现代的API和对POSIX的兼容性，比如Mach层fork、vfork可用来创建进程，而BSD层则定义了posix_spawn来进行进程创建，还有进程结构proc是BSD层的进程结构，扩展了Mach层的task进程结构。

POSIX:可移植操作系统接口(Portable Operation System Interface of UNIX)，POSIX标准定义了操作系统应该为应用程序提供的接口标准，是IEEE为要在各种UNIX操作系统上运行的软件而定义的一系列API标准的总称。

XNU:最底层包括Mach、BSD、主要基于开源技术的驱动器等组成了XNU。

Darwin:苹果操作系统内核的内部代号，由XNU和Darwin库等组成

dyld:动态库加载器，负责将动态库加载进内存。

dlopen:dlopen函数是以指定模式打开指定的动态库文件，并返回一个句柄给调用进程。若调用成功，指定的动态库就会被加载进内存中。

mmap:将一个文件或者其他对象映射进内存。

虚拟内存:

ASLR:Address space layout randomization是一种针对缓冲区溢出的安全保护技术，通过对堆、栈、共享库映射等线性区布局的随机化，从而增加攻击者预测目的地址的难度。

用户态和内核态:出于安全考虑，需要限制不同程序之间的访问能力、获取其他程序的数据，防止访问外围设备等，所以区分了用户态和内核态。用户态只能受限地访问内存，且不允许访问外围设备，内核态可以访问内存所有数据，包括外围设备，用户态如果需要做一些内核态的事情，需要通过系统调用机制从用户态切换到内核态。

### XNU源码分析

MacOS X 中的X指的就是XNU，即X is not Unix，XNU中的Mach是基于Unix的一个内核，上层封装了BSD和其他库组成了Darwin，Darwin上层依次还有核心框架层、应用框架层、用户体验层等。

XNU主要由4部分组成

- Mach

  微内核，主要实现基本的进程、虚拟内存管理、任务调度、进程通信和消息机制


- BSD

  对Mach层的封装和扩展

- libkern

  IOKit的驱动程序

- IOKit

  设备驱动程序运行时环境，如设备的电源信息、内存信息、CPU信息等都是在IOKit进行管理的。

### XNU加载Mach-O和dyld流程

大体流程:

1.创建进程

2.创建虚拟内存空间

3.解析和映射Mach-O

4.解析映射dyld

### 虚拟内存分布

共享动态库其实就是共享的物理内存中的那份动态库，App虚拟内存中的共享动态库并未正式分配物理内存，使用时虚拟内存会访问同一份物理内存达到共享动态库的目的。

以ARM64为例，App最大的虚拟内存空间为64GB，一个默认的PAGEZERO捕获空指针异常区4GB，Mach-O和dyld的ASLR，共享动态库的随机slide等，大体可以得出App的虚拟内存分布如下:

| 未分配的内存空间                    |
| --------------------------- |
| 共享动态库的内存空间                  |
| *dyld_shared_cache随机偏移slide |
| dyld的用户态内存空间                |
| *dyld的随机偏移ASLR              |
| Mach-O的内存空间                 |
| *主程序mach-o的随机偏移ASLR         |
| PAGEZERO                    |

### dyld源码分析

动态库也是一个静态的文件，文件格式也是Mach-O，其本身不能直接运行，需要加载器(即dyld，路径为/usr/lib/dyld)将其加载进内存空间。dyld承担了将动态库以镜像的方式映射进内存的工作。

### 沙盒权限和代码签名

