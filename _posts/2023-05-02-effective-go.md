---
title: effective go 
date: 2023-05-02 13:00:00 +0800
categories: [Notes, Golang]
tags: [Golang]
img_path: /assets/img/golang/
---

忽然发现自己写了两年 go ，当年草草看了几眼匆匆上手就开始写了，还从没有系统学过go语言的设计，也没仔细思考过 go 的优势劣势。

想到侯捷老爷子讲STL时提到林语堂的一句：“使用一个东西，却不明白它的道理，实在不高明！”

确实如此，所幸现在有时间学习整理，那就开个坑记录一下。

![roadmap](golang-developer-roadmap.png)

## 语言设计

> Less can be more 大道至简,小而蕴真

Go 语言由 google 推出，其目的是解决软件开发难度过大的问题。所以 Go 语言拥有简洁的语法规则，并具有自动垃圾回收机制，与此同时还能够充分发挥出多核处理器的并行能力。

虽然 Go 的语法是从C/C++简化改进而来，但差别非常巨大：
- 语法层面支持并发/并行，并提供了channel作为通信机制
- 内存管理由GC自动完成，不需要开发者手动管理内存
- 内存模型更简化，只有值类型和引用类型，没有指针类型
- 舍弃了类、多态和继承等面向对象的概念，提供了接口，采用组合的方式实现代码复用

## 并发支持

go 语言最核心的特点之一就是并发编程，它的并发编程模型是 goroutine + channel。

goroutine 本质上是 golang 通过运行时提供的一个‘线程池’，也就是说，golang runtime 同时维护着多个内核工作线程，并实现了一套调度系统将 goroutine 分配到这些内核线程上执行。这套机制名为 GPM 模型。

### G: goroutine

操作系统提供的线程一般都有2MB的栈空间，而 goroutine 初始状态只有2KB的栈空间，并且处于用户态由runtime进行调度，所以 goroutine 的创建、销毁、调度和上下文切换都非常轻量级。

goroutine 的栈空间能够支持栈增长，最大可达到1GB，所以不用担心栈溢出问题。

由于绝大多数的 goroutine 都不会消耗太多的栈空间，因此go在这里的设计是非常合理的。

### M: kernel-level threads

M 指的是内核级线程，每个M都有自己独有的g0，负责调度、发起系统调用等任务。

### P: logical processor (actually a task queue)

P 指的是逻辑处理器，它的核心是一个goroutine队列，用于存放等待执行的goroutine。早期的golang版本中并没有P这一层抽象，所有的g都放在一个公共的队列中，这样就会由于大量的资源争用导致性能下降，而P的出现就可以解决这一问题。

### scheduler

- work stealing 机制：当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程。

- hand off 机制：当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行。

- 利用并行：GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行。GOMAXPROCS 也限制了并发的程度，比如 GOMAXPROCS = 核数/2，则最多利用了一半的 CPU 核进行并行。

- 抢占：在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这是 goroutine 不同于 coroutine 的一个地方。
  - stackPreempt 机制：在函数调用时，栈空间需要自然增长，golang 在每一个函数头部都插入了栈增长检查的代码，当栈空间不足时，会触发栈扩容。当调度器希望某个goroutine让出cpu时，就会将这个 G 的 stackguard 设置为一个标志值，从而在函数调用的栈增长检查时主动让出cpu。（过于依赖栈增长，无法解决死循环占用cpu的问题）
  - 异步抢占：在 Go 1.14 中引入了异步抢占，当一个 goroutine 执行时间过长时，sysmon 会向协程关联的 M 发送 sigPreempt 信号，M 会进入sigHandler流程，然后让出 CPU。

全局 G 队列：在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G。

### channel

channel 是 goroutine 之间的通信机制，它的本质是一个队列，遵循先进先出的原则。

```go
type hchan struct {
    qcount   uint           // 队列中的元素个数
    dataqsiz uint           // 队列的容量
    buf      unsafe.Pointer // 队列的指针
    elemsize uint16         // 队列中元素的大小
    closed   uint32         // channel是否已关闭
    elemtype *_type         // channel中元素的类型
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 接收等待队列
    sendq    waitq          // 发送等待队列
    lock mutex              // 互斥锁
}

ch := make(chan int, 5)

ch <- 1 // 阻塞式发送

select {
case ch <- 1:
    // 非阻塞式发送
default:
    // ...
}

<- ch // 阻塞式接收
v := <- ch // 阻塞式接收
v, ok := <-ch // 阻塞式接收，ok 为false时表示channel已关闭

select {
case v := <- ch:
    // 非阻塞式接收
default:
    // ...
}
```

