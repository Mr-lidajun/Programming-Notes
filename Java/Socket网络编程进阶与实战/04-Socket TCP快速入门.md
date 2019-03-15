# 第4章 Socket TCP快速入门
## 4.1 TCP是什么、能做什么
### 4.1.1 TCP是什么
* 英语: Transmission Control Protocol, 缩写为TCP
* TCP是**传输控制协议**；是一种**面向连接**的、**可靠的**、**基于字节流**的传输层通信协议，由IETF的RFC 793定义
* 与UDP一样完成第四层传输层所指定的功能与职责

### 4.1.2TCP的机制
* 三次握手、四次挥手
* 具有校验机制、可靠、数据传输稳定

### 4.1.3 TCP链接、传输流程
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040101.png)

### 4.1.4 TCP能做什么
* 聊天消息传输、推送
* 单人语音、视频聊天等
* 几乎UDP能做的都能做，但需要考虑复杂性、性能问题
* 限制: 无法进行广播，多播等操作
  
## 4.2 TCP核心API讲解
* socket(): 创建一个socket
* bind(): 绑定一个socket到一个本地地址和端口上
* connect(): 连接到远程套接字
* accept(): 接受一个新的连接
* write(): 把数据写入到Socket输出流
* read(): 从Socket输入流读取数据

### 4.2.1 Socket客户端创建流程
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040201.png)

### 4.2.2 Socket服务端创建流程
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040202.png)

### 扩展-Socket与进程关系
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040203.png)

## 4.3 TCP连接可靠性-三次握手、四次挥手
> SYN：发起连接的命令
> ACK：回送命令
> FIN：发起断开的命令
> rand(): 随机数
> u：随机值

### 4.3.1 连接可靠性-三次握手
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040301.png)

### 4.3.2 三次握手-数据随机的必要性
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040302.png)

随机数是为了确定对应的客户端，客户端向服务端发起连接的时候，会把随机数带给服务端，服务端在回送的时候会把随机数加1回送给客户端，以便两者之前的连接时可靠的，没有进行一个错误的回送

### 4.3.3 三次握手-四次挥手
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040303.png)

## 4.4 TCP传输可靠性-排序、丢弃、重发
* 排序、顺序发送、顺序组装
* 丢弃、超时
* 重发机制-定时器

### 4.4.1 数据片发送流程
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040401.png)

### 4.4.2 TCP可靠性示意图（一）
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040402.png)

* 排序
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040403.gif)

* 丢弃
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040404.gif)

* 重发
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040405.gif)


### 4.4.3 TCP可靠性示意图（二）
* 服务器端一对多的连接可靠性示意图
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/040406.gif)
