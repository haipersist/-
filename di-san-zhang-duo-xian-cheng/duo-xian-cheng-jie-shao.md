# 多线程介绍

#### 线程介绍

在第二章中介绍了多进程，那和线程有什么区别呢，两者有什么联系呢？

在网上你会经常看到这样的话术：”进程是资源分配的基本单元，但系统调度的最小单位是线程“。这句话多数情况是对的，但我却不是十分的苟同。这里拿Linux来说。linux中无论是进程还是线程都是同样类型的结构体，叫task\_struct，因此线程也被称作轻量级进程。两者的区别是一个进程task\_struct被创建时所有信息都是独立的，包括地址空间，文件描述符等等信息，而线程再被创建的时候是完全copy进程的数据，只不过每个线程\_会有自己的线程ID，线程栈，程序计数器等。

<figure><img src="../.gitbook/assets/image (2) (2).png" alt=""><figcaption></figcaption></figure>

进程中的pid就是进程id，而线程中的pid是线程id，tgid则是对应的进程id。我们都知道创建子进程直接使用fork，创建线程使用pthread\_create。但实际上两者最终调用的都是同一个函数叫做do\_fork，只不过传入的参数不同。\_

进程创建时：

return do\_fork(SIGCHLD, 0, 0, NULL, NULL);

线程创建时：

```c
int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGNAL
    | CLONE_SETTLS | CLONE_PARENT_SETTID
    | CLONE_CHILD_CLEARTID | CLONE_SYSVSEM
    | 0);

 int res = do_clone (pd, attr, clone_flags, ...);
```

两者区别就是创建线程时一通拷贝，其中：

* CLONE\_VM: 新 task 和父进程共享地址空间
* CLONE\_FS：新 task 和父进程共享文件系统信息
* CLONE\_FILES：新 task 和父进程共享文件描述符表

看到这里应该可以直到，在Linux，线程同样也是以进程的方式进行调度，因此被称作轻量级进程。 这也是我不是特别认同在最开始上所说的那句话的原因。也许在windows或者其他系统，的确是这样的，至少在Linux上并不是，Linux把这种并发编程都称作是多任务处理，multi task，重点是这个task。

其实，上面已经说了进程和线程的联系，共同点和不同点了，总结就是，线程是进程的一部分，同一个进程内的线程共享资源信息，而进程之间是不共享的。此外线程上下文切换和进程上下文切换也是有区别的。

进程的上下文切换成本是比较大的，他要完全保留用户空间的虚拟内存、栈等信息，还要保存内核空间的栈，寄存器等信息。

而线程，如果是切换的线程在不同的进程，则和进程上下文切换是一致的，但如果多个线程是同一个进程的，则成本要小的多，只需要保留线程相关的上下文信息就足够了，包括程序计数器，线程栈等信息。当然这也不会差太多。

**线程的实现**

通常，线程主要包括内核线程和用户线程两种。内核线程是属于操作系统负责管理的，用户线程对于内核来说是无感知的。

通过主线程可以创建多个其他的线程，被称作对等线程。多个线程会在单个进程内执行，线程会使用同一个虚拟地址空间，即同样的资源。因为线程之间是可以实现资源共享的。

和进程一样，线程同样具备不同的状态，操作系统中规定了线程的几种状态和进程完全一样，包括就绪、运行以及阻塞态。

而在JAVA中，指定了6种状态，包括初始化，运行，阻塞，等待，超时等待以及终止。下面是JDK源码，注释很清晰得说明了每个线程状态的意义。

```
public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```

注意上面的Runnable，当注意IO阻塞等操作时，虽然让出了CPU，但是其仍然是RUNNABLE状态，并不是BLOCKED状态，BLOCKED状态是当线程在争取锁失败时，会进入BLOCKED状态。

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

#### 多线程介绍

