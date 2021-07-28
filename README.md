read go source.  
源码之前，了无秘密  

## [IO](./IO)
###### [网络轮询器](./IO/netPoll/net.md)
###### [定时器](./IO/timer/timer.md)
因为定时器与网络轮询器联系紧密，放一块了。

## [GMP](./GMP)
###### [协程的调度](./GMP/sched.md)


## [datastructure](./datastructure)
#### **make三大件**
###### [channel](./datastructure/chan/chan.md)
###### [map](./datastructure/map/map.md)
###### [slice](./datastructure/slice/slice.md)


## [tool](./tool)
###### [flag](./tool/flag/flag.md)
###### [bufio](./tool/bufio/bufio.md)

## TODO
chan : 内存分配的方式（是否包含指针）& 栈收缩  
interface ： 编译器中接口的数据初始化 （类型信息处理）  
map : key的更新 needkeyupdate  
调度: gc相关  

## 调试
runtime代码大部分可以直接调试（我使用的vscode+gdb插件，直接断点调试）。多协程可能难以调试，但是可以通过修改runtime代码来打印日志进行调试（使用root权限改就行了）。
