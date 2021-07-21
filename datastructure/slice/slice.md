# slice
切片是我们经常使用的一种数据结构，其本质就是一个数组，与cpp的 `std::vector` 类似。

## 1. 使用
`slice` 切片与数组类似，但是go语言中的数组是无法修改长度的，长度为数组的类型信息。slice可以动态增长。
```
// 1. 从数组中生成一个切片
var b [3]int
var c []int = b[:]

// 2. 使用make函数创建切片
var b []int = make([]int, 10)

// 3. 从其它切片中生成一个切片
var c []int = b[:]
```
使用切片需要注意的是第二种创建方法。此时变量 `b` 中已经有10个元素，每个元素的值为0，而不是空数组
```
fmt.Println(b)
//输出： [0 0 0 0 0 0 0 0 0 0]
```

## 2. 实现

### 2.1 切片定义
```
type slice struct {
    array unsafe.Pointer // 指向底层数组的指针
    len   int  // 长度
    cap   int  // 容量
}
```

### 2.2 切片的创建
#### 2.2.1 通过数组或切片创建切片
使用方法1或方法3创建切片，汇编如下：
```
0x0083 00131 (main.go:11)    MOVQ    "".&b+104(SP), AX
0x0088 00136 (main.go:11)    TESTB    AL, (AX)
0x008a 00138 (main.go:11)    JMP    140
0x008c 00140 (main.go:11)    MOVQ    AX, "".c+208(SP)
0x0094 00148 (main.go:11)    MOVQ    $3, "".c+216(SP)
0x00a0 00160 (main.go:11)    MOVQ    $3, "".c+224(SP)
```
可以看到通过编译将数组 `b` 的地址直接赋值给切片变量 `c`，并将数组的长度赋值给切片的 `len` 和 `cap`。根据切片的定义，清晰明了。

#### 2.2.2 通过make创建切片
```
0x0032 00050 (main.go:9)        LEAQ    type.int(SB), AX
0x0039 00057 (main.go:9)        MOVQ    AX, (SP)
0x003d 00061 (main.go:9)        MOVQ    $10, 8(SP)
0x0046 00070 (main.go:9)        MOVQ    $10, 16(SP)
0x004f 00079 (main.go:9)        PCDATA  $1, $0
0x004f 00079 (main.go:9)        CALL    runtime.makeslice(SB)
0x0054 00084 (main.go:9)        MOVQ    24(SP), AX
0x0059 00089 (main.go:9)        MOVQ    AX, "".b+352(SP)
0x0061 00097 (main.go:9)        MOVQ    $10, "".b+360(SP)
0x006d 00109 (main.go:9)        MOVQ    $10, "".b+368(SP)

func makeslice(et *_type, len, cap int) unsafe.Pointer {
    // 检查内存
    mem, overflow := math.MulUintptr(et.size, uintptr(cap))
    if overflow || mem > maxAlloc || len < 0 || len > cap {
        mem, overflow := math.MulUintptr(et.size, uintptr(len))
        if overflow || mem > maxAlloc || len < 0 {
            panicmakeslicelen()
        }
        panicmakeslicecap()
    }

    return mallocgc(mem, et, true)
}
```
可以看到就是简单的将类型大小"乘以"容量参数得到需要的内存。然后分配内存。之后将 `len` 信息和 `cap` 信息存储到分配好的内存的+8和+16处，形成了定义的slice结构体。内存中即为8字节的数组指针，两个8字节的 `int` 类型。  
**注意：不是整个数组再+两个int类型的一整块内存，刚开始理解错误以为是分配内存是类型大小*cap+16字节。可以通过unsafe包进行内存模型的验证。见最后补充**