`select` 语法支持多路复用，可以同时等待多个 channel 的读写操作，当多个 case 同时满足时，会按顺序给chan加锁（避免死锁），然后乱序轮询case来确定哪一个分支可以操作。

假如所有channel都不可操作，那么select会阻塞，当前协程会被添加到所有channel的sendq/recvq中，直到至少有一个channel可操作。

## 内存分配与GC

### 内存分配

golang 编译器如果不能在编译期确认数据对象的大小或者生命周期会超出当前函数，那么就会将这个对象分配在堆上。

### GC

目前主流的垃圾回收算法都是使用“可达性”来近似等于“存活性”，即从根对象出发，能够到达的对象都是存活的，不能到达的对象都是垃圾。

golang 采用的是三色标记法，将所有对象分为三种颜色：白色、黑色和灰色。

- 白色：未被 GC 标记的对象，可能是垃圾，也可能是存活对象；当 GC 标记阶段结束后，白色对象就是垃圾。
- 黑色：已被 GC 标记的对象，是存活对象。
- 灰色：已被 GC 标记的对象，但是其引用的对象还没有被标记，是存活对象。

标记清理算法实现比较简单，但是容易造成内存碎片化，面对大量不连续的小块内存，要找到一块合适的内存的代价更高。
- 这一问题可以基于BiBOP的思想解决：将内存块划分为多种大小规格，对相同规格的内存块进行统一管理。
- 也有人提出增加compact流程来解决该问题：即将存活内存移动到一起。
  - 有效的解决了内存碎片化的问题
  - 但是多次扫描开销和数据移动开销非常大
- 复制式回收：将堆内存分为两块，每次只使用其中一块，当这一块内存用完了，就将存活对象复制到另一块上，然后清理当前块的所有对象。
  - 这种方案也需要移动数据，能解决内存碎片化问题
  - 但内存利用率不高，只有一半的内存是可用的（通常会跟其他垃圾回收算法搭配使用）
- 分代回收：基于弱分代假说，即大部分对象都在年轻时死亡----新生代对象成为垃圾的概率高于老年代对象，因此可以降低老年对象进行垃圾回收扫描的频率，不用每次扫描全部对象
  - 对于新生代对象可以采用复制式回收算法

引用计数法：（与C++ RAII 同理）
- 可以及时回收无用内存
- 高频的更新引用计数会造成较大的额外开销
- 如果出现循环引用，会导致内存泄漏

前面介绍了两种常见的 GC 算法，那么GC过程中程序需要暂停运行（STW）吗？

golang 可以通过增量式垃圾回收来将 STW 的时间分摊到多次 GC 过程中，从而减少单次 GC 的 STW 时间。

实现增量式垃圾回收则需要避免在交替执行 GC 和用户程序时，用户程序修改了 GC 标记的对象，从而导致 GC 无法正确标记对象。
- 强三色不变式：避免出现黑色对象到白色对象的引用
- 弱三色不变式：允许出现黑色对象到白色对象的引用，但保证通过灰色对象可以抵达白色对象

实现强/弱三色不变式的通常做法是建立读写屏障：
- 插入写屏障：在写操作中插入指令，目的是在用户程序修改对象时能够通知给GC，因此有一个记录集来记录这些修改操作，GC 在扫描时会遍历这个记录集，检查是否有白色指针向黑色对象的写入操作，如果有则将这个白色指针标记为灰色/或者将黑色对象标记为灰色。
- 删除写屏障：要删除灰色对象到白色对象的引用时，可以把白色对象标记为黑色。
- 读屏障：非移动式GC不需要读屏障，移动式GC需要读屏障来避免用户程序读取到移动过程中的老对象。

多核场景下如何做到并行垃圾回收？如何做到并发垃圾回收？

## 常用框架

## 常见问题

## 工具链

https://blog.laisky.com/p/go-perf/

## 基础语法简记

