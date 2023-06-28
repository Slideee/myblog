---
title: 《100 Go Mistakes How to Avoid Them》总结
date: 2023/06/28
updated: 2023/06/28
tag: Go
author: Slide
categories: Go
---

本文主要总结《100 Go Mistakes How to Avoid Them》上面常见的错误，旨在发现日常工作中的代码问题，规范Go代码编程。

<!--more-->

### 注意 shadow 变量

**Bad Code**

```
    var client *http.Client
    if tracing {
        client, err := createClientWithTracing()
        if err != nil {
            return err
        }
        log.Println(client)
    } else {
        client, err := createDefaultClient()
        if err != nil {
            return err
        }
        log.Println(client)
    }
```

在上面这段代码中，声明了一个 client 变量，然后使用 tracing 控制变量的初始化，可能是因为没有声明 err 的缘故，使用的是 `:=`进行初始化，那么会导致外层的 client 变量永远是 nil。这个例子实际上是很容易发生在我们实际的开发中，尤其需要注意。

**Good Code**

```
// 把判断语句中的声明语法换成赋值语法，同样在最外层声明err，可以做到减少代码量的效果
var client *http.Client
var err error
if tracing {
    client, err = createClientWithTracing()
} else {
    client, err = createDefaultClient()
}
if err != nil {
    return err
}
log.Println(client)
```

或者干脆内层的变量声明换一个变量名字，这样就不容易出错了。

### embed types优缺点

embed types 指的是我们在 struct 里面定义的匿名的字段，如：

```
type Foo struct {
    Bar
}
type Bar struct {
    Baz int
}
```

那么在上面这个例子中，我们可以通过 `Foo.Baz`直接访问到成员变量，当然也可以通过 `Foo.Bar.Baz`访问。

这样在很多时候可以增加我们使用的便捷性，如果没有使用 embed types 那么可能需要很多代码，如下：

```
type Logger struct {
        writeCloser io.WriteCloser
}

func (l Logger) Write(p []byte) (int, error) {
        return l.writeCloser.Write(p)
}

func (l Logger) Close() error {
        return l.writeCloser.Close()
}

func main() {
        l := Logger{writeCloser: os.Stdout}
        _, _ = l.Write([]byte("foo"))
        _ = l.Close()
}
```

如果使用了 embed types 我们的代码可以变得很简洁：

```
type Logger struct {
        io.WriteCloser
}

func main() {
        l := Logger{WriteCloser: os.Stdout}
        _, _ = l.Write([]byte("foo"))
        _ = l.Close()
}
```

但是同样它也有缺点，有些字段我们并不想 export ，但是 embed types 可能给给我们带出去，例如：

```
type InMem struct {
    sync.Mutex
    m map[string]int
}

func New() *InMem {
     return &InMem{m: make(map[string]int)}
}
```

Mutex 一般并不想 export， 只想在 InMem 自己的函数中使用，如：

```
func (i *InMem) Get(key string) (int, bool) {
    i.Lock()
    v, contains := i.m[key]
    i.Unlock()
    return v, contains
}
```

但是这么写却可以让拿到 InMem 类型的变量都可以使用它里面的 Lock 方法：

```
m := New()
m.Lock()
```

1. 简单来说，类型嵌入可以避免编写大量的模板代码，使用嵌入类型的原则是，如果不想暴露成员的访问权，就尽量不使用类型嵌入。
2. 嵌入接口的话，嵌入类型还能够默认实现嵌入接口。

### Functional Options Pattern传递参数

这种方法在很多Go开源库都有看到过使用，比如kubernetes等。

在很多场景下下，我们需要对对象的成员进行配置化，往往我们会在NewXXX函数中的参数中罗列配置化的值，这样就出现很多问题。

**Bad Code**

```
func NewServer(addr string, port int) (*http.Server, error) { 
    // ...
}
```

