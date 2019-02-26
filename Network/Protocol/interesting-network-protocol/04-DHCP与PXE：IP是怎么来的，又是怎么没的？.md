# 以问答写笔记：

1. 正确配置IP?
CIDR、子网掩码、广播地址和网关地址。

2. 在跨网段调用中，是如何获取目标IP的mac地址的？
从源IP网关获取所在网关mac,
然后又替换为目标IP所在网段网关的mac,
最后是目标IP的mac地址

3. 手动配置麻烦，怎么办？
DHCP！Dynamic Host Configuration Protocol！
DHCP, 让你配置IP，如同自动房产中介。

4. 如果新来的，房子是空的(没有操作系统)，怎么办？
PXE， Pre-boot Execution Environment.
"装修队"PXE，帮你安装操作系统。

