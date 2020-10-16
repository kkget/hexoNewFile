---
title: 重视Netty源码解析
date: 2020-08-03 15:01:15
tags: Netty
---
第一次学习Netty可追溯到2019年1月，现在重新阅读多篇资料和源码后自己以个人能够理解的方式总结下Netty，参考多篇资料
1.Netty是啥
官网定义
Netty是 一个异步事件驱动的网络应用程序框架，用于快速开发可维护的高性能协议服务器和客户端。
学习Netty之前，通篇讲述了Channel，selector，selectionKey，以便于理解Netty的Reactor模型，并且从AIO，BIO,NIO讲解
从服务端代码入手
```java
@Component
public class WSServer {

    public static void main(String[] args) throws Exception {

        EventLoopGroup mainGroup = new NioEventLoopGroup();
        EventLoopGroup subGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap server = new ServerBootstrap();
            server.group(mainGroup, subGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new WSServerInitialzer());

            ChannelFuture future = server.bind(8088).sync();
            if (future.isSuccess()) {
                System.out.println("Netty启动成功..............");
            }
            future.channel().closeFuture().sync();
        } finally {
            mainGroup.shutdownGracefully();
            subGroup.shutdownGracefully();
        }
    }
}
```
1：这里官方定义的mainGroup和subGroup命名为boss和worker，从这里第一时间就想到了Nginx的工作原理master-workers机制，像不像？相似度90%。
首先，对于每个worker进程来说，独立的进程，不需要加锁，
所以省掉了锁带来的开销，同时在编程以及问题查找时，也会方
便很多。
其次，采用独立的进程，可以让互相之间不会影响，一个进程
退出后，其它进程还在工作，服务不会中断，master进程则很快启
动新的worker进程。当然，worker进程的异常退出，肯定是程序有
bug了，异常退出，会导致当前worker上的所有请求失败，不过不
会影响到所有请求，所以降低了风险
并且Nginx也采用了多路Io复用机制

#Reactor线程模型
Reactor 对应的叫法: 1. 反应器模式 2. 分发者模式(Dispatcher) 3. 通知者模式(notifier)

Netty模型 Reactor  
1.Reactor对象通过select监控客户端请求事件，收到后通过dispatch分发
2.建立连接请求，由acceptor通过accept处理，创建一个handler对象处理完成连接后的事件。
3.如果不是连接请求，则由Reactor对象分发调用连接对应的handler响应
4.handler只负责响应事件，不做具体业务处理，通过read方法读取数据后分发给work线程池里的线程处理
5.work线程池分配独立线程完成真正的业务，并将结果返回
优点：充分利用多核CPU能力
缺点：多个线程会数据共享和访问比较复杂，多线程场景出现性能瓶颈

主从reactor多线程模式
1.主线程只管连接
2.子线程管处理请求，一个子线程监听多个client，io读取，业务处理往下分配
3.当有新事件发生，subReactor调用对应的handler处理

2：NioEventGroup类图
![](http://47.105.116.133:8080/upload/NioEventGroup.png)
最上层是什么？不就是线程池的执行接口么，启动时，真正执行的方法是其父类的父类MultithreadEventExecutorGroup的方法调用
```java
 protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }
```
调到这时，传入的nThreads为0，DEFAULT_EVENT_LOOP_THREADS进行赋值
```java
 private static final int DEFAULT_EVENT_LOOP_THREADS;

    static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
        //CPU核心数的数量乘2
        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }

```
那么到MultithreadEventExecutorGroup方法时就变成了8  executor为null
![](http://47.105.116.133:8080/upload/debug1.png)
![](http://47.105.116.133:8080/upload/debug2.png)
```java
 protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }
		executor初始化
        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
       children 在最上层定义// private final EventExecutor[] children;
        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
            	//进入循环就是8次么
            	//第一次success为false
            	.
            	.
            	.
            	i=7时  success=true
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);
        //任务监听
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }
        //addAll()方法用于将所有指定的元素添加到指定的集合中
        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```
```java
ChannelFuture future = server.bind(8088).sync();
异步绑定端口号
mainGroup.shutdownGracefully();
优雅关闭
```
实际应用个人流程总结
![](http://47.105.116.133:8080/upload/nettyserver.png)
这里的流程基本就是企业实际应用的流程
注解@ChannelHandler.Sharable，标识此Handler可应用于多个NettyClientHandlerInitializer
下面代码来自芋道源码github
```java
@Component
public class NettyClientHandlerInitializer extends ChannelInitializer<Channel> {

    /**
     * 心跳超时时间
     */
    private static final Integer READ_TIMEOUT_SECONDS = 60;

    @Autowired
    private MessageDispatcher messageDispatcher;

    @Autowired
    private NettyClientHandler nettyClientHandler;

    @Override
    protected void initChannel(Channel ch) {
        ch.pipeline()
                // 空闲检测
                .addLast(new IdleStateHandler(READ_TIMEOUT_SECONDS, 0, 0))
                .addLast(new ReadTimeoutHandler(3 * READ_TIMEOUT_SECONDS))
                // 编码器
                .addLast(new InvocationEncoder())
                // 解码器
                .addLast(new InvocationDecoder())
                // 消息分发器
                .addLast(messageDispatcher)
                // 客户端处理器
                .addLast(nettyClientHandler)
        ;
    }
}
```