### 2.3 slice的扩容
在使用切片的过程中会遇到容量不够的情况，切片就会自动扩容（调用 `append` 函数）。在编译过程中 `append` 函数会分析是否超过容量，不超过直接将数据拷贝到数组对应位置并增加长度，否则调用下面的函数进行扩容：
```
// 参数cap 是调用append的时候 计算出来的所需的容量，比如当前slice的len和cap都为10，调用 append(slice, 1)，此函数cap就是11。调用 append(slice, 1 ,2)，此函数cap就是12
func growslice(et *_type, old slice, cap int) slice {
    if raceenabled {
        callerpc := getcallerpc()
        racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
    }
    if msanenabled {
        msanread(old.array, uintptr(old.len*int(et.size)))
    }

    if cap < old.cap {
        panic(errorString("growslice: cap out of range"))
    }

    if et.size == 0 {
        // 类型长度为0，特殊处理，底层数组不用扩容
        return slice{unsafe.Pointer(&zerobase), old.len, cap}
    }

    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.cap < 1024 {
            // 小于1k 简单扩容2倍
            newcap = doublecap
        } else {
            // 否则每次增加过去的 1/4 直到满足所需容量。 0 < newcap 为了检查溢出
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            // 溢出的话就用所需的cap
            if newcap <= 0 {
                newcap = cap
            }
        }
    }

    var overflow bool
    var lenmem, newlenmem, capmem uintptr
    // 这里是做了优化，分别对类型长度为1，系统指针长度，2的次方分别做了优化。1的时候不用乘除法，系统指针长度编译器会优化为静态的位运算，2的次方通过位运算处理。
    switch {
    case et.size == 1:
        lenmem = uintptr(old.len)
        newlenmem = uintptr(cap)
        capmem = roundupsize(uintptr(newcap))
        overflow = uintptr(newcap) > maxAlloc
        newcap = int(capmem)
    case et.size == sys.PtrSize:
        lenmem = uintptr(old.len) * sys.PtrSize
        newlenmem = uintptr(cap) * sys.PtrSize
        capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
        overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
        newcap = int(capmem / sys.PtrSize)
    case isPowerOfTwo(et.size):
        var shift uintptr
        if sys.PtrSize == 8 {
            // Mask shift for better code generation.
            shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
        } else {
            shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
        }
        lenmem = uintptr(old.len) << shift
        newlenmem = uintptr(cap) << shift
        capmem = roundupsize(uintptr(newcap) << shift)
        overflow = uintptr(newcap) > (maxAlloc >> shift)
        newcap = int(capmem >> shift)
    default:
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
        capmem = roundupsize(capmem)
        newcap = int(capmem / et.size)
    }

    // 检查内存溢出
    if overflow || capmem > maxAlloc {
        panic(errorString("growslice: cap out of range"))
    }

    var p unsafe.Pointer
    if et.ptrdata == 0 {
        p = mallocgc(capmem, nil, false)
        // 如果类型中不包含指针，清空cap - len部分的数据
        memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
    } else {
        p = mallocgc(capmem, et, true)
        if lenmem > 0 && writeBarrier.enabled {
            // 写屏障相关
            bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
        }
    }
    // 旧的数组数据移动到新的内存
    memmove(p, old.array, lenmem)

    return slice{p, old.len, newcap}
}
```
可以看到只要走到这个函数，那么就会重新分配一个更大的，符合我们所需的内存。因此我们使用 `append` 函数，一定要 `slice = append(slice, x)` 这样子来使用，否则可能旧的内存已经失效（gc可能给你回收掉了）  
如果切片是通过其它切片创建的，那么在未扩容的时候，更改新的切片会使老的切片也被改变。因此要注意扩容需要再次赋值，没扩容，老的切片要小心数据的使用（并发或者不是预期的值）

## 3. 总结
`slice` 是我们写代码过程中一定会用到，并且使用及其频繁的，因此对于切片中的一些细节要掌握，可以避免写代码过程中出现错误。虽然整个 `slice` 的实现比较简单易懂，但是扒汇编的过程中还是学到不少的。`slice` 中的数组可能会在堆上，但是 `slice` 本身这个结构体以及其属性的值还是在栈上的，非常容易迷惑。

### 3.1 验证内存模型补充
```
func main() {
    var b []int = make([]int, 10, 15)
    for i := 0; i < 10; i++ {
        b[i] = i
    }
    d := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&b)) + 8))
    f := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&b)) + 16))
    fmt.Println(d)
    fmt.Println(f)
}
// 输出：10 15
```

### 3.2 一点小补充
```
//go:noinlne
func test() {
    var s1 []int
    s2 := []int{}
    s3 := make([]int, 0, 0)
    fmt.Println(unsafe.Sizeof(s1), unsafe.Sizeof(s2), unsafe.Sizeof(s3))
    fmt.Println(s1, s2, s3)
}
```
思考一下哪种方式更好？扒汇编！扒汇编！扒汇编！
