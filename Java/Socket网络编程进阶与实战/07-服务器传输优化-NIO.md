# 第7章 服务器传输优化-NIO
## 7.1 阻塞IO和⾮非阻塞IO
* 1.阻塞IO线程消耗
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070101.png)

* 2.非阻塞IO线程优势
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070102.png)

* 3.非阻塞IO
    * NIO全称：Non-blocking I/O
    * JDK 1.4 引入全新的输入输出标准库NIO，也叫New I/O
    * 在标准Java代码中提供了高速的、可伸缩性的、面向块的、非阻塞的IO操作

## 7.2 NIO Family一览
> * Buffer缓冲区：用于数据处理的基础单元，客户端发送与接收数据都需通过Buffer转发进行
> * Channel通道：类似于流；但，不同于IN/OUT Stream；流具有独占性与单向性；通道则偏向于数据的流通多样性
> * Selectors选择器：处理客户端所有事件的分发器

### 7.2.1 Charset扩展部分
* Charset 字符编码：加密、解密
* 原生支持的、数据通道级别的数据处理方式，可以用于数据传输级别的数据加密、解密等操作

### 7.2.2 NIO-Buffer
* Buffer包括
    * ByteBuffer, CharBuffer, ShortBuffer, IntBuffer, LongBuffer, FloatBuffer, DoubleBuffer
* 与传统不同，写数据时先写到Buffer->Channel；读则反之
* 为NIO块状操作提供基础，数据都按”块“进行传输
* 一个Buffer代表一”块“数据

### 7.2.3 NIO-Channel
* 可从通道中获取数据也可输出数据到通道；按“块”Buffer进行
* 可并发可异步读写数据
* 读数据时读取到Buffer，写数据则必须通过Buffer写数据。
* 包括: FileChannel、SocketChannel、 DatagramChannel等。

### 7.2.4 Channel-Buffer
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070201.png)

## 7.3 NIO常用API学习
### 7.3.1 NIO-职责
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070301.png)

### 7.3.2 Selector注册事件
* SelectionKey.OP_CONNECT连接就绪
* SelectionKey.OP_ACCEPT接受就绪
* SelectionKey.OP_READ读就绪
* SelectionKey.OP_WRITE写就绪

### 7.3.3 Selector使用流程
* open()开启一个选择器，可以给选择器注册需要关注的事件
* register()将一个Channel注册到选择器，当选择器触发对应关注事件时回调到Channel中，处理相关数据
* select()/selectNow()一个通道Channel，处理一个当前的可用、待处理的通道数据
* selectedKeys()的到当前就绪的通道
* wakeUp()唤醒一个处于select状态的选择器
* close()关闭一个选择器，注销所有关注的事件

### 7.3.4 Selector注意事项
* 注册到选择器的通道必须为非阻塞状态
* FileChannel不能用于Selector，因为FileChannel不能切换为非阻塞模式；套接字通道可以

### 7.3.5 Selector SelectionKey
* Interset集合、Ready集合
* Channel通道
* Selector选择器
* obj附加值

## 7.4 NIO重写服务器器-1 TCPServer改造
代码略

## 7.5 NIO重写服务器器-2 ClientHandler改造
代码略

## 7.6 NIO重写服务器-3测试、ByteBuffer
### 7.6.1 ByteBuffer put写
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070601.png)

* 1.初始情况下：
    * position: 0
    * limit: 末尾（最大容量）
    * capacity: 末尾（最大容量）
* 2.put:
    * position: 定位到1
    * limit: 末尾（最大容量）
    * capacity: 末尾（最大容量）

### 7.6.2 ByteBuffer get读
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070602.png)

* 1.当前：
    * position: 0
    * limit: 定位到第5位
    * capacity: 末尾（最大容量）
* 2.get:
    * position: 定位到第1位
    * limit: 定位到第5位
    * capacity: 末尾（最大容量）
    
### 7.6.3 ByteBuffer clear
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070603.png)

* 1.当前：
    * position: 定位到第3位
    * limit: 定位到第5位
    * capacity: 末尾（最大容量）
* 2.clear:
    * position: 定位到第0位
    * limit: 定位到末尾（最大容量）
    * capacity: 末尾（最大容量）
    * 覆盖操作：数据并没有被清理，只会更改坐标值，后面的操作只是覆盖这些旧值

### 7.6.4 ByteBuffer flip
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070604.png) 

* 1.当前put操作后：
    * position: 定位到第5位
    * limit: 末尾（最大容量） 
    * capacity: 末尾（最大容量）
* 2.flip:
    * position: 定位到第0位
    * limit: 定位到之前position的位置（第5位）
    * capacity: 末尾（最大容量）

    
### 7.6.5 ByteBuffer 写模式与读模式
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070605.png)

## 7.7 NIO服务器Thread优化-1
* 1.现有线程模式
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070701.png)

* 2.单Thread模型
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070702.png)

* 3.监听与数据处理线程分离
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/070703.png)

## 7.8 NIO服务器Thread优化-2

## 7.9 NIO服务器Thread优化-3

## 7.10 NIO服务器Thread优化-4

## 7.11 NIO服务器Thread优化-5

## 7.12 NIO服务器Thread优化-6

## 7.13 NIO知识归纳梳理
### 7.13.1 NIO vs IO
* IO: 以流为导向，阻塞IO
* NIO：以缓冲区为导向，非阻塞IO，Selector选择器

| IO | NIO |
| :-- | :-- |
| Stream oriented | Buffer oriented |
| Blocking IO  | Non blocking IO  |
|   | Selectors |

### 7.13.2 阻塞IO线程处理数据
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/071301.png)

### 7.13.3 NIO线程处理数据
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/071302.png)

### 7.13.4 阻塞IO处理多客户端连接
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/071303.png)

### 7.13.5 NIO处理多客户端连接
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/071304.png)