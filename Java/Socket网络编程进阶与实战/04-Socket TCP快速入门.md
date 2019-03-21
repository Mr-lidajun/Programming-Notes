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

## 4.5 案例实操-TCP传输初始化配置
* 1.TCP传输初始化配置
    * 初始化服务器TCP链接监听
    * 初始化客户端发起链接操作
    * 服务器Socket链接处理

* 2.客户端与服务器交互
    * 客户端发送简单字节
    * 服务器接收客户端发送数据
    * 服务器回送消息，客户端识别回送消息

* 3.客户端初始化配置
 
    ```java
    private static Socket createSocket() throws IOException {
        /*
        // 无代理模式，等效于空构造函数
        Socket socket = new Socket(Proxy.NO_PROXY);
    
        // 新建一份具有HTTP代理的套接字，传输数据将通过www.baidu.com:8080端口转发
        Proxy proxy = new Proxy(Proxy.Type.HTTP,
                new InetSocketAddress(Inet4Address.getByName("www.baidu.com"), 8800));
        socket = new Socket(proxy);
    
        // 新建一个套接字，并且直接链接到本地20000的服务器上
        socket = new Socket("localhost", PORT);
    
        // 新建一个套接字，并且直接链接到本地20000的服务器上
        socket = new Socket(Inet4Address.getLocalHost(), PORT);
    
        // 新建一个套接字，并且直接链接到本地20000的服务器上，并且绑定到本地20001端口上
        socket = new Socket("localhost", PORT, Inet4Address.getLocalHost(), LOCAL_PORT);
        socket = new Socket(Inet4Address.getLocalHost(), PORT, Inet4Address.getLocalHost(), LOCAL_PORT);
        */
    
        Socket socket = new Socket();
        // 绑定到本地20001端口
        socket.bind(new InetSocketAddress(Inet4Address.getLocalHost(), LOCAL_PORT));
    
        return socket;
    }
    
    private static void initSocket(Socket socket) throws SocketException {
        // 设置读取超时时间为2秒
        socket.setSoTimeout(2000);
    
        // 是否复用未完全关闭的Socket地址，对于指定bind操作后的套接字有效
        socket.setReuseAddress(true);
    
        // 是否开启Nagle算法
        socket.setTcpNoDelay(true);
    
        // 是否需要在长时无数据响应时发送确认数据（类似心跳包），时间大约为2小时
        socket.setKeepAlive(true);
    
        // 对于close关闭操作行为进行怎样的处理；默认为false，0
        // false、0：默认情况，关闭时立即返回，底层系统接管输出流，将缓冲区内的数据发送完成
        // true、0：关闭时立即返回，缓冲区数据抛弃，直接发送RST结束命令到对方，并无需经过2MSL等待
        // true、200：关闭时最长阻塞200毫秒，随后按第二情况处理
        socket.setSoLinger(true, 20);
    
        // 是否让紧急数据内敛，默认false；紧急数据通过 socket.sendUrgentData(1);发送
        socket.setOOBInline(true);
    
        // 设置接收发送缓冲器大小
        socket.setReceiveBufferSize(64 * 1024 * 1024);
        socket.setSendBufferSize(64 * 1024 * 1024);
    
        // 设置性能参数：短链接，延迟，带宽的相对重要性
        socket.setPerformancePreferences(1, 1, 0);
    }
    ```
    
* 4.服务端初始化配置

```java
private static ServerSocket createServerSocket() throws IOException {
    // 创建基础的ServerSocket
    ServerSocket serverSocket = new ServerSocket();

    // 绑定到本地端口20000上，并且设置当前可允许等待链接的队列为50个
    //serverSocket = new ServerSocket(PORT);

    // 等效于上面的方案，队列设置为50个
    //serverSocket = new ServerSocket(PORT, 50);

    // 与上面等同
    // serverSocket = new ServerSocket(PORT, 50, Inet4Address.getLocalHost());

    return serverSocket;
}

private static void initServerSocket(ServerSocket serverSocket) throws IOException {
    // 是否复用未完全关闭的地址端口
    serverSocket.setReuseAddress(true);

    // 等效Socket#setReceiveBufferSize
    serverSocket.setReceiveBufferSize(64 * 1024 * 1024);

    // 设置serverSocket#accept超时时间
    // serverSocket.setSoTimeout(2000);

    // 设置性能参数：短链接，延迟，带宽的相对重要性
    serverSocket.setPerformancePreferences(1, 1, 1);
}
```
    
## 4.6 案例实操-TCP基础数据传输
* 1.基础类型数据传输
    * byte、char、short
    * boolean、int、long
    * float、double、string

* 2.客户端数据传输

```java
private static void todo(Socket client) throws IOException {
    // 得到Socket输出流
    OutputStream outputStream = client.getOutputStream();


    // 得到Socket输入流
    InputStream inputStream = client.getInputStream();
    byte[] buffer = new byte[256];
    ByteBuffer byteBuffer = ByteBuffer.wrap(buffer);

    // byte
    byteBuffer.put((byte) 126);

    // char
    char c = 'a';
    byteBuffer.putChar(c);

    // int
    int i = 2323123;
    byteBuffer.putInt(i);

    // bool
    boolean b = true;
    byteBuffer.put(b ? (byte) 1 : (byte) 0);

    // Long
    long l = 298789739;
    byteBuffer.putLong(l);


    // float
    float f = 12.345f;
    byteBuffer.putFloat(f);


    // double
    double d = 13.31241248782973;
    byteBuffer.putDouble(d);

    // String
    String str = "Hello你好！";
    byteBuffer.put(str.getBytes());

    // 发送到服务器
    outputStream.write(buffer, 0, byteBuffer.position() + 1);

    // 接收服务器返回
    int read = inputStream.read(buffer);
    System.out.println("收到数量：" + read);

    // 资源释放
    outputStream.close();
    inputStream.close();
}
```

* 3.服务端处理客户端消息

```java
/**
 * 客户端消息处理
 */
private static class ClientHandler extends Thread {
    private Socket socket;

    ClientHandler(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        super.run();
        System.out.println("新客户端连接：" + socket.getInetAddress() +
                " P:" + socket.getPort());

        try {
            // 得到套接字流
            OutputStream outputStream = socket.getOutputStream();
            InputStream inputStream = socket.getInputStream();

            byte[] buffer = new byte[256];
            int readCount = inputStream.read(buffer);
            ByteBuffer byteBuffer = ByteBuffer.wrap(buffer, 0, readCount);

            // byte
            byte be = byteBuffer.get();

            // char
            char c = byteBuffer.getChar();

            // int
            int i = byteBuffer.getInt();

            // bool
            boolean b = byteBuffer.get() == 1;

            // Long
            long l = byteBuffer.getLong();

            // float
            float f = byteBuffer.getFloat();

            // double
            double d = byteBuffer.getDouble();

            // String
            int pos = byteBuffer.position();
            String str = new String(buffer, pos, readCount - pos - 1);

            System.out.println("收到数量：" + readCount + " 数据："
                    + be + "\n"
                    + c + "\n"
                    + i + "\n"
                    + b + "\n"
                    + l + "\n"
                    + f + "\n"
                    + d + "\n"
                    + str + "\n");

            outputStream.write(buffer, 0, readCount);
            outputStream.close();
            inputStream.close();

        } catch (Exception e) {
            System.out.println("连接异常断开");
        } finally {
            // 连接关闭
            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        System.out.println("客户端已退出：" + socket.getInetAddress() +
                " P:" + socket.getPort());

    }
}
```