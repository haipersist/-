# 线程池原理

## JAVA线程池

JAVA线程池的设计是非常成熟的，所有的代码都在concurrent包中。

![](https://p3-sign.toutiaoimg.com/tos-cn-i-qvj2lq49k0/b1487789654d49fcaebacd768fdf91a5\~noop.image?\_iz=58558\&from=article.pc\_detail\&x-expires=1664882032\&x-signature=ahME5EOtjrImov8gs46E8AYA7X0%3D)

Executor继承图

最顶部Executor接口，ExecutorService是其继承接口，提供了更丰富多样的功能，包括返回值处理，线程池状态管理等。

```
1，execute（Runnable command）：执行Runable类型的任务,
2，submit（task）：用来提交Callable或Runnable任务，并返回代表此任务的Future对象
3，shutdown（）：在之前的任务都已经提交，且没有新任务接受。
4，shutdownNow（）：停止所有任务，并返回正在等待的任务列表。
5，isTerminated（）：测试是否所有任务都履行完毕了。
6，isShutdown（）：测试是否该ExecutorService已被关闭。
```

当调用线程池的shutdown方式时，会见线程池状态变成shutdown，并中断空闲线程，此时不会中断正在执行任务的线程，队列中的任务也会被执行。判断是否是空闲线程的方法是通过Worker的AQS实现的。在中断线程时，如果tryLock失败，就不会中断（线程在执行任务的时候会执行tryLock类似加锁，如果tryLock失败，证明这个线程以及那个在执行了）。

```java
 public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //更新线程池状态
            advanceRunState(SHUTDOWN);
            //中断空闲线程
            interruptIdleWorkers();
            //添加的钩子函数
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```

中断空闲线程的操作：

```java
 private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                 //如果该worker正在执行，tryLock会失败
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```



当调用shutdownNow时，线程池的状态会变为shutdownNow，并尝试中断所有任务，即向所有线程发出intterupt，不过注意，线程并不一定会立即停止，只是会给线程发个标志位，中断标志位变为true，如果是sleep，join，wait等，即不是runable，会直接抛出中断异常，并清除中断标志位。但如果是执行状态，还是要看线程本身会何时终止。





**线程池实现原理**

ThreadPoolExecutor为核心实现类（java也提供了几个可用的线程池类，如FixThreadoolExecutor等，但最好还是自定义，除非要使用\
ScheduleThreadPoolExecutor等带有定时功能的线程池），java中的线程池的创建都是围绕该类进行的。该类有几个重要的参数：

```
    /**
     对于超过核心线程数的线程，空闲时间超过多久就会回收
     */
    private volatile long keepAliveTime;

    /**
     是否允许核心线程在空闲时也被回收，默认时不回收。
     */
    private volatile boolean allowCoreThreadTimeOut;

    /**
     *核心线程数量。在线程池中的线程数达到该值前，会一直创建线程
     */
    private volatile int corePoolSize;

    /**
     * 最大线程数量
     */
    private volatile int maximumPoolSize;

   /**
   * 同步队列，存储即将要执行的任务
    /
    private final BlockingQueue<Runnable> workQueue;
    
    /**
    拒绝策略，当线程池的线程达到最大且队列已满或者线程池shutdown会执行该handle
    private volatile RejectedExecutionHandler handler;
```

上面的这几个参数是整个线程池实现的核心配置，缺一不可。

1、线程数的选择决定了并发性能，过低发挥不了多处理器的优势，过多会增加线程的创建和销毁过程以及上下文切换次数；在并发编程实战中提到了线程数的决定。但终究还是要以实际情况为准。是IO密集型任务还是CPU密集型任务，当然正常业务基本上都是前者。此外还要看具体任务的执行时间和等待时间的关系。

2、队列的选择也很重要，过小则导致可处理任务太多，过大可能会出现OOM。目前常用的队列有：

```
ArrayBlockingQueue：是一个数组结构的有界阻塞队列，此队列按照先进先出原则对元素进行排序

LinkedBlockingQueue：是一个单向链表实现的阻塞队列，此队列按照先进先出排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool使用了这个队列。

SynchronousQueue ：一个不存储元素的阻塞队列，每个插入操作必须等到另外一个线程调用移除操作，否则插入操作一直处理阻塞状态，吞吐量通常要高于LinkedBlockingQueue，Executors.newCachedThreadPool使用这个队列。这个有点像golang中的channel。

PriorityBlockingQueue：优先级队列，时基于数组的堆结构，进入队列的元素按照优先级会进行排序。
```

3、拒绝策略

```
    /**
     *拒绝所任务，并抛出异常
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

    /**
     *拒绝任务，但不抛出异常
     */
    public static class DiscardPolicy implements RejectedExecutionHandler {
       
    }

    /**
     *删除队列最老的，并执行该任务
     */
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

看一下线程池的实现原理：

![](https://p3-sign.toutiaoimg.com/tos-cn-i-qvj2lq49k0/9c53f0891aa04675ad157d5e40218f84\~noop.image?\_iz=58558\&from=article.pc\_detail\&x-expires=1664882032\&x-signature=0tL45YMBb0BuHw%2FOg76EttaNZEQ%3D)

线程池执行过程

1、如果当前线程池中的线程数小于corePoolSize,就会新建线程；

2、如果当前队列未满，就向队列添加任务；

3、如果队列已满，且线程数小于maxPoolSize,就新建线程；

4、如果队列已满，且线程数大于maxPoolSize，就执行拒绝策略。

核心代码是这一段：

```
 public void execute(Runnable command) {
      //ctl是一个AtomicInteger变量，初始值是0
        int c = ctl.get();
       //判断当前线程数是不是大于核心线程数配置，每新增一个线程,ctl的值就会+1
        if (workerCountOf(c) < corePoolSize) {
           //addWorker是具体创建线程的操作.Worker对象是封装的线程
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
       //判断当前线程池是不是运行状态，此外向队列中试着添加任务
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
           //如果当前有效线程大于0，则直接返回。队列中的任务后续会被线程执行到
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
      //队列满了，试着创建新线程，要看是不是超过了最大线程数配置。
        else if (!addWorker(command, false))
            reject(command);
    }
```

在添加线程的过程中，会再次涉及到线程数的判断。保留最重要（不考虑当前线程池状态）的代码：

```
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {   
            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
               //CAS增加线程数变量，成功则跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
              //线程增加使用互斥锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                   int rs = runStateOf(ctl.get());
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                       //虽然hashset不是线程安全的，但前面已经加了锁。
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        return workerStarted;
    }
```

线程池中的线程都会放到HashSet中存储：

```
    private final HashSet<Worker> workers = new HashSet<Worker>();
```

线程池中的线程执行任务的流程如下：

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption><p>线程任务处理流程</p></figcaption></figure>

上面注意的是核心线程（或者核心线程和队列都满了之后，创建线程成功时）在创建时会带上firstTask。

\\

**线程生命周期**

好了，上面是整个线程池的运行原理。再说下线程池的生命周期。线程池一共存在以下几个生命周期：

```
RUNNING:  -1 << COUNT_BITS，即高3位为111  接收新任务，也会处理队列中的任务
SHUTDOWN: 0 << COUNT_BITS，即高3位为000 不会接收新任务，但会执行队列中的任务
STOP:     1 << COUNT_BITS，即高3位为001 完全停止，不会执行任何任务，队列中的也会被忽略
TIDYING:   2 << COUNT_BITS，即高3位为010,  所有任务都停止了，线程数也是0，就会进入该状态，并且下一步就进入TERMINATED
TERMINATED:3 << COUNT_BITS，即高3位为011, 线程池任务结束了。
```

表示线程状态的字段是ctl，是一个原子类，AtomicInteger。可以看到，线程池的状态是采用高3位表示，这是因为这个字段不仅仅表示线程池状态，还可以表示实际的线程数量，由于是个AtomicInteger类型，所以最多的线程只能是2^29 -1，不过官方也说过，未来可以将其类型改成AtomicLong。

<figure><img src="../../.gitbook/assets/image (15).png" alt=""><figcaption><p>线程池生命周期</p></figcaption></figure>

这里要提到Worker使用了AQS，目的是对于运行种的线程使用独占锁，时，并不会终止运行中的线程，只会终止空闲线程。原理就是在run Worker时会调用worker的lock方法，获取独占锁，在释放锁之前，此时调用interruptIdleWorker时也会tryLock，但会失败（不会自旋，也不会进入阻塞队列）。

**线程池关闭**

通常我们在项目中会静态初始化线程池或者初始化Bean，在应用程序运行中，线程池会一直保持Runn状态。但它我们kill应用或者因为其他原因down掉，以及正常重启。为了实现线程的优雅关闭，可以为线程池增加钩子函数，shutDownHook。类似：

```
        Runtime.getRuntime().addShutdownHook(new Thread(JVMShutdownHookTest::doSomething));
```

Spring就借用了JVM的shutdownhook实现了自己的优雅停机。通过优雅停机可以实现：

1、当前运行的一些程序执行都可以继续执行完，比如正在消费消息，正在处理响应；

2、释放资源，如各种连接池，线程池、配置中心下线等等；

spring在启动时会注册自己的钩子函数，在关闭时会自动执行。在2.3.0版本之后，可以配置：

```
server:
  shutdown: graceful #开启优雅停机
spring-lifecycle:
      timeout-per-shutdown-phase: 50 //最长要等多久
```

应用启动时会注册钩子函数：

```
 public void registerShutdownHook() {
        if (this.shutdownHook == null) {
            this.shutdownHook = new Thread("SpringContextShutdownHook") {
                public void run() {
                    synchronized(AbstractApplicationContext.this.startupShutdownMonitor) {
                        AbstractApplicationContext.this.doClose();
                    }
                }
            };
           //注册钩子函数
            Runtime.getRuntime().addShutdownHook(this.shutdownHook);
        }
    }
```

在执行关闭时（kill pid),会优先处理钩子，也就是处理正在进行中的线程，随后再做关闭处理（销毁bean等操作）。这里就不做源码分析了。感兴趣的可以参考这篇文章：[Spring boot 2.3优雅下线，距离生产还有多远？-阿里云开发者社区](https://developer.aliyun.com/article/776108)

另外注意在Spring配置线程池，最好带上下面这两个配置：

```
 @Bean(name = "porateAsyncExecutor")
    public Executor porateAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //服务关闭时，线程池的终止策略。
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(10);
    }
```

这两个参数决定服务下线时，如何终止线程池，是调用shutdown，还是shutdownNow()，以及最长的等待时间。

JAVA中常见的线程池：

1. newFixedThreadPool
2. newSingleThreadExecutor
3. newCachedThreadPool

## 3、Dubbo线程池

随着技术的发展，目前的应用系统已经从最早的单体应用发展为分布式系统，系统按照某种方式划分初多个子系统，其中子系统会以微服务的方式部署，最早时微服务之间通信会使用thrift，但如果一个业务系统比较复杂，划分的微服务可能有几十个，上百个，或者几百个。因此，微服务治理至关重要，其中Dubbo和配置中心如Nacos，ZK等结合，提供了很好的微服务治理较好的方案。

本文主要介绍Dubbo的线程池模型，看Dubbo如何通过多线程来实现高并发的。

下面是Dubbo的一个大体工作流程：

![](https://p3-sign.toutiaoimg.com/tos-cn-i-qvj2lq49k0/dedf7c2933704230b93d00a6272f3b5b\~noop.image?\_iz=58558\&from=article.pc\_detail\&x-expires=1664882032\&x-signature=sJu4WQe3SyIlSAghA7LJAuwIraU%3D)

Dubbo工作流程

不论在消费端还是服务端，Dubbo都采用了多线程的模式完成请求响应处理。当然底层主要采用了Netty，且使用了经典的Reactor模式。这里先说消费端。

在之前（2.7.5之前）的版本中，消费端对于响应的处理是由统一的消费端线程池负责的，而这个线程池是newCachedThreadPool，是个无界队列，这就导致如果消费端有大量的并发发生，就有可能出现OOM。因此后续的版本中增加了一个ThreadlessExecutor,该类实际上不是一个完整的意义的线程池，只是利用了同步队列，当服务端返回响应后，会放到该队列中，随后由业务线程负责反序列化响应数据等处理。也就是说，不必再额外使用消费端的线程池了。

![](https://p3-sign.toutiaoimg.com/tos-cn-i-qvj2lq49k0/76820cc5e620422cbfde1902c00ea3e6\~noop.image?\_iz=58558\&from=article.pc\_detail\&x-expires=1664882032\&x-signature=OP1DRTPXAmbxvhbSHzL0cvQ3bNk%3D)

Dubbo消费端线程池模型

当消费端业务线程发起请求后，会调用ThreadlessExecutor的wait方法阻塞住，等待结果返回。

```
              //消费端发起请求
              ExecutorService executor = getCallbackExecutor(getUrl(), inv);
                CompletableFuture<AppResponse> appResponseFuture =
                        currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
                // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
                FutureContext.getContext().setCompatibleFuture(appResponseFuture);
                AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
                //设置处理线程池
                result.setExecutor(executor);

   
    //这个是调用get阻塞住
    public Result get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        if (executor != null && executor instanceof ThreadlessExecutor) {
            ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
            //这里的waitAndDrain会一直等待到结果返回
            threadlessExecutor.waitAndDrain();
        }
        return responseFuture.get(timeout, unit);
    }
