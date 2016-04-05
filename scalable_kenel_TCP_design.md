scalable Kernel TCP Design and Implementation for Short-Lived Connections 阅读报告
============

<center>毛海宇 2015310607 </center>
## 问题提出

随着网络带宽和单台机器上核数的提高，应用API模型性能提升的关键在于更短周期的短连接，可扩展TCP协议栈的实现。虽然现在有很多的方案已经被提出来了，但是生产环境依旧需要支持反向兼容的，自底向上的并行TCP协议栈的设计。 而且随着单机核数的增多， 网络协议栈的可扩展性在多核平台上网络应用的性能方面扮演着非常重要的角色。对于长连接来说，新连接建立的元数据管理操作的频率并不足以造成比较严重的竞争，所以在论文中并不去讨论长连接状况下的协议栈的可扩展性的话题，然而，对于短连接来说，情况就大不相同了，它会包含大量的TCP连接的建立和连接终止，会造成在TCB和VFS中的严重共享资源竞争。而完整妥善地处理这个问题的并且保持反向兼容一直是个公开的挑战。 

## 解决方案

论文提出了Fastsocket， 也就是为了解决前面提到的可扩展性和反向兼容性问题，主要通过三大修改来实现这个目的。
> * 全局共享数据结构， Listen Table 和 established table 的分割。
> * 正确掌握流入数据包以任意连接的连接局部性。
> * 为在VFS中的Socket提供一条快捷的路径来解决可扩展性问题，并且将之前提到TCP的设计封装到完整的BSD Socket API 中去。


##主要贡献

> * 引入了一种内核网络协议栈的设计实现了TCB管理的表级分割，同时也实现了任意连接的连接局部性。
> * 论证了对商用硬件配置来说BSD Socket API并不应该是网络协议栈的可扩展性的障碍。
> * 论证了通过对kernel的适量修改，可以建立一个高可扩展性的内核网络协议栈。


##设计实现

Fastsocket 提供三个部分，TCB数据结构的分割，接收流发送器和Fastsocket-aware的VFS。这个三个部分共同提供Per-Core Process Zones. 抽象实现如下：

> * Fastsocket 完全分割全局TCB表的数据结构和管理的分割，因此，当NET_RX_SoftIRQ 到达TCP层，Fastsocket使用每核本地Local listen Table 和每核本地Established table 实现TCB管理的分割。

>* 在一个流入数据包进入网络协议栈之前，Fastsocket 使用RFD将数据包送到合适的核上，这样做使得fastsocket最大化连接局部性最小化CPU的cache bouncing.

>* 内核VFS层是Socket API抽象和兼容性的关键，Fastsocket-aware VFS绕过了不必要的锁密集 VFS 例程，使用socket-special 快捷路径消除可扩展性瓶颈，同时保留足够多的状态提供所有socket API的功能设计。