1. 如果不传递port的时候，我们希望使用默认值，但是这里port又必须要填写一个值
2. 额外增加参数的时候，会对函数签名有不兼容的修改

我们期望配置*http.Server的时候，可以做到下面几点：

1. 如果port没有设置，我们使用默认值
2. 如果port是负数，抛出错误，这很重要
3. 如果port是0，那我们就随机生成端口
4. 其他场景，我们使用默认值

**解决方案一：Config struct**

```
type Config struct {
		Port *int //如果不是指针类型，int为0时无法判断是否是用户配置
}

func NewServer(addr string, cfg Config) {
} 
```

```
port := 0
// 但是用户使用的时候需要传递指针类型给Config结构体，会给人造成困惑，
// 配置一个端口需要传递一个int指针，所以这种设计并不API友好
config := httplib.Config{
    Port: &port,
}
// 当我们不想传递配置的时候，也不行，必要给一个空的结构体
httplib.NewServer("localhost", httplib.Config{})
```

**解决方案二**：**Builder pattern**

```
type Config struct {
	Port int
}

type ConfigBuilder struct {
	port *int
}

func (b *ConfigBuilder) Port(port int) *ConfigBuilder {
	b.port = &port //使用函数封装用户传递int值转化为指针类型
	return b // 返回*ConfigBuilder，可以使用bulder.Foo("f").Bar("b")这种链式写法
}

func (b *ConfigBuilder) Build() (Config, error) { //在Build函数中检查参数合法性
	cfg := Config{}

	if b.port == nil {
		cfg.Port = defaultHTTPPort
	} else {
			// 检查Port的合法性
		if *b.port == 0 {
			cfg.Port = randomPort()
		} else if *b.port < 0 {
			return Config{}, errors.New("port should be positive")
		} else {
			cfg.Port = *b.port
		}
	}

	return cfg, nil
}

func NewServer(addr string, config Config) (*http.Server, error) {
	return nil, nil
}
```

```
func client() error {
	builder := ConfigBuilder{}
	builder.Port(8080)
	cfg, err := builder.Build()
	if err != nil {
		return err
	}

	server, err := NewServer("localhost", cfg) //这里仍然需要传递一个cfg
	if err != nil {
		return err
	}

	_ = server
	return nil
}
```

**解决方案三**：**Functional options pattern**

通过提供可选参数的函数式参数，使我们能够创建灵活的 API。

- 设置一个不导出的 struct 叫 options，用来存放配置参数；
- 创建一个类型 `type Option func(options *options) error`，用这个类型来作为返回值；

比如我们现在要给 HTTP server 里面设置一个 port 参数，那么我们可以这么声明一个 WithPort 函数，返回 Option 类型的闭包，当这个闭包执行的时候会将 options 的 port 填充进去：

```
type options struct {
        port *int
}

type Option func(options *options) error

func WithPort(port int) Option {
         // 所有的类型校验，赋值，初始化啥的都可以放到这个闭包里面做
        return func(options *options) error {
                if port < 0 {
                        return errors.New("port should be positive")
                }
                options.port = &port
                return nil
        }
}
```

假如我们现在有一个这样的 Option 函数集，除了上面的 port 以外，还可以填充timeout等。然后我们可以利用 NewServer 创建我们的 server：

```
func NewServer(addr string, opts ...Option) (*http.Server, error) {
        var options options
            // 遍历所有的 Option
        for _, opt := range opts {
                    // 执行闭包
                err := opt(&options)
                if err != nil {
                        return nil, err
                }
        }

        // 在这个阶段，构建了options结构体并包含了配置,
        // 接下来可以填充我们的业务逻辑，比如这里设置默认的port 等等
        var port int
        if options.port == nil {
                port = defaultHTTPPort
        } else {
                if *options.port == 0 {
                        port = randomPort()
                } else {
                        port = *options.port
                }
        }

        // ...
}
```

初始化 server：

