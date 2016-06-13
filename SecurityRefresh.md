Sequrity Refresh: Prevent Malicious Wear-out and Increase Durability for Phase-Change Memory with Dynamically Randomized Address Mapping 阅读报告
============

<center>毛海宇 2015310607 </center>
-----------------------------------------------------
##背景##
>* 还没有多少人关注PCM硬件安全设计问题
>* 利用旁路，很容易得到PCM硬件中的平衡磨损策略
>* 基于平衡磨损策略对PCM进行攻击，快速摧毁一个PCM
>* 迫切需要一个低功耗的，安全的，磨损平衡的方法来使PCM有一个高的利用率

##本文贡献##
>1. 阐述了PCM设计中缺乏安全性以及其可能引起的严重问题
>2. 分析以前各种能使得PCM在可观时间内磨损的计算机安全隐患
>3. 提出一个动态的低功耗的能使磨损平衡的机制，称为security refresh，防止恶意磨损，实现安全和寿命的折中

##现有磨损管理的安全隐患##
现有的考虑减少PCM写磨损的技术分为两类：
>1. 去掉多余的写
>2. 磨损平衡

仅有刚提出的RBSG方法考虑到了安全问题。现有的这些技术，包括RBSG，都存在安全隐患。

####没有保护的PCM的安全隐患####
用比cache大小大一位的连续地址写访问，来一直不停的写那一块地方，大约２分钟写坏一块PCM。

####减少冗余写技术的PCM的安全隐患####
缓存写，数据比较写等技术用于PCM来减少冗余写。但是同样存在上述问题，攻击者能通过不停地写PCM cell的固定一块来加速PCM磨损。同样，只要大约2分钟就可以写坏一块PCM。

####磨损平衡的PCM的安全隐患####
当OS是compromised时，恶意设计的攻击能使PCM快速磨损至坏。




##Security Refresh算法##
####算法所需硬件结构####
>* 用security refresh controller(SRC) 来作为refresh的控制器；
>* 把SRC放在PCM里面，防止side-channel攻击以及为提高性能做准备等；

####算法流程####
>Step1 随机数硬件生成器生成一个随机数k1，硬件寄存器中保存有上一轮的随机数k0。

>Step2 从region第一个地址开始，每个地址称为MAi(i为位置编号)，判断MAi异或k0再异或k1的值是否小于MAi。如果小于，则跳过；否则，对MAi异或上k1，得到RMAi。

>Step3 判断是否所有的MAi都被遍历过，如果是，则结束这一轮的refresh；如果不是，则返回Step2。

####对应硬件实现####
见paper中硬件实现结构图。

##security refresh 中的 trade-off##
PCM设计中有很多trade-off，例如一次security refresh长，其中写的次数超过了PCM剩下可以承受的极限，攻击者就能在下一个refresh之前把这个PCM磨损掉；如果security refresh周期短，就会不停的做swap操作，而swap操作会有很多写，会影响PCM的性能。本文用简单的分析模型对PCM的耐用性和写overhead做trade-off。在分析中发现：
>1. 更大的区域会将局部写分散到更大的内存空间上。
>2. 更大的区域需要更小的refresh间隔来增加随机map的改变；如果refresh周期太长，则会保持一个map太久，从而可能被攻击者攻击到。
>3. 更短的refresh周期会有更多的swap，性能下降严重。

根据第一条，本文用一个bank作为一个region；根据第二三两条，在一个bank大小的region上，写overhead到了不能忍受的地步，所以要探索其他技术来缓解写overhead带来的影响。

##两层security refresh##
提出一个两层security refresh来在保持大region的情况下缓解写overhead。
把一个region分成多个sub-region，用来代替很小的refresh间隔。每个sub-region有自己的sub-region SRC。给定一个refresh间隔，小的region有更高的refresh频率。
操作流程：
>1. region SRC接受到一个请求，如果是R/W请求，则把地址remap到IRMA；如果是重新开一个refresh的请求，则把请求发送到sub-region SRC，
>2. 如果sub-region SRC收到的是R/W请求，则根据IRMA用上述算法来访问自己的sub-region；如果sub-region　SRC收到的是新refresh请求，则用上述算法对自己的sub-region进行refresh，在每个sub-region结束refresh前，外部的region SRC不能接受任何请求，以保证其完整性。

##评价##
两层的security refresh能使得一个256B的内存区域，用128个外层刷新间隔64个内层刷新间隔的PCM使用长达5年。而且他的性能只下降了1.2%。






