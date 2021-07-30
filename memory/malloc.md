# malloc
`go` 的内存管理最初是基于 **[tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)** 的，但是也有很多不同的地方。  

-----------------------------------------------------------------------
首先我们对于整个内存分配管理有一个大概的印象，可能不够准确和全面，但是有助于理解源码：  
内存管理主要由页组成。大小对象分别管理内存分配。主要的数据结构描述：
`fixalloc` 固定大小内存对象分配器，用于管理 分配器 的内存  
`mheap` 以页为单位的内存堆  
`mspan` 一组由 `mheap` 管理的正在使用的页  
`mcentral` 一个给定类型的所有 `mspan` 的集合  
`mcache` 每个 `P` 上的 `mspan` 的缓存  
`mstats` 内存分配的统计信息  
`mheap` 是全局的一大块内存，它是由一个个页组成，这些页通过 `mspan` 管理。而 `mheap` 通过 `mcentral` 管理 `msapn`，每个处理器`P`上都有一个 `mcache`，里面是有空闲位置的 `mspan`，可以避免多线程分配内存加锁。超过32k的对象通过 `mheap` 来分配和释放。小于32k的对象分配流程如下：
1. 从 `P` 中的`mcache` 获取 `mspan` 来存放。
2. `mcache` 中没有，去 `mcentral` 拿一个 `mspan`
3. `mcentral` 中没有，去 `mheap` 拿一些页来用作 `mspan`
4. `mheap` 中没有，去操作系统申请一组页
------------------------------------------------------------------------

## 数据结构
`runtime/sizeclasses.go` 文件中定义了小对象的相关信息：
```
// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0     87.50%
//     2         16        8192      512           0     43.75%
//     3         24        8192      341           8     29.24%
//     4         32        8192      256           0     21.88%
...............................................................
//    64      24576       24576        1           0     11.45%
//    65      27264       81920        3         128     10.00%
//    66      28672       57344        2           0      4.91%
//    67      32768       32768        1           0     12.50%
```
`class` 表示对象的 `id`，后面内存都需要使用对应的 `class` 来进行内存的管理。  
`bytes/obj` 表示对象的大小，单位为字节  
`bytes/span` 表示存储此对象的内存块的大小  
`objects` 表示这个内存块能放多少个对象。相当于这个内存块有这么多个槽可以放对象  
`tail waste` 表示这个内存块装满对象后还有多少内存没有被使用，被浪费了  
`max waste` TODO:最大的浪费率，`maxWaste := float64((c.size-prevSize-1)*objects+tailWaste) / float64(spanSize)`. 计算方式有点不懂。  
以 `class` 为4举例：分配内存为32字节大小的对象，内存管理中内存块大小为8192字节，可以分配256个对象。对于小对象的内存管理就是基于此表来进行的。  

### mspan
`mspan` 就是上表中的 `span` 内存块，用来管理一个内存块，可以看到不同 `class` 的内存块大小不同，一个内存块中的基本代为是 `page` ，一个页大小为8k，因此不同类型的内存块可能对应一个或多个页。
```
type mspan struct {
	next *mspan     // next
	prev *mspan     // pre

	startAddr uintptr // 内存块的起始地址
	npages    uintptr // 内存块包含的page数量

	manualFreeList gclinkptr // list of free objects in mSpanManual spans
	freeindex uintptr  // 下一次开始查找空位的槽
	nelems uintptr // 内存块中的槽数，即可以存放的对象的数量

	// Cache of the allocBits at freeindex. allocCache is shifted
	// such that the lowest bit corresponds to the bit freeindex.
	// allocCache holds the complement of allocBits, thus allowing
	// ctz (count trailing zero) to use it directly.
	// allocCache may contain bits beyond s.nelems; the caller must ignore
	// these.
	allocCache uint64

	allocBits  *gcBits // 内存分配详情
	gcmarkBits *gcBits // gc标记

	// sweep generation:
	// if sweepgen == h->sweepgen - 2, the span needs sweeping
	// if sweepgen == h->sweepgen - 1, the span is currently being swept
	// if sweepgen == h->sweepgen, the span is swept and ready to use
	// if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
	// if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
	// h->sweepgen is incremented by 2 after every GC
	sweepgen    uint32

	divMul      uint16        // for divide by elemsize - divMagic.mul
	baseMask    uint16        // if non-0, elemsize is a power of 2, & this will get object allocation base
	allocCount  uint16        // 已经分配的对象数量
	spanclass   spanClass     // class对象id
	state       mSpanStateBox // mSpanInUse etc; accessed atomically (get/set methods)
	needzero    uint8         // needs to be zeroed before allocation
	divShift    uint8         // for divide by elemsize - divMagic.shift
	divShift2   uint8         // for divide by elemsize - divMagic.shift2
	elemsize    uintptr       // class对象大小或块内存大小（class == 0）
	limit       uintptr       // 内存块尾地址 = startAddr + npages*8k
	speciallock mutex         // guards specials list
	specials    *special      // linked list of special records sorted by offset.
}
```
