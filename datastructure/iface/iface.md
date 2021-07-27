# interface
接口是 `go` 中很重要的一个类型。

## 1. 使用
TODO：指针类型方法和非指针类型方法 接口的赋值判断是否实现接口
接口的赋值都是编译器确定，当我们给一个接口赋值时，编译器会生成对应的数据赋值给接口：
```
type x interface {
	Mytest()
}

type tte struct {
    i int
}

func (t tte) Mytest() {
	fmt.Println("mytest")
}

//go:noinline
func test() {
	var j interface{}

	t := tte{1}

// 对于j = t，因为j是空接口，直接将数据类型信息与数据赋值给j
0x0026 00038 (main.go:24)	MOVQ	$1, ""..autotmp_3+16(SP)
0x002f 00047 (main.go:24)	LEAQ	type."".tte(SB), AX
0x0036 00054 (main.go:24)	MOVQ	AX, "".j+40(SP)
0x003b 00059 (main.go:24)	LEAQ	""..autotmp_3+16(SP), AX
0x0040 00064 (main.go:24)	MOVQ	AX, "".j+48(SP)
	j = t
	_ = j

// 对于 k = t，k为非空接口，与空接口数据不同，后面会介绍到
0x004d 00077 (main.go:28)	MOVQ	"".t(SP), AX
0x0051 00081 (main.go:28)	MOVQ	AX, ""..autotmp_4+8(SP)
0x0056 00086 (main.go:28)	LEAQ	go.itab."".tte,"".x(SB), AX
0x005d 00093 (main.go:28)	MOVQ	AX, "".k+24(SP)
0x0062 00098 (main.go:28)	LEAQ	""..autotmp_4+8(SP), AX
0x0067 00103 (main.go:28)	MOVQ	AX, "".k+32(SP)
	var k x
	k = t
	_ = k
}
```

## 2. 定义
内部实现有两个数据结构，分别为不包含方法的接口和包含方法的接口，但是接口大小都是16个字节：
```
// 非空接口
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype  // 接口类型信息
	_type *_type // 数据类型信息
	hash  uint32 // 数据类型哈希值
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}

type imethod struct {
	name nameOff
	ityp typeOff
}

type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}

// 空接口
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

### 2.1 空接口
空接口比较简单，一个是数据类型信息，一个是数据指针。假设有如下代码：
```
i := 1
var j interface{}
j = i
// _ = j
// fmt.Println(j)
```
上面代码非常简单，有兴趣的可以对注释的两行分别进行一下汇编，可以看到汇编出来的 `j = i` 这一行的代码并不相同，因为 `fmt.Println` 实现的原因，会对结果有非常大的影响。  
看以下代码：
```
// 这里把代码放出来，后面研究 fmt.Println 再来看 TODO：fmt
//go:noinline
func test1() {
	var j interface{}
	i := 1
	j = i
	*(*(**int)(unsafe.Pointer(uintptr(unsafe.Pointer(&j)) + 8))) = 2
	fmt.Println(i) // 2
	println(i)     // 1
	println(j)     // (0x4a2140,0x5406e8)
	fmt.Println(j) // 2

	var k interface{}
	x := 1
	fmt.Println(x) // 2   注：跟fmt有关
	println(x)     // 1   其实x还是1
	k = x
	fmt.Println(k) // 2
	println(k)     // (0x4a2140,0x5406e8)
}
```
可以看到空接口 `j` 与 `k` 是一模一样的地址。上面汇编的时候，当我们使用 `fmt.Println` 导致变量逃逸，因此使用了 `runtime.convT64` 函数。
```
func convT64(val uint64) (x unsafe.Pointer) {
	if val < uint64(len(staticuint64s)) {
		x = unsafe.Pointer(&staticuint64s[val])
	} else {
		x = mallocgc(8, uint64Type, false)
		*(*uint64)(x) = val
	}
	return
}
```
`staticuint64s` 其实是一个256大小数组，避免太小的数进行内存分配，那么我们上面 `test1()` 的代码所进行修改的正是这个数组，因此打印的 `k` 的值和地址都与 `j` 没有任何不同。  
因为数组最大255，那么我们把 `i` 改成256，可以看到跟上面不同的结果，以为 `else` 语句中重新分配了内存。 `runtime/iface.go` 中还有其它转换函数，根据各个内置类型的空变量尽量节省内存。**从上面实现中也可以看出，空接口中的数据与赋值给接口的数据没有任何关系，但是一定要注意fmt.Println导致的，而以为变量相同（上面的代码）**

## 非空接口
