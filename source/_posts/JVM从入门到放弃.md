---
title: JVM从入门到放弃
date: 2020-10-13 11:36:03
tags: JVM
---
{% asset_img 1.png This is an example image %}
虚拟机规范
https://docs.oracle.com/javase/specs/index.html
操作系统不识别字节码指令，虚拟机将其编译为机器指令
第一次编译将代码编译成字节码
第二次将字节码编译成机器指令并缓存进方法区
类加载器分类：启动类加载器，拓展类加载器，应用类加载器
双亲委派机制：
沙箱安全：
GC发生在方法区和堆
年轻代：复制算法，幸存者0-1区
eden区满时触发第一次GC(Yong GC)，把活着的对象拷贝到SurvivorFrom区，当eden区再次触发GC(Full  GC)的时候，会扫描Eden区和from区进行垃圾回收，经过这次回收还活着的复制到To区，对象年龄+1
复制次数到达15次还活下来的存入老年代
优点：无内存碎片
缺点：浪费空间
标记清除：mark-->sweep
优点：
缺点：会产生磁盘碎片
标记压缩：
缺点：移动对象需要成本
分代收集：
GCroot:
1.垃圾回收的时候如何确定垃圾？什么是GC Roots
可达性分析算法：从GC Roots对象向下搜索，如果遍历对象到GC Roots没有任何引用，则说明此对象不可用
2.那些对象可以作为GC Roots

GC Roots对象

JVM调优
标配: -version
-X：
-XX：+/-   开启关闭
查看jvm参数细节
jps -l
jinfo 
jinfo -flag PrintGCDetails 15460是否开启
K V 设值
-XX:MetaspaceSize=
-XX:MaxTenuringThreshold=15
永久区几乎不会被回收，但不是不回收
只是回收的条件比较苛刻 

虚拟机栈  局部变量，操作数栈，动态链接
java指令是基于栈设计的
每个线程创建时都会创建一个虚拟机栈
》线程私有
》保存的一个个栈帧

方法与栈帧一对一
方法执行：
1.正常结束
2.抛异常
javp -v xx.class  此处没有分号
反编译时，方法需是public得
返回值是int  ireturn
方法虽然是void，但是可以写return，底层指令是存在的
47:栈帧的内部结构
一共五部分
局部变量表+操作数栈的大小影响单个栈帧的大小，单个栈帧的大小影响栈的大小，以及何时出现异常
48:局部变量表

编译器确定，一但确定不会被更改

Specific info 
java代码 与字节码指令的对应关系  Slot 变量槽
占两个变量槽时，使用起始索引

静态方法中不能引用this，因为this也是一个变量不存在与当前方法的局部变量表中
变量的分类

操作数栈Operand Stack
栈访问数据只能通过入栈出栈来访问
上来放的就是1的位置，因为非静态方法，0放入this
 i++和++i区别



栈顶缓存
将栈顶元素缓存到CPU中，提高执行引擎效率
动态链接与常量池

大部分字节码质量在执行的时候，都需要常量池的访问
指向运行时常量池的方法引用
{% asset_img 3.png This is an example image %}
方法的绑定机制
静态链接:编译期可确定

动态链接:编译期无法确定

早期绑定:

晚期绑定:

方法调用指令区分虚方法与非虚方法

invokeinterface  虚方法
invokedynamic指令 java7
动态类型语言vs静态类型语言

java仍然是静态的
方法重写本质

虚方法表
非虚方法不需要，因为已经确定了哪个方法
方法返回地址

本地方法接口，本地方法库



一个进程对应一个jvm实例，一个jvm实例对应一个运行时数据区，包含多个进程所以堆方法区共享
jvm启动的时候就被创建了，大小也就确定了
物理上不连续，逻辑上连续
TLAB是私有的

几乎所有的对象实例都是这里分配内存
GC频率过高，影响用户线程，stop the world
堆得细分结构

分代收集
jvisualvm

-Xms： X 参数
     ms ：memory start
查看某个进程或服务的GC情况
jps 
jmap -heap 2082460

对象分解过程

YGC   STW
{% asset_img 4.png This is an example image %}
对象分配    老年代 ↓
{% asset_img 5.png This is an example image %}
Promotion   晋升 
{% asset_img 6.png This is an example image %}

from 区存在垃圾回收，但不触发minor Gc
养老区的Major Gc都无法保存的对象触发OOM
minor GC老年代回收Full gc还是整堆回收
```java
public static void main(String[] args) {
    ArrayList<MessageServiceImpl> list = new ArrayList<MessageServiceImpl>();
    while (true){
        list.add(new MessageServiceImpl());
        try {
           Thread.sleep(10);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
Minor GC会触发STW

内存分配自适应
TLAB:Thread Local Allotion Buffer 线程本地分配缓冲区

# 堆空间一定是共享的吗？
TLAB是线程私有的
堆空间的参数设置
堆是分配对象的唯一选择吗？
对象经过逃逸分析后并没有逃逸出方法的话，有可能优化成栈上分配
线程私有的，无同步执行可能
优化也是最终希望减少GC
如何快速判断是否发生逃逸分析，看new的对象是否有可能在方法外被调用

同步省略  锁销除
标量替换
逃逸分析本身也非常耗时
# 如何解决OOM
{% asset_img 7.png This is an example image %}
# 方法区的内部结构
{% asset_img 8.png This is an example image %}
{% asset_img 9.png This is an example image %}
涉及类得加载器也存在与方法区
记录方法信息的各个信息
异常信息
运行时常量池:包含字面量，类型，属性，方法的符号引用
字节以符号出现
#15   都是使用常量池里的
"count =" +count  底层会new StringBuilder
运行时常量池:动态性，字节码加载时的动态表现形式
方法区的演进
![jvm分区](http://47.105.116.133:8080/upload/jvm/12.png)
![jvm分区](http://47.105.116.133:8080/upload/jvm/13.png)
```java
http://openjdk.java.net/jeps/122
```
![jvm分区](http://47.105.116.133:8080/upload/jvm/14.png)
```java
Motivation
This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation.
因为别人没有   我也没有
```
# 为什么要替换元空间？
1.永久代设置空间大小很难确定，动态还是不动态都很难
2.对永久代调优很困难
# 为什么字符常量池和静态变量发生了变化？
因为永久代回收效率比较低，当创建大量字符串的时候，回收效率低，导致内存不足
![jvm分区](http://47.105.116.133:8080/upload/jvm/15.png)
方法区存在垃圾回收，只不过是回收的条件比较苛刻
# 对象实例化的几种方式
![jvm分区](http://47.105.116.133:8080/upload/jvm/16.png)