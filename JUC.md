# 多线程

## sychronized和Lock的区别

### Lock

- 通过CAS操作属于自旋锁，不停的在while
- 可重入锁
- 公平和非公平可以自己设定

### sychronized

- 可重入锁
- 非公平锁

### 什么时候用Lock,什么时候用Sychronized

- 线程数少的时候，使用Lock（自旋锁），消耗CPU
- 线程数多的时候，使用sychronized
- 如果操作的消耗时间长，那么就是用Sychronized

## sychronized的四种状态（锁升级）

无锁->偏向锁->自旋锁->重量级锁

````java 
sychronized (Object o){

}
````

- 如果说当前只有第一个线程来进行访问这个锁，现在o的头部记录这个线程的ID（偏向锁），此时是没有对这这个线程进行加锁的，如果未来没有第二个线程来抢占这个锁的话，那么每次只要是该线程来进行访问，通过id鉴别，直接进入方法，此时叫做偏向锁。
- 如果获取锁失败，就升级为 **CAS 轻量级锁**，如果失败就会短暂**自旋**，有一个while循环，一直在不停的检测当前得到锁的线程是否释放。默认循环十次。此时是消耗CPU，防止线程被系统挂起。最后如果以上都失败就升级为**重量级锁**。
- 如果获取锁失败，就升级为 **CAS 轻量级锁**，如果失败就会短暂**自旋**，有一个while循环，一直在不停的检测当前得到锁的线程是否释放。默认循环十次此时是消耗CPU，防止线程被系统挂起。最后如果以上都失败就升级为**重量级锁**。

- ![img](.\image\三种锁的区别)

- ![img](.\image\20190311172007882.png)

## CAS

- Atomic类都是根据CAS操作来保证线程安全的即通过unsafe类中的  `compareAndSwapInt`方法保证线程安全的，这个是CPU原语的支持，是原子操作。
- 但是有可能触发ABA问题。可以通过加版本号的方式来解决 

## LongAdder

- 使用了`分段锁` ,分段锁也是CAS操作。

## 公平锁

- `ReentrantLock lock  =new ReentrantLock(true);` ，谁先来，谁先获得锁。线程都在队列里，先进先出。默认是不公平的。

- 公平锁：一个线程先进入到等待队列中，如果队列中没有线程等待，那么就该线程先执行，如果队列中有线程，那么就让队列中的线程先执行。与等待的时间无关

  

## AQS

- 核心是**CAS+volatile**（修饰 **int类型的state**）
- AQS实现了一个**双向链表（等待队列），链表里装的是线程**
  - reetrantlock就是通过AQS实现的，如果是非公平的，也就是默认的，那么不通过等待队列，直接去尝试拿锁，如果得到了，state会变为1，如果得不到进入等待队列

## 四种引用 

### 强、软、弱、虚

- 正常我们创建的对象都是强引用对象，只有在这个对象为空的时候，JVM才会进行回收
- 如果创建了一个软引用对象，那么，只有在系统内存不够用的时候，JVM才会进行回收
- 如果创建了一个弱引用，那么，当JVM进行回收的时候，就会进行回收，无论这个引用是否有指针指向。
  - ThreadLocal应用了弱引用
-  虚引用：
  -  `PhantomReference<M> phantomReference = new PhantomReference<>(new M(), QUEUE);`
  - 当 `phantomReference `被回收的时候（也就是`new M()`这个对象为空的时候），这个引用`phantomReference `会被放到`QUEUE`中，虚引用主要是为了NIO的堆外内存回收，因为堆外内存是在操作系统中，不被JVM管理，那么，把一些对象放在这个 弱引用中，`new PhantomReference<>(对象, QUEUE)`，当我们把一些对象指向堆外内存的时候，我们可以去检测`QUEUE`，如果队列中有弱引用存在，我们去查看这个弱引用指向的是哪个对象，然后去堆外内存中回收这个空间。

## 容器

### HashTable

- 自带锁,很多方法都是加了synchronizde

### Vector

- 很多方法都是加了synchronizde，读和写都加锁

### Collections.synchronizedMap

- 加锁的hashmap

### ConcurrentHashMap

无序

### ConcurrentSkipListMap

有序，底层跳跃表

![image-20210718160057826](.\image\image-20210718160057826.png)

### CopyOnWriteArrayList

写时复制，写的时候加锁，读不加锁

### CopyOnWriteArraySet

## 面试题

1. **线程池的设计用了什么设计模式？wait/notify体现了什么设计模式**

