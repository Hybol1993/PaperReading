Non-scalable locks are dangerous 阅读报告
============

<center>毛海宇 2015310607 </center>

##问题提出

部分操作系统需要依赖Non-scalable spin locks来序列化，Linux内核就需要使用ticket spin locks, 尽管Scalable locks理论上来说有较好的性能。而且比较general常识是Non-scalable locks，比如说简单的spin locks，当资源竞争非常高的时候，性能是非常差的。尽管如此，许多操作系统遇到相同情况时候还是选择使用Non-scalable locks. 然而现在会遇到很多由于Non-scalable locks导致系统吞吐率陡然下降的场景。 那么这篇论文主要想阐述的一个观点就是，Non-scalable locks 是危险，为了具象描述这种危险，主要以linux 内核中的lock机制展开描述。

论文从三个方面说明了Non-scalable的问题。
> * Non-scalable locks 会严重影响整体系统性能。
> * Non-scalable locks 会使得性能的下降会随着核数的增多陡然下降。
> * Non-scalable locks 竞争区域非常短，不容易被捕获。

## 解决方案

Non-scalable 有那么多的问题，那么我们自然而然的就会想到大家都应该使用scalable locks，特别是在操作系统内核中，因为里面的负载和特权级的竞争是比较难以控制的。 为了证明这件事，论文作者们使用scalable MCS locks 代替linux 内核中spin locks，然后重新去运行那种会出现性能突然下跌的软件。 从测评结果可以看出，MCS lock提高了可扩展性，像预期那样避免了性能的突然下降。

这种方法是有人反对的，反对者认为Non-scalable lock的问题应该通过修改软件本身来消除序列化本身的瓶颈。而scalable lock并没有本质上解决问题，只是推迟修改软件本身的需求。
这种想法是没错的，但是实际上想要一次性消除内核中隐藏的竞争问题是不可能的。即使某个内核在某类问题上具有较好的可扩展性，同样的内核很可能不适应下一代的硬件从而出现可扩展的瓶颈。 应该把scalable lock当作是一种短暂性能提高内核性能的一种基本方法。

## 论文的主要贡献

这篇论文的主要贡献是进一步扩展之前的研究工作的结论--Non-scalable locks是危险的， 它们不但自身性能差，而且可能会导致整个系统的性能下降。
说得具体点可以分成三个点：

> * 用实际用例表明了non-scalable locks 会因为自身性能比较差的原因导致在实际工作中的性能的突然下降。
> * 论文提出了一个独立的针对Non-scalable spin locks 行为的综合模型，用来记录所有操作的状态。
> * 在多核x86 的处理器上验证了MCS Lock可以最大限度的提高可扩展性，在影响性能的情况下。


