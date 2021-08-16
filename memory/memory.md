# memory
TODO:
1. mspan如何管理内存的 (down)
2. 关于mspan结构体的分配
3. span内存块是如何管理的
4. span内存块是如何分配的
5. mcentral是什么
6. L1 L2 内存分级是什么
7. 页是如何申请 和 管理的

## mspan
`mspan` 是小对象分配内存的管理者。  

### 数据结构
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
`max waste` 最大的浪费率，比如我们存储25字节大小，使用内存即为32字节。最大浪费率即为`1-24/32=0.21875`  
举例：当分配内存为(24,32]字节大小的对象时，对应 `class` 为4，内存管理中内存块大小为8192字节，可以分配256个对象。对于小对象的内存管理就是基于此表来进行的。  
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
    // if sweepgen == h->sweepgen - 2, 这个span需要清扫
    // if sweepgen == h->sweepgen - 1, 这个span正在清扫
    // if sweepgen == h->sweepgen, 这个span已经清扫完毕，可以使用
    // if sweepgen == h->sweepgen + 1, 这个span在清扫开始前被mcache缓存了，需要清扫
    // if sweepgen == h->sweepgen + 3, 这个span已经被清扫，而且被mcache缓存了
    // h->sweepgen is incremented by 2 after every GC
    sweepgen    uint32

    divMul      uint16        // 根据地址算槽号用
    baseMask    uint16        // 根据地址算槽号用
    allocCount  uint16        // 已经分配的对象数量
    spanclass   spanClass     // class对象id
    state       mSpanStateBox // mSpanInUse etc; accessed atomically (get/set methods)
    needzero    uint8         // 分配内存的时候是否需要将内存清零
    divShift    uint8         // 根据地址算槽号用
    divShift2   uint8         // 根据地址算槽号用
    elemsize    uintptr       // class对象大小或块内存大小（class == 0）
    limit       uintptr       // 内存块尾地址 = startAddr + npages*8k
    speciallock mutex         // guards specials list
    specials    *special      // linked list of special records sorted by offset.
}
```
一个 `class` 对应的内存块被分为若干个槽，以 `class == 1` 即分配8字节对象为例，一个内存块 `span` 可以分配 `8192/8=1024` 个对象，那么就有连续的 `1024` 个槽可以使用。当我们要分配内存的时候，就需要从这 `1024` 个槽中找到空闲槽，此时就需要一个算法保证查找空槽速度快，并且不会遗漏空槽，否则的话就浪费了内存。  

### 槽的管理
我们很容易想到管理内存状态的方法，使用数组表示状态，假设起始地址为 `0` ，维护一个大小为 `1024` 的数组 `array`，数组下标对应槽的位置，例如当 `array[10]=0` 时表示槽 `10` 即内存块中 `80-88` 字节的内存尚未使用，分配给调用方后 `array[10]=1` 标识这块内存已经被使用。查找空槽只需要从头遍历这个数组即可。  

`mspan` 实现的算法： 首先位图 `allocBits` 的每一个 `bit` 表示一个槽的状态，初始化为 `0` 表示所有槽都是空的，那么是如何表示桶的顺序呢？
```
// n为桶号
func (b *gcBits) bitp(n uintptr) (bytep *uint8, mask uint8) {
    return b.bytep(n / 8), 1 << (n % 8) // 注意这个左移
}
```
**从上面代码可以看到第一个字节表示 `[0,8)` 号桶，第二个字节就是 `[8,16)`，以此类推。余数左移表示某个桶在这个字节中的位置，比如`0`号桶在这个二进制中是`xxxx xxxx1`，最右边的`bit`表示 `0` 号桶，`7` 号桶这个算法就是 `1xxx xxx`，也就是说桶的顺序是在字节序中从右往左的**  
`allocCache` 所有 `bit位` 初始化为 `1` 表示所有槽都可以存放，是位图的取反，相应的 `freeindex` 初始化为 `0` 表示起始位置就有空位。每次查找空槽会检查 `allocCache` 低位最近的 `1` 的位置，加上 `freeindex` 即为下一个空槽位置。这样通过位标记以及位运算能快速找到空的槽来存放数据：
```
func nextFreeFast(s *mspan) gclinkptr {
    // 获取allocCache从右往左的连续的0的个数，直到找到1
    theBit := sys.Ctz64(s.allocCache) // Is there a free object in the allocCache?
    if theBit < 64 {
        // 通过freeindex快速定位空槽位置
        result := s.freeindex + uintptr(theBit)
        if result < s.nelems {
            freeidx := result + 1
            // 因为allocCache是64bit，只能表示64个槽的情况
            if freeidx%64 == 0 && freeidx != s.nelems {
                return 0
            }
            // allocCache将尾数的0 右移消掉
            s.allocCache >>= uint(theBit + 1)
            // 赋值新的freeindex
            s.freeindex = freeidx
            s.allocCount++
            return gclinkptr(result*s.elemsize + s.base())
        }
    }
    return 0
}
```
以上就是快速查找空槽算法，当我们分配了`63`个槽的时候，因为 `allocCache` 只有 `64bit` 的原因，会触发内存重新填充 `allocCache` 的逻辑：
```
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
	s = c.alloc[spc]
	shouldhelpgc = false
    // 重新填充allocCache
	freeIndex := s.nextFreeIndex()
	if freeIndex == s.nelems {
		// mspan真的满了
		if uintptr(s.allocCount) != s.nelems {
			println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
			throw("s.allocCount != s.nelems && freeIndex == s.nelems")
		}
        // mcache根据spanClass 重新请求mspan
		c.refill(spc)
		shouldhelpgc = true
        // 这里就是新的mspan了
		s = c.alloc[spc]

		freeIndex = s.nextFreeIndex()
	}
    // 检查新的mspan对不对
	if freeIndex >= s.nelems {
		throw("freeIndex is not valid")
	}
    // 返回新的mspan中的槽的地址
	v = gclinkptr(freeIndex*s.elemsize + s.base())
	s.allocCount++
	if uintptr(s.allocCount) > s.nelems {
		println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
		throw("s.allocCount > s.nelems")
	}
	return
}

