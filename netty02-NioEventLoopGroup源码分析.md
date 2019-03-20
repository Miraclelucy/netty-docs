# NioEventLoopGroup源码分析
1. 一般可以将给boss的构造函数设置参数值1；如果不设置，则boss这个线程池的线程个数等于系统cpu的核数。两个线程池的初始化只是设置了一些初始值并没有开启任何线程。
```java
EventLoopGroup boss =new NioEventLoopGroup();
EventLoopGroup worker =new NioEventLoopGroup();
```
2. EventLoopGroup 底层就是一个死循环，不停地去侦测和等待事件的发生；
该接口中3个最重要的方法：
- EvenTLoop next();
- ChannelFuture register(Channel channel);
- ChannelFuture register(ChannelPromise channel);
3. MultithreadEventLoopGroup的父类是MultithreadEventExecutorGroup
MultithreadEventExecutorGroup的构造函数中把创建线程和线程需要执行的任务解耦了，它们各自定义并各自维护；
executor=new ThreadPerTaskExecutor(newDefaultThreadFactory());
executor.execute();
4. Bootstrap与ServerBootstrap
```java
ServerBootstrap serverBootstrap=new ServerBootstrap();
serverBootstrap.group(boss,worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new TestServerInitializer());
```
