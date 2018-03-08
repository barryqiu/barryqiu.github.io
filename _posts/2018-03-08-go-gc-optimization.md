---
title: Go服务GC优化小记
layout: post
categories: 编程语言
---

# Go服务GC优化小记

##  业务介绍

该服务核心逻辑如下：

* 输入：评论文章ID（`targetid`）和文章下的多个评论ID（`commentid`）
* 输出： 包含评论内容（`content`）和属性（`attribute`）的评论列表

评论的内容和属性数据存储在 redis 和 DB 中，其中 redis 缓存了热门数据，redis 中没有的数据透传到 DB 查询然后回写到 redis ，在 redis 和 DB 中取到内容和属性进行合并后返回给用户，其中内容数据不会变更使用了序列化之后的字符串以 KV 结构存储在 redis 中，评论属性的数据包含了审核状态，点赞数，回复数各种标记信息可能会发生变更，因此以 hash 的结构存储在 redis 中。

整体逻辑上比较简单，开发完毕测试完功能后之后进行了我们压力测试，压测过程中暴露出了一些性能问题。

## 性能问题和原因分析

开发完毕后第一次进行压力测试在24核的机器上，当QPS大约4500时，CPU打满，延时和负载都很高，性能严重不符合预期。因此我们使用了Go语言提供的 profile 工具对性能瓶颈进行了分析，下图是profile top 的结果，展示出了CPU 占用最多的几个操作