### 内置类型、函数、运算符等

- 内置类型
  - 值类型：bool int int8 int16 int32 int64 uint uint8 uint16 uint32 uint64 byte float32 float64 complex64 complex128 string array
    - 注意，array也属于值类型，是固定长度的数组
  - 引用类型：slice map channel
- 内置函数：make new len cap append copy delete close panic recover real imag print println
- 内置接口：error
- 内置运算符
  - 算术运算符：`+ - * / %`
  - 关系运算符：`== != > < >= <=`
  - 逻辑运算符：`&& || !`
  - 位运算符：`& | ^ << >>`
  - 赋值运算符：`= += -= *= /= %= <<= >>= &= ^= |=`

### 变量/常量

### iota 技巧

iota:

```go
const (
  _  = iota
  KB = 1 << (10 * iota)
  MB = 1 << (10 * iota)
  GB = 1 << (10 * iota)
  TB = 1 << (10 * iota)
  PB = 1 << (10 * iota)
  )
```

### 指针类型

Go 语言中的指针更像是 C/C++ 中的引用，指针不能进行偏移和运算，是安全指针。

除了代码中声明的指针外，slice、map、channel都是指针。

### array/slice/map

数组的长度必须是常量，并且是数组类型的组成部分，不可动态变化（string底层也是一个byte的数组）。

golang 中的数组是值类型，而slice是引用类型。值类型在赋值和传参的时候会整个复制，而不是传递指针。

由于 array 是值类型，所以 array 是支持 `== !=` 操作符的（think more：长度不同的数组作比较能正确编译吗？）。

切片是数组的一个变长引用，cap可以求出slice最大扩张容量，不能超出数组限制。0 <= len(slice) <= len(array)，其中array是slice引用的数组。

```go
var arr [3]*int // 指针数组的元素是 int 指针
var arr *[3]int // 数组指针的元素是[3]int数组类型

var arr = [...]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
var slice0 []int = arr[2:8]

var slice []type = make([]type, len)
slice  := make([]type, len)
slice  := make([]type, len, cap)

s1 := []int{0, 1, 2, 3, 8: 100} // 通过初始化表达式构造，可使用索引号。
```

append函数会智能地处理底层数组的容量增长,当len超出cap限制时就会重新分配底层数组。

### struct & method

Golang 中的结构体和 C/C++ 中的结构体非常相似，并且 Go 没有引入 class 的概念，也不支持 class 的继承等面向对象概念，只有 struct。

Golang 主要是通过结构体的组合来实现代码复用和多态等特性，比起面向对象更简洁且扩展性和灵活性都更好。

Golang 中method的接收者可以是指针类型也可以是值类型，指针类型的接收者可以改变对象的值。而值类型的接收者就跟常规的值传递一样，只是传递的副本（会发生复制）而已，无法修改原对象的值。

### select

```go
//比如在下面的场景中，使用全局resChan来接受response，如果时间超过3S,resChan中还没有数据返回，则第二条case将执行
var resChan = make(chan int)
// do request
func test() {
    select {
    case data := <-resChan:
        doData(data)
    case <-time.After(time.Second * 3):
        fmt.Println("request time out")
    }
}

func doData(data int) {
    //...
}
```

### 匿名函数 & 闭包

Golang 闭包的变量捕获采用的是引用捕获（不能采用其他方式捕获）。

闭包的实现是非常依赖GC的，被捕获的局部变量是在堆上分配的（go编译器会检查局部变量的生命周期，如果函数结束后局部变量还可能被访问，那么就需要在堆上分配了）。

### 接口

空接口：

```go
var v1 interface{}
v1, _ = os.Open("a.txt")

runtime.eface: // empty interface
type eface struct {
    _type *_type  --------> *os.File 的类型元数据,可以找到方法元数据数组
    data  unsafe.Pointer -> *os.File 的指针
}
```

非空接口：

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// 可复用的接口类型元数据，用哈希表缓存
type itab struct {
    inter *interfacetype
    _type *_type  ----> *os.File // 接口的动态类型元数据
    hash  uint32  // 类型哈希值，用于快速比较两个类型是否相等，拷贝自_type
    _     [4]byte
    fun   [1]uintptr // 方法地址数组

type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod // 接口要求的方法集合
}
```
