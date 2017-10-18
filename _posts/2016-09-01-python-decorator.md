---
layout: post
title: python装饰器
tags:
    - python
    - 编程
categories: python
description: python装饰器
---


[引言]在代码运行期间动态增加功能的方式，称之为“装饰器”（Decorator）


## 闭包与装饰器

事实上，装饰器就是一种的闭包的应用，只不过其传递的是函数：

```
def deco_one(fun):
    def _deco():
        return "<A>" + fun() + "<A>"
    return _deco


def deco_two(fun):
    def _deco():
        return "<B>" + fun() + "<B>"
    return _deco

@deco_one
@deco_two
def hello():
    return "hello world!"

print hello()
```

输出结果是: `<A><B>hello world!<B><A>`

这个执行的流程是： 

@deco_two装饰器将函数 hello 传递给函数 deco_two, 函数 deco_two 执行完毕后返回被包装后的hello函数，这个过程就是通过闭包实现的。

如果不使用装饰器就是显式的闭包,如下：

```
def deco_one(fun):
    def _deco():
        return "<A>" + fun() + "<A>"
    return _deco


def deco_two(fun):
    def _deco():
        return "<B>" + fun() + "<B>"
    return _deco

def hello():
    return "hello world!"


hello = deco_two(hello)
hello = deco_one(hello)

print hello()
```


## 闭包的作用

**闭包的最大特点是可以将父函数的变量与内部函数绑定，并返回绑定变量后的函数（也即闭包），此时即便生成闭包的环境（父函数）已经释放，闭包仍然存在，这个过程很像类（父函数）生成实例（闭包），不同的是父函数只在调用时执行，执行完毕后其环境就会释放，而类则在文件执行时创建，一般程序执行完毕后作用域才释放，因此对一些需要重用的功能且不足以定义为类的行为，使用闭包会比使用类占用更少的资源，且更轻巧灵活.**


现举一例：假设我们仅仅想打印出各类动物的叫声，分别以类和闭包来实现：


```
# 类实现
class Animal():
    def __init__(self, animal):
        self.animal = animal

    def sound(self, voice):
        print self.animal, ":", voice, "..."

dog = Animal('dog')
dog.sound('wangwang')
dog.sound('wowo')


# 闭包实现
def voice(animal):
    def sound(voc):
        print animal, ":",  voc, "..."

    return sound

dog = voice('dog')
dog.sound('wangwang')
dog.sound('wowo'

```

可以看到输出结果是完全一样的，但显然类的实现相对繁琐，且这里只是想输出一下动物的叫声，定义一个 Animal 类未免小题大做。










## 什么是decorator

**装饰器的功能是将被装饰的函数当作参数传递给与装饰器对应的函数（名称相同的函数），并返回包装后的被装饰的函数**

![image](/images/document_images/decorator.png)


@a 就是将 b 传递给 a()，并返回新的 b = a(b)


在理解这个之前，首先要明确的是Python `一切皆对象`， 那么有了这个概念， 一个函数也是一个对象， 函数可以被随意的传递。

```python
def hi(name="yasoob"):
    def greet():
        return "now you are in the greet() function"

    def welcome():
        return "now you are in the welcome() function"

    if name == "yasoob":
        return greet    # 这里返回的是greet, 而不是greet()
    else:
        return welcome

```

在if/else语句中我们返回greet和welcome，而不是greet()和welcome()。为什么那样？这是因为当你把一对小括号放在后面，这个函数就会执行；然而如果你不放括号在它后面，那它可以被到处传递，并且可以赋值给别的变量而不去执行它。

看下面这个例子：

```python
def new_decorator(func):

    def wrap_func():
        print("I am doing some boring work before executing a_func()")

        func()

        print("I am doing some boring work after executing a_func()")

    return wrap_func


def a_function_requiring_decoration_one():
    print("I am the function which needs some decoration to remove my foul smell ONE")

@new_decorator
def a_function_requiring_decoration_two():
    print("I am the function which needs some decoration to remove my foul smell TWO")


a_function_requiring_decoration_one = new_decorator(a_function_requiring_decoration_one)
a_function_requiring_decoration_one()


a_function_requiring_decoration_two()

```


上面的例子，很好的说明了装饰器是什么。

其实

```
a_function_requiring_decoration_one = new_decorator(a_function_requiring_decoration_one)
a_function_requiring_decoration_one()
```

就是把函数a_function_requiring_decoration_one()作为一个对象传递给new_decorator, 然后经过一系列的处理又返回给a_function_requiring_decoration_one，
这就是一个装饰器的功能；而

```
@new_decorator
def a_function_requiring_decoration_two():
    print("I am the function which needs some decoration to remove my foul smell TWO")
```

就是简化上诉功能的替代品而已。



## 步步深入decorator

#### 第一步：最简单的函数，准备附加额外功能

```python
def myfunc():
    print("myfunc() called.")
myfunc()
myfunc()
```

#### 第二步：使用装饰函数在函数执行前和执行后分别附加额外功能

```python
#!/usr/bin/env python
def deco(func):
        print("before myfunc() called.")
        func()
        print("after myfunc() called.")
        return func
def myfunc():
    print("myfunc() called.")
 
myfunc = deco(myfunc)
myfunc()
myfunc()
```

#### 第三步：使用语法糖@来装饰函数

```python
# -*- coding:gbk -*-
'''示例3: 使用语法糖@来装饰函数，相当于“myfunc = deco(myfunc)”
但发现新函数只在第一次被调用，且原函数多调用了一次'''
  
def deco(func):
    print("before myfunc() called.")
    func()
    print("after myfunc() called.")
    return func
  
@deco
def myfunc():
    print(" myfunc() called.")
  
myfunc()
myfunc()
```


