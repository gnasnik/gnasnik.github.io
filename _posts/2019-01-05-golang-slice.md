---
layout:     post
title:      "Go 源码阅读之 slice"
subtitle:   ""
date:       2019-01-05 11:00:00
author:     "frank"
header-style: text
tags:
    - golang
    - 编程
---

## 用法

```
s := make([]int32,0,10)
```

在执行 make 的时候，发生了什么事？先来看下 make 的定义

```
type Type int

...
...

// 返回的是 Type 不是 value 
func make(t Type, size ...IntegerType) Type

// new 返回的是 Type 的指针
func new(Type) *Type
```

内建函数 make 分配并且初始化 一个 slice, 或者 map 或者 chan 对象， 并且只能是这三种对象。 和 new 一样，第一个参数是 Type，不是一个 value。 但是 make 的返回值就是这个 Type（即使一个引用类型），而不是指针。 具体的返回值，依据传入的具体类型:
- slice  返回指定长度的 slice ,如果不指定 cap 大小，cap 为 len 的值
- map 会预先分配好足够内存给 map, 不需要指定大小
- chan 如果指定长度，就会返回一个带 buffer  的 channle , 如果没的指定长度，那返回的是没有 buffer 的 channel

可以知道内置函数 make 执行的时候会根据传入的 Type 来进行初始化类型，并且预先分配内存，当初始化 slice 时，cap 是可以省略的，默认值是 len 的大小， len 不可以缺省。

**在初始化 slice 的时候，尽量指定 cap 的大小，避免扩容重新申请内存造成的资源损耗**


## 结构
```
type slice struct {
    array unsafe.Pointer // 指向数组指针
    len   int            // 长度
    cap   int            // 容量
}
```
可以看出来 slice 是由三部分组成的， 一个指向数组的指针和 len, cap。

**golang 传递参数都是值传递，在传递大元素的时候使用 slice 会比 array 效率高的多，因为拷贝一个指针地址要比拷贝一块内存数据快的多**

slice 执行 append 的时候，是由 growslice 函数来追加底层数组的元素，下面部分无关代码直接略去，比如 race 检测，对应的代码中的 raceenabled。

```
func growslice(et *_type, old slice, cap int) slice {
	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}
    
    // 如果元素不需要存储空间，比如类型struct{}
    // 直接创建一个新的slice，不需要内存分配
	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}
    
    // 
    // 容量分配规则
    // 
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
    
    //
    // 内存分配规则
    // 
	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

    // 
    // 数据复制
    //

	var p unsafe.Pointer
	if et.ptrdata == 0 {
	 // 分配新的底层数组，这里false指不需要内存清零
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		 // 新数组中未被使用的内存清零
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		// 开启了写屏障
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	
    // 直接拷贝内存
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

可以看出，每次 append 返回的都是一个新分配内存的 slice（也就是说每次 append 之后新 slice 的地址都变了），复制旧 slice 的数据，如果 slice.array 的长度不够添加新的元素，那么会申请一块新的内存，长度为新 slice 的容量大小 ， 旧的 slice 被回收。

新的 slice 容量分配规则：  
- 当 cap 足够的时候，追加的元素直接添加到数组里
- 当 cap 不足并且 len 小于 1024 时，会重新分配一块内存，大小为 2 倍 len
- 当 cap 不足并且 len 超过 1024 时，会重新分配一块内存， 大小为 1/4 len


所以在没有指定容量的情况下，要避免踩坑，下面打印出来的结果是什么？

```

func main() {
    s := []int{5}
    s = append(s, 7)
    s = append(s, 9)
    x := append(s, 11)
    y := append(s, 12)

    fmt.Println(s, x, y)
}
```


结果是:

```
# [5 7 9] [5 7 9 12] [5 7 9 12]
```

因为：<br />
  s := []int{5}   `cap = 1 `<br />
  s = append(s, 7) `cap = 2`, 容量不足，申请新内存 <br />
  s = append(s, 9) `cap = 4`, 容量不足，申请新内存<br />
  x := append(s, 11)   容量足够添加到数据<br />
  y := append(s, 12)   容量足够添加到数据<br />
再思考一下为什么 s、 x 和 y 的数组指针都指向同一个地址，而 s 打印出来的却不是 [5,7,9,12] ？<br />
那就是因为 s 的 len 是 3， 而 x, y 的 len 是 4 

再看下面， 打印的是什么？

```
func assign(s []int) {
	s = []int{6, 6, 6}
}

func appendslice(s []int) {
	s = append(s, 888)
}

func main() {
	var s = []int{1, 2, 3, 4, 5, 6}
	
	assign(s)
	fmt.Println(s)
	
	appendslice(s)
	fmt.Println(s)
}


// 结果是: 
// [1 2 3 4 5 6]
// [1 2 3 4 5 6]
```

因为 golang 中的参数传送都是值传递的，所以修改的并不是原来的 slice ，
append 返回的也是新的 slice , 虽然 append 可以修改 slice 的底层数组，但是也要注意扩容的问题。