func (s *mspan) nextFreeIndex() uintptr {
    sfreeindex := s.freeindex
    snelems := s.nelems
    if sfreeindex == snelems {
        // 说明满了 freeindex 移动到了 nelems
        return sfreeindex
    }
    if sfreeindex > snelems {
        throw("s.freeindex > s.nelems")
    }

    aCache := s.allocCache
    // 如果allocCache表示的槽已经全部被分配
    bitIndex := sys.Ctz64(aCache)
    for bitIndex == 64 {
        // 对 sfreeindex 做一个64的对齐
        sfreeindex = (sfreeindex + 64) &^ (64 - 1)
        if sfreeindex >= snelems {
            // 这个span满了
            s.freeindex = snelems
            return snelems
        }
        // 如果span没满，那么1bit表示一个槽，说明这个span多于64个槽。
        // whichByte是获取位图中的第几个字节。
        whichByte := sfreeindex / 8
        // 重新填充allocCache
        s.refillAllocCache(whichByte)
        aCache = s.allocCache
        // 再次检查新填充的64个槽
        bitIndex = sys.Ctz64(aCache)
    }
    result := sfreeindex + uintptr(bitIndex)
    if result >= snelems {
        // 检查新找到的槽位是否越界
        s.freeindex = snelems
        return snelems
    }
    // 如果没有，则更新allocCache，说明一个空槽被找到了
    s.allocCache >>= uint(bitIndex + 1)
    sfreeindex = result + 1
    // 如果sfreeindex是64的倍数，说明allocCache用完了，再次填充allocCache
    if sfreeindex%64 == 0 && sfreeindex != snelems {
        whichByte := sfreeindex / 8
        // 填充其实就是使用位图 allocBits中的下一个64bit 取反 赋值给allocCache
        // 因为在位图中 1 表示已分配， 0表示未分配 allocCache中1表示未分配
        s.refillAllocCache(whichByte)
    }
    s.freeindex = sfreeindex
    return result
}

func (s *mspan) refillAllocCache(whichByte uintptr) {
    // 越过wichByte 即8字节的倍数
    bytes := (*[8]uint8)(unsafe.Pointer(s.allocBits.bytep(whichByte)))
    aCache := uint64(0)
    aCache |= uint64(bytes[0])
    aCache |= uint64(bytes[1]) << (1 * 8)
    aCache |= uint64(bytes[2]) << (2 * 8)
    aCache |= uint64(bytes[3]) << (3 * 8)
    aCache |= uint64(bytes[4]) << (4 * 8)
    aCache |= uint64(bytes[5]) << (5 * 8)
    aCache |= uint64(bytes[6]) << (6 * 8)
    aCache |= uint64(bytes[7]) << (7 * 8)
    // 取反
    s.allocCache = ^aCache
}
```
上面的算法通过结合几个变量实现了 `O(1)` 的空槽寻找， **主要思想就是拓印一份位图（与数组思想相同）的一部分（以`64bit`为一组），这个拓印出来的位图是原位图的取反。**   
举一个例子：
起始原位图`allocBits`为：`0101 0010` ，那么拓印 `allocCache` 就是 `1010 1101` ，此时 `freeindex` 为`0`。根据 `allocBits` 的表示方法，初始状态为 `1`号、`4`号、`6`号桶表示已经被分配。  
当分配一个槽的时候，计算 `allocCache` 后面有几个零，此时为 `0`，那么 `freeindex + 0` 即`0`号为可以分配给用户的槽。分配后 `allocBits` 为 `0101 0011`，而 `allocCache` 右移`1`位后为 `0101 0110`，`freeindex` 为 `1`。再次分配，`allocCache` 后面有`1`个零，则 `freeindex + 1` 即`2`号槽可以分配给用户，可以看到跳过了本身就有数据的`1`号桶。  
**通过将位图单个字节中槽从右往左排列，对拓印进行右移来表示当前槽的状态，实现快速查找定位空槽的算法。位运算+槽倒排完美实现了快速查找而不用去遍历位图**  
**tips：分配内存的时候并没有实时修改`allocBits`位图，详情需要TODO：gc**

### mspan的其它
`mspan` 通过上一节的管理算法提供了一些功能函数，例如：根据内存地址查找槽号，根据槽号获取槽的状态位并可以设置是否分配，根据槽号获取槽的分配状态等等。
```
// p是对象地址
func (s *mspan) objIndex(p uintptr) uintptr {
    byteOffset := p - s.base()
    if byteOffset == 0 {
        return 0
    }
    if s.baseMask != 0 {
        // s.baseMask is non-0, elemsize is a power of two, so shift by s.divShift
        return byteOffset >> s.divShift
    }
    return uintptr(((uint64(byteOffset) >> s.divShift) * uint64(s.divMul)) >> s.divShift2)
}
```
简单提一下根据对象地址返回槽号，这里的 `baseMask`、`divShift` 等等也都是预先定义好的，主要描述了对象在`span`中存在的布局，根据上面的算法可以根据某类对象的地址算出槽号。更多的就不深入了，了解即可。

## mcache
`mspan` 对于小对象的内存管理搞明白了，接下来是对于 `mspan` 的管理，每一个 `mspan` 对应一种小对象，在处理器 `P` 中包含一个 `mcache` 的结构体来管理各个 `mspan`，这样当我们分配内存的时候，只需要通过 `P` 来进行分配内存，而不用加锁。

### 数据结构
```
type mcache struct {
    // The following members are accessed on every malloc,
    // so they are grouped here for better caching.
    nextSample uintptr // 分配这么多内存后 触发 内存采样 这个数字是一个随机数字符合泊松分布（注释说的）
    scanAlloc  uintptr // 需要扫描的内存的大小

    // 微小对象分配相关
    tiny       uintptr // tiny块 16字节 从class2 noscan 的span中取的一个槽
    tinyoffset uintptr // 当前tiny已被分配的内存位移
    tinyAllocs uintptr // 微小对象分配次数

    // The rest is not accessed on every malloc.

    alloc [numSpanClasses]*mspan // p本地管理的mspan，大小136

    stackcache [_NumStackOrders]stackfreelist // 栈分配相关

    // flushGen indicates the sweepgen during which this mcache
    // was last flushed. If flushGen != mheap_.sweepgen, the spans
    // in this mcache are stale and need to the flushed so they
    // can be swept. This is done in acquirep.
    flushGen uint32
}
```
`alloc` 字段就是管理所有本处理器的 `mspan`。这个数组的大小是136，因为我们小对象是从1开始，总共67个，大小为68，那么这个数组就存储了两倍的小对象对应的内存块。
```
func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
    return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}
