> For the impatient, golang is aways **pass-by-value**.
>
> Java is **strictly** pass-by-value, exactly as in C. 

## map是pass-by-value的

### 代码

```go
package main

import "fmt"

func fixmap(m map[string]string) {
	m["a"] = "aaa"
	m = make(map[string]string)
	m["b"] = "bbb"
}

func main() {
	amap := map[string]string{
		"a": "aa",
		"b": "bb",
	}

	fixmap(amap)

  for k, v := range amap {
		fmt.Println(k, v)
	}
}
//output:
//a aaa
//b bb
```

### map不是pass-by-reference的 

乍一看,在fixmap函数,`m["a"] = "aaa"`,最终amap["a"] == "aaa";符合pass-by-reference的定义;

但是 `m = make(map[string]string); m["b"] = "bbb"`却没有对amap生效。这是为什么呢？

这就需要了解一下map的实现逻辑

`m := make(map[int]int)` 的实际底层实现为`runtime.makemap`,其signature如下:

`func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap`

实际上map是一个指向hmap接口的指针。

> A map value is a pointer to a `runtime.hmap` structure.

所以map是pass by value的,只不过这里的map本身是个指针;chan,slice(稍微有点特殊)，array也是类似原理。

```go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

```go
package main

func modifySlice1(intslice []int)  {
	intslice[1] = 2
	// 虽然修改了底层数据,由于slice是传值调用,没有修改到slice.len
	intslice = append(intslice, 3)
	// 扩容了不再指向原来的数据,所以修改未对intarray生效
	intslice = append(intslice, 4)

}

func main() {
	intarray := [3]int{}
	intslice := intarray[:0]
	println("len(intslice): ", len(intslice), "  cap(intslice):", cap(intslice))
	intslice = append(intslice, 1)
	intslice = append(intslice, 1)
	println("len(intslice): ", len(intslice), "  cap(intslice):", cap(intslice))
	modifySlice1(intslice)
	println("len(intslice): ", len(intslice), "  cap(intslice):", cap(intslice))

	println("intarray[2]:", intarray[2])

	for k, v := range intslice {
		println("intslice k:", k, " v:", v)
	}
    println(intslice[2])  // panic: runtime error: index out of range
	println(intslice[2:3][0])

	for k, v := range intarray {
		println("intarray k:", k, " v:", v)
	}
}
```

## Pass-By-Value vs Pass-By-Reference

区别于`primitive types` `reference types`

|  类型  | 值类型/引用类型 |        描述        |
| :----: | :-------------: | :----------------: |
| struct |     值类型      | Mutex、WaitGroup等 |
| array  |   value type    |                    |
|  chan  | reference type  |                    |
| slice  | reference type  |    SliceHeader     |
|  map   | reference type  |  runtime.makemap   |

区别于`call by name` `call by value` `lazy` `val` `def` in scala

`reference` `dereference`

> Unless you are programming in Fortran or Visual Basic (C++ ?), it's not the default behavior, and in most languages in modern use, true call-by-reference is not even possible.

## sync.WaitGroup & sync.Mutex & Struct

> A WaitGroup must not be copied after first use.
>
> A Mutex must not be copied after first use.

golang中struct是值传递的,特别是WaitGroup和Mutex作为参数传递时,一定要用指针。

## 附录

[Is Java “pass-by-reference” or “pass-by-value”?](https://stackoverflow.com/questions/40480/is-java-pass-by-reference-or-pass-by-value/)

[If a map isn’t a reference variable, what is it?](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)

[What's the difference between passing by reference vs. passing by value?](https://stackoverflow.com/questions/373419/whats-the-difference-between-passing-by-reference-vs-passing-by-value)

[Java is Pass-by-Value, Dammit!](http://www.javadude.com/articles/passbyvalue.htm)

