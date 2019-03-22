# ChannelOption、AttributeKey的作用分析
1. ChannelOption作用是以一种类型安全的方式来配置ChannelConfigue。
- ChannelOption<T>主要是配置一些底层的设置，比如TCP协议相关的设置，其中T标识某一个设置项的值的类型。
- ChannelOption<T>继承了抽象类AbstractConstant：class ChannelOption<T> extends AbstractConstant<ChannelOption<T>>
- AbstractConstant实现了Constant<T>接口：class AbstractConstant<T extends AbstractConstant<T>> implements Constant<T>；其中Constant<T>是由ConstantPool创建和管理的
2. ChannelOption与Constant、ConstantPool之间的关系：
- ChannelOption申明了一个ConstantPool，通过方法valueOf(string key)可以根据给定的常量key获取常量value值
```java
//ChannelOption中的核心代码片段：
/**
 * 申明了一个ConstantPool.
 */
 private static final ConstantPool<ChannelOption<Object>> pool = new ConstantPool<ChannelOption<Object>>() {
        @Override
        protected ChannelOption<Object> newConstant(int id, String name) {
            return new ChannelOption<Object>(id, name);
        }
    };

/**
 * Returns the {@link ChannelOption} of the specified name.
 */
@SuppressWarnings("unchecked")
public static <T> ChannelOption<T> valueOf(String name) {
    return (ChannelOption<T>) pool.valueOf(name);
}
```
- ConstantPool中获取或者新建常量的方法getOrCreate(String name)，核心代码如下：
```java
//ConstantPool中的核心代码片段：
//以双重为空判定的方式，实现了线程安全。
private T getOrCreate(String name) {
    T constant = constants.get(name);
    if (constant == null) {
        final T tempConstant = newConstant(nextId(), name);
        constant = constants.putIfAbsent(name, tempConstant);
        if (constant == null) {
            return tempConstant;
        }
    }

        return constant;
    }
```
- ChannelOption与ChannelConfig是channel所有配置属性的集合：
> ChannelConfig是Channel所有配置属性的集合；
> ChannelConfig中的boolean setOptions(Map<ChannelOption<?>, ?> options) 方法用于从一个key/value键值对集合中设置Channel的属性；

3. AttributeKey作用是
- class AttributeKey<T> extends AbstractConstant<AttributeKey<T>>
- interface Attribute<T> 
- interface AttributeMap 
> Channel继承了AttributeMap，其中包含了一个ServerSocketChannelConfig config和一组attributes；
> ChannelHandlerContext也继承了AttributeMap；
> 那么问题来了，两个AttributeMap的作用域是怎么样的？
实际上netty4.0这2者是分开的；netty4.1之后，ChannelHandlerContext中的AttributeMap直接调用了Channel中的AttributeMap。
参见：[netty4.1中的新特性](https://netty.io/wiki/new-and-noteworthy-in-4.1.html)

```java
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

