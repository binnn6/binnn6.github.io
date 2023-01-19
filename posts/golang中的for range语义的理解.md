golang中for range经常会被用来遍历slice、map、chan、array,但是由于在某些情况下,其内部实现并不是你想的那样(make sence),所以使用时还是需要特别注意。

以下是两个错误使用for range的场景

## 场景1 - 在for range中获取循环变量的地址

### 代码

```go
package main

func main() {
	vals := []int{0, 1, 2}
	valptr := make([]*int, 0)

	for _, v := range vals {
		//v := v
		valptr = append(valptr, &v)
	}

	for _, v := range valptr {
		println(v)
		println(*v)
	}
}
```

### 输出

```bash
0xc000016140
6
0xc000016140
6
0xc000016140
6
```

### 分析

> The iteration variables in `for`statements are reused in each iteration. This means that each closure (aka function literal) created in your `for` loop will reference the same variable (and they'll get that variable's value at the time those goroutines start executing).

go runtime中for range循环只会为v分配一次内存，后续只是给`v`赋值；跟for的语义是一样一样的,如下这样理解起来就容易多了。

```go
package main

func main() {
	for i := 0; i < 3; i++ {
		println("&i=", &i, " i=", i)
	}
}
```

### 解决方式

```go
package main

func main() {
	vals := []int{0, 1, 2}
	valptr := make([]*int, 0)

	for _, v := range vals {
		//修复方式1 
		//v := v
		valptr = append(valptr, &v)
	}
	
	//修复方式2
	//for k, _ := range vals {
	//	valptr = append(valptr, &vals[k])
	//}

	for _, v := range valptr {
		println(v)
		println(*v)
	}
}
```



## 场景2 - 在closure中捕获循环变量

### 代码

```go
package main

func main() {
	kvs := map[string]int{
		"zero": 0,
		"one":  1,
		"two":  2,
	}

	for _, v := range kvs {
		//v := v
		defer func() {
			println("defer func(): ", v)
		}()
	}
}
```

### 输出

```bash
# go run main.go 
go func():  2
go func():  2
go func():  2
# go vet
# github.com/username/gomodule
./main.go:19:27: loop variable v captured by func literal
```

### 分析

closure(func literal)中捕获变量是通过引用的方式,因此也会像场景1中那样,`v`的地址虽然没变,但是值会随着循环变化。

同时,在场景2中,go vet会有对应的提示:

`./main.go:19:27: loop variable v captured by func literal`

在出现以上提示时,需要特别小心。同时这里要说下：

**`go vet`是一个很有价值的工具,可以帮助我们发现很多代码问题**

但是场景1是没有的提示, 也不能完全依赖go vet来发现错误。

除了通过增加`v := v`的方式,还可以通过以下方式：

```go
package main

func main() {
	kvs := map[string]int{
		"zero": 0,
		"one":  1,
		"two":  2,
	}

	for _, v := range kvs {
		defer func(v int) {
			println("defer func(): ", v)
		}(v)
	}
}
```

其实原理是一样的，

- `v (v1) := v (v2)` [括号是注释]是显式的创建了一个v的副本,也叫v;  这里两个v的生命周期不同, v2的生命周期是整个for循环,v1的生命周期是for循环中的一个循环,但是这里由于closure对于v1的引用,所以在一个循环结束后,v会发生逃逸,并不会被立即回收;
- 把v作为closure的参数,通过golang的pass-by-value,隐式的创建了一个v的副本;

大部分刚入门golang的开发者都会犯类似错误,讲道理这个可以算是语言的缺陷了，毕竟让用户少犯错也是语言的义务。因此社区中会很多proposal想要解决这个问题。




## Proposal

[proposal: spec: redefine range loop variables in each iteration](https://github.com/golang/go/issues/20733)

已经有人提了类似Proposal, 有可能`Go 2.0`中会彻底解决这个问题。

如下：

```go
for k, v := range vals {
  // ...
}
```

should be equivalent to

```go
for k, v := range vals {
  k := k
  v := v
  // ...
}
```

## 参考

[Iterating Over Slices In Go](https://www.ardanlabs.com/blog/2013/09/iterating-over-slices-in-go.html)

[golang-range-and-pointers](https://tam7t.com/golang-range-and-pointers/)

