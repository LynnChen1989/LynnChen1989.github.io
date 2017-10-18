---
layout: post
title: go传值和传引用
tags:
    - GO
    - 指针
categories: golang
description:  go传值和传引用
---

### 结论

首先, Go语言的函数调用参数全部是传值的, 包括 slice/map/chan 在内所有类型, 没有传引用的说法.

# 指针

Go语言保留着C中值和指针的区别，但是对于指针繁琐用法进行了大量的简化，引入引用的概念。所以在Go语言中，你几乎不用担心会因为直接操作内寸而引起各式各样的错误。
Go语言的指针，基本上只剩下用于区分 byref 和 byval 语义。
运算符就是简单的 `& 和 * 一个取地址、一个解析地址`

~~~go
package main

import (
	"fmt"
)

func main() {
	var i int = 1
	var p *int //定义一个整型指针
	p = &i //p指向i的地址
	fmt.Printf("i=%d, p=%d, *p=%d\n", i, p, *p)

	*p=2  //p 的值为 [[i的地址]的指针] (其实就是i嘛),这行代码也就等价于 i = 2
	fmt.Printf("i=%d, p=%d, *p=%d\n", i, p, *p)

	i=3 // 验证想法
	fmt.Printf("i=%d, p=%d, *p=%d\n",i, p, *p)
}

~~~

**结果**

<p>
i=1, p=826814939592, *p=1<br>
i=2, p=826814939592, *p=2<br>
i=3, p=826814939592, *p=3<br>
</P>

**函数的参数传递**

~~~go
package main

import "fmt"

type Data struct{
    value int
}

func (d Data) FunA() { //传入值
    d.value = 1
    fmt.Printf("A: %d\n", d.value)
}

func (d *Data) FunB() { //传入引用
    fmt.Printf("B: %d\n", d.value)
    d.value = 2
    fmt.Printf("C: %d\n", d.value)
}

func (d *Data) FunC() { //传入引用
    fmt.Printf("D: %d\n", d.value)
}

func main(){
    obj := Data{}
    obj.FunA() //调用FunA时，value的值为1
    obj.FunB()
    obj.FunC()
}
~~~

**结果**

<p>
A: 1 <br>
B: 0 <br>
C: 2 <br>
D: 2 <br>
</p>

### 传值与传指针

当我们传一个参数值到被调用函数里面时，实际上是传了这个值的一份copy，当在被调用函数中修改参数值的时候，调用函数中相应实参不会发生任何变化，因为数值变化只作用在copy上。
传指针比较轻量级 (8bytes),只是传内存地址，我们可以用指针传递体积大的结构体。
如果用参数值传递的话, 在每次copy上面就会花费相对较多的系统开销（内存和时间）。
所以当你要传递大的结构体的时候，用指针是一个明智的选择。
Go语言中string，slice，map这三种类型的实现机制类似指针，所以可以直接传递，而不用取地址后传递指针。
（注：若函数需改变slice的长度，则仍需要取地址传递指针）
要访问指针 p 指向的结构体中某个元素 x，不需要显式地使用 * 运算，可以直接 p.x ；

# 类方法传递引用和值

引用个同事写的代码

~~~go
package main

import(
	"fmt"
)

type Person struct {
	Health int
}



func (p *Person) Fuck() {  //p Person 和 p *Person的结果完全不同，就是能不能修改到结构体中Health值得问题
	p.Health--
	fmt.Println("do fuck, then health=", p.Health)
}



func main(){
	smart := Person{
		Health: 100,
	}
	smart.Fuck()
	smart.Fuck()
	smart.Fuck()
	smart.Fuck()
	smart.Fuck()
	smart.Fuck()
	smart.Fuck()
}
~~~


# 引用

### 什么是传引用

~~~go
package main

import (
	"fmt"
)

func Change(value *int) {
	*value = *value + 2
}

func main() {
	var i int = 1
	Change(&i)
	fmt.Println(i)
}
~~~

