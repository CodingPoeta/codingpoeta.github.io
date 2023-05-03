---
title: effective go 
date: 2023-05-02 13:00:00 +0800
categories: [Blogging, Golang]
tags: [Golang]
img_path: /assets/img/golang/
---

忽然发现自己写了两年 go ，当年草草看了几眼匆匆上手就开始写了，还从没有系统学过go语言的设计，也没仔细思考过 go 的优势劣势。

想到侯捷老爷子讲STL时提到林语堂的一句：“使用一个东西，却不明白它的道理，实在不高明！”

确实如此，所幸现在有时间学习整理，那就开个坑记录一下。

![roadmap](golang-developer-roadmap.png)

## 语言设计

> Less can be more 大道至简,小而蕴真

## 内存分配与GC

## 并发支持

## 常用框架

## 常见问题

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