```
以上函数解释了为什么会是2倍大小，同一个小对象 `class` 分为 `scan` 和 `noscan` 两种， `noscan := typ == nil || typ.ptrdata == 0` 也就是说如果分配内存类型为空或类型中不包含指针的，都是 `noscan`。这里先放下为什么要区分，后面会看到。根据算法这里数组就是 `scan class1` `noscan class1` `scan class2` `noscan class2`......这样的布局。这样 `noscan` 块中的内存无需进行扫描。一个 `sizeclass` 对应两个 `spanclass` ，对应关系为 `spanclass = sizeclass*2 + 0或1`
```
// 在P初始化的时候会对mcahe进行初始化
func allocmcache() *mcache {
    var c *mcache
    systemstack(func() {
        lock(&mheap_.lock)
        // 全局mheap上分配一个sizeof(mcache)大小的内存
        c = (*mcache)(mheap_.cachealloc.alloc())
        c.flushGen = mheap_.sweepgen
        unlock(&mheap_.lock)
    })
    for i := range c.alloc {
        // 初始化所有mspan 到一个空的mspan上
        c.alloc[i] = &emptymspan
    }
    c.nextSample = nextSample()
    return c
}
```
`mspan` 的创建会在创建对应 `class` 的时候判断 `mcache` 上的对应的 `mspan` 是否为空或者已满。然后去 `mcentral` 中请求一个 `span`  
```
// 上面mcache没有mspan的时候，会调用refill重新申请一个mspan
func (c *mcache) refill(spc spanClass) {
	s := c.alloc[spc]

	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// Mark this span as no longer cached.
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		mheap_.central[spc].mcentral.uncacheSpan(s)
	}

	// 从mcentral中获取一个新的mspan
	s = mheap_.central[spc].mcentral.cacheSpan()
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// 此次gc不清扫，下次gc再清扫
	s.sweepgen = mheap_.sweepgen + 3

	// 假设新取的mspan中的空闲槽都会被分配出去，等还回来的时候再调整
	stats := memstats.heapStats.acquire()
	atomic.Xadduintptr(&stats.smallAllocCount[spc.sizeclass()], uintptr(s.nelems)-uintptr(s.allocCount))
	memstats.heapStats.release()

	// heap_live同上
	usedBytes := uintptr(s.allocCount) * s.elemsize
	atomic.Xadd64(&memstats.heap_live, int64(s.npages*pageSize)-int64(usedBytes))

	// 如果是微小对象的话 把已经分配的微小对象数量刷进内存状态中
	if spc == tinySpanClass {
		atomic.Xadd64(&memstats.tinyallocs, int64(c.tinyAllocs))
		c.tinyAllocs = 0
	}

	// 更新堆扫描size
	atomic.Xadd64(&memstats.heap_scan, int64(c.scanAlloc))
	c.scanAlloc = 0

	if trace.enabled {
		// heap_live changed.
		traceHeapAlloc()
	}
	if gcBlackenEnabled != 0 {
		// heap_live and heap_scan changed.
		gcController.revise()
	}
    // 更新mcache中的mspan
	c.alloc[spc] = s
}
```

### 微小对象分配
`mcache` 对 `mspan` 仅仅是持有独一份的指针，防止多线程并发竞争需要加锁。  
值得一提的是 `mcache` 有一个微小对象分配逻辑。在 `mallocgc` 函数中可以看到微小对象分配的详细情况，这里简单介绍一下：  
1. 当分配微小对象时（小于16字节并且为`noscan`），检查 `tiny` 字段，如果为空，去 `class2 noscan` 找一个 `mspan` 即16字节大小的内存块给tiny，并将所需的内存 `size` 分配给用户，此时 `tinyoffset` 为已分配的内存大小 `size`  
2. 若 `tiny` 字段不为空，则根据要分配的内存大小 `size`，对 `tinyoffset` 进行一个内存对齐，如果剩下的内存够放，则放进去，否则重复第一步

### 栈内存分配
TODO：后面再看

## mcentral
`mcentral` 是更上一层的管理 `mspan` 的数据结构，也是 `mcache` 中的内存块的来源。
```
type mcentral struct {
	spanclass spanClass // 这里可以看到 mcentral 对应 mcache 的136个span，通过spanclass来进行管理
	partial [2]spanSet // list of spans with a free object 一个是已经清扫的spanset 一个是尚未清扫的spanset
	full    [2]spanSet // list of spans with no free objects
}

