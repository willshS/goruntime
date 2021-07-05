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
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```
