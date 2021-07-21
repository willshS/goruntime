# go flag
flag包是命令行解析工具  
## 1. 使用
以下是煎鱼大佬和p神的go语言编程之旅介绍flag使用的源码  
***以下默认编译为main，使用go run挺别扭的***
1. 直接使用flag包解析命令行参数
```
var name string
flag.StringVar(&name, "name", "Go语言变成之旅", "帮助信息")
flag.StringVar(&name, "n", "Go语言变成之旅", "帮助信息")
flag.Parse()
```
`./main -name sherock`进行测试
2. 定义子命令
```
var name string
flag.Parse()
args := flag.Args()
if len(args) <= 0 {
    fmt.Println(len(args))
    return
}
switch args[0] {
case "go":
    goCmd := flag.NewFlagSet("go", flag.ExitOnError)
    goCmd.StringVar(&name, "name", "Go语言", "帮助信息")
    goCmd.Parse(args[1:])
case "php":
    phpCmd := flag.NewFlagSet("php", flag.ExitOnError)
    phpCmd.StringVar(&name, "n", "PHP语言", "帮助信息")
    phpCmd.Parse(args[1:])
}
```
`./main go -name sherock` `./main php -n sherock` 去掉./是不是跟go run/build很像,run和build也是go命令的子命令，就像上面go和php是main的子命令  