```
server, err := httplib.NewServer("localhost",
                httplib.WithPort(8080),
                httplib.WithTimeout(time.Second)) 
```

这样写的话就比较灵活，如果只想生成一个简单的 server，我们的代码可以变得很简单：

```
server, err := httplib.NewServer("localhost") 
```

### slice 相关注意点

#### 区分 slice 的 length 和 capacity

首先让我们初始化一个带有 length 和 capacity 的 slice ：

```
s := make([]int, 3, 6)
```

在make 函数里面，capacity 是可选的参数。上面这段代码我们创建了一个 length 是 3，capacity 是 6 的 slice，那么底层的数据结构是这样的：

![image-20230612144605717](/images/mistakes/image-20230612144605717.png)

slice 的底层实际上指向了一个数组。当然，由于我们的 length 是 3，所以这样设置 `s[4] = 0` 会 panic 的。需要使用 append 才能添加新元素。

```
panic: runtime error: index out of range [4] with length 3
```

当 appned 超过 cap 大小的时候，slice 会自动帮我们扩容，在元素数量小于 1024 的时候每次会扩大一倍，当超过了 1024 个元素每次扩大 25%。

有时候我们会使用 `：`操作符从另一个 slice 上面创建一个新切片：

```
s1 := make([]int, 3, 6)
s2 := s1[1:3]
```

实际上这两个 slice 还是指向了底层同样的数组，构如下：

![image-20230612144750940](/images/mistakes/image-20230612144750940.png)



由于指向了同一个数组，那么当我们改变第一个槽位的时候，比如 `s1[1]=1`，实际上两个 slice 的数据都会发生改变：

![image-20230612144808338](/images/mistakes/image-20230612144808338.png)

但是当我们使用 append 的时候情况会有所不同：

```
s2 = append(s2, 2)

fmt.Println(s1) // [0 1 0]
fmt.Println(s2) // [1 0 2]
```



![image-20230612144844254](/images/mistakes/image-20230612144844254.png)

s1 的 len 并没有被改变，所以看到的还是3元素。

我们再继续 append s2 直到 s2 发生扩容，这个时候会发现 s2 实际上和 s1 指向的不是同一个数组了：

```
s2 = append(s2, 3, 4, 5)
fmt.Println(s1) //[0 1 0]	
fmt.Println(s2) //[1 0 2 3 4 5]
```

![image-20230612145212451](/images/mistakes/image-20230612145212451.png)

#### slice 初始化

对于 slice 的初始化实际上有很多种方式：

```go
func main() {
        var s []string
        log(1, s)

        s = []string(nil)
        log(2, s)

        s = []string{}
        log(3, s)

        s = make([]string, 0)
        log(4, s)
}

func log(i int, s []string) {
        fmt.Printf("%d: empty=%t\tnil=%t\n", i, len(s) == 0, s == nil)
}
```

输出：

```
1: empty=true   nil=true
2: empty=true   nil=true
3: empty=true   nil=false
4: empty=true   nil=false
```

前两种方式会创建一个 nil 的 slice，后两种会进行初始化，并且这些 slice 的大小都为 0 。

对于 `var s []string`这种方式来说，好处就是不用做任何的内存分配。比如下面场景可能可以节省一次内存分配：

```go
func f() []string {
        var s []string
        if foo() {
                s = append(s, "foo")
        }
        if bar() {
                s = append(s, "bar")
        }
        return s
}
```

对于 `s := []string{}`这种方式来说，它比较适合初始化一个已知元素的 slice：

```
s := []string{"foo", "bar", "baz"}
```

如果没有这个需求其实用 `var s []string` 比较好，反正在使用的适合都是通过 append 添加元素， `var s []string` 还能节省一次内存分配。

如果我们初始化了一个空的 slice， 那么最好是使用 `len(xxx) == 0` 来判断 slice 是不是空的，如果使用 nil 来判断可能会永远非空的情况，因为对于 `s := []string{}` 和 `s = make([]string, 0)`这两种初始化都是非nil的。

