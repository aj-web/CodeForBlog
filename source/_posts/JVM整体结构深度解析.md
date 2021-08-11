---
title: JVM整体结构深度解析
date: 2021年7月14日00:06:33
tags: JVM
------
>前言：为什么要学习JVM？     
> 一门语言有可能会过时，但是它的思想是不会过时的，尤其是作为JAVA跨平台的核心实现，JVM的思想值每个程序员得学习


Java Virtual Machine(JVM) 是一种抽象的计算机，基于堆栈架构，它有自己的指令集和内存管理。它加载 class 文件，分析、解释并执行字节码。基本结构如下：
![JVM结构](https://raw.githubusercontent.com/aj-web/picturebed/master/JVM%E6%9E%84%E6%88%90.png)
JVM 主要分为以上三个子系统：类加载器、运行时数据区和执行引擎，下面我们分部分展开理解

# 1.JVM之运行时数据区
它约定了在运行时程序代码的数据比如变量、参数等等的存储位置，主要包含以下几部分：