![](http://7xj536.com1.z0.glb.clouddn.com/GC1.png)

从上图可以看出来消耗CPU最多的操作都跟内存有关系，经过资料查询这些都跟Go语言的垃圾回收有关系（注：关于Golang的垃圾回收可以参考这篇文章.[golang 垃圾回收机制](https://lengzzz.com/note/gc-in-golang "golang 垃圾回收机制")）

下面这张图是使用profile生成的性能消耗树状图，从这张图可以更加明显的看出来性能都消耗在 GC 上。
![](http://7xj536.com1.z0.glb.clouddn.com/GC2.png)

**因此到这里我们基本可以确定，导致性能无法达到预期的罪魁祸首就是GC，真正的业务逻辑性能消耗反而不是主因，发现了问题，接下来我们就要想办法解决这个问题；**

## 解决方案和效果

### Golang 的版本

开发文章中心和评论中心的时候我们使用的 golang 版本是1.8.3，就在我们进行压测的时候距离1.8版本六个月后Golang 1.9 正式版本发布，版本说明有这样的一句话：

`As always, the changes are so general and varied that precise statements about performance are difficult to make. Most programs should run a bit faster, due to speedups in the garbage collector, better generated code, and optimizations in the core library.`

也就是说1.9版本相当于1.8版本的垃圾回收性能有着明显提升，由于 Go 的版本是向下兼容的，所以我们直接进行了版本升级，逻辑测试完成之后对性能进行对比，下图展示了两个版本的性能对比

![](http://7xj536.com1.z0.glb.clouddn.com/GC3.png)

图中红色框就是版本升级的时间点，可以明显看出来版本升级后CPU有了明显的下降，由此可见版本的升级直接带来了性能的提升；

**注：本文撰写时Go 1.10已经发布，这里还没有对此进行性能测试，但是我相信性能应该会比1.9更好，建议大家可以尽量使用最新版本的 Golang。**

### 减少内存的分配

上面 golang 版本的升级在一定程度上改善了性能，但是与预期还是有比较大的差距，因为这里寻求对程序进行优化。垃圾回收是为了回收分配的内存，那么如果减少了内存的分配是不是可以减少GC的次数，使得更多的CPU用于处理业务逻辑，从而提升QPS和降低延时，有了这个思路，我们首先使用 pprof 工具分析了程序的内存分配情况，下图展示了程序运行期间的内存分配情况

![](http://7xj536.com1.z0.glb.clouddn.com/GC4.jpg)

从图中可以看出是 redigo 的 readReplyEx 操作分配了大量的内存，进一步的可以使用 pprof 对运行数据集合源代码进行深入分析，可以具体到某一行代码分配了多少对象，下图展示了对 readReply 这个操作的对象分析结果

![](http://7xj536.com1.z0.glb.clouddn.com/GC5.jpg)

这里的分析需要结合 redis 的协议一起进行分析，这里简单对当前 redis  protocol 的请求和回复协议进行描述：

* **请求协议 **
一般形式：

```
*<参数数量> CR LF
$<参数 1 的字节数量> CR LF
<参数 1 的数据> CR LF
...
$<参数 N 的字节数量> CR LF
<参数 N 的数据> CR LF
```
注：命令本身也作为协议的其中一个参数来发送。
举个例子， 以下是一个命令协议的打印版本：
```
*3
$3
SET
$5
mykey
$7
myvalue
```
这个命令的实际协议值如下：
```
"*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"
```

* **回复**
Redis 命令会返回多种不同类型的回复。

通过检查服务器发回数据的第一个字节， 可以确定这个回复是什么类型：

```
  状态回复（status reply）的第一个字节是 "+"
  错误回复（error reply）的第一个字节是 "-"
  整数回复（integer reply）的第一个字节是 ":"
  批量回复（bulk reply）的第一个字节是 "$"
  多条批量回复（multi bulk reply）的第一个字节是 "*"
 ```
* **示例**
这里用LRANGE 为例来说明一下该协议
```
客户端： LRANGE mylist 0 3
服务器： *4
服务器： $3
服务器： foo
服务器： $3
服务器： bar
服务器： $5
服务器： Hello
服务器： $5
服务器： World
```

从上面的说明结合前面redis go的代码可以看出针对 redis 协议中的每一行数据返回，都会为其创建一个 slice（Go 里面的变长数组） ，那么这里的内存分配大就很好理解了，以评论属性为例，每个评论属性大约有35个字段，每次请求平均会拉取20条评论，而作为hash结构其 key 和 value 会分成两行来拉取，那么简单计算一下， **每个request仅仅在评论属性的redis读取部分就需要创建 `35*2*20=1400`个小对象**，当QPS达到5000时，假设所有请求都是从redis 获取那么每秒将会创建 7000000 个小对象，这些小对象用完就废弃了，都需要交给 GC 来回收，不累死它才怪呢，o(╯□╰)o

问题找到了，如何解决呢？ 

思路是既然这些小对象用完就废弃了，那可不可以采用一种对象池的方式来进行管理呢，提前申请好一批对象放在一个池子里，使用的时候从池子里免取出，用完了再放回去，这样就可以避免大量小对象的分配了，想法是好的，但是 Go 是利用协程进行高并发的，管理内存池必须要涉及到线程安全的问题，实现起来较为复杂而且容易出性能问题，不过幸好，Golang 为我们提供了临时对象池sync.Pool，根据官方描述该实现是线程安全并且对锁进行优化，兼具了性能和安全两方面。

有了 sync.Pool 我们就可以对上面 redigo 的内存分配进行优化，具体的实现方法可以参考下面的代码

```go
// 申明 sync.Pool 对象
var bytePool = sync.Pool{
	New: func() interface{} {
		b := make([]byte, 0, 64)
		return &b
	},
}

// 获取字节对象
func getBytes() *[]byte {
	return bytePool.Get().(*[]byte)
}

// 释放字节对象回内存池
func putBytes(b *[]byte) {
	(*b) = (*b)[:0]
	bytePool.Put(b)
}

// 当获取的对象不够长度时进行扩展
func expand(b []byte, to int) []byte {
	if b == nil || cap(b) < to {
		return make([]byte, to)
	}
	return b[:to]
}
```

经过上面的对象池的改造后，再次压测时评论中心的性能有了非常明显的提升，QPS 提升到 8000 左右，延时也降低到10ms以内（改造前5000QPS，延时直接超过500ms）

**后记**
在进行这些改造的同时，我也在开源社区四处寻找解决方案，因为 redigo 在面对高并发拉取大尺寸 hash 的性能问题肯定不是我一个人遇到，目前常用 go 的 redis 的 client 主要有两个：

* [redigo](https://github.com/garyburd/redigo "redigo")   就是上文提到的那个，也是使用最为广泛的，在  github 上 4000 多 star
*  [radix](https://github.com/juhanoi/radix "radix")    这个在 github 上只有300多 star 

很明显 redigo 更加受大家的欢迎，良好的封装，非常好的连接池管理都是 redisgo 的优点，但是在面对评论中心这样高并发拉取多个HASH key的情况下，它的性能问题非常严重，radix 的稍微小众一点，但是 mediocregopher 在radix 的基础上发展了其 V2 和 V3 版本，经过阅读源码，我发现V3版本[radix.v3](https://github.com/mediocregopher/radix.v3 "radix.v3")的 radix 就是使用了上面我提到的 sync.Pool 的技术对返回结果解析的过程进行了内存分配方面的优化，在此基础上，它还使用了由调用上层传递返回结果地址的方式来进一步减少内存的分配。

因此这里我对 V3版本的 radix 进行了本地化改造，例如加入了 名字服务支持，支持托管集群扩展的 redis 协议（主要是针对分片数据的批量获取），然后将其引入评论中心，再一次压测的结果是QPS进一步提高到1W+。

### 参数调优

评论服务上线，观察线上运行的情况发现了一些问题，如下图所示，我们的CPU大部分处在5%左右，但是每隔10几秒会出现一个峰值，达到了45%多，经过分析该毛刺是由于GC造成的，这种情况十分危险，当平均CPU上去之后，这种毛刺有可能将CPU打满造成超时和失败。
![](http://7xj536.com1.z0.glb.clouddn.com/GC6.jpg)

大部分的时候 CPU 占用只有5%，开启 GC 就飙到了35% 甚至 45%，那就说明 GC 的任务太重了，那么我就在思考，是否可以让GC提前进行，也就是说提高 GC 的频率来减少每次 GC 的负担，有没有相应的参数可以设置这个呢？ 一查官网文档，我靠~~还真有，不像 JVM 的 GC 调优可以设置多个参数， Go的GC只有一个参数可以调整， 就是

```
func SetGCPercent(percent int) int
```

关于这个参数，官网是这么说的：

`SetGCPercent sets the garbage collection target percentage: a collection is triggered when the ratio of freshly allocated data to live data remaining after the previous collection reaches this percentage. SetGCPercent returns the previous setting. The initial setting is the value of the GOGC environment variable at startup, or 100 if the variable is not set. A negative percentage disables garbage collection.`

翻译一下就是：该参数用于设置垃圾回收的目标百分比，新申请的内存对比上次GC剩余的内存达到这个百分比就进行垃圾回收，默认值是100%，如果设置为负值就是关闭了垃圾回收。

我们当然是不能关闭垃圾回收的，因为 Go 并没有提供类似 free 或者 delete 这种可以清理内存的调用，但是我们可以利用这个参数提高内存回收的频率，经过多次试验发现对于评论中心这个应用 30% 是一个比较优的点（太低的话导致CPU平均值上升的较多，太高就回导致毛刺明显），也就是当新申请的内存达到上一次 GC 之后剩余的内存的30%时 开始本次的垃圾回收，下图展示了调优前后的CPU对比

![](http://7xj536.com1.z0.glb.clouddn.com/GC7.jpg)

这张图可以明显看出来虽然设置为30%之后，平均 CPU 有所上升，但是毛刺现象好了很多，服务更加的稳定了。

## 总结

至此，就将评论中心开发测试过程中发现的 GC 性能问题以及对应的三个解决措施介绍完了，服务也顺利上线运行稳定，背调延时控制在5ms以内。