type spanSet struct {
	spineLock mutex
	spine     unsafe.Pointer // *[N]*spanSetBlock, accessed atomically 存储的是切片指针，切片中是spanSetBlock的指针
	spineLen  uintptr        // Spine array length, 数组长度
	spineCap  uintptr        // Spine array cap, 数组容量

	index headTailIndex    // spine的下标
}

type spanSetBlock struct {
	lfnode
	popped uint32

	// 大小4k 512*8字节指针长度
	spans [spanSetBlockEntries]*mspan  // 512长度的mspan数组
}
```
`mcentral` 也是通过 `spanClass` 分类管理，每一个 `mcentral` 中都存储了多个 `mspan`。这些 `mspan` 被分为4块，每一块就是一个 `spanSet` ，分别为：有空闲的并已经清扫的、有空闲的并尚未清扫的、无空闲的并已经清扫的，无空闲的并尚未清扫的。这里对 `spanSet` 的存取是通过一下函数：
```
func (c *mcentral) partialUnswept(sweepgen uint32) *spanSet {
	return &c.partial[1-sweepgen/2%2]
}

func (c *mcentral) partialSwept(sweepgen uint32) *spanSet {
	return &c.partial[sweepgen/2%2]
}
```
每次 `mheap.sweepgen` 在gc的时候+2，因此每次gc后，清扫与未清扫的 `spanSet` 自然交换。

### 获取mspan
```
func (c *mcentral) cacheSpan() *mspan {
	// Deduct credit for this span allocation and sweep if necessary.
	spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
	deductSweepCredit(spanBytes, 0)

	sg := mheap_.sweepgen

	traceDone := false
	if trace.enabled {
		traceGCSweepStart()
	}

	// 这里不能无限制的找free span，如果超过100个 新分配一个span
	spanBudget := 100

	var s *mspan

	// 去已经清扫的 有空闲的 spanset里面取一个
	if s = c.partialSwept(sg).pop(); s != nil {
		goto havespan
	}

	// 尝试去未清扫的 有空闲的 spanset里面取一个
	for ; spanBudget >= 0; spanBudget-- {
		s = c.partialUnswept(sg).pop()
		if s == nil {
			break
		}
		if atomic.Load(&s.sweepgen) == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// 获取span的所有权，进行清扫并使用
			s.sweep(true)
			goto havespan
		}
		// 说明有其它人获取了这个span的所有权正在清扫
	}
	// 尝试去没有空闲的 未清扫的 spanset里面取一个
	for ; spanBudget >= 0; spanBudget-- {
		s = c.fullUnswept(sg).pop()
		if s == nil {
			break
		}
		if atomic.Load(&s.sweepgen) == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// 与上面相同
			s.sweep(true)
			// 检查清扫结束后 是否有空槽
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				goto havespan
			}
			// 清扫结束后还是没有空槽 放入没有空闲的 已经清扫的 spanset
			c.fullSwept(sg).push(s)
		}
		// 跟partialUnswept相同，没有成功拿到所有权
	}
	if trace.enabled {
		traceGCSweepDone()
		traceDone = true
	}

	// mcentral没找到空闲的（没有空闲或者找了100个span了）  分配一个新的span
	s = c.grow()
	if s == nil {
        // 没内存了？外面会throw
		return nil
	}

	// 这个label表示已经有空闲槽位的span了