和多进程一样，多线程存在的目的也是提高资源利用率，但相比于多进程，多线程的资源消耗要更小，因为尽管也有上下文切换，但线程上下文切换所使用的资源要远远小于进程上下文切换。计算机在执行任务时，真正执行任务的是线程，就算是多进程，每个进程执行的任务仍然是线程，只是这时只有单线程。很多地方都会问到线程和进程的区别，一句话概括就是进程时资源分配的基本单元，线程是CPU调度的基本单元。

然而多线程同样存在一定的风险，上面已经提到过多线程可以共享进程内变量，这引出了一个问题，即线程安全性。在多个线程同时操作同一变量，如何保证其执行结果的可遇见性是具备一定挑战的。为了实现线程安全，有很多的实现机制，尤其是JAVA，实现了多种不同的线程同步机制，通信方式等。从volatile变量定义禁止重排序保证可预见性，到final的定义，再到synchronized原生锁，以及后续版本的演进升级，再到JAVA线程池和Executor，再到并发包种的多种Lock(ReentrantLock,读写锁等等），再到ForkJoin/Pool，Sephmare,CountDownLatch，还有就是Java的并容器.

包括ConcurrentHashMap,CopyOnWriteArrayList,阻塞队列等等。总之，JAVA在多线程方面所作的努力要远远高于其他语言。

学习多线程，一定是要学习JAVA多线程的，因为JAVA对多线程的使用非常得成熟，对于多线程的管理，线程池的使用，线程安全性，优先级等方面，JAVA都有比较深入的研究和成熟的成果。不过还有一点要明白，JAVA多线程也是最终创建操作系统的内核线程来实现的。

总结JAVA的使用，为了保证线程安全性，可以分为两大类，一个就是悲观互斥锁，通过内置的synchronized种的重量级锁，其他的基本上都是借助于volatile，CAS，AQS（同步队列）来实现的。

再说几个线程相关的现象。

线程饥饿：线程饥饿是指由于某个线程长时间的占用，而不释放，导致其他线程永远得不到执行。通常只要通过设置线程的超时时间即可。

线程死锁：这个比较好理解了，和第二章介绍的多进程死锁是一个意思，即因为相互抢占资源，互相等待的场景。

线程活锁：活锁和死锁是完全相反的。活锁的主要现象是不同线程相互推脱责任，谁也不执行，导致整体任务进行不下去。简单说就是相互谦让。

线程需要考虑的几大方面：

如何提高效率；

如何保证线程安全性；

如何实现线程通信；

linux中在之前的版本中并不支持多线程，后来是引入了用户级的线程，通常称为轻量级进程，在2.6版本之后才真正支持内核级线程。



对于多线程的学习，主要是通过JAVA实现。JAVA中只有多线程，其对多线程的应用特别完善，其兼顾了性能、安全性等多个方面。概括说JAVA所提供的功能包括。

1、性能

```
线程池ExecutorService接口，实现类ThreadPoolExecutor，同时提供了多种线程池的实现方式
       1.FixedThreadPoolExecutor.核心线程和最大线程都是固定的。
       2.newSingleThreadExecutor  只有一个线程执行
       3.newCachedThreadPool  线程数量没有限制（实际上也受到Integer.MAX_VALUE限制），使用了Sychronized队列
       4.newWorkStealingPool 这个实际上是Fork/Join框架提供的功能。
     
Future,CompletableFuture   实现异步接收数据，提供API可获取当前响应数据状态
Fork/Join框架     并行递归执行，如并行排序，并行查找。
```

2、安全性

```
volatile    保持了可见性、禁止重排序（通过内存屏障）
final   保证了初始化的可见性
锁： 
   sychronized   锁的升级
   Lock接口(主要借助于AQS,Condition)
   
         ReentrantLock  独占锁
         ReentantReadAndWriteLock  读写锁，读读可并行，读写、写写均互斥
         StampedLock  解决当读多写少的场景下，读写锁由于读写互斥可能导致的写线程的线程饥饿。其在悲观读写锁的基础上增加了乐观读。并对于每次读锁都使用了stamp。
并发容器：
   ConcurrentHashmap
   CopyOnWriteArrayList
   ConcurrentLinkedQueue
   ConcurrentLinkedDeque
   ConcurrentSkipListMap
   ConcurrentSkipListSet
   阻塞队列：
        ArrayBlockingQueue
        LinkedBlockingQueue
        PrioityBlockingQueue
        SynchronizedQueue
        DelayQueue
```

3、并发工具

```
 1、Semphore    信号量，和Linux的Semphore比较类似，但linux的信号量只能是0，1,JAVA的可以设置任意整数值，用于控制并发量。使用AQS，通过state值实现。
 2、CountDownLatch  使用AQS，通过设置等待，可以使得主线程等待其他线程执行完之后执行。主线程只能在state是0的时候才可以获取锁。
 3、CylicBarriew  相比于CountDownLatch，其可以相互等待，并且不仅可以等待一次，可以循环多次，通过nextGeneration实现，其未使用AQS,而是借助ReentrantLock以及Condition实现。
 4、Exchanger  线程间交换数据，我是没用过。我觉得对于业务编码有些增加了复杂性。其原理大致是通过设置槽数组，来实现不同线程间的数据交换。
```

可以看到上面很多重要的工具或者锁都用到了AQS，AQS是JAVA中比较重要的一个抽象类。

对于多线程的学习，主要是通过JAVA实现。JAVA中只有多线程，其对多线程的应用特别完善，其兼顾了性能、安全性等多个方面。概括说JAVA所提供的功能包括。

1、性能

```
线程池ExecutorService接口，实现类ThreadPoolExecutor，同时提供了多种线程池的实现方式
       1.FixedThreadPoolExecutor.核心线程和最大线程都是固定的。
       2.newSingleThreadExecutor  只有一个线程执行
       3.newCachedThreadPool  线程数量没有限制（实际上也受到Integer.MAX_VALUE限制），使用了Sychronized队列
       4.newWorkStealingPool 这个实际上是Fork/Join框架提供的功能。
     
Future,CompletableFuture   实现异步接收数据，提供API可获取当前响应数据状态
Fork/Join框架     并行递归执行，如并行排序，并行查找。
```

2、安全性

```
volatile    保持了可见性、禁止重排序（通过内存屏障）
final   保证了初始化的可见性
锁： 
   sychronized   锁的升级
   Lock接口(主要借助于AQS,Condition)
   
         ReentrantLock  独占锁
         ReentantReadAndWriteLock  读写锁，读读可并行，读写、写写均互斥
         StampedLock  解决当读多写少的场景下，读写锁由于读写互斥可能导致的写线程的线程饥饿。其在悲观读写锁的基础上增加了乐观读。并对于每次读锁都使用了stamp。
并发容器：
   ConcurrentHashmap
   CopyOnWriteArrayList
   ConcurrentLinkedQueue
   ConcurrentLinkedDeque
   ConcurrentSkipListMap
   ConcurrentSkipListSet
   阻塞队列：
        ArrayBlockingQueue
        LinkedBlockingQueue
        PrioityBlockingQueue
        SynchronizedQueue
        DelayQueue
        
```

3、并发工具

```
 1、Semphore    信号量，和Linux的Semphore比较类似，但linux的信号量只能是0，1,JAVA的可以设置任意整数值，用于控制并发量。使用AQS，通过state值实现。
 2、CountDownLatch  使用AQS，通过设置等待，可以使得主线程等待其他线程执行完之后执行。主线程只能在state是0的时候才可以获取锁。
 3、CylicBarriew  相比于CountDownLatch，其可以相互等待，并且不仅可以等待一次，可以循环多次，通过nextGeneration实现，其未使用AQS,而是借助ReentrantLock以及Condition实现。
 4、Exchanger  线程间交换数据，我是没用过。我觉得对于业务编码有些增加了复杂性。其原理大致是通过设置槽数组，来实现不同线程间的数据交换。
 
```