2. **线程池的七个参数**

   1. 核心线程数

   2. 最大线程数

   3. 线程存活时间

   4. 线程存活时间单位

   5. 任务队列

   6. 线程工厂

   7. 拒绝策略,拒绝策略的场景

      - ```java
        //CallerRunsPolicy：调用者处理任务
        //AbortPolicy：没有线程处理的任务直接抛异常
        //DiscardPolicy：扔掉任务，但不抛异常
        //DiscardOldestPolicy：扔掉排队时间最久的任务，不抛异常
        //如果想用自己的，那么需要实现RejectedExecutionHandler接口 
        ```
      ```
        
      - 自定义拒绝策略需要实现`RejectedExecutionHandler`
      ```

   8. ```
      假设核心线程数是2个，最大线程数是4个，队列是4个，当来了两个任务，会创建两个线程来处理，再来四个的时候，会将这四个任务放入任务队列中，如果再有任务进来会再创建线程去处理，比如进来两个，那么就会创建两个线程来去处理新来的两个线程，如果，线程数已经到了最大限度，并且任务队列也满了，那么此时就会进行拒绝策略
      ```

3. **七个参数该怎么配置最好**

   1.  CPU密集型线程数配置公式：CPU核数+1个线程的线程池
   2. IO密集型，即该任务需要大量的IO，即大量的阻塞。
      1. 配置公式：CPU核数 * 2。
      2. 配置公式：CPU核数 / 1 – 阻塞系数（0.8~0.9之间）
      3. 线程数量= cpu数量 * cpu利用率 * （1+等待时间/计算时间）

   

4. **锁的四种状态**

   1. **无锁->偏向锁->轻量级锁->重量级锁**
   2. 对象先是无锁的状态，当该对象第一次被线程获得锁的时候，发现是匿名偏向状态，会采用CAS指令，将mark word中的线程id由0改为该线程id。如果成功，则代表获取偏向锁，否则，偏向锁撤销，升级为轻量级锁（说明此时已有偏向锁）。当被偏向的线程再一次进入同步块时，通过一些简单的检查就会得到锁，而不再需要之前的CAS操作。如果由多个线程来竞争，偏向锁就会升级为轻量级锁，第二个线程会不停的循环，查看偏向锁是否释放，当自旋十次后，就会升级为重量级锁，重量级锁是由操作系统底层的同步机制来实现同步。
   3. 需要注意的是，偏向锁的解锁步骤中**并不会修改对象头中的thread id。**

5. **sychronized和Lock的区别**

   1. synchronized是**jvm层次的锁实现**，而lock是jdk层次的锁实现
   2. Synchronized的锁状态是无法在代码中直接判断的，但是ReentrantLock可以通过`ReentrantLock#isLocked`判断；
   3. Synchronized是非公平锁，ReentrantLock是可以是公平也可以是非公平的；
   4. Synchronized是不可以被中断的，而`ReentrantLock.lockInterruptibly`方法是可以被中断的；
   5. 在发生异常时Synchronized会自动释放锁（由javac编译时自动实现），而ReentrantLock需要开发者在finally块中显示释放锁；
   6. ReentrantLock获取锁的形式有多种：如立即返回是否成功的tryLock(),以及等待指定时长的获取，更加灵活；
   7. Synchronized在特定的情况下**对于已经在等待的线程**是后来的线程先获得锁（上文有说），而ReentrantLock对于**已经在等待的线程**一定是先来的线程先获得锁；
   8. 当调用一个锁对象的`wait`或`notify`方法时，**如当前锁的状态是偏向锁或轻量级锁则会先膨胀成重量级锁**。

6. **synchrnoized和reentrantlock的底层实现及重入的底层原理**

   1. 对于`synchronized`关键字而言，`javac`在编译时，会生成对应的`monitorenter`和`monitorexit`指令分别对应`synchronized`同步块的进入和退出，有两个`monitorexit`指令的原因是：为了保证抛异常的情况下也能释放锁，所以`javac`为同步代码块添加了一个隐式的try-finally，在finally中会调用`monitorexit`命令释放锁。而对于`synchronized`方法而言，`javac`为其生成了一个`ACC_SYNCHRONIZED`关键字，在JVM进行方法调用时，发现调用的方法被`ACC_SYNCHRONIZED`修饰，则会先尝试获得锁。
   2. Reentrantlock
      1. CAS+volatile
      2. 使用了模板设计模式
      3. 首先，如果是公平锁，那么先进入Reetrancklock中的内部类`FairSync`,FairSync继承了Sync，而Sync继承了AQS，这应用了模板模式。此时会调用FairSync的lock方法。然后调用acquire（1）,这个1是AQS中的state参数，表示获取锁。由于AQS中维护了一个双向队列，里面保存的是一个一个节点，代表等待的线程，由于是公平锁，如果链表里有线程等待，那么就需要等待，如果没有的话，就会获取锁。

