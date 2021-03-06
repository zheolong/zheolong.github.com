---
layout: post
title: "Garbage Collection"
modified:
categories: blog
excerpt:
tags: []
comments: true
share: true
counter: true
image:
  feature:
date: 2016-03-06T00:36:11+08:00
---


## 垃圾回收

垃圾回收（GC）是一种自动内存管理机制，垃圾收集器（garbage collector）会尝试回收程序中无用的对象，是约翰·麦卡锡在1959年前后为了去掉Lisp中的手动回收机制而发明的。

手动回收需要指定需要回收的对象。还有一些其他的混合机制，例如stack collection和region reference。自动垃圾回收会占用处理时间，所以对程序的性能有一定的影响。

除了内存，网络套接字、数据库句柄、用户交互窗口、文件和设备描述符，这些资源一般不是通过垃圾回收机制处理的。管理这些资源的通常是destructor。一些GC可以将这些资源与内存区域联系起来，通过回收内存的机制，将这些资源回收，即finalization。Finalization机制会引入一些复杂性，限制其使用，比如对于特别受限的资源，在废弃和回收之间存在无法忍受的延迟，对于哪个线程来执行具体的回收过程缺乏控制。

> ## 维持一种无穷存储的假象
> 
> 垃圾回收这个概念还是从LISP来的，SICP中简单讲解了下存储分配和垃圾回收，Lisp需要将其抽象的计算过程映射到存储的管理上，强烈地依赖于创建新数据对象的能力，无论是显示创建的，还是Lisp解释器本身创建的，例如环境和参数表等。但是计算机没有可以无穷寻址的存储器，所以想要不断创建新的对象，那就必须对已经无用的中间结果对象所占用的存储空间进行回收。这种“automatic storage allocation”必须提供一种假象，即计算机似乎拥有无限的存储空间。可以用很多方法来实现，这里只讨论垃圾回收机制。
> 
> Lisp中用了对象表的方式来维护遇到的符号，一个符号就是一个带类型的指针，所以在遇到新的字符串时，如果需要构造符号，会先查找对象表，如果已经存在，就直接返回对应的指针，否则创建并插入新的符号。
> 
> Lisp中的表操作可以代换为基本的向量操作，向量就是可以随机访问的一段空间，而堆栈可以用表来模拟，虽然大部分体系结构里为堆栈分配了单独的向量（可以通过增减上下标的方式实现）。
> 
> 如何检测哪些序对（conns）已经无用了：在Lisp解释过程中，从当前机器寄存器的指针开始，经过car和cdr能够到达的那些对象还是有用的，其他的都可以被回收了。

## 原理

许多语言需要垃圾回收机制，有些属于语言规范（例如java、C#、D、Go和多数脚本语言），或者为了实际的实现（例如函数式语言）；这些可以称为垃圾回收语言。一些语言设计时就是手动内存管理，但是也能够实现垃圾回收（例如C/C++）。有些语言是这两种方式混合，例如Ada、Modula-3和C++/CLI，有自动垃圾回收，也有单独的堆用于手动管理的对象。其他，例如D，具有自动垃圾回收，也允许手动删除对象，或者为了性能关闭垃圾回收。

### 优势

垃圾回收使得程序员不需要手动处理内存释放问题，可以消除或减少特定类型的bug。

* 野指针（Dangling pointer bugs）
* 重复释放（Double free bugs）
* 特定种类的内存泄露（Memory leaks）
* Persistent data structure的有效实现

垃圾回收解决的一些bug可能有安全隐患。

### 劣势

当然，垃圾回收也有一些缺点，消耗额外的资源，影响性能，造成程序执行过程的停顿，与手动资源管理的兼容问题。

在决定要释放哪些内存时，GC会消耗计算资源，虽然程序员已经知道要释放哪些内存了。GC免除了手动指定对象生命周期的麻烦，代价就是overhead，会导致程序性能下降或不均匀。一篇同行评议论文中给出结论，GC可能需要5倍的内存来补偿这种overhead，才能与显示内存管理运行得一样快。内存层级越复杂，常规测试可能难以预估或检测，使得这种overhead难以忍受。这种对性能的影响也是Apple在iOS中弃用垃圾回收的原因。

无用内存的实际回收时间可能无法预估，导致程序可能停顿（用来迁移/释放内存）。在实时环境、事务处理或交互程序中，无法预估的停顿会变得令人无法接受。递增、并发和实时垃圾回收通过各种权衡解决这些问题。

现在的GC都会通过用后台线程承担尽可能多的工作，来减少这种停顿的影响。GC在对内存进行标记的时候，程序还在继续执行。

