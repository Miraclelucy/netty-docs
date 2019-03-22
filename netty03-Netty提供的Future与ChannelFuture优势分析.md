# Netty提供的Future与ChannelFuture优势分析
```java
ChannelFuture channelFuture=serverBootstrap.bind(10101).sync();
```
1. bind(10101)做的事情：创建一个channel，并绑定到端口10101上,返回的是一个ChannelFuture对象。
sync()做的事情：Waits for this future until it is done。
2. netty中的Future继承了jdk1.8中的Future：Future<V> extends java.util.concurrent.Future<V> 
netty中的Future在jdk原本的基础上增加了几个重要的listener方法：
- Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
- Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
- Future<V> sync() throws InterruptedException;
- boolean isSuccess();
netty中的ChannelFuture：
- Future：The result of an asynchronous operation.
- ChannelFuture：The result of an asynchronous {@link Channel} I/O operation.
- 在ChannelFuture中推荐使用addListener和removeListener的方法，而不推荐使用await()方法。
因为addListener(GenericFutureListener)是非阻塞的，该方法将ChannelFutureListener对象添加至ChannelFuture上，当与该Future关联的异步I/O操作完成时，I/O线程将通知这些hannelFutureListener。
3. ChannelFactory：本质是一个ReflectiveChannelFactory,通过反射的方式创建新的NioServerSocketChannel
4. ChannelPromise：本质是一个ChannelFuture的子类，提供了一些写方法setSuccess，setFailure用来标记异步操作执行成功或者失败，但是这些写操作只能设置一次，第二次再设置将报异常。
5. ChannelFutureListener的operationComplete方法是由I/O线程执行的，因此要注意的是不要在这里执行耗时的操作，否则需要通过另外的线程或者线程池来执行。
6. 下面是服务启动时的核心源码
```java
//抽象类AbstractBootstrap中核心代码部分展示
//核心片段一
	private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }

//核心片段二
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();//这里通过反射的方式创建新的NioServerSocketChannel
            init(channel); //在子类ServerBootStrap和BootStrap会完成初始化，主要是将一些原始的ChannelHandle加入到ChannelPipeline中
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);
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