7. **CAS的ABA问题怎么解决**

   1. 增加版本号

8. **什么叫做阻塞队列的有界和无界**

9. **说一些AQS**

10. **JUC包里的同步组件主要实现了AQS的哪些主要方法**

11. **AtomicInteger底层用的啥？AtomicInteger用了Voliate么？

    - `private volatile int value;`value被volatile修饰
    - addAndGet和GetAndAdd都是调用Unsafe类中的相应方法，Unsafe主要使用的是CAS

12. **UnSafe类知道么？**

    - 内部的所有方法都被native修饰，有些方法没有用native修饰，但是方法的实现都是在调用native修饰的方法

13. **是每个线程都会创建一个栈还是共用一个栈？**

    - 创建一个栈

14. **介绍volatile的功能**

    - 禁止指令重排序
    - 当被修饰的值被修改时，会立刻写入到内存中

15. **总线锁的副作用**

    - 某一CPU获取了总线锁，其他CPU就不能访问内存，效率低下。

16. **AQS怎么阻塞当前线程**

17. **介绍下ConcurrentHashMap，ConcurrentHashMap底层原理**

    - `put`、`replaceNode`、`remove`等方法内部都使用了syncronzied代码块
    - 采用了 `CAS + synchronized` 来保证并发安全性。
    - put的步骤
      - 根据 key 计算出 hashcode 。
      - 判断是否需要进行初始化。
      - 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
      - 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。
      - 如果都不满足，则利用 synchronized 锁写入数据。
      - 如果数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。
    - get的步骤
      - 根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值。
      - 如果是红黑树那就按照树的方式获取值。
      - 就不满足那就按照链表的方式遍历获取值。

18. **手写生产者和消费者**

19. **集合框架**

20. **JUC包里的限流该怎么做到**

21. **Executors创建线程池的方式**

    - ```java
      newFixedThreadPool
      ```

    - ```java
      newCachedThreadPool
      ```

    - ```java
      newSingleThreadExecutor
      ```

22. **CachedThreadPool里面用的什么阻塞队列**

    - `new SynchronousQueue<Runnable>());`

    - ```java
      public static ExecutorService newCachedThreadPool() {
          return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                        60L, TimeUnit.SECONDS,
                                        new SynchronousQueue<Runnable>());
      }
      
      public static ExecutorService newSingleThreadExecutor() {
              return new FinalizableDelegatedExecutorService
                  (new ThreadPoolExecutor(1, 1,
                                          0L, TimeUnit.MILLISECONDS,
                                          new LinkedBlockingQueue<Runnable>()));
          }
      
        public static ExecutorService newFixedThreadPool(int nThreads) {
              return new ThreadPoolExecutor(nThreads, nThreads,
                                            0L, TimeUnit.MILLISECONDS,
                                            new LinkedBlockingQueue<Runnable>());
          }
      ```
      
      

23. **那你知道LinkedTransferQueue吗，和SynchronousQueue有什么区别**

24. **你还知道什么阻塞队列，能具体说说它们的特点吗**

25. **你知道新出的LongAdder吗，和AtomicLong有什么区别**

26. **那你知道LongAccumulator吗**

27. **volatile的可见性和禁止指令重排序怎么实现的**

    - volatile可以使得long和double的赋值是原子的。
    - volatile写是在前面和后面**分别插入内存屏障**，而volatile读操作是在**后面插入两个内存屏障**。
    - 写：![image-20210723113150040](.\image\image-20210723113150040.png)
    - 读：![image-20210723113233407](.\image\image-20210723113233407.png)

28. **PriorityQueue底层是什么，初始容量是多少，扩容方式呢**

29. **CopyOnWriteArrayList知道吗，迭代器支持fail-fast吗**

    - 不支持，juc下的包都支持的是**fail—safe**
    - 

30. **Hashtable和HashMap的区别**

    - HashTable的方法都是加了synchronized，效率比较低，并且HashTable的key不能是空值，但是HashMap是可以传入空值的

    - ````java
      //HashMap对空值的处理
      static final int hash(Object key) {
          int h;
          return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
      }
      ````

    - **实现方式不同**：Hashtable 继承了 Dictionary类，而 HashMap 继承的是 AbstractMap 类。

    - **初始化容量不同**：HashMap 的初始容量为：16，Hashtable 初始容量为：11，两者的负载因子默认都是：0.75。

    - **扩容机制不同**：当现有容量大于总容量 * 负载因子时，HashMap 扩容规则为当前容量翻倍，Hashtable 扩容规则为当前容量翻倍 + 1。

    - **迭代器不同**：HashMap 中的 Iterator 迭代器是 fail-fast 的，而 Hashtable 的 Enumerator 不是 fail-fast 的。

