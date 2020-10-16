---
title: JVM调优相关
date: 2020-09-24 10:35:48
tags: JVM
---
# 线上系统CPU，内存与磁盘IO暴增，你会如何调优？
①首先top命令确定到具体线程，将内存与io异常的线程记录
```java
[root@zhaokk logs]# top
top - 16:48:59 up 2 days, 23:23,  1 user,  load average: 0.00, 0.01, 0.20
Tasks: 1784 total,   2 running, 1354 sleeping,   0 stopped, 428 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1882736 total,    70640 free,  1526212 used,   285884 buff/cache
KiB Swap:        0 total,        0 free,        0 used.    29260 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                                   
17755 root      20   0  165676   4060   1600 R  3.2  0.2   0:00.79 top                                                                                                       
15120 root      10 -10  134524   6240   1140 S  1.0  0.3   1:04.54 AliYunDun                                                                                                 
 1325 root      10 -10  438960    676     80 S  0.3  0.0  12:14.54 AliSecGuard                                                                                               
 7688 root      20   0 2522072  86580   2172 S  0.3  4.6   0:33.20 java                                                                                                      
    1 root      20   0  120276  60392    640 S  0.0  3.2   4:37.34 systemd                                                                                                   
    2 root      20   0       0      0      0 S  0.0  0.0   0:00.01 kthreadd                                                                                                  
    3 root      20   0       0      0      0 S  0.0  0.0   0:31.43 ksoftirqd/0                                                                                               
    5 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 kworker/0:0H                                                                                              
    7 root      rt   0       0      0      0 S  0.0  0.0   0:00.00 migration/0                                                                                               
    8 root      20   0       0      0      0 S  0.0  0.0   0:00.00 rcu_bh                                                                                                    
    9 root      20   0       0      0      0 R  0.0  0.0   6:31.17 rcu_sched                                                                                                 
   10 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 lru-add-drain                                                                                             
   13 root      20   0       0      0      0 S  0.0  0.0   0:00.00 kdevtmpfs                                                                                                 
   14 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 netns                                                                                                     
   15 root      20   0       0      0      0 S  0.0  0.0   0:00.69 khungtaskd                                                                                                
   16 root       0 -20       0      0      0 S  0.0  0.0   0:00.03 writeback                                                                                                 
   17 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 kintegrityd                                                                                               
   18 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 bioset                                                                                                    
   19 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 bioset                                                                                                    
   20 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 bioset     
```
②查看磁盘IO 
```java   
sar -d -p 1 2

09:51:48 AM       DEV       tps     rkB/s     wkB/s   areq-sz    aqu-sz     await     svctm     %util
09:51:49 AM       vda      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:51:50 AM       vda      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:          vda      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```
其中， “-d”参数代表查看磁盘性能，“-p”参数代表将 dev 设备按照 sda，sdb……名称显示，“1”代表每隔1s采取一次数值，“2”代表总共采取2次数值。
await：平均每次设备 I/O 操作的等待时间（以毫秒为单位）。 
svctm：平均每次设备 I/O 操作的服务时间（以毫秒为单位）。
%util：一秒中有百分之几的时间用于 I/O 操作
磁盘IO标准:
正常情况下 svctm 应该是小于 await 值的，而 svctm 的大小和磁盘性能有关，CPU 、内存的负荷也会对 svctm 值造成影响，过多的请求也会间接的导致 svctm 值的增加。
await 值的大小一般取决与 svctm 的值和 I/O 队列长度以 及I/O 请求模式，如果 svctm 的值与 await 很接近，表示几乎没有 I/O 等待，磁盘性能很好，如果 await 的值远高于 svctm 的值，则表示 I/O 队列等待太长，系统上运行的应用程序将变慢，此时可以通过更换更快的硬盘来解决问题。
%util 项的值也是衡量磁盘 I/O 的一个重要指标，如果 %util 接近 100% ，表示磁盘产生的 I/O 请求太多，I/O 系统已经满负荷的在工作，该磁盘可能存在瓶颈。长期下去，势必影响系统的性能，可以通过优化程序或者通过更换更高、更快的磁盘来解决此问题
③.确定好具体线程后定位具体代码
```java
 ps -ef|grep 2588|grep -v grep
```
④定位
```java
ps -mp 2588 -o THREAD,tid,time
```
⑤换算16进制后
```java
jstack 2588 |grep a1d -A60
```
![图1](http://47.105.116.133:8080/upload/jvm/1.png)
接下来就处理是逻辑问题，还是代码，还是数据库问题了
# 你们JVM线上使用的什么垃圾回收器？CMS还是G1？
我们线上由于使用的java8，默认垃圾回收器G1，如果不知道，可以打印下GC，-XX:PrintGCDetails,
windows查看收集器
![图1](http://47.105.116.133:8080/upload/jvm/2.png)
Linux 查看
```java
java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=260062400 -XX:MaxHeapSize=4160998400 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
```
-XX:+UseParallelGC
# 垃圾收集器种类 
垃圾回收器：并行  串行   并发标记(CMS)  G1    ZGC
```java
java15 ZGC转正
ZGC 功能转正
ZGC是一个可伸缩、低延迟的垃圾回收器。
ZGC 已由JEP 333集成到JDK 11 中，其目标是通过减少 GC 停顿时间来提高性能。借助 JEP 377，JDK 15 将 ZGC 垃圾收集器从预览特性变更为正式特性而已，没错，转正了。
这个 JEP 不会更改默认的 GC，默认仍然是 G1。
```
![图](http://47.105.116.133:8080/upload/jvm/3.png)
![图](http://47.105.116.133:8080/upload/jvm/4.png)
![图](http://47.105.116.133:8080/upload/jvm/5.png)
![图](http://47.105.116.133:8080/upload/jvm/6.png)
![图](http://47.105.116.133:8080/upload/jvm/7.png)
![图](http://47.105.116.133:8080/upload/jvm/8.png)
# CMS的并发更新失败是怎么回事？如何优化？
CMS垃圾回收失败类型主要是两种：并发失效和晋升失败
并发失效
在新生代(YoungGen)发生垃圾回收时，达到晋升年龄的对象会被移动到老年代(OldGen)中。
如果老年代没有足够的空间容纳这个晋升对象，CMS为了腾出老年代空间，就会从本来的MinorGC退化成FullGC。
MinorGC只回收新生代，而FullGC不仅回收新生代，而且还会回收老年代，永久区(PermGen)或元区(MetaSpace)空间回收也可能随FullGC顺便执行。
本来只是简单的新生代回收工作扩大到老年代甚至更大。除此之外，老年代空间通常比新生代的Eden和Survivor区大得多，检查和清理无效对象的时间要多得多。
还有，FullGC回收的同时，所有进程必须StopTheWorld，并用单线程(SerialGC)开始垃圾回收。导致本来可以并发的MinorGC变得缓慢无比。
晋升失败
晋升失败同样是老年代导致的问题。
CMS开启新生代垃圾收集的时候，判断老年代似乎有足够空间容纳所有晋升对象。
然而晋升的时候才发现老年代的空间竟然都是碎片化的，根本容纳不了一个完整的晋升对象。
剩下出路只有内存整理。所有应用运行的线程停止，CMS开始对老年代进行整理和压缩。
空间压缩要通过移动里面的对象，令这些对象排列好，所以晋升失败比不需要移动对象的并发失效更加浪费时间。
完成清理的堆空间变得规整和空余，继续运行应用。
调优
并发失效调优：
令老生代垃圾回收提早，增大回收频率
增大老年代空间
增大新生代空间，提高对象滞留时间，更多新对象被回收而不是晋升。
增加更多后台回收线程
晋升失败调优：
有难度，因为CMS本身不能规整Compat内存，只能退化到SerialGC来做
尝试用G1，G1的内存模型更加先进
# JVM是任何时刻都可以STW吗？为什么？
STW即停顿类型
垃圾回收并不会阻塞我们程序的线程，他是与当前程序并发执行的。所以问题就出在这里，当GC线程标记好了一个对象的时候，此时我们程序的线程又将该对象重新加入了“关系网”中，当执行二次标记的时候，该对象也没有重写finalize()方法，因此回收的时候就会回收这个不该回收的对象。 
  虚拟机的解决方法就是在一些特定指令位置设置一些“安全点”，当程序运行到这些“安全点”的时候就会暂停所有当前运行的线程（Stop The World 所以叫STW），暂停后再找到“GC Roots”进行关系的组建，进而执行标记和清除
这些特定的指令（安全点）位置主要在：
1、循环的末尾
2、方法临返回前 / 调用方法的call指令后
3、可能抛异常的位置
那个停顿类型就是STW，至于有GC和Full GC之分，还有Full GC (System)。个人认为主要是Full GC时STW的时间相对GC来说时间很长，因为Full GC针对整个堆以及永久代的，因此整个GC的范围大大增加；还有就是他的回收算法就是我们之前说过的“标记–清除–整理”，这里也会损耗一定的时间。所以我们在优化JVM的时候，减少Full GC的次数也是经常用到的办法。
# 线上系统GC问题如何快速定位与分析？阿里巴巴的Arthas用过吗？
Arthas单独去学就好
# 单机几十万并发的系统JVM如何优化？
既然说了单机了，服务单机，就把中间件去集群化
redis+二级缓存+mq+数据库优化
# 高并发系统为何建议选择G1垃圾收集器？
上面G1特点已经讲过了
# 能说说Mysql索引底层B+树结构与算法吗？
B+树中的B代表平衡（balance），而不是二叉（binary）
B-Tree：平衡二叉树 
特点：
        1.具有数据节点       
        2.指向下层指针
        3.指向数据指针
缺页查询,产生IO
B+Tree：
特点:
       1.具有数据节点       
       2.指向下层指针
命中数据3层查找后查询数据指针
加载更快，产生更少IO
效率：BTree更高，但从IO角度，Mysql选择B+Tree

B+tree算法
插入算法
删除算法

https://www.codedump.info/post/20200615-btree-2/
# 聚集索引与覆盖索引与索引下推到底是什么？
聚集索引和组合索引
索引
表的数据量比较大时，查询操作会很耗时。建立索引是加快查询速度的有效手段。
数据库索引就类似于书签，可以快速定位到要查询的内容。数据库索引类型
有顺序文件索引，B+树索引，散列索引，位图索引。其中B+树索引应用广泛。
在B+树上的查找，删除，插入的代价为O ( l o g N ) O(log N)O(logN)。建立索引有好处，当然也有
缺点。索引会占额外存储空间。每次数据更新时，也要用额外的时间来维护索引。

# 聚集索引
一张表里面只能有一个聚集索引，一般设置主键为索引。
数据库中行数据的物理顺序和索引顺序相同。这样的索引称为聚集索引。
一张表只有一个物理顺序，也就只能有一个聚集索引。
# 组合索引(覆盖索引)
定义：包含两个或多个属性列的索引称为复合索引。
# 索引下推 Index Condition Pushdown 
官网描述
```java
The goal of ICP is to reduce the number of full-record 
reads and thereby reduce IO operations. For InnoDB clustered indexes,
the complete record is already read into the InnoDB buffer.
Using ICP in this case does not reduce IO.
ICP的目标是减少完整记录读取的次数，从而减少IO操作。对于InnoDB聚集索引，
完整的记录已经被读入InnoDB缓冲区。在这种情况下使用ICP不会减少IO。
```
# 能说说Mysql并发支撑底层Buffer Pool机制吗？
一个缓冲池，来来回回还是缓存+池化思想那一套
# 一线互联网公司的数据库架构是如何设计的?
我愿意咋设计就咋设计(*￣︶￣)
先从业务逻辑上避免复杂关联库，冗余字段等
记录表等多次查询的字段建索引
多用标识性字段不用 is null等操作
阿里微服务分布式事务Seata源码深度剖析
![图](http://47.105.116.133:8080/upload/jvm/9.png)
![图](http://47.105.116.133:8080/upload/jvm/10.png)
# 动态链接与常量池
大部分字节码质量在执行的时候，都需要常量池的访问
指向运行时常量池的方法引用
方法的绑定机制
静态链接:编译期可确定

动态链接:编译期无法确定

早期绑定:同理

晚期绑定:同理

方法调用指令区分虚方法与非虚方法
![图](http://47.105.116.133:8080/upload/jvm/11.png)