---
title: 《Go in action》读书笔记
layout: post
categories: 编程语言
---

# Go 语言学习笔记 

## 简介

* Go语言使用了更加智能的编译器，Go语言简化了解决依赖的算法，最终提供了更快的编译速度。
* goroutine 使用的内存比线程更少，Go语言运行时会自动在配置的一组逻辑处理器上调度执行。
* goroutine 之间使用 channel 通道传输数据，这种传输方式不需要任何锁或者同步机制，但是如果传输的是指向数据的指针时，且读和写在不同的goroutine内完成则仍然需要额外的同步动作。
*  Go语言中最常使用的接口之一是 io.Reader
*  所有处于同一个文件夹里面的代码必须使用同一个包名。
*  引用的包必须使用，除非以 _ 作为引用的别名，这种引用别名是为了在不使用某个包时对其进行初始化（执行包下所有代码文件的init函数）
*  map 是Go语言中的一个引用类型，需要使用 make 来构造。如果不先构造 map 并将构造后的值赋给变量，会在使用这个 map 变量时收到出错信息。
*  对于引用类型来说，所引用的底层数据结构会被初始化为对应的零值，但是被声明为其零值的引用类型的变量，会返回nil作为其值。
*  如果需要声明初始值为零值的变量，应该使用 var 关键字声明变量；如果提供确切的非零值初始化变量或者使用函数返回值创建变量，应该使用简化变量声明运算符（:=）。
*  因为有了闭包，函数可以直接访问那些没有作为参数传入的变量。匿名函数并没有拿到这些变量的副本，而是直接访问外部函数作用域中声明的这些变量本身。
*  命名接口的时候应该遵循Go语言的命名规范，如果接口类型只有一个方法，那么这个接口类型的名字以 er 结尾。
*  空结构在创建实例的时候不会分配任何内存，这种结构很适合创建没有任何状态的类型。
*  使用range读取通道，会一直阻塞，直到有结果写入，一旦通道被关闭，for 循环就回终止
    *  for result := range results 
* 程序里面的所有 init 方法都会在 main 函数启动前被调用。
* GOPATH 可以使用:分隔设置多个，当引用某个包时，会首先查找 Go 的安装目录，然后才会按照顺序查找 GOPATH 变量里列出的目录，一旦编译器找到一个符合条件的包，就会停止进一步查找。
* 可以为不同的工程设置不同的 GOPATH，以保持源代码和依赖的隔离。


## 数组、切片和映射

* 切片底层包含三个部分：
    * 指向底层数组的指针
    * 切片的元素个数（即长度）
    * 切片的允许增长到的元素个数（即容量）
* 切片的底层内存是在连续块中分配的，所以切片还能获得索引、迭代和为垃圾回收优化的好处。
* 如果基于某个切片创建新的切片，新切片和原有切片共享底层数据。
* nil 切片和空切片，两者的区别在于前者指针为nil，后者指针指向一个地址
    * var slice [] int // nil 切片
    * slice := make([]int,0) // 使用make创建空切片
    * slice := []int{} // 使用切片字面量创建空切片
* 空切片在底层数组包含0个元素，也没有分配任何存储空间。想表示空集合时空切片很有用。不管是nil切片还是空切片，对其调用内置函数 append， len 和 cap 的效果是一样的。
* 切片只能访问到其长度内的元素。试图访问超出其长度的元素将会导致语言运行时异常，与切片的容量相关联的元素只能用于增长切片，在使用这部分元素之前必须将其合并到切片的长度里（使用appen函数）。

* 函数 append 会智能处理底层数组的容量增长。在切片的容量小于 1000 个元素时，总是会成倍的增加容量。一旦元素超过 1000 个，容量的增长因子会设为1.25，当然随着语言的发展这种增长算法可能会有所改变。
* reslice 可以传三个参数，例如 slcie := source[2:3:4] // 将第三个元素切片，并限制容量， slcie 长度为一个元素，且容量为两个元素，这个参数可以设定的和长度一样，保证 reslice 之后的 slice 再第一次 append 的时候就重新分配内存，避免影响老的 slice
* append 也是一个可变参数的函数，如果使用 ... 运算符，可以将一个切片的所有元素追加到另外一个切片里。
* 内置函数 len ， cap 可以用于处理数组、切片和通道。
* 在 64 位架构的机器上一个切片需要 24 字节的内存，指针字段需要 8 字节，长度和容量分别需要 8 字节，与切片关联的数据包含在底层数组里，不属于切片本身，所以将切片字复制到任意函数的时候，对底层的数据大小都不会有影响。复制时只复制切片本身，不会涉及底层数组。
* 映射是无序的，每次迭代映射的时候顺序也可能不一样。无序的原因是映射的实现使用了散列表。散列函数使用键来生成一个索引，这个索引的低位用于选择桶，高位选择散列键。
* 可以通过声明一个未初始化的映射来创建一个值为 nil 的映射（称为 nil 映射），nil 映射不能用于存储键值对，否则会产生一个运行时错误
* 在 Go 语言里，通过键来索引映射时，即使这个键不存在也总会返回一个值。在这种情况下，返回的是该值对应的类型的零值（不是 nil ）。
* 数组是构造切片和映射的基石。
* 切片有容量限制， 可以通过也只可以通过 append 函数扩容。


