---
layout: post
title: golang OOP
tags:
    - GO语言
    - 面向对象
categories: golang
description:  GO语言的面向对象
---

有过C++语言学习经历的朋友都知道，面向对象主要包括了三个基本特征：封装、继承和多态。

<ul>
<li>封装，就是指运行的数据和函数绑定在一起，C++中主要是通过this指针来完成的；</li>
<li>继承，就是指class之间可以相互继承属性和函数；</li>
<li>多态，主要就是用统一的接口来处理通用的逻辑，每个class只需要按照接口实现自己的回调函数就可以了。</li>
</ul>

作为集大成者的go语言，自然不会在面向对象上面无所作为。
相比较C++、java、C#等面向对象语言而言，它的面向对象更简单，也更容易理解。
下面，我们不妨用三个简单的例子来说明一下go语言下的面向对象是什么样的。

### 封装
~~~go
package main

import "fmt"

type Data struct  {
	value int
}

func (dt *Data) Set(num int) {
	dt.value = num
}

func (dt *Data) Show()  {
	fmt.Println(dt.value)
}

func main()  {
	d := &Data{value:5}
	d.Set(6)
	d.Show()
}
~~~

### 继承
~~~go
package main

import "fmt"

type Parent struct  {
	value int
}

type Child struct  {
	Parent
	num int
}

func main()  {
	var c Child
	c = Child{Parent{1}, 2}
	fmt.Println(c.value)
	fmt.Println(c.num)
}
~~~

### 多态
~~~go
package main

import "fmt"

type act interface {

    write()
}

type xiaoming struct {

}

type xiaofang struct {

}

func (xm *xiaoming) write() {

    fmt.Println("xiaoming write")
}

func (xf *xiaofang) write() {

    fmt.Println("xiaofang write")
}

func main() {

    var w act;

    xm := xiaoming{}
    xf := xiaofang{}

    w = &xm
    w.write()

    w = &xf
    w.write()
}
~~~
