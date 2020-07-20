---
layout:     post
title:      "sync.Pool 源码阅读"
subtitle:   ""
date:       2020-07-19 16:03:50
author:     "frank"
# header-style: text
header-img: "img/golang.png"
tags:
    - 编程
    - golang 
---

# sync.Pool 源码阅读

[TOC]

>  Pool's purpose is to cache allocated but unused items for later reuse,
relieving pressure on the garbage collector. That is, it makes it easy to
build efficient, thread-safe free lists. However, it is not suitable for all free lists.        
>
>A Pool is safe for use by multiple goroutines simultaneously.

从官方的文档可以知道, sync.Pool 主要用来重复使用对象,减少 GC 压力,减少内存分配和内存消耗,从而提升性能，重要的是它是并发安全的，非常适合高并发的业务场景的优化。

## sync.Pool 的用法

定义一个对象池      
```golang
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}
```

然后可以从这个对象池中获取一个对象，假如取不到对象，那么它会调用 New 的方法创建新的对象，然后保存在池中。       
```golang
buf := bufferPool.Get().(*bytes.Buffer)
```

使用完之后要记得把对象放回对象池

```golang
buf.Flush()
bufferPool.Put(buf)
```

## sync.Pool 结构

```golang
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{} // Can be used only by the respective P.
	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

local 的真实类型是 [P]poolLocal， P指的是 GPM 模型中的 P, 所以每个 P 都会维护一个 poolLocal， poolLocal 中避免了 [false sharing (伪缓存)](https://www.cnblogs.com/cyfonly/p/5800758.html) 可能导致的性能问题， 然后具体的对象存储在 poolLocalInternal 里。

poolLocalInternal 里维护一个 private 私有的对象和一个 share 公开访问的队列，这个队列自己从头部推取，其他 P 来获取的时候从尾部取。

victim 缓存的是上一个 GC 周期的对象。

## Get

```golang
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	l, pid := p.pin()
	x := l.private
	l.private = nil
	if x == nil {
		// Try to pop the head of the local shard. We prefer
		// the head over the tail for temporal locality of
		// reuse.
		x, _ = l.shared.popHead()
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}

func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// Try to steal one element from other procs.
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Try the victim cache. We do this after attempting to steal
	// from all primary caches because we want objects in the
	// victim cache to age out if at all possible.
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Mark the victim cache as empty for future gets don't bother
	// with it.
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}

```

从代码可以看到，从对象池中获取一个对象的流程是这样的：      
1. 先从自身的 private 里取对象，取到了直接返回 
2. 假如 private 是空的，那么从 share 公开的队列的头部获取一个
3. 假如自身的 share 队列里仍然是空的，那么会尝试从其他 P 的 share 队列的尾部获取
4. 假如仍然取不到，那么就从上一个 GC 周期的缓存对象 victim 中获取
5. 假如 victim 也是空的，那么就通过调用 New 函数来创建一个新的对象

## Put
```golang
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
	l, _ := p.pin()
	if l.private == nil {
		l.private = x
		x = nil
	}
	if x != nil {
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}
```
 Put 的流程：
 1. 尝试把对象放到 private 里
 2. 假如 private 已经有对象，那么再放到 share 队列里头部


## 对象回收
```
func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.

	// Because the world is stopped, no pool user can be in a
	// pinned section (in effect, this has all Ps pinned).

	// Drop victim caches from all pools.
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// Move primary cache to victim cache.
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	// The pools with non-empty primary caches now have non-empty
	// victim caches and no pools have primary caches.
	oldPools, allPools = allPools, nil
}
```

在 STW 之前，会先执行 poolCleanup , 把 victim 指向 local,把当前的对象放到 victim 里存起来，避免了 GC 的立即回收，但假如两轮 GC 之间没有任何操作，下一次 STW 的时候， victim 仍然会被回收，再获取对象时会调用 New 方法来创建，所以对象池技术只适合在高并发场景下才有效提高性能，并发不高的场景性能不会有大的提升。


## 性能 

使用 sync.Pool 和没有使用 sync.Pool 性能区别

bufio 来读 benchmark 测试结果
```
BenchmarkWriteBufioWithPool-8      	41551393	        29.1 ns/op	       0 B/op	       0 allocs/op
BenchmarkWriteBufioWithoutPool-8   	 2104039	       559 ns/op	    4160 B/op	       2 allocs/op
```

性能提升17倍，使用 sync.Pool 内存消耗为 0 op/s，不使用为 4160B/op


## 应用1：使用 Go 秒级别处理 16GB 大小文件

[Reading 16GB File in Seconds, Golang](https://medium.com/swlh/processing-16gb-file-in-seconds-go-lang-3982c235dfa2)

## 总结：

- 使用之后要放回对象池
- sync.Pool 用了原子性操作，所以是并发安全的
- sync.Pool 主要解决的是对象复用的问题,比较适合高并发的场景
- GC 时对象不会被立即收回，会滞后一轮