31. **什么是快速失败机制**

    - **快速失败（fail—fast）**是java集合中的一种机制， 在用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行了修改（增加、删除、修改），则会抛出Concurrent Modification Exception。
    - **安全失败（fail—safe）**大家也可以了解下，java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。





## 两个线程进行i++

i++语句只需要执行一条指令

但当有多个线程时，并不能保证多个线程i++，操作同一个i。因为还有寄存器的因素，多个cpu对应多个寄存器。每次要先把i从内存复制到寄存器，然后++，然后再把i复制到内存中，这需要至少3步。

如此，假设两个线程的执行步骤如下：

i++在两个线程中执行100次的时候，由于，对于多线程，线程共用一个内存，如果线程A在寄存器中执行了加一操作而没有写入内存，那么将会切入到另一个线程进行加一操作，在进程来回转换的过程中很可能导致原来内存中的值被覆盖，因此，此段代码执行的结果为：

最小值为：2

最大值为：200

在次范围内所有的结果都是正确的；

具体分析如下：

1.最小值的情况

线程A执行第一次i++操作，取出内存中的i，值为0；放到cpu1寄存器中执行加1操作（不写回内存），寄存器中的值为1，内存中的值为0；


线程B执行第一次i++操作，取出内存中的i，值为0；放到cpu2寄存器中执行加1操作（不写回内存），寄存器中的值为1，内存中的值为0；



线程A继续执行第99次i++，每执行一次都将其值写回内存，此时cpu1寄存器中的值为99，内存中的值为99.



线程B由于未写回内存，继续执行第一次i++，将其值放入内存，此时cpu2寄存器中的值为1，内存中的值为1（线程B写回时覆盖了原来的99）；



线程A执行第100次i++，此时cpu1寄存器中的值为2（不写入内存），内存中的值为1；




线程B继续执行完所有的操作，此时cpu2寄存器中的值为100，内存中的值为100；



此时A线程进行最后一次操作，将cpu1寄存器中的值2放入内存，此时内存中的值为2；



即此操作的最小值为2；

2.最大值情况200；

最大值的情况即为当线程A和线程B进行i++时，每进行一次i++，都写入到内存当中，这样就不会存在覆盖情况



即这种情况下取得最大值200；



拓展：当执行 -- 操作时，思路和上面一样当i=;100，各执行50次减减操作，最终取值范围为0到98；












1、 线程A执行第一次i++，取出内存中的i，值为0，存放到寄存器后执行加1，此时CPU1的寄存器中值为1，内存中为0；（对于多线程，线程共用一个内存，如果线程A在寄存器执行操作后而没有写入内存，则会切换到另一个线程。）

2.线程B执行第一次i++，取出内存中的i，值为0；存放到寄存器中，进行加1操作，cpu2中的值为1，内存中的值为0

3、线程A继续执行完成第99次i++，并把值放回内存，此时CPU1中寄存器的值为99，内存中为99.

1
4、线程B继续执行第一次i++，将其值放回内存，此时CPU1中的寄存器值为1，内存中为1；
注：由于线程B之前未写回内存，因此停留在第一次，而当其写回时，覆盖了原来的99，内存
中的值变为1
1
2
1
2
5、线程A执行第100次i++，将内存中的值（现在是1）取回CPU1的寄存器！！！，并执行加1，此时CPU1的寄存器中的值为2，内存中为1；

6、线程B执行完所有操作，并将其放回内存，此时CPU2的寄存器值为100，内存中为100；

注：B执行完100次i++，则CPU2寄存器值为100，写入内存，100覆盖了原来的1
1
1
7、线程A执行100次操作的最后一部分，将CPU1中的寄存器值放回内存（值为2，见第5步），内存中值为2；

8、结束！

注：最大值200的情况：线程A和B每次执行完i++均写入内存，交替进行
1
1
所以该题目便可以得出最终结果，最小值为2，最大值为200。

相似的题目：

j=100，两个线程j–，均执行50次，可能值是多少？

思路：过程和上题类似。第一次j–：两个线程从内存取到100值，执行j–,此时寄存器值都为99，内存值100。后边线程A和B交替执行j–，不写入内存，直到执行到第50次j–时，线程A先将内存中的值（100）取到CPU1的寄存器，再执行减1，变为99，写入内存；线程B将内存中的值（99）取到CPU2的寄存器，再执行减1，变为98，写入内存，结束。此时内存值最大为98！！！ 
因此，该题的范围是0到98之间！！！！！！！！





