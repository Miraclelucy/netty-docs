# 一、IO体系架构系统与装饰模式
1. 输入流的分类：
- 节点流（从特定的地方读写的流类）
- 过滤流（使用一个已经存在的输入流或输出流创建）
2. 在java.io中InputStream的类层次
![image](https://github.com/Miraclelucy/funny-java/blob/master/images/netty01-01.PNG)
3. 在java.io中OnputStream的类层次
![image](https://github.com/Miraclelucy/funny-java/blob/master/images/netty01-02.PNG)
4. I/O流的链接
![image](https://github.com/Miraclelucy/funny-java/blob/master/images/netty01-03.PNG)

# 二、IO体系架构中的装饰模式
1. 装饰模式又叫做wrapper模式，可以在不增加子类的情况下，动态地扩展一个对象的功能。
2. 装饰模式的角色：
- 抽象构件角色
- 具体构建角色
- 装饰角色
- 具体装饰角色
3. 装饰模式的特点：
- 装饰对象包含了一个对真实对象的引用
- 装饰对象接受来自所有客户端的请求，然后把请求转发给真实的对象
4. 示例见[demo](https://github.com/Miraclelucy/deepjava-project/tree/master/src/main/java/designpattern/ch01decorator)

# 三、Java NIO深入详解与体系分析
1. Java.io中的核心概念是流，一个流要么是输出流，要么是输入流。不可能既是输出流又是输入流。
2. Java.nio中有3个核心的概念：Selector,Channel,Buffer。java.nio中，我们是面向块（Block）或者缓冲区（Buffer）编程的。Buffer本身就是一块内存块，底层实现上，它实际上就是个数组。数据的读和写都是通过Buffer实现的，即Buffer既可以读又可以写。与java.io中不同，流不可能既是输出流又是输入流。
> 注意：Buffer从读块变成写块时，需要调用flip方法，实现翻转才能从读块变成写块，或者从写块变成读块。
3. Java.nio中8种基本数据类型都有对应的Buffer,如IntBuffer,LongBuffer,ByteBuffer及CharBuffer等。
4. Channel是指可以向其写入数据或是读取数据的对象，它类似于java.io中的stream。不同的是stream只可能是InputStream或者OutputStream，而Channel是双向的，Channel打开后可以读取、写入、读写。在Linux系统中，底层操作系统的独写通道也是双向的。
5. 示例见[NIOTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
