---
layout: 10道大厂必考java
title: 面试题---小马哥次灵均阁
date: 2020-07-21 14:40:31
tags: 面经
---
# java允许多继承吗？
![](http://47.105.116.133:8080/upload/Q2.png)
必须针对Autocloseable
FileInputStream   最终继承Autocloseable
找到字节码文件  javap -p  -v  .class
invokrvitrual   java 7编译器做的字节码提升
![](http://47.105.116.133:8080/upload/Q3.png)
intern   reference   unique
B
![](http://47.105.116.133:8080/upload/Q3-1.png)

//变量，对象，引用
//变量：局部变量，成员变量(实例，static)，lable标签（变量名称，变量类型，引用指向）
//对象：java  Heap new 实例
//引用：强软弱虚
![](http://47.105.116.133:8080/upload/Q4.png)
D
Finally 除了JVM退出  都会执行
如果Boolean flag=null;
![](http://47.105.116.133:8080/upload/Q5.png)
muti -catch 无法使用子类关联
RuntimeException  非checked 异常  不需要throws
![](http://47.105.116.133:8080/upload/Q7.png)
CopyOnwriteArrayList
![](http://47.105.116.133:8080/upload/Q8.png)
![](http://47.105.116.133:8080/upload/Q8-1.png)
![](http://47.105.116.133:8080/upload/Q8-2.png)
![](http://47.105.116.133:8080/upload/Q9.png)
否
![](http://47.105.116.133:8080/upload/Q8-4.png)
![](http://47.105.116.133:8080/upload/Q8-5.png)

![](http://47.105.116.133:8080/upload/Q10.png)

D
# HashMap并发读是线程安全的
![](http://47.105.116.133:8080/upload/Q10-1.png)
![](http://47.105.116.133:8080/upload/Q10-2.png)
//合理关闭线程池
//Daemon线程

