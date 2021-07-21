# map
`map` 是 `go` 语言实现的哈希表。可以与[redis的哈希表](https://github.com/willshS/redis-/blob/main/datastruct/dict.md)进行对比，更容易理解。

## 1. map的使用
`map` 的细节有点多，下面的例子可能不够全面：
```
// panic map必须通过make 或 显式赋值
var m map[int]int
m[0] = 0

// map访问不存在的key时，会返回对应value类型的"零值"
m := make(map[int]int)
i := map[0]
i, ok := map[0] // 应该通过第二个参数来判断是否存在

// 《go专家编程》 p27
func MapCRUD() {
    m := make(map[string]string, 10)
    m["apple"] = "red"        // 添加
    m["apple"] = "green"      // 修改   NOTICE:如果“apple”不存在，会添加
    delete(m, "apple")        // 删除   NOTICE:m为nil或指定的键不存在，也不会报错，相当于空操作
    v, exist := m["apple"]    // 查询
    if exist {
        fmt.Printf("apple-%s\n", v)
    }
}
```
还要注意的是 `map` 并不支持并发。

## 2. 实现
`map` 与 `redis` 的 `dict` 都使用拉链法解决冲突。

### 2.1 map的定义
```
// A header for a Go map.
type hmap struct {
    count     int // map中的键值对的数量
    flags     uint8
    B         uint8  // 桶个数的对数 即 buckets = 2^B
    noverflow uint16 // 溢出桶的近似数量
    hash0     uint32 // hash seed

    buckets    unsafe.Pointer // 桶数组的指针，当count == 0时，可能为nil
    oldbuckets unsafe.Pointer // 旧桶数组的指针
    nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

    extra *mapextra // optional fields
}

// 桶的首地址
type bmap struct {
    tophash [bucketCnt]uint8
}

func (b *bmap) overflow(t *maptype) *bmap {
    // 获取溢出桶的指针
    return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize))
}

func (b *bmap) setoverflow(t *maptype, ovf *bmap) {
    // 设置溢出桶的指针
    *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize)) = ovf
}

func (b *bmap) keys() unsafe.Pointer {
    return add(unsafe.Pointer(b), dataOffset)
}
```

### 2.2 map的创建
`map` 的创建有两种，根据静态变量 `bucketCnt == 8` 来判断，如果至多为8，则使用 `makemap_small`，否则使用 `makemap`
```
// a := make(map[int]int, 5) // 可以看到在汇编过程中因为参数小于8，直接被忽略了，调用 makemap_samll 即可
0x003e 00062 (main.go:9)        PCDATA  $1, $0
0x003e 00062 (main.go:9)        NOP
0x0040 00064 (main.go:9)        CALL    runtime.makemap_small(SB)
0x0045 00069 (main.go:9)        MOVQ    (SP), AX
0x0049 00073 (main.go:9)        MOVQ    AX, "".a+120(SP)

// small的创建非常简单，new 出一个 hmap 的指针，并算一个哈希种子。这里h肯定在堆上 逃逸了。
func makemap_small() *hmap {
    h := new(hmap)
    h.hash0 = fastrand()
    return h
}


// a := make(map[int]int, 9) // 与下面的makemap 函数比对，可知参数
0x003e 00062 (main.go:9)        LEAQ    type.map[int]int(SB), AX
0x0045 00069 (main.go:9)        MOVQ    AX, (SP)
0x0049 00073 (main.go:9)        MOVQ    $9, 8(SP)
0x0052 00082 (main.go:9)        MOVQ    $0, 16(SP)
0x005b 00091 (main.go:9)        PCDATA  $1, $0
0x005b 00091 (main.go:9)        NOP
0x0060 00096 (main.go:9)        CALL    runtime.makemap(SB)
0x0065 00101 (main.go:9)        MOVQ    24(SP), AX
0x006a 00106 (main.go:9)        MOVQ    AX, "".a+120(SP)

// 返回*hmap 表示我们的map对象就是个8字节的指针，与chan不同。
func makemap(t *maptype, hint int, h *hmap) *hmap {
    mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
    if overflow || mem > maxAlloc {
        hint = 0
    }

    // initialize Hmap
    if h == nil {
        h = new(hmap)
    }
    h.hash0 = fastrand()

    /* 根据负载因子 loadFactorNum 与 loadFactorDen 算出桶数量的对数
    loadFactorNum == 13 && loadFactorDen == 2 负载因子表示最大负载允许2个桶装13个元素。 */

    /* return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
    因为B是对数 因此永远可以整除2，这样就可以使用两个int表示6.5这个最大负载因子了。*/
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

    // 给哈希表分配内存，如果B是0，不在这里分配，而是等插入数据的时候在mapassign惰性分配
    // 如果hint很大，这里耗费时间也要长
    if h.B != 0 {
        var nextOverflow *bmap
        h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
        if nextOverflow != nil {
            // 有溢出桶
            h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow
        }
    }

    return h
}

// 分配内存函数 // dirtyalloc可以为nil或之前同样的t和b分配的桶数组（内存复用）
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    base := bucketShift(b)
    nbuckets := base
    // 当b比较小的时候，不太可能溢出，避免计算开销
    if b >= 4 {
        // 计算溢出桶的估计值
        nbuckets += bucketShift(b - 4)
        sz := t.bucket.size * nbuckets
        // 内存调整一下
        up := roundupsize(sz)
        if up != sz {
            nbuckets = up / t.bucket.size
        }
    }

    if dirtyalloc == nil {
        buckets = newarray(t.bucket, int(nbuckets))
    } else {
        // 之前分配的内存清理一下，复用
        buckets = dirtyalloc
        size := t.bucket.size * nbuckets
        if t.bucket.ptrdata != 0 {
            memclrHasPointers(buckets, size)
        } else {
            memclrNoHeapPointers(buckets, size)
        }
    }

    if base != nbuckets {
        // 预分配溢出桶，nbuckets - base 就是溢出桶的数量
        nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
        // 最后一个溢出桶的最后8个字节，指向数组头指针，表示是最后一个溢出桶了
        last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
        last.setoverflow(t, (*bmap)(buckets))
    }
    return buckets, nextOverflow
}
```
上面是 `map` 的创建，这里我们看到如何去创建一个哈希表，并分配相关内存。下面增删改查可以进一步了解 `map` 的实现细节。

### 2.2 增
添加一个元素的表达式为 `map[x] = x` 。通过扒汇编，可以看到最终调用 `mapassign` 函数来进行处理：
```
// b[1] = 1
0x00a1 00161 (main.go:15)       LEAQ    type.map["".mapkey]int(SB), CX
0x00a8 00168 (main.go:15)       MOVQ    CX, (SP)
0x00ac 00172 (main.go:15)       MOVQ    AX, 8(SP)
0x00b1 00177 (main.go:15)       LEAQ    ""..autotmp_7+240(SP), AX
0x00b9 00185 (main.go:15)       MOVQ    AX, 16(SP)
0x00be 00190 (main.go:15)       PCDATA  $1, $1
0x00be 00190 (main.go:15)       NOP
0x00c0 00192 (main.go:15)       CALL    runtime.mapassign(SB)

func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if h == nil {
        // 不要访问 nil map哦
        panic(plainError("assignment to entry in nil map"))
    }
    if raceenabled {
        callerpc := getcallerpc()
        pc := funcPC(mapassign)
        racewritepc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
    if msanenabled {
        msanread(key, t.key.size)
    }
    // 并发检测
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }
    // 计算 key 的哈希值
    hash := t.hasher(key, uintptr(h.hash0))
    // 因为计算哈希可能会panic，因此计算完哈希再设置状态
    h.flags ^= hashWriting

    if h.buckets == nil {
        h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
    }

again:
    // 拿到桶的下标
    bucket := hash & bucketMask(h.B)
    // 如果在rehash，rehash一下
    if h.growing() {
        growWork(t, h, bucket)
    }
    // 获取到桶的位置的指针，从这里可以看出来桶的首地址是 bmap
    b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
    // 拿到哈希值的高8位
    top := tophash(hash)

    var inserti *uint8
    var insertk unsafe.Pointer
    var elem unsafe.Pointer
bucketloop:
    for {
        // bucketCnt == 8 表示一个桶里面有8个元素， 现在遍历这个桶中的所有元素
        for i := uintptr(0); i < bucketCnt; i++ {
            // 桶中的top哈希对比
            if b.tophash[i] != top {
                // 桶中的这个没有元素，并且我们插入位置还没找到
                if isEmpty(b.tophash[i]) && inserti == nil {
                    // 这三行代码再结合bmap结构体定义以及方法。一个桶的内存模型就水落石出了。
                    // 插入top哈希的位置
                    inserti = &b.tophash[i]
                    // 插入key的位置 桶首地址+8（tophash）+i*keysize
                    insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
                    // value的位置 桶首地址+8（tophash）+8*keysize+i*elemsize
                    elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                }
                // 没有更多非空的位置了 也就是后面的都是空的位置
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            // 这里找到了top哈希相同的
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // 判断存储key的指针还是key本身
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
            // 判断key是否相同，不相同继续遍历桶
            if !t.key.equal(key, k) {
                continue
            }
            // 是否需要更新key （TODO:暂时不清楚这个机制是什么意思，需要看go的编译器实现，测试了一个大的结构体（里面多个string类型）为key，这里是进去了的）
            if t.needkeyupdate() {
                typedmemmove(t.key, k, key)
            }
            // 找到值的位置
            elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
            goto done
        }
        // 遍历完桶了 没找到 并且没有空位置，去溢出桶里找
        ovf := b.overflow(t)
        if ovf == nil {
            break
        }
        b = ovf
    }

    // 走到这里的 是没找到key的（可能找到了空位置，也可能没有），找到的直接goto了

    // 如果没有扩容 并且 （超过负载因子了 或 太多溢出桶）
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        // 扩容 并且重新找位置 哪怕已经找好位置了
        hashGrow(t, h)
        goto again // Growing the table invalidates everything, so try again
    }

    if inserti == nil {
        // 没找到空位置，新建溢出桶
        newb := h.newoverflow(t, b)
        // 插入到溢出桶的新位置
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        elem = add(insertk, bucketCnt*uintptr(t.keysize))
    }

    // store new key/elem at insert position
    if t.indirectkey() {
        // 如果存的是指针，new一块key类型的内存，插入位置指向这块内存
        kmem := newobject(t.key)
        // insertk 是要插入key的位置的内存地址，也就是说insertk的地址在哈希表的数组里面，不能变，这一句其实就是将kmem的值（指向一个key的地址）放在了insertk的地址上。
        // 比如insertk本来是0x60，要在这里放key，它的值现在是初始化状态0x00，kmem是0x80，这一句就是将0x60的值0x00改为0x80。变成了key的二级指针（小补充我们扒汇编！）
        *(*unsafe.Pointer)(insertk) = kmem
        // 让insertk真正的指向要放key的指针
        insertk = kmem
    }
    if t.indirectelem() {
        // 跟上面一样 剩下那一句在函数最后，要照顾上面直接goto到done的elem
        vmem := newobject(t.elem)
        *(*unsafe.Pointer)(elem) = vmem
    }
    // 真正的将key 放入到哈希表中
    typedmemmove(t.key, insertk, key)
    // 放tophash
    *inserti = top
    h.count++

done:
    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    // 去掉写的状态
    h.flags &^= hashWriting
    if t.indirectelem() {
        elem = *((*unsafe.Pointer)(elem))
    }
    return elem
}
```
上面的函数返回 `elem` 指向值要插入的位置。`value` 的赋值就是编译器用汇编指令做的，一个赋值指令。  
以上就是 `map` 的添加元素的核心代码，修改与添加没有任何区别。在这个过程中，如果桶没有空位置了，会新增溢出桶：
```
// 新建溢出桶
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
    var ovf *bmap
    if h.extra != nil && h.extra.nextOverflow != nil {
        // h.b >= 4 的时候我们已经预分配了溢出桶，这里就直接用
        ovf = h.extra.nextOverflow
        if ovf.overflow(t) == nil {
            // 这个桶不是最后一个桶，更新额外的溢出桶
            h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
        } else {
            // 最后一个桶更新它的最后的8个字节的指针，以及额外溢出桶置为nil
            ovf.setoverflow(t, nil)
            h.extra.nextOverflow = nil
        }
    } else {
        // 新分配一个桶
        ovf = (*bmap)(newobject(t.bucket))
    }
    // 这个函数增加h.noverflow，如果h.B >= 16，这是个近似值。反之是个精确值
    h.incrnoverflow()
    if t.bucket.ptrdata == 0 {
        // 如果桶不包含指针类型，创建溢出桶 使用 extra.overflow管理
        h.createOverflow()
        *h.extra.overflow = append(*h.extra.overflow, ovf)
    }
    // 新的溢出桶 挂在 插入的时候找到的 满了的桶后面
    b.setoverflow(t, ovf)
    return ovf
}
```
从上面可以看到，溢出桶是可能存在两种情况的，一种是我们创建 `map` 的时候参数较大（b>=4,即参数大于4*13），会预分配出一些溢出桶，这些桶会被挂在整个数组的最后。另一种是我们溢出桶用完了或者没有预分配的溢出桶的时候，重新分配一个桶。

--------  
**tips:从新增就能看出哈希表的整个内存模型了。**
1. **首先是一整块内存，根据kv类型大小被分为等大的桶。每个桶的大小为：`[8]uint8+8*(keysize+valsize)+uintptr`。**
2. **桶中的内存模型即8字节的tophash+8个key+8个val+8字节的指向溢出桶的指针**
3. **当桶中空间不够以及没有预分配溢出桶或预分配的溢出桶被用完，会单独开辟一块内存为一个桶，与主哈希的数组不在一起的内存，通过桶中的尾指针进行关联**

--------  

### 2.3 删
`map` 删除是调用内置函数 `delete`，经过汇编会调用 `mapdelete` （有兴趣的自己扒汇编，不贴了）：
```
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
    if raceenabled && h != nil {
        callerpc := getcallerpc()
        pc := funcPC(mapdelete)
        racewritepc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
    if msanenabled && h != nil {
        msanread(key, t.key.size)
    }
    if h == nil || h.count == 0 {
    // 可以看到 nil map或空map随便你调用delete 不会panic
    // 关于这个 issue 有兴趣的也可以看看
        if t.hashMightPanic() {
            t.hasher(key, 0) // see issue 23734
        }
        return
    }
    // 竞争检查
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }

    // 计算哈希 更改哈希状态
    hash := t.hasher(key, uintptr(h.hash0))
    h.flags ^= hashWriting
    // 找桶
    bucket := hash & bucketMask(h.B)
    if h.growing() {
        growWork(t, h, bucket)
    }
    b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
    bOrig := b
    top := tophash(hash)
search:
    // 与增加不一样，只需要查找桶b以及其溢出桶
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                // 优化,后面没了 不用找了
                if b.tophash[i] == emptyRest {
                    break search
                }
                continue
            }
            // top哈希相同的k的位置
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            k2 := k
            // 处理key
            if t.indirectkey() {
                k2 = *((*unsafe.Pointer)(k2))
            }
            // 比较key
            if !t.key.equal(key, k2) {
                continue
            }
            // 如果是指针，gc来回收，然后只需要处理key中包含的指针
            if t.indirectkey() {
                *(*unsafe.Pointer)(k) = nil
            } else if t.key.ptrdata != 0 {
                memclrHasPointers(k, t.key.size)
            }
            // 处理val
            e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
            if t.indirectelem() {
                *(*unsafe.Pointer)(e) = nil
            } else if t.elem.ptrdata != 0 {
                memclrHasPointers(e, t.elem.size)
            } else {
                memclrNoHeapPointers(e, t.elem.size)
            }

            // 这里是精髓，我们通过tophash的状态值可以做很多优化。
            // 这里当我们删除一个元素，如果这个元素的下一个元素是 emptyRest,那么这个被删除的元素也是emptyRest，否则仅仅是emptyOne
            b.tophash[i] = emptyOne
            if i == bucketCnt-1 {
                if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
                    goto notLast
                }
            } else {
                if b.tophash[i+1] != emptyRest {
                    goto notLast
                }
            }
            // 自己的状态搞定了，要往前看，如果前面一个是emptyOne,那么现在也一定是emptyRest了
            for {
                b.tophash[i] = emptyRest
                if i == 0 {
                    // bOrig是主桶，检查b是否是溢出桶
                    if b == bOrig {
                        break // beginning of initial bucket, we're done.
                    }
                    // 溢出桶的话得找到当前桶的前一个溢出桶的最后一个元素
                    c := b
                    for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
                    }
                    i = bucketCnt - 1
                } else {
                    // 不管在哪个桶，不是第一个元素，前面就有元素
                    i--
                }
                if b.tophash[i] != emptyOne {
                    break
                }
            }
        notLast:
            h.count--
            break search
        }
    }

    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    h.flags &^= hashWriting
}
```
删除函数 `delete` 对于不存在的键和 `nil` 哈希都不会panic  
通过删除函数，`tophash` 的部分作用就暴漏出来了：
1. **首先做为比较键的前置，如果 `tophash` 不同，则键一定不同，大大加速了键的比较**
2. **其次 `tophash` 不存在元素的时候，可以作为所代表的 `cell` 的状态，这个状态也能标识后面 `cell` 的状态，让我们遍历的时候可以不需要遍历所有数据**  

这仅仅是部分作用，其它作用会在后面也凸显出来。

### 2.4 rehash
这里要先看如何rehash的，然后再去看查询，因为不论是增和删，都只需要在新的 `buket` 进行操作。
```
func hashGrow(t *maptype, h *hmap) {
    // 扩容
    bigger := uint8(1)
    if !overLoadFactor(h.count+1, h.B) {
        // 如果容量其实够，就不进行扩容，说明溢出桶太多。
        bigger = 0
        h.flags |= sameSizeGrow
    }
    // 生成新的桶数组 新的B = oldB 或 B = oldB + 1 即扩大二倍
    oldbuckets := h.buckets
    newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

    // 处理迭代器状态
    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }
    // 更新哈希表的数据，以前的桶数组变成旧桶
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
    h.nevacuate = 0
    h.noverflow = 0

    if h.extra != nil && h.extra.overflow != nil {
        // 以前的溢出桶现在变成旧的溢出桶
        if h.extra.oldoverflow != nil {
            throw("oldoverflow is not nil")
        }
        h.extra.oldoverflow = h.extra.overflow
        h.extra.overflow = nil
    }
    // 溢出桶
    if nextOverflow != nil {
        if h.extra == nil {
            h.extra = new(mapextra)
        }
        h.extra.nextOverflow = nextOverflow
    }
}
```
上面的函数的功能是生成新的桶数组，并且修改哈希表的相关数据状态。真正的扩容工作，是在增和删的函数中做的。加入我们要新增或者删除某个桶的数据，那么先对这个桶做rehash。因此保证了增和删操作都是在新的桶数组上进行！
```
func growWork(t *maptype, h *hmap, bucket uintptr) {
    // 这个函数扩容，参数bucket是新的数组的桶号，根据这个桶号算出在旧的桶数组中的桶号
    //（因为是二倍扩容，比如以前10个桶，扩容到20，那么以前5号桶的数据就分散到了现在的5号桶和15号桶）
    evacuate(t, h, bucket&h.oldbucketmask())

    // evacuate one more oldbucket to make progress on growing
    if h.growing() {
        evacuate(t, h, h.nevacuate)
    }
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
    newbit := h.noldbuckets()

    if !evacuated(b) {
        // 这个桶没有迁移 这里有个作者的TODO，复用溢出桶当没有迭代器使用的时候

        // 这里找到一个新桶的位置，一个是x 原来的桶号 另一个是y 原来的桶号+原来的总桶数
        x := &xy[0]
        x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
        x.k = add(unsafe.Pointer(x.b), dataOffset)
        x.e = add(x.k, bucketCnt*uintptr(t.keysize))

        if !h.sameSizeGrow() {
            // 仅在增大容量的时候计算y，否则y会越界
            y := &xy[1]
            y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
            y.k = add(unsafe.Pointer(y.b), dataOffset)
            y.e = add(y.k, bucketCnt*uintptr(t.keysize))
        }

        // 遍历旧桶和旧桶的溢出桶
        for ; b != nil; b = b.overflow(t) {
            k := add(unsafe.Pointer(b), dataOffset)
            e := add(k, bucketCnt*uintptr(t.keysize))
            // 遍历桶中的所有数据
            for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
                top := b.tophash[i]
                // 这个位置没有数据
                if isEmpty(top) {
                    b.tophash[i] = evacuatedEmpty
                    continue
                }
                // 注意，没有迁移的桶的数据 不可能是 1 - 5 即 不会有迁移状态
                if top < minTopHash {
                    throw("bad map state")
                }
                k2 := k
                if t.indirectkey() {
                    k2 = *((*unsafe.Pointer)(k2))
                }
                var useY uint8
                if !h.sameSizeGrow() {
                    // 再算一次hash
                    hash := t.hasher(k2, uintptr(h.hash0))
                    if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
                        // 当key无法比较的时候，根据tophash迁移位置，并再次计算top哈希，这样扩容多次能保证数据均匀
                        useY = top & 1
                        top = tophash(hash)
                    } else {
                        // 根据哈希的低位和旧桶数量来决定迁移到哪个桶
                        if hash&newbit != 0 {
                            useY = 1
                        }
                    }
                }
                // 静态变量的状态检查，是否是预期的。
                if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
                    throw("bad evacuatedN")
                }

                // 以下代码就是迁移某个k-v到对应的新桶中
                // 旧桶中的tophash被改为迁移状态
                b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
                dst := &xy[useY]                 // evacuation destination

                // 目的桶满了，创建新的溢出桶
                if dst.i == bucketCnt {
                    dst.b = h.newoverflow(t, dst.b)
                    dst.i = 0
                    dst.k = add(unsafe.Pointer(dst.b), dataOffset)
                    dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
                }
                // 更新新桶的对应tophash
                dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
                if t.indirectkey() {
                    // 指针类型的话赋值指针就行了，key的真是数据还是那一块内存
                    *(*unsafe.Pointer)(dst.k) = k2 // copy pointer
                } else {
                    typedmemmove(t.key, dst.k, k) // copy elem
                }
                if t.indirectelem() {
                    *(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
                } else {
                    typedmemmove(t.elem, dst.e, e)
                }
                // 新桶的key的数量+1
                dst.i++
                // 更新新桶的下一个k-v的地址。如果是最后一个也没关系，因为桶的最后8个字节是溢出桶的指针，不会越界
                dst.k = add(dst.k, uintptr(t.keysize))
                dst.e = add(dst.e, uintptr(t.elemsize))
            }
        }
        // 清楚溢出桶和k-v的数据，帮助gc
        if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
            b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
            // tophash还要用。k-v+溢出桶指针 都清理掉
            ptr := add(b, dataOffset)
            n := uintptr(t.bucketsize) - dataOffset
            memclrHasPointers(ptr, n)
        }
    }

    // 更新迁移状态
    if oldbucket == h.nevacuate {
        advanceEvacuationMark(h, t, newbit)
    }
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
    h.nevacuate++
    // 这里为了做个保护，不是太明白 TODO：
    // Experiments suggest that 1024 is overkill by at least an order of magnitude.
    // Put it in there as a safeguard anyway, to ensure O(1) behavior.
    stop := h.nevacuate + 1024
    if stop > newbit {
        stop = newbit
    }
    // 更新nevacuate，如果后面桶也被迁移
    for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
        h.nevacuate++
    }
    // 迁移完成
    if h.nevacuate == newbit {
        // 释放旧的桶内存和溢出桶内存
        h.oldbuckets = nil
        if h.extra != nil {
            h.extra.oldoverflow = nil
        }
        h.flags &^= sameSizeGrow
    }
}
```
可以看到rehash有两种，一种是溢出桶太多，一种是负载因子超过6.5。每次访问某个桶的时候，都对这个桶进行迁移。这样渐进式rehash可以分步进行，与 `redis` 的是同样的。这里 `tophash` 的其他3种状态也就明白了。`nevacuate` 的使用可以记录迁移进度，小于 `nevacuate` 一定被迁移了，大于 `nevacuate` 可能被迁移。

### 2.5 查
查询分为两种，一种是根据键获取值，一种是遍历哈希表。
#### 2.5.1 search k
根据键获取值
```
// v := map[k]
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if raceenabled && h != nil {
        callerpc := getcallerpc()
        pc := funcPC(mapaccess1)
        racereadpc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
    if msanenabled && h != nil {
        msanread(key, t.key.size)
    }
    if h == nil || h.count == 0 {
        // nil map不会panic
        if t.hashMightPanic() {
            t.hasher(key, 0) // see issue 23734
        }
        return unsafe.Pointer(&zeroVal[0])
    }
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    hash := t.hasher(key, uintptr(h.hash0))
    m := bucketMask(h.B)
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // 如果 oldbuckets 不为空，说明正在rehash
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            // 如果不是同样大小的rehash，说明就的桶比新的少一倍
            m >>= 1
        }
        // 获取旧的桶
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        // 判断这个桶是否已经rehash了，true表示已经rehash
        if !evacuated(oldb) {
            b = oldb
        }
    }
    top := tophash(hash)
bucketloop:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                // 优化
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            // 获取tophash相同的key
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
            // 比较key
            if t.key.equal(key, k) {
                // 获取指向val的指针
                e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                if t.indirectelem() {
                    e = *((*unsafe.Pointer)(e))
                }
                return e
            }
        }
    }
    // 没找到，返回 零值
    return unsafe.Pointer(&zeroVal[0])
}
```
另一个函数 `mapaccess2` 与 `mapaccess1` 大同小异，只是多了一个是否存在的 `bool` 变量。

#### 2.5.2 遍历
`map` 的遍历，经过编译器转换后，调用 `mapiterinit` 函数，这里使用了一个叫 `hiter` 的迭代器，因此先看一下迭代器的定义：
```
// 迭代器结构体
type hiter struct {
    key         unsafe.Pointer // 编译器指定好的，必须第一个位置。遍历结束赋值为nil
    elem        unsafe.Pointer // 第二个位置
    t           *maptype             // map 类型
    h           *hmap                     // map 指针
    buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
    bptr        *bmap          // current bucket
    overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
    oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
    startBucket uintptr        // bucket iteration started at
    offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
    wrapped     bool           // already wrapped around from end of bucket array to beginning
    B           uint8
    i           uint8
    bucket      uintptr
    checkBucket uintptr
}
```
迭代器中的字段很多看起来都很眼熟，与哈希表的比较像，我们看迭代过程源码逐一理解其含义。首先，先扒一下迭代汇编：
```
// k,v := range m
0x013f 00319 (main.go:14)    MOVQ    "".m+96(SP), AX
0x0144 00324 (main.go:14)    MOVQ    AX, ""..autotmp_11+144(SP) // m是我们的map 拷贝一个（map本身就是8字节的指针）
0x014c 00332 (main.go:14)    LEAQ    ""..autotmp_12+416(SP), DI
0x0154 00340 (main.go:14)    XORPS    X0, X0
0x0157 00343 (main.go:14)    PCDATA    $0, $-2
0x0157 00343 (main.go:14)    LEAQ    -32(DI), DI
0x015b 00347 (main.go:14)    NOP
0x0160 00352 (main.go:14)    DUFFZERO    $273
0x0173 00371 (main.go:14)    PCDATA    $0, $-1
0x0173 00371 (main.go:14)    MOVQ    ""..autotmp_11+144(SP), AX // map放到 AX
0x017b 00379 (main.go:14)    LEAQ    type.map["".test1]int(SB), CX // map类型放到CX
0x0182 00386 (main.go:14)    MOVQ    CX, (SP) // 类型入栈
0x0186 00390 (main.go:14)    MOVQ    AX, 8(SP) // m指针入栈
0x018b 00395 (main.go:14)    LEAQ    ""..autotmp_12+416(SP), AX
0x0193 00403 (main.go:14)    MOVQ    AX, 16(SP) // 根据参数顺序，匿名变量12 就是 *hiter了，这说明hiter是栈变量
0x0198 00408 (main.go:14)    PCDATA    $1, $3
0x0198 00408 (main.go:14)    CALL    runtime.mapiterinit(SB)
0x019d 00413 (main.go:14)    JMP    415
0x019f 00415 (main.go:14)    CMPQ    ""..autotmp_12+416(SP), $0
0x01a8 00424 (main.go:14)    JNE    431
0x01aa 00426 (main.go:14)    JMP    1115
0x01af 00431 (main.go:14)    MOVQ    ""..autotmp_12+416(SP), AX
0x01b7 00439 (main.go:14)    TESTB    AL, (AX)
0x01b9 00441 (main.go:14)    MOVQ    (AX), CX
0x01bc 00444 (main.go:14)    MOVQ    8(AX), DX
0x01c0 00448 (main.go:14)    MOVQ    16(AX), AX
0x01c4 00452 (main.go:14)    MOVQ    CX, "".k+264(SP)
0x01cc 00460 (main.go:14)    MOVQ    DX, "".k+272(SP)
0x01d4 00468 (main.go:14)    MOVQ    AX, "".k+280(SP)
0x01dc 00476 (main.go:14)    MOVQ    ""..autotmp_12+424(SP), AX
0x01e4 00484 (main.go:14)    TESTB    AL, (AX)
0x01e6 00486 (main.go:14)    MOVQ    (AX), AX
0x01e9 00489 (main.go:14)    MOVQ    AX, "".v+64(SP)
```

```
func mapiterinit(t *maptype, h *hmap, it *hiter) {
    if raceenabled && h != nil {
        callerpc := getcallerpc()
        racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiterinit))
    }

    if h == nil || h.count == 0 {
        // 空map不需要遍历
        return
    }

    if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
        throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
    }
    // 存储map相关信息到迭代器
    it.t = t
    it.h = h

    // grab snapshot of bucket state
    it.B = h.B
    it.buckets = h.buckets
    if t.bucket.ptrdata == 0 {
        // 这里是为了拿到除了主数组外其它的哈希内存，可以参考上面的内存模型的总结
        h.createOverflow()
        it.overflow = h.extra.overflow
        it.oldoverflow = h.extra.oldoverflow
    }

    // 注意，这里随机一个值，用这个值来选择从哪开始遍历
    r := uintptr(fastrand())
    if h.B > 31-bucketCntBits {
        r += uintptr(fastrand()) << 31
    }
    // 随机的开始桶下标
    it.startBucket = r & bucketMask(h.B)
    // 开始桶的开始偏移
    it.offset = uint8(r >> h.B & (bucketCnt - 1))

    // 迭代中的桶下标
    it.bucket = it.startBucket

    // 这里给map的状态加上迭代状态
    if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
        atomic.Or8(&h.flags, iterator|oldIterator)
    }

    mapiternext(it)
}

func mapiternext(it *hiter) {
    // 开始迭代
    h := it.h
    if raceenabled {
        callerpc := getcallerpc()
        racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiternext))
    }
    if h.flags&hashWriting != 0 {
        throw("concurrent map iteration and map write")
    }
    t := it.t
    bucket := it.bucket
    b := it.bptr
    i := it.i
    checkBucket := it.checkBucket

next:
    if b == nil {
        if bucket == it.startBucket && it.wrapped {
            // 遍历结束
            it.key = nil
            it.elem = nil
            return
        }
        if h.growing() && it.B == h.B {
            // 如果正在rehash，并且rehash没有完成，遍历下标对应的旧桶
            oldbucket := bucket & it.h.oldbucketmask()
            b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
            if !evacuated(b) {
                checkBucket = bucket
            } else {
                // 旧桶已经被转移了，拿新桶
                b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
                checkBucket = noCheck
            }
        } else {
            // 没有rehash，直接拿桶
            b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
            checkBucket = noCheck
        }
        bucket++
        // 遍历到数组末尾了，继续从数组头遍历，因此it.wrapped就是表示是否到数组头了
        if bucket == bucketShift(it.B) {
            bucket = 0
            it.wrapped = true
        }
        i = 0
    }
    for ; i < bucketCnt; i++ {
        // 根据随机的偏移，获取元素
        offi := (i + it.offset) & (bucketCnt - 1)
        if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
            // 桶这个位置不存在元素
            continue
        }
        // 获取k和v
        k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
        if t.indirectkey() {
            k = *((*unsafe.Pointer)(k))
        }
        e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
        if checkBucket != noCheck && !h.sameSizeGrow() {
            // 这里我们遍历是以新桶为准，如果k迁移到其他新桶，跳过，等遍历到它的新桶的时候，再遍历
            if t.reflexivekey() || t.key.equal(k, k) {
                // 判断是否到这个新桶
                hash := t.hasher(k, uintptr(h.hash0))
                if hash&bucketMask(it.B) != checkBucket {
                    continue
                }
            } else {
                // 如果是不可比较的key类型，通过tophash的最后一位判断，与迁移的逻辑一致
                // checkBucket是新桶的桶号，它的位数如果为 B-1 表示 0-2^(B-1)桶号，左移后是0，如果位数是B 则表示 2^(B-1)到2^B，左移后是1。
                // tophash & 1 为1，则k被迁移到 2^(B-1)到2^B。这样遍历顺序还是跟这新桶的桶号走，不会出现重复遍历或少遍历
                if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
                    continue
                }
            }
        }
        if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
            !(t.reflexivekey() || t.key.equal(k, k)) {
            // 如果当前元素没有被迁移或当前元素是不可比较的那种，可以返回结果
            it.key = k
            if t.indirectelem() {
                e = *((*unsafe.Pointer)(e))
            }
            it.elem = e
        } else {
            // 遍历开始后，这个桶又进行迁移了，调用mapaccessK查找。其实就是我们开始遍历的时候这个桶还没有迁移，但是遍历过程中这个桶迁移了。
            // 比如遍历过程中 "插入数据" 或 "将数据删除" 。这个过程其实与查找完全一样，再次检验桶的状态（这个元素被迁移表示这个桶一定被迁移了）。
            // 理论上来讲mapaccessK函数不会再去遍历旧桶，但是这个函数还是去检验桶的状态了。
            rk, re := mapaccessK(t, h, k)
            if rk == nil {
                continue // key has been deleted
            }
            it.key = rk
            it.elem = re
        }
        // 记录遍历状态，返回数据
        it.bucket = bucket
        if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
            it.bptr = b
        }
        it.i = i + 1
        it.checkBucket = checkBucket
        return
    }
    b = b.overflow(t)
    i = 0
    goto next
}
```
遍历过程非常复杂，因为涉及到rehash，甚至在遍历过程中rehash。下面总结一下遍历过程：
1. 临时栈变量迭代器 `hiter`，随机选择基于主桶数组 `h.bucket` 的某个桶号 `hiter.bucket` 和某个偏移 `hiter.i` 作为起始开始遍历（这里就像获取了个快照）。
2. 开始遍历：判断是否正在rehash，获取对应的桶地址，遍历其中的数据。
    1. 若正在rehash，且当前桶未迁移，遍历对应的旧桶，过滤不会被迁移到当前 `bucket` 的数据，找到下一个偏移 `i++`。将旧桶号作为当前遍历的桶号 `bucket`，新桶号作为检查桶号 `checkBucket`
    2. 若桶已经迁移或不需要迁移，遍历对应的新桶，找到对应的偏移 `i`
3. 根据偏移 `i` 再次检查 `tophash` 状态:
    1. 若不是迁移状态或键值无法比较，选择这对k-v赋值给迭代器 `hiter.key` 和 `hiter.elem`。（因为不可比较key无法在遍历过程中迁移，因为你根本拿不到）
    2. 若在遍历过程中 迁移或删除，重新查找对应的数据，并赋值给迭代器。（因为迭代器的状态，下次遍历这个桶的状态还是未迁移，真实情况已经迁移了，但是只需要继续走这条逻辑就能不漏数据）
4. 更新迭代器的当前遍历的桶号 `hiter.bucket` 和 `hiter.checkBucket`，以便下次迭代时可以继续进行，所以如果继续遍历此桶，状态未变，知道这个桶遍历完，继续第2步。

## 3. 总结
`map` 的实现其实以遍历最难，要考虑到太多种情况，非常难以理解。源码中的 `tophash`、`flags`、`overflow` 等字段都使用的非常巧妙，极大地提升了效率。`mapextra` 的设计来管理所有溢出桶可以让不包含指针类型并且大小小于128的哈希主数组避免被整个扫描。  
对于 `uint32`、`uint64`、`string` 类型的键，`go` 单独做了优化。有兴趣的可以自己去翻阅。整个流程与上面 `map` 差别不大，仅仅是做了类型优化。

### 3.1 unsafe.Pointer的骚操作
"关于大于128字节的key的赋值" 测试代码及其汇编
```
package main

import (
    "unsafe"
)

func main() {
    i := 1
    iptr := &i
    uniptr := unsafe.Pointer(iptr)

    jptr := new(int)
    *jptr = 2
    unjptr := unsafe.Pointer(jptr)

    *(*unsafe.Pointer)(uniptr) = unjptr
    uniptr = unjptr
}

// 汇编
0x0021 00033 (main.go:8)    MOVQ    $1, "".i+16(SP)    // 给i赋值1
0x002a 00042 (main.go:9)    LEAQ    "".i+16(SP), AX    // i的地址给ax
0x002f 00047 (main.go:9)    MOVQ    AX, "".iptr+48(SP) // ax的值给iptr
0x0034 00052 (main.go:10)    MOVQ    AX, "".uniptr+32(SP) // iptr与uniptr一模一样
0x0039 00057 (main.go:12)    LEAQ    type.int(SB), AX
0x0040 00064 (main.go:12)    MOVQ    AX, (SP)
0x0044 00068 (main.go:12)    PCDATA    $1, $1
0x0044 00068 (main.go:12)    CALL    runtime.newobject(SB) //jptr
0x0049 00073 (main.go:12)    MOVQ    8(SP), AX
0x004e 00078 (main.go:12)    MOVQ    AX, "".jptr+40(SP)
0x0053 00083 (main.go:13)    MOVQ    $2, (AX) //给 jptr 赋值
0x005a 00090 (main.go:14)    MOVQ    "".jptr+40(SP), AX
0x005f 00095 (main.go:14)    MOVQ    AX, "".unjptr+24(SP)
0x0064 00100 (main.go:16)    MOVQ    "".uniptr+32(SP), DI // 将uniptr的值给DI
0x0069 00105 (main.go:16)    MOVQ    DI, ""..autotmp_5+56(SP)
0x006e 00110 (main.go:16)    TESTB    AL, (DI)
0x0070 00112 (main.go:16)    MOVQ    "".unjptr+24(SP), AX // unjptr的值给AX
0x0075 00117 (main.go:16)    PCDATA    $0, $-2
0x0075 00117 (main.go:16)    CMPL    runtime.writeBarrier(SB), $0
0x007c 00124 (main.go:16)    JEQ    130
0x007e 00126 (main.go:16)    NOP
0x0080 00128 (main.go:16)    JMP    155
0x0082 00130 (main.go:16)    MOVQ    AX, (DI) // DI 中的值指向AX的值
0x0085 00133 (main.go:16)    JMP    135
0x0087 00135 (main.go:17)    PCDATA    $0, $-1
0x0087 00135 (main.go:17)    MOVQ    "".unjptr+24(SP), AX
0x008c 00140 (main.go:17)    MOVQ    AX, "".uniptr+32(SP)
```
以上最重要的一句就是 `MOVQ    AX, (DI)` ，如果不太懂可以参照 `*jptr = 2` 的汇编赋值那一句 `MOVQ    $2, (AX)`。

### 3.2 iter的小补充
注：key为一个结构体：
```
type test1 struct {
    a int
    b string
}
```
我们分析一下2.5.2中的汇编。首先 `hiter` 是一个指针，匿名变量。第一个和第二个参数分别是键和值的指针。在调用 `runtime.mapiterinit(SB)` 返回后，将 `hiter` 8个字节赋值给 `AX`，然后对 `AX` 进行取值操作，前8个字节给 `CX`，第二个8个字节给 `DX`，第三个8个字节给 `AX`。然后将这24个字节分别赋值给变量 `k` 。等下次遍历的时候，我们依然使用的是 `hiter` 的指针，第二次遍历同第一次一样，将数据赋值给变量 `k`。因此每次遍历 `k` 的值是不同的，但是地址是同一个，归根结底都是 `hiter` 的前8个字节的地址，而 `hiter` 是一个栈临时变量，一直没有变动！因此以下用法是错误的：
```
// k 和 v 道理一样。
test := make([]*int, 0, 10)
for k, v := range m {
    _ = k
    test = append(test, &v)
}
```
