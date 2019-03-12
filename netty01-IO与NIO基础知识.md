# 一、IO体系架构
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

# 四、Java NIO中Buffer深入详解
1. Buffer的三个属性：capacity,limit,position.
- capacity:指的是元素的个数，是不会变化的。
- limit：不能再读或者再写的第一个元素的索引，永远不会为负数，永远不会超过capacity。
- positon:下一个将要被读或者写的元素的索引，永远不会为负数，永远不会超过limit。
2. Buffer中的flip()方法：改变了position和limit的位置。position指向0；limit指向position之前指向的位置。示例见[BufferFlipTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
3. 通过NIO读取文件的步骤：
- 从FileInputStream获取到FileChannel对象
- 创建Buffer
- 将数据从Channel中读到Buffer中
4. 通过NIO写入文件的步骤：
- 从FileOutputStream获取到FileChannel对象
- 创建Buffer
- 将Buffer写入到FileChannel中；
示例见[BufferClearTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
5. Buffer中的其他方法：
- putChar(),putShort(),putInt(),putLong(),putDouble()
- getChar()),getShort()),getInt()),getLong()),getDouble()
示例见[BufferMethodTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
6. Buffer中的slice()方法：
- sclie后的新buffer一般是子集，他们共享相同的底层数组。
- 新buffer上的修改会反应到旧buffer上，反之亦然。
- 两个buffer的capacity,limit,position值是相互独立的，不相关。
示例见[BufferSliceTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
7. 只读buffer:示例见[BufferReadOnlyTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
- 创建方式：ByteBuffer onlyReadBuffer=byteBuffer.asReadOnlyBuffer();
- 原byteBuffer上的修改会反应到onlyReadBuffer上
8. 直接buffer:
- 创建方式：ByteBuffer onlyReadBuffer=ByteBuffer.allocateDirect(1024)；实际返回的是一个DirectByteBuffer对象
- DirectByteBuffer类的内部有一个address变量，指向了堆外内存的地址。这样使得DirectByteBuffer可以直接使用堆外内存（不在jvm虚拟机内的内存），
实现了零拷贝。示例见[BufferDirectTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
9. 内存映射文件MappedByteBuffer。示例见[BuffeMappedTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
10. buffer数组 示例见[BuffeScatteringTest.java](https://github.com/Miraclelucy/netty-project/tree/master/src/main/java/com/tenglu/ch00/niodemo)
