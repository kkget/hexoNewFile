---
title: Nginx安装与原理
date: 2020-06-02 11:25:02
tags: Nginx
---

官网http://nginx.org/
需要的素材
```
pcre-8.37.tar.gz
openssl-1.0.1t.tar.gz
zlib-1.2.8.tar.gz
nginx-1.11.1.tar.gz
```
1.1.安装pcre
解压缩pcre-xx.tar.gz包
进入解压缩目录，执行./configure
如果提示，需要提前安装gcc++
进入安装光盘目录的软件包(/media/CentOSXX/Package)
执行
```
rpm -ivh libstdc++-devel-4.4.7-17.el6.x86_64.rpm
rpm -ivh gcc-c++-4.4.7-17.el6.x86_64.rpm
```
./configure完成后，回到pcre目录下执行make，再执行make install
2. 安装openssl
```java
1、 解压缩openssl-xx.tar.gz包。
2、 进入解压缩目录，执行./config
3、 make && make install
3. 安装zlib
1、 解压缩zlib-xx.tar.gz包。
2、 进入解压缩目录，执行./configure。
3、 make && make install
4. 安装nginx
1、 解压缩nginx-xx.tar.gz包。
2、 进入解压缩目录，执行./configure。
3、 make && make install
```
nginx无法启动: libpcre.so.1/libpcre.so.0: cannot
open shared object file解决办法
解决方法：
ln -s /usr/local/lib/libpcre.so.1 /lib64
32位系统则：
ln -s /usr/local/lib/libpcre.so.1 /lib
```
在/usr/local/nginx/sbin目录下
执行 ./nginx
启动命令 在/usr/local/nginx/sbin目录下
执行 ./nginx
关闭命令 在/usr/local/nginx/sbin目录下
执行 ./nginx -s stop
重新加载命令 在/usr/local/nginx/sbin目录下
执行 ./nginx -s reload
设置nginx为自启动服务
修改linux 启动脚本/etc/rc.d/rc
加入 :
/usr/local/nginx/sbin/nginx
```
5、配置nginx.conf
```java
http {
......
upstream myserver{
ip_hash;//ip取哈希码  与反向代理服务器取模 分在那一台
server 115.28.52.63:8080 weight=1;
server 115.28.52.63:8180 weight=1;
}
.....
server{
location / {
.........
proxy_pass http://myserver;
proxy_connect_timeout 10;
}
.........
}
}
```
master-workers的机制的好处
首先，对于每个worker进程来说，独立的进程，不需要加锁，
所以省掉了锁带来的开销，同时在编程以及问题查找时，也会方
便很多。
其次，采用独立的进程，可以让互相之间不会影响，一个进程
退出后，其它进程还在工作，服务不会中断，master进程则很快启
动新的worker进程。当然，worker进程的异常退出，肯定是程序有
bug了，异常退出，会导致当前worker上的所有请求失败，不过不
会影响到所有请求，所以降低了风险
需要设置多少个worker
Nginx 同redis类似都采用了io多路复用机制，每个
worker都是一个独立的进程，但每个进程里只有一个主线
程，通过异步非阻塞的方式来处理请求， 即使是千上万个
请求也不在话下。每个worker的线程可以把一个cpu的性
能发挥到极致。
所以worker数和服务器的cpu数相等是最为适宜的。设
少了会浪费cpu，设多了会造成cpu频繁切换上下文带来的
损耗。

//静态资源请求    2个
//动态资源请求    4个

#设置worker数量。
worker_processes 4
#work绑定cpu(4 work绑定4cpu)。
worker_cpu_affinity 0001 0010 0100 1000
#work绑定cpu (4 work绑定8cpu中的4个) 。
worker_cpu_affinity 0000001 00000010 00000100
00001000
连接数worker_connection
• 这个值是表示每个worker进程所能建立连接的最大值，所以，一个nginx
能建立的最大连接数，应该是worker_connections * worker_processes。
当然，这里说的是最大连接数，对于HTTP请求本地资源来说，能够支持的
最大并发数量是worker_connections * worker_processes，如果是支持
http1.1的浏览器每次访问要占两个连接，所以普通的静态访问最大并发数
是： worker_connections * worker_processes /2，而如果是HTTP作
为反向代理来说，最大并发数量应该是worker_connections *
worker_processes/4。因为作为反向代理服务器，每个并发会建立与客
户端的连接和与后端服务的连接，会占用两个连接。

worker_connections * worker_processes /2  静态
worker_connections * worker_processes /4  动态

work最先处理请求 nobody表示权限最低  路人甲
use epoll 