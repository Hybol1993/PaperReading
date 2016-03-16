﻿IX阅读报告
=========
<center>毛海宇    2015310607</center>

------------------------------------------------------

 
 **背景和研究目的**
很多数据中心的应用包括搜索，社交网络还有电子交易平台都在重新定义对系统软件的需求， 新的需求包括对短消息的high packet rates 以及在保证较小尾延迟的情况下对远端请求的毫秒级应答，还有对大量连接数的支持等。当然，系统还需要有一个较强的保护模块以及对资源的弹性使用即允许其他应用使用当前共享在cluster中的空闲资源。 总结起来就是以下几点：
    1. high packet rates
    2. 毫秒级tail latency
    3. 安全保护
    4. 资源的高效利用
    论文作者分析了当前时期的硬件状况，认为在当前丰富的硬件资源的形势下，对于数据中心的应用来说，低延迟和high packet rates 是完全可以实现的，而不幸的是目前的商用操作系统的设计前提设想是适用于各种硬件环境，因此内核调度，网络API，网络协议栈设计的前提是多个应用共享一个处理器核，而且数据包传输时间是远远大于中断和系统调用的延迟。 造成的结果是这类的操作系统只有通过细粒度的资源调度才能实现latency和throughput的tradeoff. 而应用对多核的细粒度资源调度会增加CPU和系统Mem的Overhead,从而限制了throughput.
    由于商用内核的网络协议栈并不能高效利用丰富的硬件资源，因此就涌现了一些解决方案，当然每个方案都只解决了一部分的问题，也并没有针对数据中心应用的全部需求。论文总结了四种解决方式：
    1. 用户态网络协议栈
    2. 远程内存访问（Alternatives to TCP, RDMA)
    3. MegaPipe (Alternatives to POSIX API)
    4. 操作系统的增强。(SO REUSEPORT, epoll) 等。
基于此，论文作者提出了IX，其设计的系统可实现高吞吐率，低延时， 强保护以及资源的高效利用。 在保证有较高performance的情况下，还保留有非常大量的API的兼容性，同时也能够提供一个现代操作系统例如linux提供的服务。

**IX的设计方法**
   在前面提到两个需求(毫秒级延迟，High packet rates)中, 这种需求也并不是数据中心应用独有的，而且这些需求其实已经在一些中间设备（如防火墙，负载均衡器，软件路由等）的设计中被解决了，解决的办法就是将网络协议栈和应用程序整合一个单个的数据层。 而剩下的两个需求（安全防护，资源高效利用）在中间设备的设计过程中是没有被解决的，因为是专用系统，用户也不可以之间访问到。
   很多中间设备的设计原则也是不同于传统操作系统的。其设计原则有两条：
   1. Run each packets to completion（即所有的网络协议和应用在处理一个数据包的时候才能处理下一个数据包）
   2.  synchronization-free operation.(网络流量通过流的一致性哈希被分配到清晰的队列中，一般数据包的处理不需要在不同核之间进行同步，然而传统操作系统的同步操作是非常频繁的）
   IX扩展了这种数据层架构使得其支持不可信的，通用的应用程序，同时满足在前面提到的所有需求。那么它的设计原则有以下四点：
   1. Separation and protection of control and data plane （IX 将内核的控制函数（如负责资源配置的，调度的，监控的等函数，原本运行在网络协议栈上和应用程序逻辑里）从数据层分离出来，控制层在数据层之间复用和调度资源，它也负责弹性调整不同数据层的资源分配）
   2. Run to completion with adaptive batching （在使用了run each packets to comletion同时，添加adaptive patch机制，两者结合的意思是说进入的数据包队列只能在NIC边建立，在每个数据包进行处理之前，网络协议栈发送ACK到peers的速度要跟应用程序处理它们的速度一样快，应用程序处理速度的下降会很快导致在peers中的窗口收缩。 数据层也可以监控队列深度，也可以向控制层请求资源分配。）
   3. Native, zero-copy API with explicit flow control（IX不效仿POSIX API对网络提供接口，数据层kernel 和应用在协调迁移点的通信是通过存储在mem中的消息实现的。 IX的API支持双向的Zero-Copy操作，数据层会和应用协同管理一个消息缓冲池。 数据层强制要求流控制的正确性并可能调整requests的传输导致超出当前的滑动窗口大小，应用控制Buffer的传输）
   4. Flow consistent, synchronization-free processing （使用多队列的带有接受边扩展的NICs将income 数据包流一致性哈希到不同的硬件队列。）
   总体上来说，IX数据层的设计在当前服务器多核并在核数不断上升的情况下具有良好的扩展性，大大地提高了packet rate 和latency. 这种方法没有制约应用程序的内存模块，因此可以很好的利用shared mem 实现不同核之间的信息交换和同步。

**IX的实现**

论文作者这个部分讲述了IX 设计的具体实现，着重关注控制层和多数据层的分离。硬件环境就不多提了，IX控制层包含Linux kernel 和IXCP（也就是IX设计的数据层，是一个用户级的程序。linux kernel 初始化什么的也不多提了，实现过程中也使用了Dune Module 使得数据层可以直接访问硬件特性，而且它为控制层，数据层和不可信的应用程序代码之间提供了全方位的安全防护。
当然作者也提到了IX 数据层的实现，它不同于那种为单一程序的高性能而定制的内核，IX数据层仍然提供很多常见的内核级服务。它实现了内存管理，实现了自己的一套虚拟内存的转换，也实现了按等级划分的时间片机制。值得一提的是， 当前的数据层实现是基于Dune的，并要求硬件系统支持VT-x虚拟化特性的。 数据层还实现了自己的RFC，支持UDP，ARP,ICMP. 

同时，IX实现了基于数据层的API和Opreation, 它实现了一系列系统调用和event condition API, Bactch过系统调用和event condition都通过shared mem, 由用户和内核各自管理。 实现细节就不多说了。

实现具有良好多核扩展性， 主要因为以下：
   1. 系统调用的实现是synchronization-free的
   2. API的实现都是经过优化的，因为系统中不会有并发执行生产者和消费者，实现过程不包含同步原语的原子操作。
   3. 在NICs上流一致性哈希保证了TCP数据流的不相交子集中的每个弹性的线程操作。因此，对于一个服务器程序来说，在Income的request中没有同步操作。
   4. 从论文描述看来， 同步操作仅仅发生在控制层和数据层进行资源分配的时候发生，这种事件发生是很少见的。
 
IX也实现了安全模块

IX API的实现在用户程序代码和网络协议栈之间使用一个协作的流控制模型，恶意程序只能攻击自己，它不能破坏网络协议栈或者影响其他程序。所有在IX中的应用程序代码都运行在用户模式，而数据层代码放在被保护的ring0中，应用程序不能直接访问数据层的mem, 除非使用只读消息buffer.而且数据层也可以用来加强网络安全策略，比如防火墙和访问控制列表。