## 2. 实现
flag包有一个全局变量 `CommandLine = NewFlagSet(os.Args[0], ExitOnError)`，跟我们定义子命令一模一样，都是 `FlagSet` 对象，`flag.Parse()` 就是使用 `CommandLine` 来进行命令行解析
```
type FlagSet struct {
    Usage func() // 解析失败的时候调用
    name          string // flagset的名字
    parsed        bool // 是否解析完成
    actual        map[string]*Flag // 标识赋值集合
    formal        map[string]*Flag // 标识绑定集合
    args          []string // 剩下没解析的args
    errorHandling ErrorHandling // 错误处理
    output        io.Writer // nil means stderr; use Output() accessor
}
```
### 2.1 变量绑定
解析命令行前我们需要对命令行可能会有的标识进行配置，不同类型的变量使用不同的接口，下面是 `int` 类型的变量绑定。
```
// p 是我们将命令行解析出来的数据存放的地址，name是要解析的标识，value是默认值，usage是帮助信息
// 将默认值value先赋值给要绑定的变量p
func (f *FlagSet) IntVar(p *int, name string, value int, usage string) {
    f.Var(newIntValue(value, p), name, usage)
}

// 例： ./main -myint=5。 那么程序就会对命令行中出现的-myint的值进行解析，赋值给i
var i int
flag.IntVar(&i, "myint", 0, "int类型变量")
```
`newIntValue` 类型为如下：
```
// 其实就是int类型的别名
type intValue int
// 简单的进行赋值
func newIntValue(val int, p *int) *intValue {
    *p = val
    return (*intValue)(p)
}
// Set接口
func (i *intValue) Set(s string) error {
    v, err := strconv.ParseInt(s, 0, strconv.IntSize)
    if err != nil {
        err = numError(err)
    }
    *i = intValue(v)
    return err
}
// Get接口
func (i *intValue) Get() interface{} { return int(*i) }

func (i *intValue) String() string { return strconv.Itoa(int(*i)) }
```
然后我们再来看var函数的实现：
```
type Flag struct {
    Name     string // 标识名字
    Usage    string // 帮助信息
    Value    Value  // value
    DefValue string // 默认值，string类型的，因此value接口要有String()函数
}

type Value interface {
    String() string
    Set(string) error
}

func (f *FlagSet) Var(value Value, name string, usage string) {
    // 这里先生成一个Flag对象，注意我们的intValue到这里是Value类型了，Value是一个接口
    flag := &Flag{name, usage, value, value.String()}
    // 检查变量是否已经在标识集合中存在了
    _, alreadythere := f.formal[name]
    if alreadythere {
        // 已经存在就报错
        var msg string
        if f.name == "" {
            msg = fmt.Sprintf("flag redefined: %s", name)
        } else {
            msg = fmt.Sprintf("%s flag redefined: %s", f.name, name)
        }
        fmt.Fprintln(f.Output(), msg)
        panic(msg) // Happens only if flags are declared with identical names
    }
    // 第一个绑定的变量
    if f.formal == nil {
        f.formal = make(map[string]*Flag)
    }
    // 存储绑定的变量
    f.formal[name] = flag
}
```
以上是变量绑定，涉及的非常多，首先 `FlagSet` 中的 `formal` 字段是用来保存标识和标识值的。其次，标识 `Flag` 对象保存了这个标识用户设定的信息，`value` 字段是一个接口，因此我们无论是 `int` 类型还是 `string` 类型或其它类型，都可以表示为 `Flag`，只要我们实现了这个接口。因此我们也可以自定义类型来进行**变量绑定**，类似于 `intValue`，只要我们实现了 `Value` 接口，我们的类型与 `intValue` 从语法来讲并没有任何区别。
## 3. 解析
变量绑定仅仅是将标识信息，默认值，帮助信息注册到标识集合 `FlagSet` 中，绑定结束后，我们需要从命令行对应的标识读取到用户输入的数据，赋值给程序的变量，这样才形成一个完整的逻辑。
```
func (f *FlagSet) Parse(arguments []string) error {
    f.parsed = true
    f.args = arguments
    // 循环分析所有参数，直至结束，结束标志就是err
    for {
        seen, err := f.parseOne()
        if seen {
            continue
        }
        if err == nil {
            break
        }
        // 根据标识集合的错误处理，做不同的处理
        switch f.errorHandling {
        case ContinueOnError:
            return err
        case ExitOnError:
            if err == ErrHelp {
                os.Exit(0)
            }
            os.Exit(2)
        case PanicOnError:
            panic(err)
        }
    }
    return nil
}

func (f *FlagSet) parseOne() (bool, error) {
    // 参数分析完毕了。返回err == nil 告诉上层函数
    if len(f.args) == 0 {
        return false, nil
    }
    // 获取第一个参数，标识一定是 '-' 开头，比如 "-name" 或 "--name"
    // 如果不符合规则，err == nil 后面不解析了
    s := f.args[0]
    // 长度小于2 最多有个 '-' 标识名字都没有
    if len(s) < 2 || s[0] != '-' {
        return false, nil
    }
    numMinuses := 1
    if s[1] == '-' {
        // 对应 "--name" 这种情况
        numMinuses++
        if len(s) == 2 { // "--" terminates the flags
            f.args = f.args[1:]
            return false, nil
        }
    }
    // -减号后面的字符串取出来，如果还有-，就3个减号了，如果是 = 就没有标识名字了
    name := s[numMinuses:]
    if len(name) == 0 || name[0] == '-' || name[0] == '=' {
        return false, f.failf("bad flag syntax: %s", s)
    }

    // 这里确定参数列表中的第一个肯定是个标识了，下面判断这个字符串中是否有值，"--name=sherock"
    // args推进
    f.args = f.args[1:]
    hasValue := false
    value := ""
    for i := 1; i < len(name); i++ { // equals cannot be first
        // 通过等号判断是否有值
        if name[i] == '=' {
            value = name[i+1:]
            hasValue = true
            name = name[0:i]
            break
        }
    }
    m := f.formal
    // 看标识集合中我们是否绑定了这个标识
    flag, alreadythere := m[name] // BUG
    if !alreadythere {
        // 特殊处理一下help，不需要我们自己去绑定
        if name == "help" || name == "h" { // special case for nice help message.
            f.usage()
            return false, ErrHelp
        }
        return false, f.failf("flag provided but not defined: -%s", name)
    }
    // 这里就取出来我们绑定的flag了，这里特殊处理一下bool类型 "--bool" 可以不用值
    if fv, ok := flag.Value.(boolFlag); ok && fv.IsBoolFlag() {
        if hasValue {
            if err := fv.Set(value); err != nil {
                return false, f.failf("invalid boolean value %q for -%s: %v", value, name, err)
            }
        } else {
            if err := fv.Set("true"); err != nil {
                return false, f.failf("invalid boolean flag %s: %v", name, err)
            }
        }
    } else {
        // 不是bool变量，其它变量标识后面必须跟着值
        if !hasValue && len(f.args) > 0 {
            // 所以下一个参数就是上面标识的值了，顺便推进args
            hasValue = true
            value, f.args = f.args[0], f.args[1:]
        }
        // 参数不够
        if !hasValue {
            return false, f.failf("flag needs an argument: -%s", name)
        }
        // 把解析出来的string类型的value，通过接口的函数赋值
        if err := flag.Value.Set(value); err != nil {
            return false, f.failf("invalid value %q for flag -%s: %v", value, name, err)
        }
    }
    if f.actual == nil {
        f.actual = make(map[string]*Flag)
    }
    // actual 字段也出来了
    f.actual[name] = flag
    return true, nil
}
```
解析函数非常简单，就是对字符串进行解析，然后将结果赋值。这里可以看到赋值是通过 `Value` 接口的 `Set` 函数。那么我们自定义变量只要实现了 `Value` 接口就能使用flag包了。  
## 4. 总结
flag包内容基本就这么多了，还有一些辅助的函数下面总结一下，有兴趣的自行了解。  
`FlagSet` ：
`Usage` 这个其实也讲到了，每次解析错误调用，failf函数的时候都会调用到这里，打印出帮助信息，flag包也提供了默认的帮助信息打印，通过绑定的集合，打印标识和默认值。参见 `PrintDefaults`  
`name` 标识集合的名字，唯一表示一个标识集合，如果我们有两个同样的标识集合名字会发生什么？  
`parsed` 解析标志  
`actual` 所有调用接口的`Set`的标识集合  
`formal` 所有标识集合，我们绑定的就是所有  
`args` 发生错误时剩下的没分析完的  
`errorHandling` 错误句柄，`Parse` 函数中很明显三种句柄不同的结果  
`output` `io.Writer` 错误输出位置  

### 4.1 一点小补充
`name` 相同会发生什么？结合上面的分析和下面的代码就明白了
```
var name string
var name1 string

func main() {
    a := []string{"--name", "sherock"}
    goCmd := flag.NewFlagSet("go", flag.ContinueOnError)
    goCmd.StringVar(&name, "name", "gogogo", "help")
    goCmd.Parse(a)
    phpCmd := flag.NewFlagSet("go", flag.ContinueOnError)
    phpCmd.StringVar(&name1, "n", "phpphpphp", "help")
    phpCmd.Parse(a)
    fmt.Println("name: " + name)
}
```
`output` 是有接口可以通过 `SetOutput` 设置的，这样我们可以自定义输出位置。
### 4.2 两点小补充
自定义变量
```
type myFlag struct {
    a int
    b string
}

func (f *myFlag) Set(value string) error {
    f.a = 1
    f.b = value
    return nil
}

func (f *myFlag) String() string {
    return fmt.Sprintf("myFlag a(%d) b(%s)", f.a, f.b)
}

func main() {
    var f myFlag
    flag.Var(&f, "myflags", "mmmmhelp")
    flag.Parse()
    fmt.Println(f)
}

//output: {1 hello}
```
读源码的时候，flag包文件夹下面的test文件可以结合着看。