对于 `[]string(nil)` 这种初始化的方式，使用场景很少，一种比较方便的使用场景是用它来进行 slice 的 copy：

```go
src := []int{0, 1, 2}
dst := append([]int(nil), src...)
```

对于 make 来说，它可以初始化 slice 的 length 和 capacity，如果我们能确定 slice 里面会存放多少元素，从性能的角度考虑最好使用 make 初始化好，因为对于一个空的 slice append 元素进去每次达到阈值都需要进行扩容，下面是填充 100 万元素的 benchmark：

```
BenchmarkConvert_EmptySlice-4                22     49739882 ns/op
BenchmarkConvert_GivenCapacity-4             86     13438544 ns/op
BenchmarkConvert_GivenLength-4               91     12800411 ns/op
```

可以看到，如果我们提前填充好 slice 的容量大小，性能是空 slice 的四倍，因为少了扩容时元素复制以及重新申请新数组的开销。

#### copy slice

```
src := []int{0, 1, 2}
var dst []int
copy(dst, src)
fmt.Println(dst) // []
```

使用 copy 函数 copy slice 的时候需要注意，上面这种情况实际上会 copy 失败，因为对 slice 来说是由 length 来控制可用数据，copy 并没有复制这个字段，要想 copy 我们可以这么做：

```
src := []int{0, 1, 2}
dst := make([]int, len(src))
copy(dst, src)
fmt.Println(dst) //[0 1 2]
```

除此之外也可以用上面提到的：

```
src := []int{0, 1, 2}
dst := append([]int(nil), src...)
```

#### slice capacity内存释放问题

先来看个例子：

```go
type Foo struct {
    v []byte
}

func keepFirstTwoElementsOnly(foos []Foo) []Foo {
    return foos[:2]
}

func main() {
    foos := make([]Foo, 1_000)
    printAlloc()

    for i := 0; i < len(foos); i++ {
        foos[i] = Foo{
            v: make([]byte, 1024*1024),
        }
    }
    printAlloc()

    two := keepFirstTwoElementsOnly(foos)
    runtime.GC()
    printAlloc()
    runtime.KeepAlive(two)
}
```

上面这个例子中使用 printAlloc 函数来打印内存占用：

```go
func printAlloc() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("%d KB\n", m.Alloc/1024)
}
```

上面 foos 初始化了 1000 个容量的 slice ，里面 Foo struct 每个都持有 1M 内存的 slice，然后通过 keepFirstTwoElementsOnly 返回持有前两个元素的 Foo 切片，我们的想法是手动执行 GC 之后其他的 998 个 Foo 会被 GC 销毁，但是输出结果如下：

```
387 KB
1024315 KB
1024319 KB
```

实际上并没有，原因就是实际上 keepFirstTwoElementsOnly 返回的 slice 底层持有的数组是和 foos 持有的同一个：

![image-20230628103934187](/images/mistakes/image-20230628103934187.png)

所以我们真的要只返回 slice 的前2个元素的话应该这样做：

```go
func keepFirstTwoElementsOnly(foos []Foo) []Foo {
        res := make([]Foo, 2)
        copy(res, foos)
        return res
}
```

不过上面这种方法会初始化一个新的 slice，然后将两个元素 copy 过去。不想进行多余的分配可以这么做：

```go
//给元素中的指针成员置为nil，能够让GC自动回收
func keepFirstTwoElementsOnly(foos []Foo) []Foo {
        for i := 2; i < len(foos); i++ {
                foos[i].v = nil
        }
        return foos[:2]
}
```

### defer

#### 注意defer的调用时机

有时候我们会像下面一样使用 defer 去关闭一些资源：

