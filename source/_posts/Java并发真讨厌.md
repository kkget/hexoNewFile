---
title: Java并发真讨厌
date: 2020-06-03 13:44:19
tags: 并发编程
---
volatile 解决的是多核CPU带来的缓存与CPU之间数据的可见性
![Alt text](http://47.105.116.133:8080/upload/concurrent/1.png)
如何创建一个线程
new Thread
如何创建一个进程
runtime 
runtime.exec("");
java无法销毁一个线程，只能改变线程状态
thread.Alive() 返回false时已经被销毁
thread_main_inner
![Alt text](http://47.105.116.133:8080/upload/concurrent/2.png)
![Alt text](http://47.105.116.133:8080/upload/concurrent/3.png)
2.如何通过JAVA API启动线程
thread.start()
3.如何指定代码执行顺序
![Alt text](http://47.105.116.133:8080/upload/concurrent/4.png)
join是不管用的  线程必须执行完成。必须start.join
![Alt text](http://47.105.116.133:8080/upload/concurrent/5.png)

countdownLatch可以  信号量可以
![Alt text](http://47.105.116.133:8080/upload/concurrent/6.png)
java1.5怎么实现？
threadLoop
自旋


sleep


Wait

![Alt text](http://47.105.116.133:8080/upload/concurrent/7.png)

到底谁通知了？调用了thread.notify()
join方法的实现
线程中止方法
![Alt text](http://47.105.116.133:8080/upload/concurrent/8.png)

thread停止

1.加开关
![Alt text](http://47.105.116.133:8080/upload/concurrent/9.png)

![Alt text](http://47.105.116.133:8080/upload/concurrent/10.png)
2.isInterrupted  *.cpp是C的实现
中断仅仅是设置状态，而非中止线程

为什么放弃stop？  
防止死锁ddd
3.0说明Thread interrupt（）  isinterrupted（）interrupted 的区别和含义

Thread.interrupt()   设置状态
isInterrupted()    判断 返回Boolean
interrupted 即判断又清除

JavaThread  是一个包装  他由GC做垃圾回收
JVM thread 可能是一个OS thread JVM 管理
![Alt text](http://47.105.116.133:8080/upload/concurrent/11.png)

3.当遇到异常时，线程池如何捕捉
线程池及时关闭
![Alt text](http://47.105.116.133:8080/upload/concurrent/12.png)




如何获取线程的资源消费情况
代码层面
JMX
ThreadMXBean   threadInfo()












javap -v -p 

字节码的区别

RPC 
Request\Response=Command
一问一答  命令模式
Http  Streaming 
同步好于异步

无论是dubbo还是grpc 都是私有协议

UncaughtExceptionHandler
3.请解释偏向锁对 synchronized 与 ReentrantLock 的价值？
偏向锁？对synchronize有用

ReentrantLock已经实现了偏向锁

UseBiaseLocking  java5 6版本

OpenJDKviki
把线程id 加入头里面  java6-7之间默认打开的



设计如此
wait() 获取锁的对象 释放锁 当前线程被阻塞
 #LockSupport  park()死锁
notify()已经获取锁，唤起被阻塞的线程
 #LockSupport unpark()  
 #LockSupport park()
Java 代码模拟实现 wait() 和 notif y() 以及 notif yAll() 的语
义？


1.当主线程退出时，守候子线程会执行完毕吗？
不一定执行
ti.setDaemon(true);
守候线程执行依赖于执行时间
请说明 ShutdownHook 线程的使用场景，以及如何触发执行？
比如dubbo，在static方法块里面注册了自己的关闭钩子，完全不可控。在进程退出时直接就把长连接给断开了，导致当前的执行线程无法正常完成，源码如下：
```java
static {
    Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
        public void run() {
            if (logger.isInfoEnabled()) {
                logger.info("Run shutdown hook now.");
            }
            ProtocolConfig.destroyAll();
        }
    }, "DubboShutdownHook"));
}

从Runtime.java和ApplicationShutdownHooks.java的源码中，我们看到并没有一个可以遍历操作shutdownHook的方法。 
Runtime.java仅有的一个removeShutdownHook的方法，对于未写线程名的匿名类来说，无法获取对象的引用，也无法分辨出彼此。 
Runtime runtime=new Runtime();
runtime.addShutDownHook(new Thread(()->{
System.out.println("")
),"");
```


如何确保在主线程退出前，所有线程执行完毕？
countdownlatch
thread.getThreadGroup();


如何将普通 Li s t、Set 以及 Map 转化为线程安全对象？
把普通的集合转为线程安全的 
Collections.synchronizedList()方法，都被同步





Vector所有方法都是同步的  返回fastfail

CopyOnWriteArrayList   弱一致性的实现

Arrays#asList(Object. . .) 方法是线程安全的吗？如果不是的话，如何实现替代方案？
并非线程安全，三种解决
请至少举出三种线程安全的 Set 实现？
synchronizedSet
CopyOnWriteArraySet
CopyOnWriteArrayList
CopyOnWriteArraySet
ConcurentSkipListSet
ConcurrentHashMap 与 ConcurrentSkipListMap
ConcurrentHashMap 加锁
ConcurrentSkipListMap  不需要加锁，浪费空间，
从性能来说，synchronizedSet，Collections.synchronizedList()方法原理都是加了
synchronized关键字，性能低下，推荐ConcurrentHashMap系列



java并发框架

为什么会出现？
知识点穿插
reentrantLock与reentrantReadWriteLock的区别
不光要去看源码也要去看如何命名的?设计是一部分，
重进入的体现:acquire调了N次
2020年5月8日21:32:53   优先级 
相似：都是可进入的，都是互斥的，都是锁
2020年3月3日11:04:25更新
[
都是互斥
lock，unlock
其中 arg视为信号量
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

↓
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

 实现判定是否是公平锁
 CAS也是一个锁  即内存屏障
 memory 
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
}
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

//增加一个等待者
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
//入队
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
thread[main]->thread[0]
同一个线程不需要锁

T1 T2 T3
T1lock
waited Queue ->head ->T2 next ->T3 


]
synchronized实际也是可重入的只不过是jvm层次的 
ReentrantLock是代码层次的
写锁：互斥，
读锁：并行，数据的可见性
AQS
1.trAcquire 判断锁的状态，cas，是否是公平锁
2.acquireQueue 入队  Node线程的状态语义，共享/独占模式
3.addWaiter 
acquire（int  arg：信号量）
公平锁和非公平锁
偏向锁的实现方案
UseBiaseLock在java5的6版本

getstate默认是0
CAS   也可以看做一个锁  内存地址锁掉	  内存屏障
共享模式    独占模式
addwaiter  入队  aqs   
为什么是重进入？
acquire调了N次



renentrantlock   workThrogh
Lock与LockInterruptibly的区别
获取不到锁的时候进行入队，如果当前线程已被其他线程调用interrupt()方法的时候，这时会返回true，selfInterrupt将状态设置回去
AQS设置完会清空状态

LockInterruptibly判断是否被中断  中断则抛异常

显式的恢复中断标识
抛出interruptedException状态会被清掉

关注当前线程是否互斥
Condition:条件变量

thread结束后  自动notify了
并不会中断或者阻塞线程，但会被JVM利用，然后抛出异常

JAVA1.4实现countdownlatch
--LegacyCountDownLatch
--模仿countdownLatch
--当count>0 等待
--当count<1  return  count--;
--当count==0 notifyall









会发生死锁







cyclicBarrier  
reset 
breakBarrier
nextGeneration

尽可能不要执行完成再reset
reset不要轻易去用
releaseshared  计数

lock.condition() 	
condition.signalAll


线程池  THREADPOOL
threadpoolExecutors
ScheduledExecutorService

ctrl +alt +b

ForkJoinPool

即时线程池很大  但是存活时间不长
newCatchedThreadPool()


拒绝策略



如何得到正在获取的线程
AbstractExecutorService   netty 实现
threadPool   beforeExecutor
子线程的创造来自于factory
Future

Future get

如何优雅的停止Futrue
高并发-------------
1.线程安全
2.减少线程同步竞争
3.合理利用状态位
4.线程池
5.超时意识


Volatile  内存屏障   一段内存的   一个变量的原子性
可见性  
原子性？？  原生类型都是原子性

long    double    是线程安全的

屏障==锁？

AtomicInteger  为什么用int 
boolean的真是实现就是int      jvm虚拟机
volatile.set  就是线程安全的


CAS	操作是锁非常重的操作
synchronized 是瘦锁
http://wiki.openjdk.java.net/display/HotSpot/synchronization

偏离锁避免CAS操作
cpmchg   汇编指令
资料参考自：小马哥blibli公开课直播，议题为《面试虐我千百遍，Java 并发真讨厌》
