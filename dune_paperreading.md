Dune阅读报告
==========

<center>毛海宇    2015310607</center>

---

- Dune想要解决什么问题
  - 为什么有“直接使用特权CPU硬件”这个需求
     - 一些应用需要频繁使用特权硬件，如垃圾回收需要访问页表
     - 系统调用需要进行用户态-内核态切换，开销大
  - 现有使用特权硬件特性的两种方式及其缺点
     - 根据应用需求修改内核
     太过直接，如果修改不恰当的话影响稳定性。如果多个应用都根据自己的需要修改，这些修改不一定相容
     -  把应用部署到使用特定内核的虚拟机上 
     虚拟机与host系统的整合不好，把应用部署到虚拟机上可能丧失一些功能，得不偿失
  - Dune的两个目标
        -  目标一: *User-level Access to Privileged CPU Features*，用户态的进程可以直接使用内核态的CPU特性，如页表、中断、异常。
        -  目标二: *Safe*，做到有效的隔离，防止系统遭到破坏，保持稳定性
- Dune是怎样解决的
  - 手中的筹码:VT-x
        - VT-x将CPU划分为两种状态，VMX root和VMX non-root
        - VMX root状态用来运行VMM，并且不改变CPU的行为(除了增加管理VT-x的指令之外)
        - VMX non-root状态限制CPU的行为，用来运行虚拟化的guest操作系统
        - guest可以访问页表基地址寄存器CR3，这样就可以通过设置页表来访问任意物理地址，包括VMM的内存。所以，VT-x提供了一个EPT(拓展页表)，用于确保guest系统的内存隔离(*目标二*)
  - 体系结构
     - Dune的核心是Dune内核模块(实现为可动态加载的Linux内核模块)，此模块管理VT-x并且向用户程序提供访问内核态硬件的能力
     - 内核(包括Dune内核模块)和普通的进程运行运行在VMX root状态，使用Dune的进程(如Garbage collector)运行在VMX non-root状态
     - 在Dune进程和Dune内核模块之间，提供了一个libDune，这个non-root状态的库封装了一些操纵硬件的指令，便于用户编程 
  - 内存管理
     - 为Dune的进程提供一个可以访问、修改的用户页表，此页表将guest-virtual-memory映射到guest-physical-memory
     - EPT将guest-physical-memory映射到host-physical-memory
     - 从guest-virtual到host-physical需要两层映射(*代价*)
  - 提供访问硬件的能力(*目标一*)
     - Exceptions、Privilege modes在VMX non-root状态下隐含，不需额外配置
     - 虚拟内存需要访问CR3，可以在VMCS(VM control structure)中配置。而且每一个Dune进程可以设置、维护自己的VMCS。
  - 保留OS接口
     - 目的是避免已有软件的大幅改动 
     - Dune的进程使用VMCALL进行系统调用，VMCALL比普通system call的开销大(*代价*)
     




