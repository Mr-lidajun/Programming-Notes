#本篇所得
1. hub 集线器，会把包 复制到每台连接的电脑上，不需要包的电脑运动会发生，占用通讯资源 造成了 浪费

2. 交换机 具有记忆功能，可以定期缓存 IP与Mac的数据
3. ip封装了tcp、udp、http等数据
4. IP已知 可以使用 ARP协议 用ip获得mac地址，包里有目标mac地址 最好，没有 可以运用ARP协议 通过 吼 获得 目标mac地址
5. 通讯的第一个问题 是，谁发送、发给谁、谁接受
6. 信息发送 有三种方式 1）信道划分 每个信道一个路径 互不影响 2）轮流协作 比如单双号 发送 3）随机接入协议，先发送 交通堵塞，避开 或者 错开高峰。

回答老师的问题一：ARP协议，已知IP可获得 Mac地址，RARP已知Mac 可获得 IP地址，管理员 可以用Mac定位 定位IP终端设备
问题二：一个局域网有多个交换机 ，广播摸索 回循环 从A-B-C-...-A，这样消耗网络资源，直至资源资源枯竭。

