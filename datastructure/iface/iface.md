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
	fun   [1]uintptr // 保存数据类型实现接口类型的方法，如果fun==0，则_type没有实现了inter类型的函数。
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
`staticuint64s` 其实是一个256大小数组，避免太小的数进行内存分配，那么我们上面 `test1()` 的代码所进行修改的正是这个数组，`fmt.Println` 函数也调用了 `convT64` ，因此 `x` 打印出来也是2，因此打印的 `k` 的值和地址都与 `j` 没有任何不同。  
因为数组最大255，那么我们把 `i` 改成256，可以看到跟上面不同的结果，以为 `else` 语句中重新分配了内存。 `runtime/iface.go` 中还有其它转换函数，根据各个内置类型的空变量尽量节省内存。**从上面实现中也可以看出，空接口中的数据与赋值给接口的数据没有任何关系。**

## 非空接口
可以从定义中看到，非空接口中多了一个接口类型信息。接口相关的数据都是编译期确定，需要深入编译器源码进行解读，这里我们只拿接口转换来解析：
```
// 根据汇编可以看到 inter 是要转换成的接口， i是待转换接口
func assertI2I(inter *interfacetype, i iface) (r iface) {
	tab := i.tab
	if tab == nil {
		// explicit conversions require non-nil interface value.
		panic(&TypeAssertionError{nil, nil, &inter.typ, ""})
	}
    // 类型一致
	if tab.inter == inter {
		r.tab = tab
		r.data = i.data
		return
	}
    // 通过要转换成的接口类型和待转换的数据类型生成一个itab
	r.tab = getitab(inter, tab._type, false)
	r.data = i.data
	return
}
```
在 `getitab` 中会对 `itab` 进行缓存，如果存在同样的，直接使用，否则会新分配一个 `itab` ，并进行初始化，在初始化过程中检查数据类型是否实现了接口类型：
```
func (m *itab) init() string {
	inter := m.inter
	typ := m._type
    // 数据类型的方法信息
	x := typ.uncommon()
    // 接口类型的方法个数
	ni := len(inter.mhdr)
    // 数据类型的方法个数
	nt := int(x.mcount)
	xmhdr := (*[1 << 16]method)(add(unsafe.Pointer(x), uintptr(x.moff)))[:nt:nt]
	j := 0
	methods := (*[1 << 16]unsafe.Pointer)(unsafe.Pointer(&m.fun[0]))[:ni:ni]
	var fun0 unsafe.Pointer
imethods:
    // 两层for循环检查
	for k := 0; k < ni; k++ {
		i := &inter.mhdr[k]
		itype := inter.typ.typeOff(i.ityp)
		name := inter.typ.nameOff(i.name)
		iname := name.name()
		ipkg := name.pkgPath()
		if ipkg == "" {
			ipkg = inter.pkgpath.name()
		}
		for ; j < nt; j++ {
			t := &xmhdr[j]
			tname := typ.nameOff(t.name)
            // 数据类型的方法 与 接口类型的方法 相同
			if typ.typeOff(t.mtyp) == itype && tname.name() == iname {
				pkgPath := tname.pkgPath()
				if pkgPath == "" {
					pkgPath = typ.nameOff(x.pkgpath).name()
				}
				if tname.isExported() || pkgPath == ipkg {
					if m != nil {
						ifn := typ.textOff(t.ifn)
						if k == 0 {
							fun0 = ifn // we'll set m.fun[0] at the end
						} else {
                            // 存储到m.fun0
							methods[k] = ifn
						}
					}
					continue imethods
				}
			}
		}
		// didn't find method
		m.fun[0] = 0
		return iname
	}
	m.fun[0] = uintptr(fun0)
	return ""
}
```