## go语言的类型系统

* 和 map 不同，struct 总是会被初始化
* 使用字面量初始化 struct 时，可以使用字段名:值的形式，每一行以 "," 结尾，也可以把所有值写在一行，按照字段顺序赋值，结尾不需要 ","。 
* Duration 是一种描述时间间隔的类型，单位是纳秒（ns），这个类型使用内置的 int64 类型作为其表示，但是 Duration 和 int64 是两种类型，不可以赋值和比较。
* byte 和 int8，tune 和 int32 是相同类型，可以比较和赋值
* 如果一个函数有接收者，这个函数被称为方法
* 字符串（sting）就像整数、浮点数和布尔值一样，本质上一中很原始的数据值，所以在函数或者方法内外传递时，要传递字符串的一个副本。
* 编译器只允许为命名的用户定义的类型声明方法。
* io.Copy 函数会把 Reader 数据分成小段，源源不断的传送给另外一个 Writer
* 接口值是一个两个字长度的数据结构，第一个字包含一个指向内部表的指针，这个内部表叫做 iTable，包含了所有存储的值的类型信息。 iTable 包含了已存储的值的类型信息以及与这个值相关联的一组方法。第二个字是一个指向所存储值的指针。将类型信息和指针组合在一起，就将这两个值组成了一种特殊的关系。
* T 类型的值的方法集只包含值接收者声明的方法。而指向 T 类型的指针的方法集既包含值接收者声明的方法，也包含指针接收者声明的方法。 因为不是总能获取一个值的地址，所以值的方法集只包括了使用值接收者实现的方法。
* 嵌套时，内部类型实现的接口会自动提升到外部类型。
* 将工厂函数命名为 New 是 Go 语言的一个习惯。
* 即便内部类型是未公开的，内部类型里声明的字段依旧是公开的，既然内部类型的标识符提升到外部类型，这些公开的字段也可以通过外部类型的字段的值来访问。

## 并发

* Go 语言的并发同步模型来自一个叫做通信顺序进程（Communicating Sequential Process, CSP）的范型（paradigm）。
* 操作系统会在物理处理器上调度线程来运行，而 Go 语言的运行时会在逻辑处理器上调度 goroutine 来运行。每个逻辑处理器都分别绑定到单个操作系统线程，在 1.5 版本以上，Go 语言的运行时默认会为每个可用的物理处理器分配一个逻辑处理器。
* 当正在运行的 goroutine 需要执行一个阻塞的系统调用，例如打开一个文件，线程和 goroutine 会从逻辑处理器上分离，该线程会阻塞等待，等待系统调用的返回，与此同时这个逻辑处理器就失去了用来运行的线程，这时调度器会创建一个新的线程，并将其绑定到该逻辑处理器上，然后调度器会从本地运行队列里素选择另一个 goroutine 来运行。也就是说发生阻塞时，是 goroutine 和当前线程一起从逻辑处理器上分离，其他 goroutine 的运行会另外创建线程，不再共享这个线程。
* 调度器对可以创建的逻辑处理器的数量没有限制，但语言运行时默认限制每个程序最多创建10000个线程。这个限制值可以通过 runtime/debug 包的 SetMaxThreads 方法来更改。如果程序试图使用更多的线程就会崩溃。
* Go 语言有个工具可以用来检测代码竞争状态， 

```
go build -race  // 用竞争状态检测器标志来编译程序
./test   // 运行程序会打印出竟态检测结果
```

* 有两个有用的原子函数 LoadInt64 和 StoreInt64，这两个函数提供了一中安全的读写一个整型值的方式。
* 锁住共享资源的方式
    * 原子函数
    * 互斥锁
* 当通道关闭后，goroutine 依旧可以从通道接收数据，但是不能向通道里发送数据。能够从已经关闭的通道接收数据这一点非常重要，因为这允许通道关闭后依旧能取出其中缓冲的全部值，而不会丢失数据。
* 无缓冲的通道保证同时交换数据，而有缓冲的通道则不做这种保证。 

## 测试和性能

* Go 语言的测试工具只会认为以 _test.go 结尾的文件是测试文件。
* `go test -v` 来执行测试， `-v`参数表示输出详细信息，如果不加这个参数除非测试失败，否则我们看不到任何输出。
* 测试函数需要以 `Test` 单词开头，并且函数的签名必须要接收一个 `testing.T` 类型的指针，并且不返回任何值。
* 示例代码，可以用于展示如何使用该某个函数等，示例基于已经存在的函数或者方法，我们需要使用 `Example` 代理 `Test` 作为函数名的开始。
* 基准测试函数必须以 `Benchmark` 开头，接受一个指向 `testing.B` 类型的指针作为唯一函数。
* 基础测试里面，一定要将所有进行基础测试的代码都放到循环里，并且循环要使用 b.N 的值，否则测试的结果将是不可靠的。示例：

```go
// BenchmarkSprintf ， 对 fmt.Sprinff 函数进行基准测试
// 参数： b *testing.B 测试基准包
func BenchmarkSprintf(b *testing.B) {
    number := 10
    
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        fmt.Sprintf("%d", number)
    }
}
```

* 执行性能测试时，可以加上 `-benchmem` 选项，这个选项可以提供每次操作分配内存的次数，以及总共分配内存的字节数。