# 第2章 Socket网络编程快速入门
## 2.1 什么是网络编程？
> **什么是网络编程?**
> * 什么是网络、计算机网络的构成是什么?
> * 什么是网络编程?
> 
> **什么是网络?**
> * 在计算机领域中,网络是信息传输、接收、共享的虚拟平台
> * 通过它把各个点、面、体的信息联系到一起,从而实现这些资源的共享
> * 网络是人类发展史来最重要的发明,提高了科技和人类社会的发展 

### 2.1.1 局域网
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020101.png)
### 2.1.2 互联网
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020102.png)

### 2.1.3 什么是网络编程
* 网络编程从大的方面说就是对信息的发送到接收
* 通过操作相应Api调度计算机硬件资源,并利用传输管道(网线)进行数据交换的过程
* 更为具体的涉及:网络模型、套接字、数据包

### 2.1.4 7层网络模型-OSI
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020103.png)

### 2.1.5 7层网络模型-编程
* 基础层:物理层(Physical)、数据链路层(Datalink)、网络层(Network)
* 传输层(Transport) : TCP-UDP协议层、Socket

### 2.1.6 网络模型-对应关系
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020104.png)

## 2.2 Socket与TCP、UDP
### 2.2.1 What is Socket?
* 简单来说是IP地址与端口的结合协议( RFC 793 )
* 一种地址与端C的结合描述协议
* TCP/IP协议的相关API的总称;是网络Api的集合实现
* 涵盖了：Stream Socket/Datagram Socket

### 2.2.2 Socket的作用与组成
* 在网络传输中用于唯一标示两个端点之 间的链接
* 端点:包括( IP+Port )
* 4个要素:客户端地址、客户端端口、服务器地址、服务器端口

### 2.2.3 Socket传输原理
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020201.png)

### 2.2.4 Socket之TCP
* TCP是面向连接的通信协议
* 通过三次握手建立连接,通讯完成时要拆除连接
* 由于TCP是面向连接的所以只能用于端到端的通讯

### 2.2.5 Socket之UDP
* UDP是面向无连接的通讯协议
* UDP数据包括目的端口号和源端口号信息
* 由于通讯不需要连接,所以可以实现广播发送，
* 并不局限于端到端

### 2.2.6 TCP传输图解
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020202.png)

### 2.2.7 UDP传输图解
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020203.png)

### 2.2.8 Client-Server Application
* TCP/IP协议中,两个进程间通信的主要模式的: CS模型
* 主要目的:协同网络中的计算机姿源、服务模式、迸程间数据共享
* 常见的：FTP、SMTP、HTTP

## Socket TCP牛刀小试
> * 构建TCP客户端、服务端
> * 客户端发送数据
> * 服务器读取数据并打印

## 2.3 Socket TCP牛刀小试-客户端实现

## 2.4 Socket TCP牛刀小试-服务端实现

## 2.5 报文、协议、Mac地址
### 2.5.1 报文段
* 报文段是指TCP/IP协议网络传输过程中，起着路由导航作用
* 用以查询各个网络路由网段、IP地址、交换协议等IP数据包
* 报文段充当整个TCP/IP协议数据包的导航路由功能
* 报文在传输过程中会不断地封装成分组、包、帧来传输
* 封装方式就是添加一些控制信息组成的首部，即报文头
### 2.5.2 传输协议
* 协议顾名思义，一种规定，约束
* 约定大于配置，在网络传输中依然适用；网络的传输流程是健壮的稳定的，得益于基础的协议构成
* 简单来说: A->B的传输数据，B能识别，反之B->A的传输数据A也能识别，这就是协议

### 2.5.3 Mac地址
* Media Access Control或者Medium Access Control
* 意译为媒体访问控制，或称为物理地址、硬件地址
* 用来定义网络设备的位置
* 形如: 44-45-53-54-00-00；与身份证类似

## 2.6 IP、端口及远程服务器
### 2.6.1 IP地址
* 互联网协议地址（英语：Internet Protocol Address , 又译为网际 协议地址），缩写为IP地址（英语：IP Address)
* 是分配给网络上使用网际协议（英语：Internet Protocol, IP)的设备的数字标签
* 常见的IP地址分为IPv4与IPv6两大类
* IP地址由32位二进制数组成，常以XXX.XXX.XXX.XXX形式表现每组XXX代表小于或等于255的10进制数
* 如: 208.80.152.2
* 分为A、B、 C、D、E五大类，其中E类属于特殊保留地址

### 2.6.2 IP地址-IPv4
* 总数量: 4,294,967,296个(即232) : 42亿个；最终于2011年2月3日尽
* 如果主机号是全1，那么这个地址为直接广播地址
* IP地址"255.255.255.255" 为受限广播地址

### 2.6.3 IP地址-IPv6
* 总共有128位长，IPv6地址的表达形式，一般采用32个十六进制数。也可以想象为1632个
* 由两个逻辑部分组成: 一个64位的网络前缀和一个64位的主机地址，主机地址通常根据物理地址自动生成，叫做EUI-64 (或者64-位扩展唯一标识)
* 2001:0db8:85a3:0000:1319:8a2e:0370:7344
* IPv4转换为IPv6-定可行，IPv6转换为IPv4不一定可行

### 2.6.4 端口
* 如果把IP地址比作一间房子，端口就是出入这间房子的门或者窗户
* 在不同门窗户后有不同的人，房子中的用户与外界交流的出口
* 外界鸽子(信息)飞到不同窗户也就是给不同的人传递信息
* 0到1023号端口以及1024到49151号端口都是特殊端口
* **2.6.4.1 端口-特殊端口**
    ![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020601.png)
* 计算机之间依照互联网传输层TCP/IP协议的协议通信，不同的协议都对应不同的端口
* 49152到65535号端口属于“动态端口”范围，没有端口可以被正式地注册占用
* 问题:端口总数数: 65536 ,连接能建立多少个? 65536个?

### 2.6.5 数据传输层次
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020602.png)

### 2.6.6 远程服务器
* 局域网：一般而言，家里的环境以及公司相互电脑之间环境都属于局域网.
* 我与你们的电脑之间属于互联网，而非局域网
* 默认的：我的电脑无法直接链接到你们的电脑

#### 2.6.6.1 连接示意图
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020603.png)

![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020604.png)

#### 2.6.6.2 Web请求流程
![](https://raw.githubusercontent.com/Mr-lidajun/Programming-Notes/master/Java/Socket网络编程进阶与实战/img/020605.png)