# JVM

- java如何从编码到执行
  - 先将Java文件通过javac编译成class文件，然后将class文件装载到内存（classloader），并且将相关的Java类库加载到classloader，调用字节码解释器或者即时编译器（JIT）进行解释或者编译，再调用执行引擎进行执行
- 只要是一门语言能够编译成class文件，那么就可以运行在jvm上，jvm跟Java没关系，只跟class文件有关

## Class文件

### 加载class

- 过程：

  - 加载->验证->准备->解析->初始化
  - 加载：
    - 加载过程既可以使用内置的引导类加载器完成，也可以自定义类加载器去完成
    - 但是数组类，数组类本身不通过类加载器创建，它是由Java虚拟机直接在内存中动态构造出来的。但是数组的类型最终还是要靠类加载器来完成加载。
    - 加载的过程
      1. 通过一个类的全限定类名获取定义此类的二进制字节流	
      2. 将这个字节流所代表的静态储存结构转换为方法区的运行时数据结构
      3. 在内存生成一个代表这个类的java.lang.class对象，作为方法去这个类的各种数据的访问入口
    - 对于数组类的加载过程
      1. 如果数组的组件类型是引用类型，那就递归采用上文的类加载过程去加载这个类型，**数组将被表示在加载该类型的类加载器的类名空间上**
      2. 如果类型不是引用类型，比如是int，那么JVM会吧数组标记为与引导类加载器关联
      3. **数组类**的可访问类型与它的组件类型的可访问性一致，如果组件类型不是引用类型，**他的数组类的可访问性将默认为public，可被所有的类和接口访问到。**

  - 验证：
    - 验证是确保class文件的字节流包含的信息符合JVM规范的全部约束要求
    - 过程：
      1. 文件格式的验证：验证字节流是否符合Class文件格式的规范
         1. 比如class文件的开头是不是**0xCAFEBABE**
         2. 等等，检查项有很多
      2. 元数据验证：对字节码描述的信息进行语义分析
         1. 比如是否有父类（Object类除外）
         2. 父类是否继承了不允许被继承的类（也就是被final修饰的类）
         3. .....
      3. 字节码验证：这是整个验证阶段最复杂的阶段，主要是通过数据流分析和控制流分析，确定程序语义是合法的，符合逻辑的，这个阶段就要对方法体进行分析
      4. 符号引用验证：
         1. 这个阶段验证发生在JVM将符号引用转化为可以直接引用的时候
  - 准备：
    - 准备阶段是正式为类中定义的变量（静态变量）分配内存并设置类变量初始值（默认值）的阶段，从概念上讲，这些变量所使用的内存都应该在方法区中进行分配。但在1.8及以后，类变量则会随着Class对象一起存放在Java堆中。
  - 解析：
    -  解析阶段是Java虚拟机将常量池内的序号引用替换为直接引用的过程，符号应用是以一组符号来描述所引用的目标，符号可以是任何的字面量
      1. 类或接口的解析
      2. 字段解析
      3. 方法解析
      4. 接口方法解析
  - 初始化
    - 类的初始化阶段是类加载过程的最后一个步骤，此时，JVM开始真正的执行类中编写的Java程序代码，将主导权移交给应用程序

## 类加载

1. 如果有自己的类加载器，那就从从cache（是该类自己实现的一个容器）中找，如果有返回，没有话，进行下一项

2. 去app的cache中找，有的话返回，没有的话去下一项

3. 再去Ext的cache中找，有的话返回，没有去下一项

4. 去bootStrapt中的cache中找，有的话返回，没有

5. 让Ext去加载，加载成功，返回，没有的话，让Ext去加载，加载成功返回，不成功的话

6. 由app加载，加载成功，返回，不成功，让自定义的去加载

7. 成功返回，不成功，返回ClassNotFindClassException

- ![image-20210615180708694](D:\面试\学习笔记\image\image-20210615180708694.png)

#### 为什么要**双亲委派**

- **双亲委派**
  
  - 指的不是父子关系，而是从自定义到Bootstrap再从Bootstrap到自定义类加载器
- 原因：
  
  - 比如object类，无论哪个类加载器来加载它，都会交给最顶层的类加载器来加载，如果不使用双亲委派，自己定义一个object，那么最基本的一些功能将无法保证。如果我们将自己写了一个Object类，并且放到了rt.jar中，那么虽然会正常编译，但是会爆错。