非确定性GC与非GC资源基于RAII的管理机制不兼容。在非确定性GC中，如果资源或资源类对象需要手动资源管理（release/close），而且此对象是其他对象的“一部分”，那么合成对象也需要进行手动资源管理（release/close）。

## 追踪垃圾回收（Tracing garbage collection）

追踪垃圾回收是最常用的垃圾回收机制，从特定的根对象开始，通过引用链标记可达（reachable）的对象，然后将其余的对象作为垃圾进行回收。具体的实现方法很多，在此不赘述。

## 引用计数（Reference counting）

每个对象会保存指向自身的引用总数，创建引用则递增，销毁引用则递减，当引用数为零则可以将对象回收。

与手动内存管理类似，而与自动垃圾回收不同的是，引用计数可以在最后一个引用被销毁时就回收内存，并且仅访问在CPU缓存、所指对象、或二者直接所指向的空间，对CPU缓存或虚拟内存管理没有显著的负面影响。

当然，引用计数也有缺点。

### 循环引用（Cycles）

CPython中使用基于引用计数的垃圾回收，采用了一种特殊的循环引用检测算法来解决这个问题。

另一种解决办法是使用弱引用（weak reference），在引用计数中，弱引用与追踪垃圾回收中的弱引用类似。弱引用是一种特殊的引用，不会增加被引用对象的引用计数，并且在所引用对象变成垃圾以后，指向其的所有弱引用都会*到期*，而不会变成野引用，其值会变成可预期的值，例如null。

### 空间代价

每个对象都需要一个引用总数，无论是在对象空间内，还是用专门的表来存储。

在一些系统，可能通过tagged pointer将引用总数存储在对象内存的无用区域。通常，一种体系结构不会允许程序访问能够存在本地指针的全部内存地址，所以指针的高位经常有固定的比特是无用的。如果对于一个对象，总有一个指针放在特定的位置，就可以利用这个指针的高位无用比特来存储引用总数。

### 速度代价（增加/减少）

在不成熟的实现中，引用的赋值，或者引用的范围变化需要更改一个或多个引用总数。然而，当引用从外层空间被拷贝到内层空间变量时，内层空间变量的生命周期受限于外层空间的，可以不增加引用总数。在C++中，可以用const引用来实现。

通常，C++用“智能指针”来实现引用计数，其构造函数、析构函数和赋值操作符管理着引用计数。可以通过引用将智能指针传入函数，这样就可以避免触发智能指针的拷贝构造函数（入函数时会增加引用总数，出函数时会减小引用总数）。

### 需要原子性

在多线程环境下使用时，这些修改（增加或减少）必须是原子操作，例如CAS，至少对于多线程共享或可能共享的对象来说如此。对于多处理器，原子操作代价较高，如果使用软件模拟，代价更高。

为了避免这种问题，可以对每个线程或每个CPU都增加引用计数，只有在本地引用计数变成或不再为0时，才会访问全局引用计数（或者，用二叉搜索树保存引用计数，或者干脆放弃确定性的destruction，去掉全局引用计数），但是，这会占用更多的内存，只在特定的情况可以使用（例如，用在内核模块的引用计数上）。

### 非实时

不成熟的实现不能提供实时操作，因为任何指针赋值都可能导致一连串的对象空间释放，此时，线程无法进行其他操作。要避免这个问题，可以将释放空间的操作交给另一个线程去做，当然，这也是有代价的。

## 逃逸分析（Escape analysis）

逃逸分析可以用来将堆分配转为栈分配，减少垃圾回收器的工作量。通过编译期间的分析，可以确定函数内分配的对象会不会在函数外的其他函数或线程访问到（逃逸）。这种情况下，可以直接在线程栈上直接分配对象空间，然后在函数返回的时候回收之。

### 编译时

编译时垃圾回收是一种静态分析形式，允许基于编译期间获悉的不变量来重用或回收空间。在Mercury编程语言中被研究过，在LLVM的自动引用计数器（ARC）中使用了。

## 可用性

一般来说，高级语言可能会具有垃圾回收机制。对于没有这种机制的语言，通常可以增加library来扩展，比如用于C和C++的Boehm垃圾收集器。这种方法也有缺点，会改变对象的创建和销毁过程。

大多数的函数式编程语言，例如ML、Haskell和APL，都内置了GC。特别是Lisp，是第一个函数式编程语言，也是第一个内置了GC的语言。

其他语言，例如Ruby，也倾向于使用GC（Perl 5和PHP < 5.3使用的是引用计数）。面向对象语言，如Smalltalk、Java、JavaScript和ECMAScript都提供了集成的GC。C++和Dephi只有destructor。