#### 第四步：使用内嵌包装函数来确保每次新函数都被调用

```python
# -*- coding:gbk -*-
'''示例4: 使用内嵌包装函数来确保每次新函数都被调用，
内嵌包装函数的形参和返回值与原函数相同，内嵌包装函数的函数名随便取，装饰函数返回内嵌包装函数对象'''
  
def deco(func):
    def _deco():
        print("before myfunc() called.")
        func()
        print("after myfunc() called.")
        # 不需要返回func，实际上应返回原函数的返回值
    return _deco
  
@deco
def myfunc():
    print(" myfunc() called.")
    return 'ok'
  
myfunc()
myfunc()
```

#### 第五步：对带参数的函数进行装饰

```python
# -*- coding:gbk -*-
'''示例5: 对带参数的函数进行装饰，
内嵌包装函数的形参和返回值与原函数相同，装饰函数返回内嵌包装函数对象'''
  
def deco(func):
    def _deco(a, b):
        print("before myfunc() called.")
        ret = func(a, b)
        print("after myfunc() called. result: %s" % ret)
        return ret
    return _deco
  
@deco
def myfunc(a, b):
    print(" myfunc(%s,%s) called." % (a, b))
    return a + b
  
myfunc(1, 2)
myfunc(3, 4)
```


#### 第六步：对参数数量不确定的函数进行装饰

```python
# -*- coding:gbk -*-
'''示例6: 对参数数量不确定的函数进行装饰，
参数用(*args, **kwargs)，自动适应变参和命名参数'''
  
def deco(func):
    def _deco(*args, **kwargs):
        print("before %s called." % func.__name__)
        ret = func(*args, **kwargs)
        print("after %s called. result: %s" % (func.__name__, ret))
        return ret
    return _deco
  
@deco
def myfunc(a, b):
    print(" myfunc(%s,%s) called." % (a, b))
    return a+b
  
@deco
def myfunc2(a, b, c):
    print(" myfunc2(%s,%s,%s) called." % (a, b, c))
    return a+b+c
  
myfunc(1, 2)
myfunc(3, 4)
myfunc2(1, 2, 3)
myfunc2(3, 4, 5)
```

#### 第七步：让装饰器带参数

```python
# -*- coding:gbk -*-
'''示例7: 在示例4的基础上，让装饰器带参数，
和上一示例相比在外层多了一层包装。
装饰函数名实际上应更有意义些'''
  
def deco(arg):
    def _deco(func):
        def __deco():
            print("before %s called [%s]." % (func.__name__, arg))
            func()
            print("after %s called [%s]." % (func.__name__, arg))
        return __deco
    return _deco
  
@deco("mymodule")
def myfunc():
    print(" myfunc() called.")
  
@deco("module2")
def myfunc2():
    print(" myfunc2() called.")
  
myfunc()
myfunc2()
```

#### 第八步：让装饰器带 类 参数

```python
# -*- coding:gbk -*-
'''示例8: 装饰器带类参数'''
  
class locker:
    def __init__(self):
        print("locker.__init__() should be not called.")
          
    @staticmethod
    def acquire():
        print("locker.acquire() called.（这是静态方法）")
          
    @staticmethod
    def release():
        print("  locker.release() called.（不需要对象实例）")
  
def deco(cls):
    '''cls 必须实现acquire和release静态方法'''
    def _deco(func):
        def __deco():
            print("before %s called [%s]." % (func.__name__, cls))
            cls.acquire()
            try:
                return func()
            finally:
                cls.release()
        return __deco
    return _deco
  
@deco(locker)
def myfunc():
    print(" myfunc() called.")
  
myfunc()
myfunc()
```

#### 第九步：装饰器带类参数，并分拆公共类到其他py文件中，同时演示了对一个函数应用多个装饰器

```python
class MyLocker(object):
    def __init__(self):
        print("{}.__init__() has been called.".format(self.__class__.__name__))
 
    @staticmethod
    def acquire():
        print("MyLocker.acquire() has been called.")
 
    @staticmethod
    def unlock():
        print("MyLocker.unlock() has been called.")
 
 
class LockRex(MyLocker):
    @staticmethod
    def acquire():
        print("LockRex.acquire() has been called")
 
    @staticmethod
    def unlock():
        print("LockRex.unlock() has been called.")
 
 
def lockhelper(cls):
    def _deco(func):
        def __deco(*args, **kwargs):
            print("before %s called." % func.__name__)
            cls.acquire()
            try:
                return func(*args, **kwargs)
            finally:
                cls.unlock()
        return __deco
    return _deco
 
 
class Example(object):
    @lockhelper(MyLocker)
    def myfunc(self):
        print("myfunc() has been called.")
 
    @lockhelper(MyLocker)
    @lockhelper(LockRex)
    def myfunc2(self, a, b):
        print("myfunc2() has been called.")
        return a + b
 
if __name__ == "__main__":
    e = Example()
    e.myfunc()
    print("#"*30)
    print(e.myfunc())
    print("#" * 30)
    print(e.myfunc2(3,4))
```
结果

```text
before myfunc called.
MyLocker.acquire() has been called.
myfunc() has been called.
MyLocker.unlock() has been called.
\##############################
before myfunc called.
MyLocker.acquire() has been called.
myfunc() has been called.
MyLocker.unlock() has been called.
None
\##############################
before __deco called.
MyLocker.acquire() has been called.
before myfunc2 called.
LockRex.acquire() has been called
myfunc2() has been called.
LockRex.unlock() has been called.
MyLocker.unlock() has been called.
7
```