```go
func readFiles(ch <-chan string) error {
            for path := range ch {
                    file, err := os.Open(path)
                    if err != nil {
                            return err
                    }

                    defer file.Close()

                    // Do something with file
            }
            return nil
} 
```

因为defer会在方法结束的时候调用，但是如果上面的 readFiles 函数永远没有 return，那么 defer 将永远不会被调用，从而造成内存泄露。

为了避免这种情况，我们可以 wrap 一层：

```go
func readFiles(ch <-chan string) error {
      for path := range ch { 
          if err := readFile(path); err != nil {
                  return err
          } 
      }
      return nil
} 

func readFile(path string) error {
      file, err := os.Open(path)
      if err != nil {
              return err
      }

      defer file.Close()

      // Do something with file
      return nil
} 
```

#### 注意defer的参数

defer 声明时会先计算确定参数的值。

```go
func a() {
    i := 0
    defer notice(i) // 0
    i++
    return
}

func notice(i int) {
  fmt.Println(i)
}
```

在这个例子中，变量 i 在 `defer`被调用的时候就已经确定了，而不是在 `defer`执行的时候，所以上面的语句输出的是 0。

所以我们想要获取这个变量的真实值，应该用引用：

```go
func a() {
    i := 0
    defer notice(&i) 
    i++
    return
}
```

### 注意range

#### copy 的问题

使用 range 的时候如果我们直接修改它返回的数据会不生效，因为返回的数据并不是原始数据：

```
type account struct {
	balance float32
}

func createAccounts() []account {
	return []account{
		{balance: 100.},
		{balance: 200.},
		{balance: 300.},
	}


func main() {
	accounts := createAccounts()
	for _, a := range accounts {
		a.balance += 1000
	}
	fmt.Println(accounts)
）
```

如果像上面这么做，那么输出的 accounts是：

```
[{100} {200} {300}]
```

所以我们想要改变 range 中的数据可以这么做：

```
for i := range accounts {
    accounts[i].balance += 1000
}

for i := 0; i < len(accounts); i++ {
    accounts[i].balance += 1000
}
```

如果range的是指针呢：

```
type account struct {
	balance float32
}

func createAccountsPtr() []*account {
	return []*account{
		{balance: 100.},
		{balance: 200.},
		{balance: 300.},
	}
}

func main() {
	accountsPtr := createAccountsPtr()
	for _, a := range accountsPtr {
		a.balance += 1000
	}
	printAccountsPtr(accountsPtr)
}
```

那么输出的 accounts是：

```
[{1100} {1200} {1300}]
```

#### 指针问题

```
type Customer struct {
    ID      string
    Balance float64
}

type Store struct {
    m map[string]*Customer
}

func main() {
    s := Store{
        m: make(map[string]*Customer),
    }
    s.storeCustomers([]Customer{
        {ID: "1", Balance: 10},
        {ID: "2", Balance: -10},
        {ID: "3", Balance: 0},
    })
    print(s.m)
}

// 问题来源：
func (s *Store) storeCustomers(customers []Customer) {
    for _, customer := range customers {
        // 每次迭代的时候 customer 都是同一个对象，只是 customer 的值会不断被新的值覆盖
        fmt.Printf("%p\n", &customer)
        // 最终map中的每个key都指向了同一个内存地址
        s.m[customer.ID] = &customer
    }
}
```

这是因为迭代器会把数据都放入到（0xc000096020 ）这块空间里面，所以取的实际上都是迭代器的指针：

