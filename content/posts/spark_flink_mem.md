---
title: spark和flink的内存管理简单介绍和对比
date: 2016-12-24T11:30:27+08:00
categories:
- big-data
- spark
tags:
- spark
- flink
- big-data
---

# 为什么要做内存管理

最近几年存储和网络硬件的升级对大数据领域相关的系统性能提升很大，在很多场景中CPU和内存的渐渐成为瓶颈。
然而大数据领域很多开源框架都使用JVM。相对 c/c++ JVM系语言对于CPU和内存的利用还是差很多的，主要体现在:
- GC的开销较大，Full GC更会极大影响性能，尤其是对于为了处理更大数据开辟了极大内存空间的JVM来说。
- Java对象的存储密度低。一个对象头就8字节，为了对齐有的时候还要补齐。Java对象的内存很难利用CPU的各级cache，对于 `avx` `simd` 等技术很难利用起来难以发挥CPU的这些优势。
- OOM的问题影响稳定性。在大数据领域经常会遇到OOM的问题，除了影响稳定性之外，JVM崩溃后重新拉起的代价也很高。

为了解决这些问题spark flink hbase hive 等各大框架都在自己管理内存。框架自己比JVM更了解自己的内存，更熟悉生命周期，拥有更多的信息来管理内存。

// 直接的意思是"绕过"JVM的内存管理机制自己管理每一个字节，就像写C一样，而不是new出来等着GC；另一方面是指通过定制的序列化工具等技术直接操作数据内容不必全部反序列化或不反序列化。
# 统一管理内存
内存中的Java对象越少, GC压力越小，而且可以保证统一管理的内存一直呆在老年代，而且也可以是堆外内存。

Spark 和 Flink 都支持堆内和堆外两种内存的管理。管理的策略都和操作系统类似，对内存进行分段分页。两者都是一级页表。

- Spark：页表的大小是 `1 << 13`, 每个页(`MemoryBlock`)的大小不确定。on-heap模式时候能表示的最大页大小受限于long[]数组的最大长度，理论上最大能表示8192*2^32*****8字节(35T)。on-heap模式时如果页超过1M会触发bufferPools来复用long数组(`HeapMemoryAllocator`)。`TaskMemoryManager`接管Task的内存分配释放，"寻址"，分配的内存页大小不固定，具体执行内存申请是由`HeapMemoryAllocator`和`UnsafeMemoryAllocator`进行。

- Flink: `MemoryManager`接管内存分配和释放，构造时页的大小固定，页表大小根据需要的内存反推。`TaskManagerServices`构造`MemoryManager`时会将页大小定为`networkBufferSize`, 默认大小为32KB, 为了配合flink的Network Buffer管理。`AbstractPagedInputView`, `AbstractPagedOutputView`以及各种InputView OutputView 用来进行跨page读写内存。

这些缓解了GC耗时和抖动问题。

# spill到内存外持久化存储
具体原理和为什么都不叫显然，两者实现也没有什么差别。

- Flink: 由内存使用者管理spill规则，并不像操作系统一样。比如: `MutableHashTable`，`SpillingBuffer`...
- Spark: 同样由各类使用者各自管理，所有内存的消费者都继承`MemoryConsumer` abstract class， 实现具体spill方法。相对Flink更整洁，统一。`BytesToBytesMap`，`ExternalAppendOnlyMap`，`ExternalSorter`会spill到disk。

# 列方式存储
根据空间局部性原理，做到缓存友好首先对数据进行向量化存储。通常我们都是对结构化数据的一"列"进行相同的处理，所以

# 缓存友好算法



# codegen (SIMD AVX)


































