---
title: 记录踩的 Go sync map 的一个坑
layout: post
categories: 编程语言
---

# 记录踩的 Go sync map 的一个坑

18年1月底上线了一个 Golang 服务最近被主调方反馈有出现502的情况，上去一查控制台日志，果然是在那个时间段出现了 crash，下面是 crash 的错误信息和调用栈信息

```
fatal error: concurrent map iteration and map write

goroutine 268674187022 [running]:
runtime.throw(0x93ee2f, 0x26)
        /data/barry_dev/go/src/runtime/panic.go:605 +0x95 fp=0xc4390d0a10 sp=0xc4390d09f0 pc=0x42ed95
runtime.mapiternext(0xc4390d0b10)
        /data/barry_dev/go/src/runtime/hashmap.go:778 +0x6f1 fp=0xc4390d0aa8 sp=0xc4390d0a10 pc=0x40d441
sync.(*Map).Range(0xc44840ff50, 0xc4390d0bd8)
        /data/barry_dev/go/src/sync/map.go:331 +0xf8 fp=0xc4390d0b80 sp=0xc4390d0aa8 pc=0x474a58
fly/components/triefilter.FilterContent(0xc473eb1260, 0x13, 0xc466b6ccf0)
        /data/barry_dev/gohome/src/fly/components/triefilter/trie.go:163 +0x277 fp=0xc4390d0d70 sp=0xc4390d0b80 pc=0x818557
fly/components/triefilter.FilterContentInLex(0xc473eb1260, 0x13, 0xc45a621ff0, 0x5, 0xc474574066, 0x5, 0x8, 0x1, 0x1)
        /data/barry_dev/gohome/src/fly/components/triefilter/words.go:59 +0x83 fp=0xc4390d0f40 sp=0xc4390d0d70 pc=0x8186d3
galaxy_access/controller.(*FilterController).ActionSentence.func1(0xc474574070, 0xc478e06200, 0xc474574080, 0xc478a672c0, 0xc45725b8a0, 0x13)
        /data/barry_dev/gohome/src/galaxy_access/controller/filter_controller.go:75 +0xdb fp=0xc4390d0fb0 sp=0xc4390d0f40 pc=0x823e8b
runtime.goexit()
        /data/barry_dev/go/src/runtime/asm_amd64.s:2337 +0x1 fp=0xc4390d0fb8 sp=0xc4390d0fb0 pc=0x45fe21
created by galaxy_access/controller.(*FilterController).ActionSentence
        /data/barry_dev/gohome/src/galaxy_access/controller/filter_controller.go:65 +0x36c

```

观察调用栈信息是 map 信息出现了竟态异常，然后使用竟态检查工具（[race-detector](https://blog.golang.org/race-detector "race-detector")）进行编译检查，却并没有报 warning，仔细观察上面的调用栈信息貌似是在进行 sync.Map 的 Range 操作的时候出现了异常，这就非常有意思了，这是官方系统的线程安全的操作库竟然出现了竟态异常然后 panic 了，这个时候谷歌了一下，发现 Go 的 github上有个人跟我出现了相似的情况并且提了一个 [issue](https://github.com/golang/go/issues/24112#issuecomment-371744007 "issue")，但是烂尾了，提 issue 的人没有继续回复，于是我就在下面回复了，

```
那个那个我也遇到这个问题，跟这个哥们的情况很类似啊，求各位 contributeor 帮忙瞅瞅啊
```

![](http://7xj536.com1.z0.glb.clouddn.com/syncmap1.png)

不得不说，大家很热心啊，很快就有了回复，无非还是如何复现这个问题，我们在 issue 下面七嘴八舌的聊了半天，试了各种办法也没有复现，下面是我写的尝试复现的代码，

```go
package main

import (
	"sync"
	"fmt"
)

var m sync.Map

func main() {

	count := 100000000

	keys := []string{"foo0", "foo1", "foo2"}

	var wg sync.WaitGroup
	wg.Add(20001)

	// Writer routine
	go func() {
		defer func() {
			fmt.Println("store finish")
			wg.Done()
		}()
		i := 0
		for {
			m.Store(keys[i % 3], i)
			i += 1
			if i > count {
				return
			}
		}
	}()

	// Range routine
	for i := 0; i < 20000; i++ {
		go func() {
			defer func() {
				wg.Done()
			}()
			m.Range(func(key, value interface{}) bool {
				return true
			})
		}()
	}
	wg.Wait()
}

```

然后跑了很久并没有 panic ，事情就此陷入了尴尬的局面，老哥说我这试了很多办法也没有复现，你也复现不了，咋查呢

![](http://7xj536.com1.z0.glb.clouddn.com/syncmap2.png)

终于一个叫做 crvv 的老哥解救了我们，他复现了，代码如下：

```
package main

import (
	"math/rand"
	"sync"
)

func main() {
	var m sync.Map

	for i := 0; i < 64; i++ {
		key := rand.Intn(128)
		m.Store(key, key)
	}
	n := m
	go func() {
		for {
			key := rand.Intn(128)
			m.Store(key, key)
		}
	}()
	for {
		n.Range(func(key, value interface{}) bool {
			return key == value
		})
	}
}
```

关键的点在于他把 sync.Map 做了一次拷贝，而正好的我逻辑里面也存在 sync.Map 的拷贝，基本确认了问题所在了，而且 老哥也指出 sync.Map 在[官方文档](https://golang.org/pkg/sync/ "官方文档")里面说了，让你们不要拷贝，


Package sync provides basic synchronization primitives such as mutual exclusion locks. Other than the Once and WaitGroup types, most are intended for use by low-level library routines. Higher-level synchronization is better done via channels and communication.

**Values containing the types defined in this package should not be copied.**

创建完成的 sync.Map 是线程安全的，但是经过拷贝之后，两个 sync.Map 里面存储的是同一个 map（就是那个原生的，线程不安全的 map）， mutex 无法起到保护作用，就线程不安全了。

但是如果真的要拷贝这个 sync.Map 应该怎么办呢？ 那就只能再创建一个，然后 Range 老的 Map 一个个把 KV 拷进去了。 

至此终于解决了，这个故事告诉我们还是应该多读文档啊

最后再次奉上 issue 的地址可以看到整个问题讨论的过程  [点击这里](https://github.com/golang/go/issues/24112#issuecomment-371744007 "点击这里")