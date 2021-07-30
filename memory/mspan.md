# mspan
`mspan` 是小对象分配内存的管理者。

## 数据结构
`runtime/sizeclasses.go` 文件中定义了小对象的内存管理相关信息：
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
举例：当分配内存为(24,32]字节大小的对象时，对应 `class` 为4，内存管理中内存块大小为8192字节，可以分配256个对象。对于小对象的内存管理就是基于此表来进行的。  

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

	allocCache uint64 // 内存分配信息的缓存

	allocBits  *gcBits // 位图 内存分配详情
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
一个 `class` 对应的内存块被分为若干个槽，以 `class == 1` 即分配8字节对象为例，一个内存块 `span` 可以分配 `8192/8=1024` 个对象，那么就有连续的 `1024` 个槽可以使用。当我们要分配内存的时候，就需要从这 `1024` 个槽中找到空闲槽，此时就需要一个算法保证查找空槽速度快，并且不会遗漏空槽，否则的话就浪费了内存。  
最方便的算法很容易想到，假设起始地址为 `0` ，维护一个大小为 `1024` 的数组 `array`，数组下标对应槽的位置，例如当 `array[10]=0` 时表示槽 `10` 即内存块中 `80-88` 字节的内存尚未使用，分配给调用方后 `array[10]=1` 标识这块内存已经被使用。查找空槽只需要从头遍历这个数组即可。  
// `mspan` 实现的算法是上述算法的扩展： 位图 `allocBits` 使用 `1bit` 标识一个槽，那么表示 `1024` 个槽只需要 `1024/8=128` 字节即可。