havespan:
	if trace.enabled && !traceDone {
		traceGCSweepDone()
	}
    // 检查span的状态
	n := int(s.nelems) - int(s.allocCount)
	if n == 0 || s.freeindex == s.nelems || uintptr(s.allocCount) == s.nelems {
		throw("span has no free objects")
	}
    // 重新填充mspan的allocCahce
	freeByteBase := s.freeindex &^ (64 - 1)
	whichByte := freeByteBase / 8
	// Init alloc bits cache.
	s.refillAllocCache(whichByte)
	s.allocCache >>= s.freeindex % 64

	return s
}
```

### 回收mspan
```
func (c *mcentral) uncacheSpan(s *mspan) {
	if s.allocCount == 0 {
		throw("uncaching span but s.allocCount == 0")
	}

	sg := mheap_.sweepgen
	stale := s.sweepgen == sg+1

	// 修正span的清扫标志
	if stale {
        // 表示此span需要清扫
		atomic.Store(&s.sweepgen, sg-1)
	} else {
		// Indicate that s is no longer cached.
		atomic.Store(&s.sweepgen, sg)
	}

	// span放在合适的spanSet中
	if stale {
		// 清扫span
		s.sweep(false)
	} else {
        // 此次gc不需要清扫，放入相应队列，下次gc清扫
		if int(s.nelems)-int(s.allocCount) > 0 {
			// Put it back on the partial swept list.
			c.partialSwept(sg).push(s)
		} else {
			// There's no free space and it's not stale, so put it on the
			// full swept list.
			c.fullSwept(sg).push(s)
		}
	}
}
```

### 申请mspan
```
func (c *mcentral) grow() *mspan {
	npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])
	size := uintptr(class_to_size[c.spanclass.sizeclass()])

    // 从堆上分配
	s := mheap_.alloc(npages, c.spanclass, true)
	if s == nil {
		return nil
	}

	// Use division by multiplication and shifts to quickly compute:
	// n := (npages << _PageShift) / size
	n := (npages << _PageShift) >> s.divShift * uintptr(s.divMul) >> s.divShift2
	s.limit = s.base() + size*n
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
```

#### spanSet
```
func (b *spanSet) push(s *mspan) {
	// 根据index，找到要存的位置
	cursor := uintptr(b.index.incTail().tail() - 1)
    // top是存在spanSet中的第几个spanSetBlock，bottom是存在spanSetBlock数组中的第几个位置
	top, bottom := cursor/spanSetBlockEntries, cursor%spanSetBlockEntries
    // 获取当前spanSet的长度
	spineLen := atomic.Loaduintptr(&b.spineLen)
	var block *spanSetBlock
retry:
	if top < spineLen {
        // 长度够 直接从spanSet切片spine中的对应spanSetBlock拿出来
		spine := atomic.Loadp(unsafe.Pointer(&b.spine))
		blockp := add(spine, sys.PtrSize*top)
		block = (*spanSetBlock)(atomic.Loadp(blockp))
	} else {
		lock(&b.spineLock)
		// 二次检查 double check
		spineLen = atomic.Loaduintptr(&b.spineLen)
		if top < spineLen {
			unlock(&b.spineLock)
			goto retry
		}
        // 是否需要扩容 与切片扩容类似
		if spineLen == b.spineCap {
			// Grow the spine.
			newCap := b.spineCap * 2
			if newCap == 0 {
				newCap = spanSetInitSpineCap
			}
            // 分配新的切片数组
			newSpine := persistentalloc(newCap*sys.PtrSize, cpu.CacheLineSize, &memstats.gcMiscSys)
			if b.spineCap != 0 {
				// 旧的切片数据移动到新的切片
				memmove(newSpine, b.spine, b.spineCap*sys.PtrSize)
			}
			// 更新spine
			atomic.StorepNoWB(unsafe.Pointer(&b.spine), newSpine)
			b.spineCap = newCap

            // 因为并发push，所以不能立即释放旧的spine
            // 根据注释 表示1TB的堆内存只用了2M，即便泄露也无所谓
            // 如果这个泄露是问题，那么可以在STW的时候进行释放旧的spine
		}

		// 分配一个spanSetBlock结构体
		block = spanSetBlockPool.alloc()

		// 加入到spine中
		blockp := add(b.spine, sys.PtrSize*top)
		// 更新即可
		atomic.StorepNoWB(blockp, unsafe.Pointer(block))
		atomic.Storeuintptr(&b.spineLen, spineLen+1)
		unlock(&b.spineLock)
	}

	// span放到spanSet中的某个spanSetBlock中对应的位置
	atomic.StorepNoWB(unsafe.Pointer(&block.spans[bottom]), unsafe.Pointer(s))
}
```
一个 `spanSetBlock` 有一个512大小（占用内存为512*8=4k）的数组，可以表示起码（按一个`mspan`只有一个`page`来算）4M的内存。  
一个 `spanSet` 按照长度或容量256大小（占用内存256*8=2k字节）计算，可以表示起码1G的内存（256个`spanSetBlock` * 4M）。
```
func (b *spanSet) pop() *mspan {
	var head, tail uint32
claimLoop:
	for {
		headtail := b.index.load()
		head, tail = headtail.split()
		if head >= tail {
			// spanSet is nil
			return nil
		}
		// 判断head还在数组里面 这里是再次检查，因为可能spine正在扩容？
		spineLen := atomic.Loaduintptr(&b.spineLen)
		if spineLen <= uintptr(head)/spanSetBlockEntries {
			return nil
		}
		// cas修改index，修改成功就可以pop
		want := head
		for want == head {
			if b.index.cas(headtail, makeHeadTailIndex(want+1, tail)) {
				break claimLoop
			}
			headtail = b.index.load()
			head, tail = headtail.split()
		}
	}
	top, bottom := head/spanSetBlockEntries, head%spanSetBlockEntries

	// 拿到第几个spanSetBlock 和 对应其中的位置
	spine := atomic.Loadp(unsafe.Pointer(&b.spine))
	blockp := add(spine, sys.PtrSize*uintptr(top))

	// 取出block
	block := (*spanSetBlock)(atomic.Loadp(blockp))
	s := (*mspan)(atomic.Loadp(unsafe.Pointer(&block.spans[bottom])))
	for s == nil {
		// 再次尝试 竞争？
		s = (*mspan)(atomic.Loadp(unsafe.Pointer(&block.spans[bottom])))
	}
	// 取出后的位置赋值为nil
	atomic.StorepNoWB(unsafe.Pointer(&block.spans[bottom]), nil)
    // spanSetBlock 被pop了512次，说明已经没有数据了，清空
	if atomic.Xadd(&block.popped, 1) == spanSetBlockEntries {
		// Clear the block's pointer.
		atomic.StorepNoWB(blockp, nil)

		// Return the block to the block pool.
		spanSetBlockPool.free(block)
	}
	return s
}
```
从这两个函数可以看出，`spanSet` 被分为一个个块，每一个块存储512个 `mspan`，其本质上是一个支持并发的无锁队列，从头取，从尾存。

## mheap
```
type mheap struct {
	// lock must only be acquired on the system stack, otherwise a g
	// could self-deadlock if its stack grows with the lock held.
	lock      mutex
	pages     pageAlloc // page allocation data structure
	sweepgen  uint32    // 清扫的代
	sweepdone uint32    // all spans are swept
	sweepers  uint32    // number of active sweepone calls

	// allspans is a slice of all mspans ever created. Each mspan
	// appears exactly once.
	//
	// The memory for allspans is manually managed and can be
	// reallocated and move as the heap grows.
	//
	// In general, allspans is protected by mheap_.lock, which
	// prevents concurrent access as well as freeing the backing
	// store. Accesses during STW might not hold the lock, but
	// must ensure that allocation cannot happen around the
	// access (since that may free the backing store).
	allspans []*mspan // all spans out there

	_ uint32 // align uint64 fields on 32-bit for atomics

	// Proportional sweep
	//
	// These parameters represent a linear function from heap_live
	// to page sweep count. The proportional sweep system works to
	// stay in the black by keeping the current page sweep count
	// above this line at the current heap_live.
	//
	// The line has slope sweepPagesPerByte and passes through a
	// basis point at (sweepHeapLiveBasis, pagesSweptBasis). At
	// any given time, the system is at (memstats.heap_live,
	// pagesSwept) in this space.
	//
	// It's important that the line pass through a point we
	// control rather than simply starting at a (0,0) origin
	// because that lets us adjust sweep pacing at any time while
	// accounting for current progress. If we could only adjust
	// the slope, it would create a discontinuity in debt if any
	// progress has already been made.
	pagesInUse         uint64  // pages of spans in stats mSpanInUse; updated atomically
	pagesSwept         uint64  // pages swept this cycle; updated atomically
	pagesSweptBasis    uint64  // pagesSwept to use as the origin of the sweep ratio; updated atomically
	sweepHeapLiveBasis uint64  // value of heap_live to use as the origin of sweep ratio; written with lock, read without
	sweepPagesPerByte  float64 // proportional sweep ratio; written with lock, read without
	// TODO(austin): pagesInUse should be a uintptr, but the 386
	// compiler can't 8-byte align fields.

	// scavengeGoal is the amount of total retained heap memory (measured by
	// heapRetained) that the runtime will try to maintain by returning memory
	// to the OS.
	scavengeGoal uint64

	// Page reclaimer state

	// reclaimIndex is the page index in allArenas of next page to
	// reclaim. Specifically, it refers to page (i %
	// pagesPerArena) of arena allArenas[i / pagesPerArena].
	//
	// If this is >= 1<<63, the page reclaimer is done scanning
	// the page marks.
	//
	// This is accessed atomically.
	reclaimIndex uint64
	// reclaimCredit is spare credit for extra pages swept. Since
	// the page reclaimer works in large chunks, it may reclaim
	// more than requested. Any spare pages released go to this
	// credit pool.
	//
	// This is accessed atomically.
	reclaimCredit uintptr

	// 所有的arenas
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	// 64位未使用
	heapArenaAlloc linearAlloc

	// 向操作系统申请内存的时候的起始地址
	arenaHints *arenaHint

	// 64位未使用
	arena linearAlloc

	// allArenas is the arenaIndex of every mapped arena. This can
	// be used to iterate through the address space.
	//
	// Access is protected by mheap_.lock. However, since this is
	// append-only and old backing arrays are never freed, it is
	// safe to acquire mheap_.lock, copy the slice header, and
	// then release mheap_.lock.
	allArenas []arenaIdx

	// sweepArenas is a snapshot of allArenas taken at the
	// beginning of the sweep cycle. This can be read safely by
	// simply blocking GC (by disabling preemption).
	sweepArenas []arenaIdx

	// markArenas is a snapshot of allArenas taken at the beginning
	// of the mark cycle. Because allArenas is append-only, neither
	// this slice nor its contents will change during the mark, so
	// it can be read safely.
	markArenas []arenaIdx

	// curArena is the arena that the heap is currently growing
	// into. This should always be physPageSize-aligned.
	curArena struct {
		base, end uintptr
	}

	_ uint32 // ensure 64-bit alignment of central

	// mcentral通过 spanclass管理
	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}

    // 以下是各种分配器，缓存了mspan mcache等对象，进行内存复用
	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	specialfinalizeralloc fixalloc // allocator for specialfinalizer*
	specialprofilealloc   fixalloc // allocator for specialprofile*
	speciallock           mutex    // lock for special record allocators.
	arenaHintAlloc        fixalloc // allocator for arenaHints

	unused *specialfinalizer // never set, just here to force the specialfinalizer type into DWARF
}
```
`mheap` 是全局变量，管理进程生命周期中的所有内存  
```
// mheap 分配span
func (h *mheap) allocSpan(npages uintptr, typ spanAllocType, spanclass spanClass) (s *mspan) {
	// Function-global state.
	gp := getg()
	base, scav := uintptr(0), uintptr(0)

	// 有些系统在分配栈的内存的时候需要跟物理页对齐
	needPhysPageAlign := physPageAlignedStacks && typ == spanAllocStack && pageSize < physPageSize

	// 不需要跟物理页面对齐的 并且小于128k 尝试使用P的pageCache来分配
	pp := gp.m.p.ptr()
	if !needPhysPageAlign && pp != nil && npages < pageCachePages/4 {
		c := &pp.pcache

		// 缓存为空，分配一个
		if c.empty() {
			lock(&h.lock)
			*c = h.pages.allocToCache()
			unlock(&h.lock)
		}

		// 从P的pageCache中分配一些页
		base, scav = c.alloc(npages)
		if base != 0 {
            // 从mspan的缓存中取一个mspan对象（上面是分配内存，这里仅仅是分配一个mspan对象的内存）
			s = h.tryAllocMSpan()
			if s != nil {
				goto HaveSpan
			}
		}
	}

	lock(&h.lock)

	if needPhysPageAlign {
		// Overallocate by a physical page to allow for later alignment.
		npages += physPageSize / pageSize
	}

	if base == 0 {
		// P中的PageCache没有内存，从全局的页里面分配内存
		base, scav = h.pages.alloc(npages)
		if base == 0 {
            // 内存不够，runtime去系统里面申请一些页
			if !h.grow(npages) {
				unlock(&h.lock)
				return nil
			}
            // 分配span
			base, scav = h.pages.alloc(npages)
			if base == 0 {
				throw("grew heap, but no adequate free space found")
			}
		}
	}
	if s == nil {
		// 通过全局获取一个mspan对象
		s = h.allocMSpanLocked()
	}

	if needPhysPageAlign {
		allocBase, allocPages := base, npages
		base = alignUp(allocBase, physPageSize)
		npages -= physPageSize / pageSize

		// Return memory around the aligned allocation.
		spaceBefore := base - allocBase
		if spaceBefore > 0 {
			h.pages.free(allocBase, spaceBefore/pageSize)
		}
		spaceAfter := (allocPages-npages)*pageSize - spaceBefore
		if spaceAfter > 0 {
			h.pages.free(base+npages*pageSize, spaceAfter/pageSize)
		}
	}

	unlock(&h.lock)

HaveSpan:
	// 以下是初始化mspan
	s.init(base, npages)
	if h.allocNeedsZero(base, npages) {
		s.needzero = 1
	}
	nbytes := npages * pageSize
	if typ.manual() {
        // 非 堆类型 内存的初始化
		s.manualFreeList = 0
		s.nelems = 0
		s.limit = s.base() + s.npages*pageSize
		s.state.set(mSpanManual)
	} else {
		// 设置spanClass
		s.spanclass = spanclass
		if sizeclass := spanclass.sizeclass(); sizeclass == 0 {
            // 特殊处理 sizeclass = 0 TODO：好像没有看到用sizeclass为0的
			s.elemsize = nbytes
			s.nelems = 1

			s.divShift = 0
			s.divMul = 0
			s.divShift2 = 0
			s.baseMask = 0
		} else {
            // 根据对象大小预定义属性
			s.elemsize = uintptr(class_to_size[sizeclass])
			s.nelems = nbytes / s.elemsize

			m := &class_to_divmagic[sizeclass]
			s.divShift = m.shift
			s.divMul = m.mul
			s.divShift2 = m.shift2
			s.baseMask = m.baseMask
		}

		// 初始化内存标记相关属性
		s.freeindex = 0
		s.allocCache = ^uint64(0) // all 1s indicating all free.
		s.gcmarkBits = newMarkBits(s.nelems)
		s.allocBits = newAllocBits(s.nelems)

		// 更新sweepgen
		atomic.Store(&s.sweepgen, h.sweepgen)

		// 修改span状态 防止gc的时候扫描到的无效指针指向此span导致的竞争
        // 这样在任何需要的场景下进行判断状态来避免并发竞争
		s.state.set(mSpanInUse)
	}

	if scav != 0 {
		// sysUsed
		sysUsed(unsafe.Pointer(base), nbytes)
		atomic.Xadd64(&memstats.heap_released, -int64(scav))
	}
	// Update stats.
	if typ == spanAllocHeap {
		atomic.Xadd64(&memstats.heap_inuse, int64(nbytes))
	}
	if typ.manual() {
		// Manually managed memory doesn't count toward heap_sys.
		memstats.heap_sys.add(-int64(nbytes))
	}
	// Update consistent stats.
	stats := memstats.heapStats.acquire()
	atomic.Xaddint64(&stats.committed, int64(scav))
	atomic.Xaddint64(&stats.released, -int64(scav))
	switch typ {
	case spanAllocHeap:
		atomic.Xaddint64(&stats.inHeap, int64(nbytes))
	case spanAllocStack:
		atomic.Xaddint64(&stats.inStacks, int64(nbytes))
	case spanAllocPtrScalarBits:
		atomic.Xaddint64(&stats.inPtrScalarBits, int64(nbytes))
	case spanAllocWorkBuf:
		atomic.Xaddint64(&stats.inWorkBufs, int64(nbytes))
	}
	memstats.heapStats.release()

	// Publish the span in various locations.

	// span存到mheap的arena
	h.setSpans(s.base(), npages, s)

	if !typ.manual() {
		// 将span的标记信息加入到arena
		arena, pageIdx, pageMask := pageIndexOf(s.base())
		atomic.Or8(&arena.pageInUse[pageIdx], pageMask)

		// Update related page sweeper stats.
		atomic.Xadd64(&h.pagesInUse, int64(npages))
	}

	// Make sure the newly allocated span will be observed
	// by the GC before pointers into the span are published.
	publicationBarrier()

	return s
}
```
到此上一个主角 `mspan` 以及相关数据结构基本明白了，而 `mspan` 的来源也清晰了，是通过 `page` 来组成的，那么 `page` 是如何来的，以及是如何管理的。  
**因为硬件和操作系统限制，linux amd64 分配最大内存为 1<<48，golang将内存分为一个个arena，每一个大小为64M（1<<26），因此总共可以有1<<22个arena，每一个arena具有8192个页（8192*8k=64M），然后通过字典树来映射arenas**

`mheap`是以页为基本单位来管理内存的，最上面一层是 `arena`，数据结构是以下定义：
```
type heapArena struct {
	// 当前heapArena的位图表示 2M
	bitmap [heapArenaBitmapBytes]byte

	// 这里的mspan每一个都是存了一个页，是为了管理页，与上面的不同
    // pagesPerArena = 8192 因此一个heapArena管理的内存大小是8192*8192 = 64M
	spans [pagesPerArena]*mspan

	// 表示哪个页在使用，每一位表示一个页
	pageInUse [pagesPerArena / 8]uint8

	// 表示哪个页上面有对象被标记
	pageMarks [pagesPerArena / 8]uint8

	// pageSpecials is a bitmap that indicates which spans have
	// specials (finalizers or other).
	pageSpecials [pagesPerArena / 8]uint8

	// debug模式调试使用
	checkmarks *checkmarksMap

	// zeroedBase marks the first byte of the first page in this
	// arena which hasn't been used yet and is therefore already
	// zero. zeroedBase is relative to the arena base.
	// Increases monotonically until it hits heapArenaBytes.
	//
	// This field is sufficient to determine if an allocation
	// needs to be zeroed because the page allocator follows an
	// address-ordered first-fit policy.
	//
	// Read atomically and written with an atomic CAS.
	zeroedBase uintptr
}
```


```
func (h *mheap) grow(npage uintptr) bool {
	assertLockHeld(&h.lock)

	// 申请512整数倍的页数，也就是最少4M
	ask := alignUp(npage, pallocChunkPages) * pageSize

	totalGrowth := uintptr(0)
	end := h.curArena.base + ask
	nBase := alignUp(end, physPageSize)
	if nBase > h.curArena.end || /* overflow */ end < h.curArena.base {
        // 当前arena空间不够
		av, asize := h.sysAlloc(ask)
		if av == nil {
			print("runtime: out of memory: cannot allocate ", ask, "-byte block (", memstats.heap_sys, " in use)\n")
			return false
		}
        
		if uintptr(av) == h.curArena.end {
			// The new space is contiguous with the old
			// space, so just extend the current space.
			h.curArena.end = uintptr(av) + asize
		} else {
			// The new space is discontiguous. Track what
			// remains of the current space and switch to
			// the new space. This should be rare.
			if size := h.curArena.end - h.curArena.base; size != 0 {
				h.pages.grow(h.curArena.base, size)
				totalGrowth += size
			}
			// Switch to the new space.
			h.curArena.base = uintptr(av)
			h.curArena.end = uintptr(av) + asize
		}

		// The memory just allocated counts as both released
		// and idle, even though it's not yet backed by spans.
		//
		// The allocation is always aligned to the heap arena
		// size which is always > physPageSize, so its safe to
		// just add directly to heap_released.
		atomic.Xadd64(&memstats.heap_released, int64(asize))
		stats := memstats.heapStats.acquire()
		atomic.Xaddint64(&stats.released, int64(asize))
		memstats.heapStats.release()

		// Recalculate nBase.
		// We know this won't overflow, because sysAlloc returned
		// a valid region starting at h.curArena.base which is at
		// least ask bytes in size.
		nBase = alignUp(h.curArena.base+ask, physPageSize)
	}

	// Grow into the current arena.
	v := h.curArena.base
	h.curArena.base = nBase
	h.pages.grow(v, nBase-v)
	totalGrowth += nBase - v

	// We just caused a heap growth, so scavenge down what will soon be used.
	// By scavenging inline we deal with the failure to allocate out of
	// memory fragments by scavenging the memory fragments that are least
	// likely to be re-used.
	if retained := heapRetained(); retained+uint64(totalGrowth) > h.scavengeGoal {
		todo := totalGrowth
		if overage := uintptr(retained + uint64(totalGrowth) - h.scavengeGoal); todo > overage {
			todo = overage
		}
		h.pages.scavenge(todo, false)
	}
	return true
}
```
