# Reactor模式与Netty
![image](https://github.com/Miraclelucy/funny-java/blob/master/images/netty04-01.PNG)
![image](https://github.com/Miraclelucy/funny-java/blob/master/images/netty04-02.PNG)
1. Reactor模式(反应器模式)-多路复用；Proactor模式()-纯异步的
Reactor模式主要的应用是在网络编程中。
Reactor本质上是一个线程；netty里面的NioEventLoopGroup就是一个Reactor
2. Reactor模式中的5大角色：
- Handler(句柄或者描述符)：本质上表示一种资源，是由操作系统提供的；该资源用于表示一个个的事件，比如说文件描述符，或是针对网络编程中的Socket描述符，事件既可以来自外部，也可以来自于内部；外部事件比如说客户端连接请求，客户端发送过来的数据等；内部事件比如操作系统产生的定时期事件，它本质上就是一个文件描述符。Handler是事件产生的发源地。
- Synchronous Event Demultiplexer(同步事件分离器):它本身是一个系统调用，用于等待事件的发生（事件可能是一个，也可能是多个）。调用方在调用它的时候会被阻塞，一直阻塞到同步事件分离器上有事件产生为止。对于Linux来说，同步事件分离器值得就是常用得IO多路复用机制，比如说select,poll,epoll等。在Java NIO领域中，同步事件分离器对应得组件就是Selector;对应的阻塞方法就是select方法。
- Event Handler (事件处理器)：本身由多个回调方法构成，这些回调方法构成了与应用相关的对于某个事件的反馈机制。Netty相比于Java NIO来说，在事件处理器这个角色山进行了一个升级，netty为我们开发者提供了大量的回调方法，供我们在特定事件产生时实现相应的回调方法进行业务逻辑的处理。比如netty中的SimpleChannelInboundHandler<HttpObject>抽象类，其中提供了channelInactive，channelActive，channelRead等大量的回调方法。
- Concrete Event Handler(具体事件处理器)：是事件处理器的实现。它本身实现了事件处理器所提供的各个回调方法，从而实现了特定于业务的逻辑。它本质上就是我们写的一个个处理器实现。比如netty中我们自己写的MyHandler。
- Initiation Dispatcher（初始分发器）：实际上就是Reactor角色。它本身定义了一些规范，这些规范用于控制事件的调度方式，同时又提供了应用进行事件处理器的注册、删除等设施。Initiation Dispatcher本身是整个事件处理器的核心所在，Initiation Dispatcher会通过同步事件分离器来等待事件的发生，一旦事件发生，Initiation Dispatcher首先会分离出每一个事件（实际就是遍历selectionKeys），然后调用事件处理器，最后调用相关的回调方法来处理这些事件。 也就是说在netty中，MainReactor对应的是boss线程池，subReactor对应的是worker线程池。而MyHandler中的channelRead0是由worker线程池来处理的。因此，如果channelRead0中需要处理逻辑特别复杂的业务，需要单独起一个线程，否则会阻塞线程。具体的实现方法是：
```java
static final {@link EventExecutorGroup} group = new {@link DefaultEventExecutorGroup}(16);
...
{@link ChannelPipeline} pipeline = ch.pipeline();
pipeline.addLast("decoder", new MyProtocolDecoder());
pipeline.addLast("encoder", new MyProtocolEncoder());
// Tell the pipeline to run MyBusinessLogicHandler's event handler methods
// in a different thread than an I/O thread so that the I/O thread is not blocked by
// a time-consuming task.
// If your business logic is fully asynchronous or finished very quickly, you don't
// need to specify a group.
pipeline.addLast(group, "handler", new MyBusinessLogicHandler());
```
3. Reactor模式的流程：
- [1]当应用向Initiation Dispatcher注册具体的事件处理器时，应用会指定该事件处理器感兴趣的事件，该事件是与Handle关联的；
- [2]Initiation Dispatcher会要求每个事件处理器向其传递内部的Handler。该Handle向操作系统标识了事件处理器。
- [3]当所有的事件处理器注册完毕后，应用会调用handle_events方法来启动Initiation Dispatcher的事件循环。这时Initiation Dispatcher会将每个注册的事件的Handle合并起来，并使用同步事件分离器等待这些事件的发生。比如说，TCP协议层会使用select同步事件分离器操作来等待客户端发送的数据到达连接的socket Handle上。
- [4]当与某个事件源对应的Handle变为ready状态时（比如，TCP socket变为等待读的状态时），同步事件分离器就会通知Initiation Dispatcher。
- [5]Initiation Dispatcher会触发事件处理器的回调方法，从而响应这个处于ready状态的Handle。当事件发生时，Initiation Dispatcher会将被事件源激活的handle作为[key]来寻找到底应该调用事件处理器的哪一个方法。
- [6]Initiation Dispatcher会回调事件处理器的handle_events回调方法来执行特定于应用的功能。（开发者自己编写的功能），从而响应这个事件。它所发生的事件的事件类型可以作为参数，来做细致的划分，用于确定特定事件类型绑定的处理。
4. netty中的核心代码解读
```java
//核心代码片段一 ServerBootstrap
@Override
void init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            // We add this handler via the EventLoop as the user may have used a ChannelInitializer as handler.
            // In this case the initChannel(...) method will only be called after this method returns. Because
            // of this we need to ensure we add our handler in a delayed fashion so all the users handler are
            // placed in front of the ServerBootstrapAcceptor.
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```
```java
//核心代码片段二 ServerBootstrapAcceptor

@Override
@SuppressWarnings("unchecked")
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() { //完成了真正的事件处理转移，从boss转到worker
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}

```
