---
title: '深入理解Java虚拟机'
date: 2023-10-27T16:34:20+08:00
draft: false
---

# 深入理解Java虚拟机

## 走近Java

### 1. JVM现状

#### 当前环境下Java的劣势

  在**微服务架构**的视角下，有了高可用的服务集群，也就无需追求单个服务7*24小时运行，反而因为Java的启动时间相对较长，需要预热才能达到最高性能等特点就显得有悖于这样的应用场景。
  比起服务，一个函数的规模通常会更小，执行时间更短，当前最热门的无服务运行环境AWS Lambda 所允许的最长运行时间仅有15分钟。

#### Java的提前编译

  **提前编译(Ahead of Time Compilation ATO)**是相对于即时编译的概念。
  提前编译能带来的最大好处是Java虚拟机加载这些已经编译成二进制库之后就能够直接调用，而无需等待即时编译器在运行时将其编译成二进制机器码。理论上，提前编译可以减少即时编译带来的预热时间。
  但是提前编译的坏处也很明显，它破坏了Java “一次编写，到处运行”的承诺，必须为不同的硬件、操作系统编译对应的发行包，必须要求加载的代码在编译期就是全部已知的，而不能在运行期才确定，否则就只能舍弃掉已经提前编译好的版本

#### Substrate VM
  Substrate VM 是在Graal VM 0.20版本里出现的一个**极小型的运行时环境**，包括了独立的议程处理、同步调度、线程管理、内存管理(垃圾回收)和JNI访问等组件，目标是替代HotSpot来支持提前编译后的程序执行。它还包含了一个本地镜像的构造器(Native Image Generator), 用于为用户程序建立基于Substrate VM的本地运行环境

  构造器采用**指针分析(Points-To Analysis)**技术，从用户提供的程序入口出发，搜索所有可达的代码，在搜索的同时，它还执行初始化代码，并在最终生成可执行文件时，将已初始化的堆保存至一个堆快照中。在执行时无需重复进行Java虚拟机的初始化过程。

## 自动内存管理

### 2. Java内存区域与内存溢出异常

#### 运行时数据区域

![Java虚拟机运行时数据区](./img/运行时数据区.png)

##### 程序计数器(Program Counter Pegister)
线程私有

    程序计数器 是流程控制的指示器，各条线程之间计数器互不影响，独立存储(线程私有)
    是唯一一个没有规定任何OutOfMemoryError(OOM)情况的区域


##### Java虚拟机栈(Java Virtual Machine Stack)
线程私有
```
Java虚拟机栈描述的是Java方法执行的线程内存模型，生命周期与线程相同
每个方法被调用直至执行完毕的过程，就对应着一个栈帧(Stack Frame)在虚拟机栈中从入栈道出栈的过程。
每个方法需要在栈帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小
```
```
如果线程请求的栈的深度大于虚拟机所允许的深度，将抛出StackOverflowError
如果Java虚拟机栈容量可以动态扩展(HotSpot栈容量不可以动态扩展)，当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError
```

##### 本地方法栈(Native Method Stack)
线程私有
```

```
```

```