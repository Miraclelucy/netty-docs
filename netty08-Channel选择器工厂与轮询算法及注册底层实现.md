# Channel选择器工厂与轮询算法及注册底层实现
```java
final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);//config().group()返回一个NioEventLoopGroup
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```
1. config().group()返回一个NioEventLoopGroup；
2. NioEventLoopGroup的父类是MultithreadEventLoopGroup；
3. 实际上会调用MultithreadEventLoopGroup中的register(Channel channel)，如下：
```java
//MultithreadEventLoopGroup中的register(Channel channel)
@Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}

@Override
public EventLoop next() {
    return (EventLoop) super.next();
}
```
4. 这里的next()方法，返回的实际对象是NioEventLoopGroup。它本质上是由DefaultEventExecutorChooserFactory.PowerOfTwoEventExecutorChooser中的next()方法产生的：
```java
/**
 * Default implementation which uses simple round-robin to choose next {@link EventExecutor}.
 */
@UnstableApi
public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {

    public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();

    private DefaultEventExecutorChooserFactory() { }

    @SuppressWarnings("unchecked")
    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

    private static boolean isPowerOfTwo(int val) {
        return (val & -val) == val;
    }

    private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];//最终调用的next()方法
        }
    }

    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
}
```
5. 实际上会调用SingleThreadEventLoop(是一个单线程的)中的register(Channel channel)，如下：
```java
//SingleThreadEventLoop中的register(Channel channel)
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```
实际上会调用本类中的
```java
//SingleThreadEventLoop中的register(final ChannelPromise promise)
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```
6. AbstractChannel.AbstractUnsafe内部类中的register()方法：
```java
//AbstractChannel.AbstractUnsafe内部类中的register()
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    }
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {//将当前正在执行注册逻辑的线程传递给EventExecutor的inEventLoop(Thread thread)，判断该线程是否和SingleThreadEventExecutor中包裹的线程是同一个线程；
        register0(promise);//最终调用的是具体Channel,比如子类AbstractNioChannel中的doRegister()
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);//最为核心的注册方法
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```
7. AbstractNioChannel中的doRegister()，这里是最底层的注册方法，如下：
```java
 @Override
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}
```
8. 为什么注册前会判断eventLoop.inEventLoop()呢？先补充一些基础知识：
- 一个EventLoopGroup当中会包含一个或多个EventLoop；
- 一个EventLoop在它的整个生命周期当中都只会与一个Thread进行绑定；
- 所有由EventLoop所处理的各种I/O事件都将在它所关联的那个Thread上进行处理；
- EventLoop与Channel是一对多的关系：
> 一个Channel在它的整个生命周期中只会注册在一个EventLoop；
> 一个EventLoop在运行过程当中，会被分配给一个或者多个Channel;（类似Java NIO中的Selector和channel的关系）
9. 重要的结论：
- Netty中Channel的实现是线程安全的；基于此我们可以存储一个Channel的引用，并且在需要向远程端点发送数据时，通过这个引用来调用Channel响应的方法；即便当时有很多线程都在使用它也不会出现多线程问题；而且，消息一定会按照顺序发送出去。
- 由以上可知，Channel上的那些ChannelHandler的处理全部是在一个线程中处理，这也就是为什么遇到逻辑比较复杂的ChannelHandler一定要单独起一个专门的业务线程池（EventExecutorGroup），否则会造成线程阻塞。业务线程池通常有两种实现方式，如下：
- [1]在ChannelHandler的回调方法中，使用定义的业务线程池，这样就可以实现异步调用。
```java
public class MyServerHandler extends SimpleChannelInboundHandler<String> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        System.out.println(ctx.channel().remoteAddress()+"  "+msg);
        //返回客户端数据
        ctx.channel().writeAndFlush("from server :" + UUID.randomUUID());
        
        //单独起线程池进行业务逻辑的处理
        ExecutorService executorService= Executors.newCachedThreadPool();
        executorService.submit(()->{
            //业务逻辑代码
        });

    }

    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("exceptionCaught");
        cause.printStackTrace();
        super.exceptionCaught(ctx,cause);
    }

}
```
- [2]借助Netty提供的向ChannelPipeline添加ChnanelHandler时调用的addLast方法来传递EventExecutor：
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
