# 第3章 Socket UDP快速入门
## 3.1 UDP是什么
* 英语: User Datagram Protocol，缩写为UDP
* 一种用户数据报协议，又称用户数据报文协议
* 是一个简单的面向数据报的传输层协议，正式规范为RFC 768 
* 用户数据协议、非连接协议

### 3.1.1 为什么不可靠
* 它一旦把应用程序发给网络层的数据发送出去，就不保留数据备份
* UDP在IP数据报的头部仅仅加入了复用和数据校验(字段)
* 发送端生产数据，接收端从网络中抓取数据
* 结构简单、无校验、速度快、容易丢包、可广播

### 3.1.2 UDP能做什么
* DNS、TFTP、SNMP
* 视频、音频、普通数据（无关紧要数据）
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/030101.png)

### 3.1.3 UDP包最大长度
* 16位->2字节存储长度信息
* 2^16-1 = 64K-1 = 65536-1 = 65535
* 自身协议占用: 32+32位 = 64位 = 8字节
* 65535-8 = 65507 byte

## 3.2 UDP核心API讲解
### 3.2.1 API-DatagramSocket
* 用于接收与发送UDP的类
* 负责发送某一个UDP包,或者接收UDP包.
* 不同于TCP，UDP并没有合并到Socket API中
* API：
    * DatagramSocket()创建简单实例，不指定端口与IP
    * DatagramSocket(int port)创建监听固定端口的实例
    * DatagramSocket(int port, InetAddress localAddr)创建固定端口指定IP的实例.
    * receive(DatagramPacket d): 接收
    * send(DatagramPacket d): 发送
    * setSoTimeout(int timeout): 设置超时,毫秒
    * close() :关闭、释放资源

### 3.2.2 API-DatagramPacket
* 用于处理报文
* 将byte数组、目标地址、目标端口等数据包装成报文或者将报文拆卸成byte数组
* 是UDP的发送实体，也是接收实体
* API：
    * DatagramPacket(byte[] buf, int offset, int length, InetAddress address, int port)
    * 前面3个参数指定buf的使用区间
    * 后面2个参数指定目标机器地址与端口
    * DatagramPacket(byte[] buf, int length, SocketAddress address)
    * 前面3个参数指定buf的使用区间
    * SocketAddress相当于InetAddress + Port
    * setData(byte[] buf, int offset, int length)
    * setData(byte[] buf)
    * setLength(int length)
    * getData()、getOffset()、getLength()
    * setAddress(InetAddress iaddr)、setPort(int iport)
    * getAddress()、getPort()
    * setSocketAddress(SocketAddress address)
    * getSocketAddress()

## 3.3 UDP单播、广播、多播-1
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/030301.png)

* IP地址类别
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/030302.png)

## 3.4 UDP单播、广播、多播-2
### 3.4.1 广播地址
* 255.255.255.255受限广播地址
* C网广播地址一般为: XXX.XXX.XXX.255 (192.168.1.255)
* D类IP地址为多播预留

### 3.4.2 网络信息
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/030401.png)

### 3.4.3 IP地址构成
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/030402.png)

### 3.4.4 广播地址运算
* 案例一：
IP: 192.168.124.7
子网掩码: 255.255.255.0
网络地址: 192.168.124.0
广播地址: 192.168.124.255

* 案例二：
IP: 192.168.124.7
子网掩码: 255.255.255.192
网络地址: 192.168.124.0
广播地址: 192.168.124.63

* 运算步骤：
    * 255.255.255.192->11111111.11111111.11111111.11000000
    * 可划分网段 : 2^2 = 4
    * O~63、64~127、128~191、192~255
    * 192.168.124.63

### 3.4.5 广播通信问题-广播地址运算问题
主机一: 192.168.124.7 ,子网掩码: 255.255.255.192
主机二: 192.168.124.100 ,子网掩码: 255.255.255.192
主机一广播地址: 192.168.124.63
主机二广播地址: 192.168.124.127

两者的广播地址不同，在主机一内发的广播，主机二并不能收到，即便发送的受限广播地址（255.255.255.255），主机二依然无法收到

## 3.5 案例实操-局域网搜索案例-1
* UDP接收消息并回送功能实现
* UDP局域网广播发送实现
* UDP局域网回送消息实现
 
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/030501.png)

## 3.6 案例实操-局域网搜索案例-2
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/030601.png)