函数Change修改i的值,上面代码运行结果为3, 然后print打印出来的也是修改后的值, 那么就可以认为Change是通过引用的方式使用了参数i.


### 为什么slice不是传引用

~~~go
package main

import (
	"fmt"
)

func main()  {
	arr := []int{1,2,3,4}
	fmt.Println(arr)
	modifySlice(arr)
	fmt.Println(arr)
}

func modifySlice(data []int)  {
	data = nil
}
~~~

**运行结果**

<p>
[1 2 3 4] <br>
[1 2 3 4] <br>
</P>

通过modifySlice并没有修改到arr的值，说明是值传递。

### 为什么误以为slice是传引用

可能是FAQ说slice是引用类型, 但并不是传引用!

下面这个代码可能是错误的根源:

~~~go
func main() {
    a := []int{1,2,3}
    fmt.Println(a)
    modifySliceData(a)
    fmt.Println(a)
}

func modifySliceData(data []int) {
    data[0] = 0
}
~~~

输出为:

<p>
[1 2 3] <br>
[0 2 3] <br>
</P>

函数modifySliceData确实通过参数修改了切片的内容.
但是请注意: 修改通过函数修改参数内容的机制有很多, 其中传参数的地址就可以修改参数的值(其实是修改参数中指针指向的数据).
并不是只有引用一种方式!

# 传指针和传引用是等价的吗

比如有以下代码:

~~~go
func main() {
    a := new(int)
    fmt.Println(a)
    modify(a)
    fmt.Println(a)
}

func modify(a *int) {
    a = nil
}
~~~

输出为:

<p>
0xc010000000 <br>
0xc010000000 <br>
</P>

可以看出指针a本身并没有变化. 传指针或传地址也只能修改指针指向的内存的值, 并不能改变指针本身在值.
因此, 函数参数传传指针也是传值的, 并不是传引用!


# 所有类型的函数参数都是传值的

包括slice/map/chan等基础类型和自定义的类型都是传值的.
但是因为slice和map/chan底层结构的差异, 又导致了它们传值的影响并不完全等同.

重点归纳如下:
<ul>
<li>GoSpec: the parameters of the call are passed by value!</li>
<li>map/slice/chan 都是传值, 不是传引用</li>
<li>map/chan 对应指针, 和引用类似</li>
<li>slice 是结构体和指针的混合体</li>
<li>slice 含 values/count/capacity 等信息, 是按值传递</li>
<li>slice 中的 values 是指针, 按值传递</li>
<li>按值传递的 slice 只能修改values指向的数据, 其他都不能修改</li>
<li>以指针或结构体的角度看, 都是值传递!</li>
</ul>


# GO语言有引用的说法么

Go语言其实也是有传引用的地方的, 但是不是函数的参数, 而是闭包对外部环境是通过引用访问的.

~~~go
package main

import (
	"fmt"
)

func main()  {
	a := new(int)
	fmt.Println(a)
	func() {
		a = nil
	}()
	fmt.Println(a)
}
~~~

因为闭包是通过引用的方式使用外部环境的a变量, 因此可以直接修改a的值.

比如下面2段代码的输出是截然不同的, 原因就是第二个代码是通过闭包引用的方式输出i变量:

~~~go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
    // Output: 4 3 2 1 0
}

fmt.Printf("\n")
    for i := 0; i < 5; i++ {
        defer func(){ fmt.Printf("%d ", i) } ()
    // Output: 5 5 5 5 5
}
~~~

像第二个代码就是于闭包引用导致的副作用, 回避这个副作用的办法是通过参数传值或每次闭包构造不同的临时变量:

~~~go
// 方法1: 每次循环构造一个临时变量 i
for i := 0; i < 5; i++ {
    i := i
    defer func(){ fmt.Printf("%d ", i) } ()
    // Output: 4 3 2 1 0
}
// 方法2: 通过函数参数传参
for i := 0; i < 5; i++ {
    defer func(i int){ fmt.Printf("%d ", i) } (i)
    // Output: 4 3 2 1 0
}
~~~








