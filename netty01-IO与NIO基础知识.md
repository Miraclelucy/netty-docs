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
具体的见[demo](https://github.com/Miraclelucy/deepjava-project/tree/master/src/main/java/designpattern/ch01decorator)
