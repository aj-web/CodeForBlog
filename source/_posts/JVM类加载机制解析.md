---
title: JVM类加载机制解析
date: 2021年7月14日00:06:33
tags: JVM
------

# 1.1类加载机制
类加载步骤：加载 > > 验证 > > 准备 > > 解析 > > 初始化 > > 使用 > > 卸载
<!--more-->
- 加载：在硬盘上查找并通过IO读入字节码文件，使用到类时才会加载，例如调用类的main()方法，new对象等等，在加载阶段会在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口
- 验证：校验字节码文件的正确性
- 准备：给类的静态变量（类变量）分配内存，并赋予默认值
- 解析：将符号引用替换为直接引用，该阶段会把一些静态方法(符号引用，比如main()方法)替换为指向数据所存内存的指针或句柄等(直接引用)，这是所谓的静态链接过程(类加载期间完成)，动态链接是在程序运行期间完成的将符号引用替换为直接引用
- 初始化:对类的静态变量初始化为指定的值，执行静态代码块

>以上是类加载的5个阶段，那么除了类加载的5个阶段，还有：`使用`，`卸载`两个阶段   
>也就是类的生命周期=类加载+`使用`+`卸载`

### 1.1类加载机制拓展理解
上面的基本概念是各网站都能搜到的，我们再结合自己进行拓展，理解一下：

**加载：**

1. 加载会通过限定名(可以简单理解为类名)获取到类的二进制字节流
2. 将二进制字节文件的数据放到方法区，然后在堆中生产一个代表这个类的java.lang.Class对象，Class 对象封装了类在方法区内的数据结构，并且向开发者提供了访问方法区内的数据结构的接口

所以我们可以认为，开发中用的是Class对象，但是当我们想要这个类的某些信息的时候，我们需要通过这个Class对象，到方法区中去找。例如类的`元数据`和`方法信息(继承信息、成员变量、静态变量、成员方法、构造函数)`。那么如何找到，就涉及到对象头中的一个Klass point:类型指针，通过类型指针来找到方法区中的元数据等。

**验证：**

1. 文件格式验证：（class文件来源不唯一(自己也可以手写)，有可能格式正确损坏虚拟机）
2. 元数据验证：（是否符合类的定义规范，例如是否继承java.lang.Object）
3. 字节码验证：（类中方法的控制流是否合法）
4. 符号引用验证：（转换为直接引用动作是否合法）

**准备：**
1. 为类变量分配内存，赋予默认值
2. 实例变量会在创建对象过程中一起被分配，详情看下一章

**解析：**
1. 将符号引用替换为直接引用
2. 符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。
3. 直接引用（Direct References）：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。如果有了直接引用，那么引用的目标一定是已经存在于内存中。

上面我们已经知道了，类的元数据和方法信息是存在方法区的，那么方法区可能存在符号引用，这个时候就需要进行解析了，通常有：1.类或接口的解析2.字段解析3.类方法解析4.接口方法解析，解析之后，针对我们解析到的内容，可能还需要进行上面的加载验证准备步骤

**初始化**
1. 为类变量赋予默认值
2. 执行静态代码块，静态方法
3. 执行构造方法


执行顺序
- 先加载父类的静态代码块和静态变量 这两个加载的顺序与代码顺序有关
- 加载子类的静态代码块和静态变量  加载的顺序也与位置有关
- 加载父类的变量和语句块
- 加载父类的构造方法
- 加载子类的变量和语句块
- 加载子类的构造方法

# 2.虚拟机的内存分配情况

1. 虚拟机栈：每个class类对应一个虚拟机栈帧（组成：局部变量表、操作数栈、返回地址、动态链接），类私有
2. 堆：存放对象
3. 方法区：存放类信息、常量、类变量、即时编译器编译后的代码
4. 常量池：是方法区的一部分，主要有字面量（常量和字符串）和符号引用（类和接口的符号引用、字段的名称和描述的符号引用、方法的名称和描述的符号引用）




# 3.类加载器和双亲委派机制

### 3.1类加载过程主要是通过类加载器来实现的，Java里有如下几种类加载器

