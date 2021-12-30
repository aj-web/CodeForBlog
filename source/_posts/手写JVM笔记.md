---
title: JVM-5：JVM底层学习
date: 2021-11-27
mtime: 2021-11-27
tags: [JVM]
categories: [JVM]
------

>前言：Java创建线程的能力，实际上是通过调用操作系统的能力实现的
<!--more-->

## Linux多线程机制：

- Linux创建线程：`pthread_create`  

- 等待子线程运行结束： `pthread_join`

- 线程属性：`pthread_attr_int`,`pthread_attr_destory`

- 线程栈大小：默认1M，可以修改

- 运行机制有两种：1.join机制   2.Detach机制：线程执行完了会自动释放资源（java中的线程运行机制就是Detach机制），又称分离线程

- 使用join机制时获取线程结果有两种，一种是通过`pthread_join`接收线程的执行结果，或者是在子线程中通过`pthread_exit`来退出子线程同时返回一个结果，但是由于JVM的线程是Detach机制创建的，所以并不能用上面两种方案实现

- 如果在Linux中创建线程使用Detach机制，那么`pthread_join`是无效的，早期Linux版本会报错。那么这个时候如何获取线程的执行结果呢？答案就是通过修改共享内存上变量的值

  

  我们都知道Java创建的线程其实是调用的操作系统的能力，那么从java中的线程对象到OS的线程对象，依次经过以下对应关系，而Linux线程操作的结果，就会存储在JVM的Java Thread对象中的_VM_result_2中，这个对象就可以理解为共享内存

  ![](https://raw.githubusercontent.com/aj-web/picturebed/master/202111271645498.png)

  

  

  ![线程对象](https://raw.githubusercontent.com/aj-web/picturebed/master/202111271641514.png)

  

  

  

  # 关于STW

  1. GC的时候会触发STW，暂停所有用户的线程，但是暂停用户线程的前提是用户到达了安全点

  2. 常见的安全点有： 1.方法返回之前  2.调用某个方法之后  3.抛出异常的位置  4.循环的末尾

  

  **GC时间=垃圾回收的时间+线程到达安全点的时间**

  

  那么安全点到底是什么？这个得去JVM的源码中看下，其实安全点的本质就是一个4字节的可读可写可执行的内存区域，其实是1字节，但是为了对齐就申请了4字节，当正常执行时，线程运行到某个特殊方法对安全点进行操作，此时不会抛出异常，***其实本质没有进行啥操作但是重要的是没有抛出信号（异常）***，但是如果GC需要阻塞线程的时候，JVM会把安全点修改成为不可读不可写不可执行，这样当线程运行到操作安全点的方法时就会触发异常。抛出信号SIGSEGV，JVM中会提前注册了这个信号，所以抛出信号后会进行宿舍，执行阻塞的操作是CAS实现的。

  ![](https://raw.githubusercontent.com/aj-web/picturebed/master/202111271724397.png)

  

  

  

  

  happens-befor------先行发生原则

  

  

  

  

  