![img](https://pic4.zhimg.com/v2-299e6ccad54b78d1b1cda953d296d463_r.jpg)

正确的遍历取指针应该是这样：

```go
//把每次迭代的值copy到一个新的临时变量
func (s *Store) storeCustomers2(customers []Customer) {
    for _, customer := range customers {
        current := customer
        s.m[current.ID] = &current
    }
}
```

### 注意break作用域

例如：

```go
for i := 0; i < 5; i++ {
      fmt.Printf("%d ", i)

      switch i {
      default:
      case 2:
              break
      }
  } 
```

上面这个代码本来想 break 停止遍历，实际上只是break 了 switch 作用域，print 依然会打印：0，1，2，3，4。

正确做法应该是通过 label 的方式break：

```go
loop:
    for i := 0; i < 5; i++ {
        fmt.Printf("%d ", i) 
        switch i {
        default:
        case 2:
            break loop
        }
    }
```

有时候我们会没注意到自己的错误用法，比如下面：

```go
	for {
		select {
		case <-ch:
			// Do something
		case <-ctx.Done():
			break
		}
	}
```

上面这种写法会导致只跳出了 select，并没有终止for循环，正确写法应该这样：

```go
loop:
	for {
		select {
		case <-ch:
			// Do something
		case <-ctx.Done():
			break loop
		}
	}
```

### string相关

#### 迭代带来的问题

在 Go 语言中，字符串是一种基本类型，默认是通过 utf8 编码的字符序列，当字符为 ASCII 码时则占用 1 个字节，其它字符根据需要占用 2-4 个字节，比如中文编码通常需要 3 个字节。

那么我们在做 string 迭代的时候可能会产生意想不到的问题：

```go
    s := "hêllo"
    for i := range s {
        fmt.Printf("position %d: %c\n", i, s[i])
    }
    fmt.Printf("len=%d\n", len(s))
```

输出：

```
position 0: h
position 1: Ã
position 3: l
position 4: l
position 5: o
len=6
```

上面的输出中发现第二个字符是 Ã，不是 ê，并且位置2的输出”消失“了，这其实就是因为 ê 在 utf8 里面实际上占用 2 个 byte：

| s         | h    | ê     | l    | l    | o    |
| :-------- | :--- | :---- | :--- | :--- | :--- |
| []byte(s) | 68   | c3 aa | 6c   | 6c   | 6f   |

所以我们在迭代的时候 s[1] 等于 c3 这个 byte 等价 Ã 这个 utf8 值，所以输出的是 `hÃllo` 而不是 `hêllo`。

那么根据上面的分析，我们就可以知道在迭代获取字符的时候不能只获取单个 byte，应该使用 range 返回的 value值：

```go
    s := "hêllo"
    for i, v := range s {
        fmt.Printf("position %d: %c\n", i, v)
    }
```

或者我们可以把 string 转成 rune 数组，在 go 中 rune 代表`Unicode`码位，用它可以输出单个字符：

```go
    s := "hêllo"
    runes := []rune(s)
    for i, _ := range runes {
        fmt.Printf("position %d: %c\n", i, runes[i])
    }
```

输出：

```
position 0: h
position 1: ê
position 2: l
position 3: l
position 4: o
```

#### 截断带来的问题

在上面我们讲 slice 的时候也提到了，在对slice使用 `：`操作符进行截断的时候，底层的数组实际上指向同一个，在 string 里面也需要注意这个问题，比如下面：

```go
func (s store) handleLog(log string) error {
            if len(log) < 36 {
                    return errors.New("log is not correctly formatted")
            }
            uuid := log[:36]
            s.store(uuid)
            // Do something
    } 
```

这段代码用了 `：`操作符进行截断，但是如果 log 这个对象很大，比如上面的 store 方法把 uuid 一直存在内存里，可能会造成底层的数组一直不释放，从而造成内存泄露。

为了解决这个问题，我们可以先复制一份再处理：

```go
func (s store) handleLog(log string) error {
            if len(log) < 36 {
                    return errors.New("log is not correctly formatted")
            }
            uuid := strings.Clone(log[:36])  // copy一份
            s.store(uuid)
            // Do something
    }
```

### 应多关注goroutine何时停止

很多同学觉得 goroutine 比较轻量，认为可以随意的启动 goroutine 去执行任何而不会有很大的性能损耗。这个观点基本没错，但是如果在 goroutine 启动之后因为代码问题导致它一直占用，没有停止，数量多了之后可能会造成内存泄漏。

比如下面的例子：

```go
ch := foo()
    go func() {
            for v := range ch {
                    // ...
            }
    }()
```

如果在该 goroutine 中的 channel 一直没有关闭，那么这个 goroutine 就不会结束，会一直挂着占用一部分内存。

还有一种情况是我们的主进程已经停止运行了，但是 goroutine 里面的任务还没结束就被主进程杀掉了，那么这样也可能造成我们的任务执行出问题，比如资源没有释放，抑或是数据还没处理完等等，如下：

```go
func main() {
            newWatcher()

            // Run the application
    }

    type watcher struct { /* Some resources */ }

    func newWatcher() {
            w := watcher{}
            go w.watch()
    }
```

上面这段代码就可能出现主进程已经执行 over 了，但是 watch 函数还没跑完的情况，那么其实可以通过设置 stop 函数，让主进程执行完之后执行 stop 函数即可：

```go
func main() {
            w := newWatcher()
            defer w.close()

            // Run the application
    }

    func newWatcher() watcher {
            w := watcher{}
            go w.watch()
            return w
    }

    func (w watcher) close() {
            // Close the resources
    }
```

### Channel

#### select & channel

select 和 channel 搭配起来往往有意想不到的效果，比如下面：

```
// 发送数据
for i := 0; i < 10; i++ {
       messageCh <- i
}
disconnectCh <- struct{}{}


for {
		select {
		case v := <-messageCh:
			fmt.Println(v)
		case <-disconnectCh:
			fmt.Println("disconnection, return")
			return
		}
}
```

上面代码中接受了 messageCh 和 disconnectCh 两个 channel 的数据，如果我们想要上面的 select 先输出完 messageCh 里面的数据，然后再 return，实际上可能会输出：

```
0
1
disconnection, return
```

解决方案可以是使用for-select嵌套：

```go
for {
		select {
		case v := <-messageCh:
			fmt.Println(v)
		case <-disconnectCh:
			for {
				select {
				case v := <-messageCh:
					fmt.Println(v)
				default:
					fmt.Println("disconnection, return")
					return
				}
			}
		}
}
```

在读取 disconnectCh 的时候里面再套一个循环读取 messageCh，读完了之后会调用 default 分支进行return。

#### 不要使用nil channel

使用 nil channel 进行收发数据的时候会永远阻塞，例如发送数据：

```go
var ch chan int
ch <- 0 //block
```

接收数据：

```go
var ch chan int
<-ch //block
```

### 错误使用 sync.WaitGroup

`sync.WaitGroup`通常用在并发中等待 goroutines 任务完成，用 Add 方法添加计数器，当任务完成后需要调用 Done 方法让计数器减一。等待的线程会调用 Wait 方法等待，直到`sync.WaitGroup` 内计数器为零。

需要注意的是 Add 方法是怎么使用的，如下：

```go
func listing1() {
	wg := sync.WaitGroup{}
	var v uint64

	for i := 0; i < 3; i++ {
		go func() {
			wg.Add(1)
			atomic.AddUint64(&v, 1)
			wg.Done()
		}()
	}

	wg.Wait()
	fmt.Println(v)
}

```

这样使用可能会导致 v 不一定等于3，因为在 for 循环里面创建的 3 个 goroutines 不一定比外面的主线程先执行，从而导致在调用 Add 方法之前可能 Wait 方法就执行了，并且恰好`sync.WaitGroup`里面计数器是零，然后就通过了。

正确的做法应该是在创建 goroutines 之前就将要创建多少个 goroutines 通过 Add 方法添加进去。

    func listing3() {
    	wg := sync.WaitGroup{}
    	var v uint64
    
    	for i := 0; i < 3; i++ {
    		wg.Add(1)
    		go func() {
    			atomic.AddUint64(&v, 1)
    			wg.Done()
    		}()
    	}
    
    	wg.Wait()
    	fmt.Println(v)
    }
### HTTP body忘记 Close 导致的泄露

```go
type handler struct {
        client http.Client
        url    string
}

func (h handler) getBody() (string, error) {
        resp, err := h.client.Get(h.url)
        if err != nil {
                return "", err
        }

        body, err := io.ReadAll(resp.Body)
        if err != nil {
                return "", err
        }

        return string(body), nil
}
```

上面这段代码看起来没什么问题，但是 resp 是 `*http.Response` 类型，里面包含了 `Body io.ReadCloser` 对象，它是一个 io 类，必须要正确关闭，否则是会产生资源泄露的。一般我们可以这么做：

```go
defer func() {
        err := resp.Body.Close()
        if err != nil {
                log.Printf("failed to close response: %v\n", err)
        }
}()
```

### 不要拷贝 sync 类型

sync 包里面提供一些并发操作的类型，如 mutex、condition、wait gorup 等等，这些类型都不应该被拷贝之后使用。这是因为这些类型通常需要在多个 goroutine 中共享，如果拷贝之后会导致共享状态的不一致性和竞态条件的出现。

有时候我们在使用的时候拷贝是很隐秘的，比如下面：

```go
type Counter struct {
    mu       sync.Mutex
    counters map[string]int
}

func (c Counter) Increment1(name string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counters[name]++
}

func NewCounter() Counter {
    return Counter{counters: map[string]int{}}
}

func main() {
    counter := NewCounter()
    go counter.Increment("aa")
    go counter.Increment("bb")
}
```

receiver 是一个值类型，所以调用 Increment 方法的时候实际上拷贝了一份 Counter 里面的变量。这里我们可以将 receiver 改成一个指针，或者将 `sync.Mutex` 变量改成指针类型。

所以如果：

1. receiver 是值类型；
2. 函数参数是 sync 包类型；
3. 函数参数的结构体里面包含了 sync 包类型；

遇到这种情况需要注意检查一下，我们可以借用 go vet 来检测，比如上面如果并发调用了就可以检测出来：

```
➜ go vet main.go 
# command-line-arguments
./main.go:30:9: Increment1 passes lock by value: command-line-arguments.Counter contains sync.Mutex
```

### receiver应该使用什么样的类型

提供几个场景供我们决定选择receiver的类型：

**一定要用指针类型的场景**

- 方法需要修改receiver的值
- 如果receiver的成员不能被拷贝

**应该使用指针类型的场景**

- 如果receiver是一个大的对象，使用指针类型可以更高效的处理程序，这里大的具体数值不好界定，需要看实际的场景，这里可以使用benchmark来估计。

**一定要使用值类型的场景**

- 如果强制要求receiver是不可变的
- 如果receiver是一个map、function、channel类型，否则会有编译错误

**应该使用值类型的场景**

- receiver是一个slice，并且一定需要修改
- receiver是一个array或者是一个没有可变字段的struct，如time.Time
- receiver是基本类型，例如int、float64或string。

**结构体组成复杂的场景**

```
type customer struct {
	data *data
}

type data struct {
	balance float64
}

func (c customer) add(operation float64) { //c虽然是值类型，但是c的成员为指针类型，指向成员的真实地址，实际上是可以直接修改成员的
	c.data.balance += operation
}

func main() {
	c := customer{data: &data{
		balance: 100,
	}}
	c.add(50.)
	fmt.Printf("balance: %.2f\n", c.data.balance)
}

```

- 这种场景建议receiver的类型设为指针类型，能明确的告诉使用者这个方法会修改成员的值

### Reference

https://yangsoon.github.io/100-go-mistakes-and-how-to-avoid-them--1

https://yangsoon.github.io/100-go-mistakes-and-how-to-avoid-them--2

https://www.luozhiyun.com/archives/797