1. 引导类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如rt.jar、charsets.jar等
2. 扩展类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的ext扩展目录中的JAR类包
3. 应用程序类加载器：负责加载ClassPath路径下的类包，主要就是加载你自己写的那些类
4. 自定义加载器：负责加载用户自定义路径下的类包


### 3.2类加载器初始化过程：
参见类运行加载全过程图可知其中会创建JVM启动器实例sun.misc.Launcher。
在Launcher构造方法内部，其创建了两个类加载器，分别是sun.misc.Launcher.ExtClassLoader(扩展类加载器)和sun.misc.Launcher.AppClassLoader(应用类加载器)。
JVM默认使用Launcher的getClassLoader()方法返回的类加载器AppClassLoader的实例加载我们的应用程序。

```
//Launcher的构造方法
public Launcher() {
    Launcher.ExtClassLoader var1;
    try {
        //构造扩展类加载器，在构造的过程中将其父加载器设置为null
        var1 = Launcher.ExtClassLoader.getExtClassLoader();
    } catch (IOException var10) {
        throw new InternalError("Could not create extension class loader", var10);
    }

    try {
        //构造应用类加载器，在构造的过程中将其父加载器设置为ExtClassLoader，
        //Launcher的loader属性值是AppClassLoader，我们一般都是用这个类加载器来加载我们自己写的应用程序
        this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
    } catch (IOException var9) {
        throw new InternalError("Could not create application class loader", var9);
    }

    Thread.currentThread().setContextClassLoader(this.loader);
    String var2 = System.getProperty("java.security.manager");
    。。。 。。。 //省略一些不需关注代码

}
```



### 3.3双亲委派机制
JVM类加载器是有亲子层级结构的，如下图
![双亲委派机制](https://raw.githubusercontent.com/aj-web/picturebed/master/%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE.png)

这里类加载其实就有一个双亲委派机制，加载某个类时会先委托父加载器寻找目标类，找不到再委托上层父加载器加载，如果所有父加载器在自己的加载类路径下都找不到目标类，则在自己的类加载路径中查找并载入目标类。
比如我们的Math类，最先会找应用程序类加载器加载，应用程序类加载器会先委托扩展类加载器加载，扩展类加载器再委托引导类加载器，顶层引导类加载器在自己的类加载路径里找了半天没找到Math类，则向下退回加载Math类的请求，扩展类加载器收到回复就自己加载，在自己的类加载路径里找了半天也没找到Math类，又向下退回Math类的加载请求给应用程序类加载器，应用程序类加载器于是在自己的类加载路径里找Math类，结果找到了就自己加载了。。
双亲委派机制说简单点就是，先找父亲加载，不行再由儿子自己加载

### 3.4双亲委派机制的好处   
- 沙箱安全机制：自己写的java.lang.String.class类不会被加载，这样便可以防止核心API库被随意篡改
- 避免类的重复加载：当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次，保证被加载类的唯一性

### 3.5类加载源码分析
我们来看下应用程序类加载器AppClassLoader加载类的双亲委派机制源码，AppClassLoader的loadClass方法最终会调用其父类ClassLoader的loadClass方法，该方法的大体逻辑如下：
首先，检查一下指定名称的类是否已经加载过，如果加载过了，就不需要再加载，直接返回。
如果此类没有加载过，那么，再判断一下是否有父加载器；如果有父加载器，则由父加载器加载（即调用parent.loadClass(name, false);）.或者是调用bootstrap类加载器来加载。
如果父加载器及bootstrap类加载器都没有找到指定的类，那么调用当前类加载器的findClass方法来完成类加载。

```
//ClassLoader的loadClass方法，里面实现了双亲委派机制
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 检查当前类加载器是否已经加载了该类
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {  //如果当前加载器父加载器不为空则委托父加载器加载该类
                    c = parent.loadClass(name, false);
                } else {  //如果当前加载器父加载器为空则委托引导类加载器加载该类
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                //都会调用URLClassLoader的findClass方法在加载器的类路径里查找并加载该类
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {  //不会执行
            resolveClass(c);
        }
        return c;
    }
}
```

### 3.6全盘负责委托机制
“全盘负责”是指当一个ClassLoder装载一个类时，除非显示的使用另外一个ClassLoder，该类所依赖及引用的类也由这个ClassLoder载入。