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

## 内存分配
我们可以从 `slice`、`map`、`chan` 等实现中看到，申请内存使用的函数都是 `mallocgc` ：
```
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    if gcphase == _GCmarktermination {
        throw("mallocgc called with gcphase == _GCmarktermination")
    }

    if size == 0 {
        return unsafe.Pointer(&zerobase)
    }

    // Set mp.mallocing to keep from being preempted by GC.
    mp := acquirem()
    mp.mallocing = 1

    shouldhelpgc := false
    dataSize := size
    c := getMCache()
    if c == nil {
        throw("mallocgc called without a P or outside bootstrapping")
    }
    var span *mspan
    var x unsafe.Pointer
    noscan := typ == nil || typ.ptrdata == 0
    if size <= maxSmallSize {
        // 小于16字节 并且不存在指针
        if noscan && size < maxTinySize {
            // tiny 分配
            off := c.tinyoffset
            // 对tiny中已经存在的数据进行内存对齐，方便寻址
            if size&7 == 0 { // 如果要存的对象是8的倍数，以8字节对齐 以下对齐同理
                off = alignUp(off, 8)
            } else if sys.PtrSize == 4 && size == 12 {
                off = alignUp(off, 8)
            } else if size&3 == 0 {
                off = alignUp(off, 4)
            } else if size&1 == 0 {
                off = alignUp(off, 2)
            }
            // 对齐后的内存 + 要分配的内存是否大于16字节 不大于就能存进去了，相当于一个16字节的槽存了两个tiny对象
            if off+size <= maxTinySize && c.tiny != 0 {
                // 直接给x就行
                x = unsafe.Pointer(c.tiny + off)
                c.tinyoffset = off + size
                c.tinyAllocs++
                mp.mallocing = 0
                releasem(mp)
                return x
            }
            // tinySpanClass == 5
            span = c.alloc[tinySpanClass]
            // 查找空槽
            v := nextFreeFast(span)
            if v == 0 {
                v, span, shouldhelpgc = c.nextFree(tinySpanClass)
            }
            x = unsafe.Pointer(v)
            // 手动内存清零
            (*[2]uint64)(x)[0] = 0
            (*[2]uint64)(x)[1] = 0
            // 新取出来的空槽赋值给tiny字段 这个if貌似能去掉，没什么意义
            if size < c.tinyoffset || c.tiny == 0 {
                c.tiny = uintptr(x)
                c.tinyoffset = size
            }
            size = maxTinySize
        } else {
            // 小对象分配
            var sizeclass uint8
            // 这里是根据需要分配的大小获取对应的classid
            if size <= smallSizeMax-8 {
                // 表中小于1024的，按8为最小单位分割（即最小增长）
                sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
            } else {
                // 大于1024的，按128字节为分割
                sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
            }
            // classid对应的对象大小
            size = uintptr(class_to_size[sizeclass])
            // 获取在mcache中的对应的span
            spc := makeSpanClass(sizeclass, noscan)
            span = c.alloc[spc]
            // 快速获取当前span的空槽
            v := nextFreeFast(span)
            if v == 0 {
                // 需要重填allocCache 或 分配新的span
                v, span, shouldhelpgc = c.nextFree(spc)
            }
            x = unsafe.Pointer(v)
            if needzero && span.needzero != 0 {
                // 需要内存清零的 这里清一下
                memclrNoHeapPointers(unsafe.Pointer(v), size)
            }
        }
    } else {
        // 大对象分配
        shouldhelpgc = true
        span = c.allocLarge(size, needzero, noscan)
        span.freeindex = 1
        span.allocCount = 1
        x = unsafe.Pointer(span.base())
        size = span.elemsize
    }

    var scanSize uintptr
    if !noscan {
        if typ == deferType {
            dataSize = unsafe.Sizeof(_defer{})
        }
        heapBitsSetType(uintptr(x), size, dataSize, typ)
        if dataSize > typ.size {
            // Array allocation. If there are any
            // pointers, GC has to scan to the last
            // element.
            if typ.ptrdata != 0 {
                scanSize = dataSize - typ.size + typ.ptrdata
            }
        } else {
            scanSize = typ.ptrdata
        }
        c.scanAlloc += scanSize
    }

    publicationBarrier()
    if gcphase != _GCoff {
        gcmarknewobject(span, uintptr(x), size, scanSize)
    }

    if raceenabled {
        racemalloc(x, size)
    }

    if msanenabled {
        msanmalloc(x, size)
    }

    mp.mallocing = 0
    releasem(mp)

    if debug.malloc {
        if debug.allocfreetrace != 0 {
            tracealloc(x, size, typ)
        }

        if inittrace.active && inittrace.id == getg().goid {
            // Init functions are executed sequentially in a single Go routine.
            inittrace.bytes += uint64(size)
        }
    }

    if rate := MemProfileRate; rate > 0 {
        if rate != 1 && size < c.nextSample {
            c.nextSample -= size
        } else {
            mp := acquirem()
            profilealloc(mp, x, size)
            releasem(mp)
        }
    }

    if assistG != nil {
        // Account for internal fragmentation in the assist
        // debt now that we know it.
        assistG.gcAssistBytes -= int64(size - dataSize)
    }

    if shouldhelpgc {
        if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
            gcStart(t)
        }
    }

    return x
}
```
