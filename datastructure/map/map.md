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
	m["apple"] = "red"		// 添加
	m["apple"] = "green"	// 修改   NOTICE:如果“apple”不存在，会添加
	delete(m, "apple")		// 删除   NOTICE:m为nil或指定的键不存在，也不会报错，相当于空操作
	v, exist := m["apple"]	// 查询
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
		// insertk 是要插入key的位置的内存地址，也就是说insertk的地址在哈希表的数组里面，不能变，这一句其实就是将kmem的值（指向一个key的地址）放在了insertk的地址上。比如insertk本来是0x60，要在这里放key，它的值现在是初始化状态0x00，kmem是0x80，这一句就是将0x60的值0x00改为0x80。变成了key的二级指针（小补充我们扒汇编！）
		*(*unsafe.Pointer)(insertk) = kmem
		// 让insertk真正的指向要放key的指针
		insertk = kmem
	}
	if t.indirectelem() {
		// 跟上面一样 剩下那一句在326行 要照顾上面270行的elem
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

## 3. 总结
### 3.1 unsafe.Pointer的骚操作
303行的测试代码及其汇编
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
0x0021 00033 (main.go:8)	MOVQ	$1, "".i+16(SP)					// 给i赋值1
0x002a 00042 (main.go:9)	LEAQ	"".i+16(SP), AX					// i的地址给ax
0x002f 00047 (main.go:9)	MOVQ	AX, "".iptr+48(SP)			// ax的值给iptr
0x0034 00052 (main.go:10)	MOVQ	AX, "".uniptr+32(SP)		// iptr与uniptr一模一样
0x0039 00057 (main.go:12)	LEAQ	type.int(SB), AX
0x0040 00064 (main.go:12)	MOVQ	AX, (SP)
0x0044 00068 (main.go:12)	PCDATA	$1, $1
0x0044 00068 (main.go:12)	CALL	runtime.newobject(SB)		//jptr
0x0049 00073 (main.go:12)	MOVQ	8(SP), AX
0x004e 00078 (main.go:12)	MOVQ	AX, "".jptr+40(SP)
0x0053 00083 (main.go:13)	MOVQ	$2, (AX)								//给 jptr 赋值
0x005a 00090 (main.go:14)	MOVQ	"".jptr+40(SP), AX
0x005f 00095 (main.go:14)	MOVQ	AX, "".unjptr+24(SP)
0x0064 00100 (main.go:16)	MOVQ	"".uniptr+32(SP), DI			// 将uniptr的值给DI
0x0069 00105 (main.go:16)	MOVQ	DI, ""..autotmp_5+56(SP)
0x006e 00110 (main.go:16)	TESTB	AL, (DI)
0x0070 00112 (main.go:16)	MOVQ	"".unjptr+24(SP), AX			// unjptr的值给AX
0x0075 00117 (main.go:16)	PCDATA	$0, $-2
0x0075 00117 (main.go:16)	CMPL	runtime.writeBarrier(SB), $0
0x007c 00124 (main.go:16)	JEQ	130
0x007e 00126 (main.go:16)	NOP
0x0080 00128 (main.go:16)	JMP	155
0x0082 00130 (main.go:16)	MOVQ	AX, (DI)                  // DI 中的值指向AX的值
0x0085 00133 (main.go:16)	JMP	135
0x0087 00135 (main.go:17)	PCDATA	$0, $-1
0x0087 00135 (main.go:17)	MOVQ	"".unjptr+24(SP), AX
0x008c 00140 (main.go:17)	MOVQ	AX, "".uniptr+32(SP)
```
以上最重要的一句就是 `MOVQ	AX, (DI)` ，如果不太懂可以参照 `*jptr = 2` 的汇编赋值那一句 `MOVQ	$2, (AX)`。
