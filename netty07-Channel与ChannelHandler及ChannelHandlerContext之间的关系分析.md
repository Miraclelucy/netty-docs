# Channel与ChannelHandler及ChannelHandlerContext之间的关系分析
1. ChannelHandlerContext使得一个ChannelHandler可以与它所在的ChannelPipeline和其他的Handlers交互。其中最重要的3个方法：
- Channel channel();// 获取关联的Channel
- ChannelHandler handler();// 获取关联的ChannelHandler
- ChannelPipeline pipeline();// 获取关联的ChannelPipeline
2. 可以获得ChannelHandlerContext并保存，然后再其他地方使用：
```java
public class MyHandler extends {@link ChannelDuplexHandler} {
    <b>private {@link ChannelHandlerContext} ctx;</b>
    public void beforeAdd({@link ChannelHandlerContext} ctx) {
        <b>this.ctx = ctx;</b>
    }
    public void login(String username, password) {
        ctx.write(new LoginMessage(username, password));
    }
    ...
}
```
3. 一个ChannelHandler可以被添加到多个ChannelPipeline中，因此它可以有多个ChannelHandlerContext
```java
public class FactorialHandler extends {@link ChannelInboundHandlerAdapter} {
  private final {@link AttributeKey}&lt;{@link Integer}&gt; counter = {@link AttributeKey}.valueOf("counter");
  // This handler will receive a sequence of increasing integers starting
  // from 1.
  {@code @Override}
  public void channelRead({@link ChannelHandlerContext} ctx, Object msg) {
    Integer a = ctx.attr(counter).get();
    if (a == null) {
      a = 1;
    }
    attr.set(a * (Integer) msg);
  }
}
```
4. 一个ChannelHandler可以被添加到多个ChannelPipeline；
一个ChannelHandler可以被添加到同一个ChannelPipeline多次；
```java
FactorialHandler fh = new FactorialHandler();
{@link ChannelPipeline} p1 = {@link Channels}.pipeline();
p1.addLast("f1", fh);
p1.addLast("f2", fh);
{@link ChannelPipeline} p2 = {@link Channels}.pipeline();
p2.addLast("f3", fh);
p2.addLast("f4", fh);
```

5. ChannelPipeline中的addLast方法究竟做了什么？
```java
//DefaultChannelPipeline中的核心代码片段
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);//产生一个DefaultChannelHandlerContext对象

        addLast0(newCtx);//将Context对象添加到pipeline中

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);//将handler对象添加到Context对象
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```
6. ChannelInitializer是在何时执行initChannel(C ch)方法的？
一旦Channel注册，该方法将被调用。并且调用完后ChannelInitializer实例将立即从ChannelPipeline被移除。
也就是ChannelInitializer用于一次性的添加多个ChannelHandler到ChannelPipeline中，添加完成后它本身立马被移除。
