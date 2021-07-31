# memory

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
    // if sweepgen == h->sweepgen - 2, the span needs sweeping
    // if sweepgen == h->sweepgen - 1, the span is currently being swept
    // if sweepgen == h->sweepgen, the span is swept and ready to use
    // if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
    // if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
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
    scanAlloc  uintptr // bytes of scannable heap allocated

    // Allocator cache for tiny objects w/o pointers.
    // See "Tiny allocator" comment in malloc.go.

    // tiny points to the beginning of the current tiny block, or
    // nil if there is no current tiny block.
    //
    // tiny is a heap pointer. Since mcache is in non-GC'd memory,
    // we handle it by clearing it in releaseAll during mark
    // termination.
    //
    // tinyAllocs is the number of tiny allocations performed
    // by the P that owns this mcache.
    tiny       uintptr
    tinyoffset uintptr
    tinyAllocs uintptr

    // The rest is not accessed on every malloc.

    alloc [numSpanClasses]*mspan // p本地管理的mspan，大小168

    stackcache [_NumStackOrders]stackfreelist

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
以上函数解释了为什么会是2倍大小，同一个小对象 `class` 分为 `scan` 和 `noscan` 两种， `noscan := typ == nil || typ.ptrdata == 0` 也就是说如果分配内存类型为空或类型中不包含指针的，都是 `noscan`。这里先放下为什么要区分，后面会看到。根据算法这里数组就是 `scan class1` `noscan class1` `scan class2` `noscan class2`......这样的布局。
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