### 如何自定义类加载器 

- 继承ClassLoader
- 重写findClass方法

### 打破双亲委派

- 重写loadClass

## 内存模型

- 在new对象的时候，是先给对象申请空间（对象所需要的空间，在类加载的时候就已经确定），成员变量先赋默认值，再赋初始值
- 在类加载的时候，静态变量先赋默认值，再赋初始值

## 内存屏障与JVM指令

### 乱序问题

-  CPU为了提高效率，会在一条代码执行的过程中，去同时执行另一条指令，前提是两条指令没有依赖关系
-  硬件通过内存屏障可以保证不乱序

### JVM如何保证不乱序？

- LoadLoad屏障：
  	对于这样的语句Load1; LoadLoad; Load2， 

	在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。

- StoreStore屏障：

	对于这样的语句Store1; StoreStore; Store2，
	
	在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。

- LoadStore屏障：

	对于这样的语句Load1; LoadStore; Store2，
	
	在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。

- StoreLoad屏障：

- ````
  对于这样的语句Store1; StoreLoad; Load2，
  在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。	
  ````

### volatile的实现细节

1. 字节码层面
   ACC_VOLATILE

2. JVM层面
   volatile内存区的读写 都加屏障

   > StoreStoreBarrier
   >
   > volatile 写操作
   >
   > StoreLoadBarrier

   > LoadLoadBarrier
   >
   > volatile 读操作
   >
   > LoadStoreBarrier

3. OS和硬件层面
   https://blog.csdn.net/qq_26222859/article/details/52235930
   hsdis - HotSpot Dis Assembler
   windows lock 指令实现 | MESI实现

### synchronized实现细节

1. 字节码层面
   ACC_SYNCHRONIZED
   monitorenter monitorexit
2. JVM层面
   C C++ 调用了操作系统提供的同步机制
