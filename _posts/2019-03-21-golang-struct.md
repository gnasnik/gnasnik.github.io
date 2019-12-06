---
layout:     post
title:      "避免 golang struct 的坑"
subtitle:   ""
date:       2019-03-021 14:00:00
author:     "frank"
# header-style: text
header-img: "img/post-bg-halting.jpg"
tags:
    - golang
    - 编程
---

## 结构体显式与隐式字段的区别

```
type man struct {
    name string `json:"name"`
}

type man struct {
    Name string `json:"name"`
}
```

上面两种定义有什么区别呢？ golang 的结构体里小写开头的方法和属性都是私有的 ,如果你安装了插件，在编译的时候就会有提示 `struct field name has json tag but is not exported`，私有的属性无法通过 json 或其他方式来转码与解码。

```
type man struct {
    name string `json:"name"`
}

var m man

js := `{
		"name":"11"
	}`

err := json.Unmarshal([]byte(js), &m)
if err != nil {
    fmt.Println(err)
}

fmt.Println(m.Name)

```

上面的例子是无法取得到 Name 值的， 打印的结果：
```
# {}
```

## 结构嵌套

```
type man struct {
	name string
}

func (m *man) GetName() {
	fmt.Println(m.name)
	m.GetCoolName()
}

func (m *man) GetCoolName() {
	fmt.Println("Superman-" + m.name)
}

type xman struct {
	man
}

func (x *xman) GetCoolName() {
	fmt.Println("Xman-" + x.name)
}

func main() {
	var x xman
	x.name = "Clark"
	x.GetName()
}

```

打印结果：

```
Clark
Superman-Clark
```


golang 中没有继承也没有重载，只有组合，所以 xman 的方法不会覆写被组合的 man 方法。


## 结构体的指针方法和值方法

```
func (s *MyStruct) pointerMethod() { } // 指针结构的方法
func (s MyStruct)  valueMethod()   { } // 原始结构的方法
```

两者的使用场景：
- 如果需要对接收者 s 的值进行修改，那么必须使用指针
- 如果结构体比较大，那么使用指针会来进行传递会比较高效
- 如果结构体比较小，或者是 slice，map 的引用类型，使用值结构会更高效

为什么小的 struct 使用值传递会比使用指针效率更高呢？

顺便说一下，接口中 *T 的方法集 包括 T 的方法,但 T 的方法不包括 *T 的方法， 所以 T 和 *T 是两种类型。

## 结构体在 map 中的指针类型与值类型的区别

```
type man struct {
	name string
}

func main() {
	var m = map[string]man{
		"IronMan": {"Tony"},
	}
	m["IronMan"].name = "Hulk"
}

```

上面的代码在编译时候就报错了，提示 `cannot assign to struct field m["IronMan"].name in map` , 在 map 中，结构体的非指针结构被当成一个原子变量，只能整体更改，不可以修改其中的成员，如果想要修改其中的成员，必须使用指针类型。

