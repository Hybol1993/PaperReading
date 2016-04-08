mTCP 阅读报告
============

<center>毛海宇 2015310607 </center>

---
##问题背景
　　短TCP连接现在很普遍,长TCP连接(视频服务等)消耗的是带宽,短连接消耗的是TCP连接数.在一个大型蜂窝网内,90%以上的TCP流量小于32KB,一半以上小于4KB.扩展短连接处理效率,不仅对流行的面向用户的在线服务有重要意义,同时对一些后台系统(如memcached clusters)和middleboxes(ssl代理等)同样很重要,因为这些服务都需要尽高速的来处理短TCP.尽管近来这方面发展迅速,但高速的TCP事务处理,依然是一个挑战.Linux的TCP事务处理的峰值是每秒0.3million个,而如今I/O却达到10million个每秒,很明显,Linux系统内核的处理,已经是一个瓶颈.在这之前的研究,都关注在syscall的高负载或多核系统的实现上导致的资源冲突.之前的方法,彻底的改变了I/O的抽象,来缓冲syscall的消耗.这种方法的实际限制,是它需要对内核做很多改动,并迫使应用端重写.

##现存问题

> *  缺乏连接局部性，为在多核机器上提升性能,许多应用都是多线程的,但是它们共享一个socket,所以要通过锁的机制来获取socket,因此对性能影响严重.内核处理TCP连接可能与应用端代码处理的不同(lack of connection locality)导致额外的负载(CPU 缓存不命中,cache-line sharing)

> *  共享文件描述符空间， 在支持posix的操作系统中,文件描述符fd是进程内共享的,分配新的socket时,就需要查找最小可用的fd号.在一个任务繁重的server中,多线程之间需要额外的锁的开销.在socket上使用fd,需要额外的检查vfs的负载.MegaPipe通过分隔socket fd和一般文件的fd,来减少这种开销.

> * 低效的数据包处理，per-packet内存分配和DMA的开销.NUMA-unaware memory access和heavy data structures(sk_buff)是处理小包的主要瓶颈.采取批处理的方法,来减少开销.

> * 严重系统的调用开销，syscall user/kernel mode switching.频繁的syscall调用导致处理器状态污染(顶层cache,分支预测表),带来性能上的损失.Solution: syscall batching, efficient scheduling.但以上两种方法,均需要对syscall接口和语义的改动.

##论文的主要方法和贡献
本文提出一种不需要对内核现有代码的方法，即ｍTCP: 主要实现以下三个目标。
>* TCP stack的多核扩展;
>* 易于使用,应用便于移植到mTCP;
>* 易于部署,不需要内核代码的改动.

主要贡献以下两种关键方法：

>* packet-event & socket-event batching 的整合,同时包含了现有所有的优化方法.性能提升:33%SSLShader, 320%lighttpd
>* Purely done at the user level. 不需更改内核. BSD-like socket.(accept() --> mtcp_accept())

##为什么需要user-level的TCP
    文中的结果表明,Linux和MegaPipe的CPU周期的内核态占用率达80%-83%.锁,缓存管理,频繁的用户态和内核态切换是主要原因.因此kernel以及TCP协议栈实现,是主要瓶颈.而mTCP是Linux的4.3倍以上.因其内核态中和TCP栈上消耗的CPU周期更少.
本文从以下三个方面思考了以上问题：
>* 是否可以设计出一种用户态的TCP协议栈,并整合现有的优化策略于一体?
>* 如果设计了这样一个系统,性能方面的提升有多少?
>* 现有的packet I/O libraries 能否用到TCP stack上,对性能有较大提升

为什么User-level的TCP那么具有吸引力呢，主要有以下两点：

> * 把TCP stack从复杂的内核中解放出来， 可以允许把已有的high-performance packet I/O library直接拿来就用.
> * 在进行batch processing的时候， 不需要对内核代码进行更改，也不需要对系统状态进行切换，允许mTCP stack 向后兼容，支持BSD-like socket 接口。


##设计
本文的设计目标是在多核系统上高扩展性,并保持向多线程,事件驱动的应用兼容.

###User-level 的Packet I/O Libary
    Polling(轮询)极度浪费CPU周期.我们需要多网卡之间高效的多路复用.比如发数数据包时,不希望TX队列被阻塞住,减少重传.mTCP扩展了PacketShader I/O engine(PSIO)来支持event-driven packet I/O interface.利用RSS(把收到的包根据flow放到不同的RX队列中,flow层的亲缘关系来最小化cpu内核的资源竞争).允许mTCP线程同一时间从多个NIC接口的RX/TX队列中等待事件.PSIO减小了syscall和上下文切换的开销,还减少了包内存分配和DMA的开销.PSIO中,包是批量发送的,减少了类似于DMA地址映射和IOMMU查找的开销.

### User-level 的 TCP Stack
使用用户层的TCP协议栈,自然避免了syscall的开销(比如socket I/O).实现用户层tcp协议栈的一个方法:做成一个library,并做为应用层程的主线程.这种方法的局限性是TCP进程间处理的正确性依赖于从application及时唤醒TCP functions.在mTCP中,采取创建一个独立的TCP线程来避免这种事情.Application通过Shared TCP buffers来使用mTCP library.但这种方法会产生来自于管理并发数据结构和application和mTCP线程切换的额外开销.不幸的是这种额外开销,甚至远远大于syscall(19倍).
### 应用编程接口

该系统的主要目标之一,就是减小现有应用的移植工作.保持接口及接口语义的一致性.

**User-level socket API**:类似BSD socket的接口.accept() --> mtcp_accept(). other funcs: e.g.. fcntl() ioctl(). mtcppipe().socket描述符空间是mTCP线程局部的.每个mTCP socket都关联一个线程上下文.这样节省了进程上下文之间的锁的开销.另外,存找最小fd也是开销,直接返回一个可用的fd即可.省节这一部分的开销.

**User-level event system**: 开发了epoll()-like的事件系统.该事件处理系统只是将多个flow的事件指量处理,不改变事件处理逻辑的语义.mtcpepollwait() --> epoll_wait().

**Application**: mTCP在不改动内核代码,保持接口一致性的前提下,整合了现今几乎所有已知的优化技术.因此应用程序可以很方便的扩展使用,而无需改变程序逻辑.因为shared TCP buffers的存在,application中的漏洞,会影响到TCP Stack.mTCP会绕过现有的Linux Kernel的一些服务,如防火墙等.如今的原型目前只支持单个应用程序.