```

代码：

```
 public void waitAndDrain() throws InterruptedException {
      
        Runnable runnable;
       //阻塞直到获取到数据
        try {
            runnable = queue.take();
        } catch (InterruptedException e) {
            setWaiting(false);
            throw e;
        }

        synchronized (lock) {
            setWaiting(false);
            //执行任务
            runnable.run();
        }
        runnable = queue.poll();
        while (runnable != null) {
            runnable.run();
            runnable = queue.poll();
        }
        // mark the status of ThreadlessExecutor as finished.
        setFinished(true);
    }
```

假如结果现在返回了，会调用如下函数：

```
 @Override
    public void received(Channel channel, Object message) throws RemotingException {
       //这获取到的就是ThreadPoollessExecutor
        ExecutorService executor = getPreferredExecutorService(message);
        try {
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            if(message instanceof Request && t instanceof RejectedExecutionException){
                sendFeedback(channel, (Request) message, t);
                return;
            }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }
 
```

随后，业务线程从队列中获取到任务后就开始执行了，如反序列化处理等。感兴趣的可看：[消费端线程池模型 | Apache Dubbo](https://dubbo.apache.org/zh/docs/advanced/consumer-threadpool/)

\\

再看下服务端线程池。

服务端在处理请求时，主要时采用了IO多路复用的经典模式多线程Reactor模式，即IO线程和Work线程池。在实际开发中可以进行配置派发模式。

\\

```
all 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。
direct 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
message 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
execution 只有请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
connection 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。
```

一般情况下都使用message模式，这样将请求相关事件和实际业务处理完全分开。业务由Worker线程完成。

\\

![](https://p3-sign.toutiaoimg.com/tos-cn-i-qvj2lq49k0/4c158322cf5c4585856f615bb5a2ae7d\~noop.image?\_iz=58558\&from=article.pc\_detail\&x-expires=1664882032\&x-signature=O7g9IRziofXrlKZvMSlqfmX8qVo%3D)

此外，Dubbo还可以选择线程池的种类，默认是FixedThreadPoolExecutor。注意模式的选择以及队列容量的配置，否则默认是0。

对于Dubbo的服务端，一定要注意线程池耗尽的问题，记得我们电商平台之前在抢购茅台的时候就出现了这个问题，而且线程池耗尽的时候，Dubbo只是输出了warn级别的日志，没做其他处理。对于这个问题，一是要单台机器的配置要做优化，此外要使用集群并实现负载均衡来实现流量分摊。

\\

## 4、Mysql线程池

Mysql同样是采用多线程来处理请求，不过其最大的特点是使用了线程组的概念，并设定了不同的优先级队列。如图：

![](https://p3-sign.toutiaoimg.com/tos-cn-i-qvj2lq49k0/ef919f313a3d49098863172b60175d2a\~noop.image?\_iz=58558\&from=article.pc\_detail\&x-expires=1664882032\&x-signature=%2F0Q5GVIM8rBuixtNzDGjKgKErKo%3D)

Mysql线程池

从上图可看 Mysql整个线程池划分初了多个线程组ThreadGroup，每个线程组包括一个低优先级队列和高优先级队列，还有listener线程以及worker线程。

优先级队列中的任务会根据优先级来执行，比如事务的任务要比非事务select等任务的优先级高。尤其是锁表操作如Lock table read肯定是要放到高优先级。

worker线程用来处理任务，listener线程主要负责监听任务，如果队列中没有任务，则直接变成worker线程；如果队列中有任务，则将任务放到队列中。

查看当前线程状态：

```
mysql> show global variables like '%thread_handling%';
+-----------------+-----------------+
| Variable_name   | Value                 |
+-----------------+-----------------+
| thread_handling | pool-of-threads |

  
  mysql> show global variables like '%pool%';
+-------------------------------------+----------------+
| Variable_name                       | Value          |
+-------------------------------------+----------------+
| innodb_buffer_pool_chunk_size       | 1048576        |
| innodb_buffer_pool_dump_at_shutdown | ON             |
| innodb_buffer_pool_dump_now         | OFF            |
| innodb_buffer_pool_dump_pct         | 80             |
| innodb_buffer_pool_filename         | ib_buffer_pool |
| innodb_buffer_pool_instances        | 8              |
| innodb_buffer_pool_load_abort       | OFF            |
| innodb_buffer_pool_load_at_startup  | ON             |
| innodb_buffer_pool_load_now         | OFF            |
| innodb_buffer_pool_size             | 8589934592     |
| thread_pool_high_prio_mode          | transactions   |
| thread_pool_high_prio_tickets       | 4294967295     |
| thread_pool_idle_timeout            | 60             |
| thread_pool_max_threads             | 100000         |
| thread_pool_oversubscribe           | 20             |
| thread_pool_size                    | 32             |
| thread_pool_stall_limit             | 50             |
+-------------------------------------+----------------+
```

当请求到来时，会根据线程id取模（threadId % threadPoolSize，这个size表示的是线程组的数量）分配到不同的线程组，从而将请求分治处理，再通过优先级队列的设置，满足了对事务的快速响应处理。
