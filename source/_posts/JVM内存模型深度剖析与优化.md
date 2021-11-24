---
title: JVM-1：内存模型深度剖析与优化
date: 2021年7月14日00:06:33
tags: JVM
------
>前言：为什么要学习JVM？     
> 一门语言有可能会过时，但是它的思想是不会过时的，尤其是作为JAVA跨平台的核心实现，JVM的思想值每个程序员得学习
<!--more-->

Java Virtual Machine(JVM) 是一种抽象的计算机，基于堆栈架构，它有自己的指令集和内存管理。它加载 class 文件，分析、解释并执行字节码。基本结构如下：
![JVM结构](https://raw.githubusercontent.com/aj-web/picturebed/master/JVM%E6%9E%84%E6%88%90.png)
JVM 主要分为以上三个子系统：类加载器、运行时数据区和执行引擎，下面我们分部分展开理解

# 1.JVM之运行时数据区
### 1.1 运行时数据区组成
它约定了在运行时程序代码的数据比如变量、参数等等的存储位置，主要包含以下几部分：   
1. 程序计数器：也是每个线程独有的，就是代码执行的位置，为什么要设计程序计数器，是为了多线程代码挂起后恢复执行
2. 本地方法栈：与 JVM 栈类似，只不过服务于 Native 方法   
3. 堆的组成：存储类实例对象和数组对象，垃圾回收的主要区域
4. 栈的组成：new出来对象的引用(对象引用) ,基本数据类型和局部变量
5. 方法区组成：又叫(元空间)，存放运行时常量池，字段和方法的数据，构造函数和方法的字节码等，在 JDK 8 中，把 interned String 和类静态变量移动到了 Java 堆

### 1.2 堆栈原理
讲了其中方法区，程序计数器，本地方法栈比较简单，下面深入理解下，堆和栈的概念：   
(1)堆由年轻代，老年代组成，年轻带占整个堆的1/3，老年代占整个堆的2/3，配比可以调整，年轻代有Eden区Survivor区，配比为8：1：1，new出来的对象一般在Eden区，Eden放满的时候会执行minor gc，回收无用的对象。
回收基本原理，从gcroot，找局部变量 静态变量引用了其他的话，就不是垃圾对象，复制到s0，分代年龄会加1，第二次Eden满的时候s0和Eden都会minor gc存活下来的对象存放到s1，分代年龄+1，当分代年龄到15，会被挪到老年代，老年代满的时候，会full gc，回收后还是满的话，就会内存溢出报错  
![堆的组成](https://raw.githubusercontent.com/aj-web/picturebed/master/%E5%A0%86%E7%9A%84%E7%BB%84%E6%88%90.png)


(2)栈是服务于方法的，当启动一个线程的时候，就会在栈中预先分配出一块空间，当线程执行方法的时候，会在这个预先分配的栈空间中创建一个
栈帧的数据结构。栈帧(Stack Frame)是用于支持虚拟机进行方法调用和方法执行的数据结构，是用来存储数据和部分过程结果的数据结构，同时也用来处理动态连接、方法返回值和异常分派。
栈帧随着方法调用而创建，随着方法结束而销毁——无论方法正常完成还是异常完成都算作方法结束栈帧由以下部分组成：  
- 局部变量表：方法的局部变量和方法参数。main方法的局部变量表中对象变量存放的是堆的地址   
- 操作数栈：局部变量的操作数的临时的内存空间   
- 动态链接：一个指向运行时常量池的引用，将 class 文件中的符号引用（描述一个方法调用了其他方法或访问成员变量）转为直接引用。符号引用一部分会在类加载阶段或者第一次使用时就直接转化为直接引用，这类转化称为静态解析。另一部分将在每次运行期间转化为直接引用，这类转化称为动态连接。  
- 方法返回：方法正常退出或抛出异常退出，返回方法被调用的位置
  ![栈帧](https://raw.githubusercontent.com/aj-web/picturebed/master/%E6%A0%88%E5%B8%A7.png)


### 1.3 JVM概览
![JVM概览](https://raw.githubusercontent.com/aj-web/picturebed/master/JVM.png)  


# 2.JVM内存参数设置
![JVM总体参数](https://raw.githubusercontent.com/aj-web/picturebed/master/JVM%E5%8F%82%E6%95%B0%E8%AE%BE%E7%BD%AE.png)     
- springboot的jvm参数设置格式(Tomcat启动直接加在bin目录下catalina.sh文件里)：
```
java -Xms2048M -Xmx2048M -Xmn1024M -Xss512K -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -jar microservice-eureka-server.jar
-Xss：每个线程的栈大小
```

- 关于元空间的JVM参数有两个：  
  -XX:MetaspaceSize=N和  
  -XX:MaxMetaspaceSize=N  
-XX：MaxMetaspaceSize： 设置元空间最大值， 默认是-1， 即不限制， 或者说只受限于本地内存大小   
-XX：MetaspaceSize： 指定元空间触发Fullgc的初始阈值(元空间无固定初始大小)， 以字节为单位，默认是21M左右，达到该值就会触发full gc进行类型卸载， 同时收集器会对该值进行调整： 如果释放了大量的空间， 就适当降低该值； 如果释放了很少的空间， 那么在不超过-XX：MaxMetaspaceSize（如果设置了的话） 的情况下， 适当提高该值。这个跟早期jdk版本的
- XX:PermSize参数意思不一样，  
  -XX:PermSize代表永久代的初始容量。   
由于调整元空间的大小需要Full GC，这是非常昂贵的操作，如果应用在启动的时候发生大量Full GC，通常都是由于永久代或元空间发生
了大小调整，基于这种情况，一般建议在JVM参数中将MetaspaceSize和MaxMetaspaceSize设置成一样的值，并设置得比初始值要大，
对于8G物理内存的机器来说，一般我会将这两个值都设置为256M。

结论：
-Xss设置越小count值越小，说明一个线程栈里能分配的栈帧就越少，但是对JVM整体来说能开启的线程数会更多

### 2.1 日均百万级订单交易系统如何设置JVM参数
![亿级流量JVM参数调优](https://raw.githubusercontent.com/aj-web/picturebed/master/%E4%BA%BF%E7%BA%A7%E6%B5%81%E9%87%8FJVM%E5%8F%82%E6%95%B0%E8%AE%BE%E7%BD%AE.png)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;结论：通过上面这些内容介绍，大家应该对JVM优化有些概念了，就是尽可能让对象都在新生代里分配和回收，尽量别让太多对象频繁进入老年代，避免频繁对老年代进行垃圾回收，同时给系统充足的内存大小，避免新生代频繁的进行垃圾回收。