3. OS和硬件层面
   X86 : lock cmpxchg / xxx
   [https](https://blog.csdn.net/21aspnet/article/details/88571740)[://blog.csdn.net/21aspnet/article/details/](https://blog.csdn.net/21aspnet/article/details/88571740)[88571740](https://blog.csdn.net/21aspnet/article/details/88571740)



## 对象内存布局

- 对象的创建过程

  1. 把创建的对象加载到内存
  2. 对class进行校验、赋默认值、解析（将类、方法、属性等符号引用解析为直接引用）
  3. 初始化：将类变量设为初始值，并执行static语句块
  4. 申请对象内存
  5. 成员变量赋默认值
  6. 调用构造方法
     1. 成员变量顺序赋初始值
     2. 执行构造方法语句，先调父类

- 内存存储布局

  - 如何观察虚拟机配置：CMD命令：`java -XX:+PrintCommandLineFlags -version`

    - ![image-20210618163756038](D:\面试\学习笔记\image-20210618163756038.png)

    - ````yaml
      -XX: G1ConcRefinementThreads=4 
      -XX: GCDrainStackTargetSize=64 
      -XX: InitialHeapSize=132560960 #初始堆大小
      -XX: MaxHeapSize=2120975360 #最大堆大小
      -XX: MinHeapSize=6815736 
      -XX: +PrintCommandLineFlags 
      -XX: ReservedCodeCacheSize=251658240 
      -XX: +SegmentedCodeCache 
      -XX: +UseCompressedClassPointers #跟对象头有关
      -XX: +UseCompressedOops #跟对象内存布局有关
      -XX: +UseG1GC 
      -XX: -UseLargePagesIndividualAllocation
      ````

  - 普通对象，含有以下数据

    - 对象头：在HostSpot称为markword ,8个字节
    - classPointer指针：指向对象.class
    - 实例数据
      - 加载成员变量
    - 对齐数据
      - 也就是填充数据，什么数据不重要

  - 数组对象

    - 对象头
    - class指针，指向是什么类型的数据的对象
    - 数组长度
    - 数据数据
    - 填充数据

  - 对象头具体包括：

### Java Agent机制

  - 一个class加载到内存时，agent代理（自己可以实现 ）可以截获class，能够获得对象大小

## JVM运行时数据区、指令集

### Java运行时的区域

- PC：program count，存放下一条指令的区域，存放的下一条指令的位置
  - 每个线程都有一个
- Head：堆
  - 每个线程共享
- JVM stacks:每一个线程对应一个栈帧
- native method stacks：本地 
- ==direct memory：直接内存==
  - 一般来说，所有的内存都由JVM管理，但1.4之后，为了提高IO的效率，JVM可以直接访问操作系统的内存。在1.4之前，我们访问一个数据，先加载到系统内存中，然后由JVM加载到JVM的内存中，但有了NIO，就直接去系统内存中获取，取消加载的过程
- method area：方法区
  - 线程共享
  - 类型信息、静态变量、常量池、静态变量、即时编译器编译后的代码缓存等数据

## 垃圾回收器

### 如何确定一个对象可以被回收

- 当一个对象没有任何引用指向它的时候，就可以回收了、

### 可达性分析算法

- 这个算法使用来判断一个对象是否能够被回收，简而言之，也就是这个对象是否还有对象引用它，很多人认为是通过计数器算法来判断一个对象是否能被回收，其实不是的。见《深入理解JVM》的P68。
- 这个算法具体是怎么做的呢？
  - 从被称为**GC Roots**的根对象作为初始节点，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为**引用链**，如果某个对象到GC Roots间没有任何引用链相连，或者用图论的话来说就是从GC Roots到某个对象不可达时，就说明这个对象已经没有任何引用指向它了。
- 可被成为GC Roots的对象有：
  - 在虚拟机栈中引用的对象
  - 方法区中类静态属性引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中JNI（也就是native修饰的方法）引用的对象
  - Java虚拟机内部的引用，比如基本数据类型对应的class对象，和一些常驻的异常对象
  - 所有被同步锁持有的对象等





## CMS

- CMS回收的阶段
  1. 初始标记
     - 通过GCRoots找到根对象，此时是Stop the World（STW，工作线程停止），但根对象不是很多，所以时间不是很长
  2. 并发标记
     - 这个阶段是最耗时的，所以进行并发操作，也就是和工作线程一块执行，此时不是STW
     - 顺着根对象向下找，如果在并发标记期间有的对象已经不再是垃圾，就会进行`重新标记`（是重新标记那些在并发标记期间不再是垃圾对象的对象）
  3. 重新标记
     - 此时是STW，由于并不是所有的垃圾都会被重新引用，所以此时的STW的时间不是很长
  4. 并发的清理
     - 清理垃圾，此时是并发的，也就是工作线程也会运行，这个阶段产生的垃圾叫做浮动垃圾，这些浮动垃圾会在下次回收的时候回收掉
  
  

## G1

- G1在逻辑上依旧是分带（老年代、存活去、伊甸区，大对象区）
  
- ![image-20210722213614521](D:\面试\学习笔记\image-20210722213614521.png)
  
- 特点
  - 并发收集
  - 压缩空闲空间不会延长GC的暂停时间
  - 更容易预测的GC暂停时间
  - 适用不需要实现很高的吞吐量的场景
  - ![image-20210722214005585](D:\面试\学习笔记\image\image-20210722214005585.png)
  - CSet
    - Collection Set
    - 一组可被回收的分区的集合
  - RSet
    - RemeberedSet：每一个Rgion都有一个表格，记录着其他Region中的对象到本Region的引用

## 垃圾回收算法

### 三色标记

- 黑色：

## 调优

- 老年代满了就会发生Full GC

### 垃圾回收器参数

- `-XX:+UseSerialGC = Serial New (DefNew) + Serial Old` 使用这两个垃圾回收器，这是单线程的
- `-XX:+UseConcMarkSweepGC = ParNew + CMS + Serial Old ` 如果CMS出现Full GC，那么使用单线程的Serial Old作为替补
- `-XX:+UseParallelGC = Parallel Scavenge + Parallel Old (1.8默认) 【PS + SerialOld】`，这是以吞吐量为优先的。
- `-XX:+UseG1GC = G1`，使用G1

#### 示例

- ````java
  import java.util.List;
  import java.util.LinkedList;
  
  public class test2 {
    public static void main(String[] args) {
      System.out.println("HelloGC!");
      List list = new LinkedList();
      for(;;) {
        byte[] b = new byte[1024*1024];
        list.add(b);
      }
    }
  }
  
  
  ````

- 使用 `java -XX:+PrintCommandLineFlags   test2` 

  - 结果

  - ````shell
    -XX:InitialHeapSize=15995136 -XX:MaxHeapSize=255922176 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops
    HelloGC!
    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
            at test2.main(test2.java:9)
    ````

  - `-XX:InitialHeapSize=15995136`起始堆大小

  - `-XX:MaxHeapSize=255922176`最大堆大小

  - `-XX:+UseCompressedClassPointers`，开启压缩类指针

  - ` -XX:+UseCompressedOops` 普通指针压缩

  - 

- 使用：`java -XX:+UseConcMarkSweepGC -XX:+PrintCommandLineFlags -XX:+PrintGCDetails test2`

  - 结果

  - ````shell
    HelloGC!
    [GC (Allocation Failure) [ParNew: 3514K->273K(4928K), 0.0023322 secs] 3514K->3347K(15872K), 0.0023722 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4457K->207K(4928K), 0.0051081 secs] 7531K->7647K(15872K), 0.0051429 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4383K->51K(4928K), 0.0036827 secs] 11822K->11587K(16900K), 0.0037371 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4230K->13K(4928K), 0.0037530 secs] 15765K->15644K(21012K), 0.0037754 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4192K->3K(4928K), 0.0020504 secs] 19824K->19730K(25124K), 0.0020710 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0026120 secs] 23911K->23825K(29236K), 0.0026333 secs] [Times: user=0.00 sys=0.01, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4183K->2K(4928K), 0.0022729 secs] 28006K->27921K(33348K), 0.0022972 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4183K->2K(4928K), 0.0030025 secs] 32103K->32017K(37460K), 0.0030227 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023565 secs] 36199K->36113K(41572K), 0.0023770 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0022594 secs] 40296K->40210K(45684K), 0.0022797 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023265 secs] 44392K->44306K(49796K), 0.0023475 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0029527 secs] 48488K->48402K(53908K), 0.0029742 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0030289 secs] 52584K->52498K(58020K), 0.0030899 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0034131 secs] 56681K->56594K(62132K), 0.0034385 secs] [Times: user=0.01 sys=0.01, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0028375 secs] 60777K->60690K(66244K), 0.0028650 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0026132 secs] 64873K->64787K(70356K), 0.0026380 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0024764 secs] 68969K->68883K(74468K), 0.0025001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0021762 secs] 73065K->72979K(78580K), 0.0021957 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023150 secs] 77161K->77075K(82692K), 0.0023376 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0022965 secs] 81258K->81171K(86804K), 0.0023258 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0028374 secs] 85354K->85267K(90916K), 0.0028626 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0022653 secs] 89450K->89363K(95028K), 0.0023050 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0030194 secs] 93546K->93460K(99140K), 0.0030406 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023233 secs] 97642K->97556K(103252K), 0.0023619 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0024058 secs] 101738K->101652K(107364K), 0.0024345 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023530 secs] 105834K->105748K(111476K), 0.0023757 secs] [Times: user=0.00 sys=0.01, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0022614 secs] 109931K->109844K(115588K), 0.0022820 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0021360 secs] 114027K->113940K(119700K), 0.0021542 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0022698 secs] 118123K->118037K(123812K), 0.0022878 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0024060 secs] 122219K->122133K(127924K), 0.0024300 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0028926 secs] 126315K->126229K(132036K), 0.0029238 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0027973 secs] 130411K->130325K(136148K), 0.0028258 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0025141 secs] 134508K->134421K(140260K), 0.0025399 secs] [Times: user=0.00 sys=0.01, real=0.01 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0024404 secs] 138604K->138517K(144372K), 0.0024643 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0030440 secs] 142700K->142613K(148484K), 0.0030676 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023089 secs] 146796K->146710K(152596K), 0.0023497 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023812 secs] 150892K->150806K(156708K), 0.0024169 secs] [Times: user=0.00 sys=0.01, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0022906 secs] 154988K->154902K(160820K), 0.0023126 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0023506 secs] 159085K->158998K(164932K), 0.0023728 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0027309 secs] 163181K->163094K(169044K), 0.0027972 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->2K(4928K), 0.0024068 secs] 167277K->167190K(173156K), 0.0024285 secs] [Times: user=0.00 sys=0.01, real=0.00 secs]
    [GC (Allocation Failure) [ParNew: 4184K->4184K(4928K), 0.0000100 secs][CMS: 167188K->168209K(168228K), 0.0312115 secs] 171373K->171282K(173156K), [Metaspace: 2972K->2972K(1056768K)], 0.0313177 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
    [Full GC (Allocation Failure) [CMS: 168209K->168209K(168640K), 0.0341261 secs] 242220K->241940K(243584K), [Metaspace: 2973K->2973K(1056768K)], 0.0341962 secs] [Times: user=0.03 sys=0.01, real=0.04 secs]
    [Full GC (Allocation Failure) [CMS: 168209K->168209K(168640K), 0.0024356 secs] 242964K->242964K(243584K), [Metaspace: 2973K->2973K(1056768K)], 0.0024662 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [Full GC (Allocation Failure) [CMS: 168209K->168197K(168640K), 0.0200833 secs] 242964K->242953K(243584K), [Metaspace: 2973K->2973K(1056768K)], 0.0201069 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
            at test2.main(test2.java:9)
    Heap
     par new generation   total 74944K, used 74817K [0x00000000f0a00000, 0x00000000f5b50000, 0x00000000f5b50000)
      eden space 66624K, 100% used [0x00000000f0a00000, 0x00000000f4b10000, 0x00000000f4b10000)
      from space 8320K,  98% used [0x00000000f4b10000, 0x00000000f5310408, 0x00000000f5330000)
      to   space 8320K,   0% used [0x00000000f5330000, 0x00000000f5330000, 0x00000000f5b50000)
     concurrent mark-sweep generation total 168640K, used 168197K [0x00000000f5b50000, 0x0000000100000000, 0x0000000100000000)
     Metaspace(元数据区)       used 3010K, capacity 4486K, committed 4864K, reserved 1056768K
      class space    used 285K, capacity 386K, committed 512K, reserved 1048576K
    [GC (CMS Initial Mark) [1 CMS-initial-mark: 168197K(168640K)] 243014K(243584K), 0.0002451 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
    [CMS-concurrent-mark-start]
    [CMS-concurrent-mark: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [CMS-concurrent-preclean-start]
    [CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    [CMS-concurrent-abortable-preclean-start]
    [CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    
    ````

  - 其中第49行后：

    - ![GCHeapDump](D:\面试\学习笔记\image\GCHeapDump.png)
    - eden space 5632K, 94% used [0x00000000ff980000,0x00000000ffeb3e28,0x00000000fff00000)
                                  后面的内存地址指的是，起始地址，使用空间结束地址，整体空间结束地址

### 调优前的概念

1. 吞吐量：执行用户代码时间 /（用户代码执行时间 + 垃圾回收时间）
2. 响应时间：STW越短，响应时间越好,G1响应时间较好，但是吞吐量低

- 所谓调优，首先确定，追求啥？吞吐量优先，还是响应时间优先？还是在满足一定的响应时间的情况下，要求达到多大的吞吐量...

### 什么是调优

- 根据需求进行JVM规划和预调优
- 优化进行JVM环境
- 解决JVM运行过程中出现的各种问题
  - 不单单是解决OOM
- 设定日志参数：举例
  - -Xloggc:/opt/xxx/logs/xxx-xxx-gc-%t.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCCause

### 优化例子

1. 有一个50万PV的资料类网站（从磁盘提取文档到内存）原服务器32位，1.5G
   的堆，用户反馈网站比较缓慢，因此公司决定升级，新的服务器为64位，16G
   的堆内存，结果用户反馈卡顿十分严重，反而比以前效率更低了
   1. 为什么原网站慢?
      很多用户浏览数据，很多数据load到内存，内存不足，频繁GC，STW长，响应时间变慢
   2. 为什么会更卡顿？
      内存越大，FGC时间越长
   3. 咋办？
      PS 换成 PN + CMS 或者 G1
2. 系统CPU经常100%，如何调优？(面试高频)
   CPU100%那么一定有线程在占用系统资源，
   1. 找出哪个进程cpu高（top）
   2. 该进程中的哪个线程cpu高（top -Hp）
   3. 导出该线程的堆栈 (jstack)
   4. 查找哪个方法（栈帧）消耗时间 (jstack)
   5. 工作线程占比高 | 垃圾回收线程占比高
3. 系统内存飙高，如何查找问题？（面试高频）
   1. 导出堆内存 (jmap) 
   2. 分析 (jhat jvisualvm mat jprofiler ... )
4. 如何监控JVM
   1. jstat jvisualvm jprofiler arthas top...



 









