# Channel与ChannelPipeline关联关系及模式运用
1. Channel可以看成是一个组件，可以执行I/O操作，比如读写、连接、绑定。
2. ChannelPipeline是一种高级的拦截过滤器，它本质上是一组ChannelHandlerContext，而ChannelHandlerContext中维护了ChannelHandler对象。
它可以拦截和处理入站和出站的操作。
但是它和以往的拦截器有不同之处：
- 一般的拦截器会处理请求，也能处理响应；
- 而ChannelPipeline的出站拦截ChannelOutboundInvoker和入站拦截ChannelInboundInvoker是分开的，使得拦截更加细粒度。
3. ChannelPipeline的处理顺序：
```java
 {@link ChannelPipeline} p = ...;
 p.addLast("1", new InboundHandlerA());
 p.addLast("2", new InboundHandlerB());
 p.addLast("3", new OutboundHandlerA());
 p.addLast("4", new OutboundHandlerB());
 p.addLast("5", new InboundOutboundHandlerX());
```
上面的例子中。入站事件会依次经过1，2，5的处理；出站事件会依次经过5，4，3的处理
4. ChannelHandler之间是如何转发的？
依靠ChannelHandlerContext对象。
入站事件处理器的传播方法：
```java
<li>Inbound event propagation methods:
    <ul>
    <li>{@link ChannelHandlerContext#fireChannelRegistered()}</li>
    <li>{@link ChannelHandlerContext#fireChannelActive()}</li>
    <li>{@link ChannelHandlerContext#fireChannelRead(Object)}</li>
    <li>{@link ChannelHandlerContext#fireChannelReadComplete()}</li>
    <li>{@link ChannelHandlerContext#fireExceptionCaught(Throwable)}</li>
    <li>{@link ChannelHandlerContext#fireUserEventTriggered(Object)}</li>
    <li>{@link ChannelHandlerContext#fireChannelWritabilityChanged()}</li>
    <li>{@link ChannelHandlerContext#fireChannelInactive()}</li>
    <li>{@link ChannelHandlerContext#fireChannelUnregistered()}</li>
    </ul>
</li>
```
出站事件处理器的传播方法：
```java
<li>Outbound event propagation methods:
    <ul>
    <li>{@link ChannelHandlerContext#bind(SocketAddress, ChannelPromise)}</li>
    <li>{@link ChannelHandlerContext#connect(SocketAddress, SocketAddress, ChannelPromise)}</li>
    <li>{@link ChannelHandlerContext#write(Object, ChannelPromise)}</li>
    <li>{@link ChannelHandlerContext#flush()}</li>
    <li>{@link ChannelHandlerContext#read()}</li>
    <li>{@link ChannelHandlerContext#disconnect(ChannelPromise)}</li>
    <li>{@link ChannelHandlerContext#close(ChannelPromise)}</li>
    <li>{@link ChannelHandlerContext#deregister(ChannelPromise)}</li>
    </ul>
</li>
```
实际中是这样使用的：
```java
public class MyInboundHandler extends {@link ChannelInboundHandlerAdapter} {
    {@code @Override}
    public void channelActive({@link ChannelHandlerContext} ctx) {
        System.out.println("Connected!");
        ctx.fireChannelActive();
    }
}
public class MyOutboundHandler extends {@link ChannelOutboundHandlerAdapter} {
    {@code @Override}
    public void close({@link ChannelHandlerContext} ctx, {@link ChannelPromise} promise) {
        System.out.println("Closing ..");
        ctx.close(promise);
    }
}
```

5. 如何构建Pipeline？如果ChannelHandler中有非常耗时的逻辑处理，该如何单独使用一个线程处理？
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
