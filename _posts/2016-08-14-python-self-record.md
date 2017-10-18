---
layout: post
title: python零碎记录
tags:
    - python
    - 编程
categories: python
description: python零碎记录
--- 

[引言] 本文介绍一些python学习过程中常用到的知识点整理


1 . python的拷贝
================
Python中的对象之间赋值时是按引用传递的，如果需要拷贝对象，需要使用标准库中的copy模块。 
+ copy.copy 浅拷贝 只拷贝父对象，不会拷贝对象的内部的子对象。 
+ copy.deepcopy 深拷贝 拷贝对象及其子对象 

```python
# -*- coding:utf-8 -*-
import copy
# 原始对象
a = [1, 2, 3, 4, ['a', 'b']]


# 赋值，传对象的引用
b = a
# 对象拷贝，浅拷贝
c = copy.copy(a)   
# 对象拷贝，深拷贝
d = copy.deepcopy(a)


# 修改对象a
a.append(5)
# 修改对象a中的['a', 'b']数组对象
a[4].append('c')
  
print 'a = ', a  
print 'b = ', b  
print 'c = ', c  
print 'd = ', d  
```

结果输出

```text
a =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
b =  [1, 2, 3, 4, ['a', 'b', 'c'], 5]
c =  [1, 2, 3, 4, ['a', 'b', 'c']]
d =  [1, 2, 3, 4, ['a', 'b']]
```

2 . basestring()
========================
说明：basestring是str和unicode的超类（父类），也是抽象类，因此不能被调用和实例化，但可以被用来判断一个对象是否为str或者unicode的实例，isinstance(obj, basestring)等价于isinstance(obj, (str, unicode))；
版本：python2.3版本以后引入该函数，兼容python2.3以后python2各版本。注意：python3中舍弃了该函数，所以该函数不能在python3中使用。


```python
# coding:utf-8
print isinstance("Hello world", str)
print isinstance("Hello world", basestring)
print isinstance(u"你好", unicode)
print isinstance(u"你好", basestring)
```

结果输出

```text
True
True
True
True
```

3 . super继承
======
 super 是用来解决多重继承问题的，直接用类名调用父类方法在使用单继承的时候没问题，但是如果使用多继承，会涉及到查找顺序（MRO）、重复调用（钻石继承）等种种问题。总之前人留下的经验就是：保持一致性。要不全部用类名调用父类，要不就全部用 super，不要一半一半。

#### 普通继承

```python
class FooParent(object):
    def __init__(self):
        self.parent = 'I\'m the parent.'
        print 'Parent'


def bar(self,message):
        print message, 'from Parent'


class FooChild(FooParent):
    def __init__(self):
        FooParent.__init__(self)
        print 'Child'


    def bar(self,message):
        FooParent.bar(self,message)
        print 'Child bar function.'
        print self.parent


if __name__=='__main__':
    fooChild = FooChild()
    fooChild.bar('HelloWorld')
```

#### super继承

```python
class A(object):
    def __init__(self, name, sex):
        self.name = name
        self.sex = sex

def display(self):
        print "My name is %s, My sex is %s" % (self.name, self.sex)

class B(A):
    def __init__(self, name, sex, age):
        super(B, self).__init__(name, sex)
        self.age = age


    def display(self):
        super(B, self).display()
        # print "hello"
        print "My name is: %s, My sex is: %s, My age is: %s" % (self.name, self.sex, self.age)

b = B(name='chenlin', sex='male', age=26)
print b.display()
```
程序运行结果相同，为：

```text

Parent
Child
HelloWorld from Parent
Child bar fuction
I'm the parent.
```


从运行结果上看，普通继承和super继承是一样的。

但是其实它们的内部运行机制不一样，这一点在多重继承时体现得很明显。

在super机制里可以保证公共父类仅被执行一次，至于执行的顺序，是按照mro进行的（E.__mro__）。

注意super继承只能用于新式类，用于经典类时就会报错。

+ 新式类：必须有继承的类，如果没什么想继承的，那就继承object
+ 经典类：没有父类，如果此时调用super就会出现错误：『super() argument 1 must be type, not classobj』
关于super用法的详细研究可参考「http://blog.csdn.net/johnsonguo/article/details/585193」 



4.三元运算符 
======

```python
x if x < y else y
x < y and x or y
```

5 . Monkey patch
=====
用来在运行时动态修改已有的代码，而不需要修改原始代码。


简单的monkey patch 实现：

```python 
#coding=utf-8 
def originalFunc(): 
    print 'this is original function!' 
     
def modifiedFunc(): 
    modifiedFunc=1 
    print 'this is modified function!' 
     
def main(): 
    originalFunc() 
     
if __name__=='__main__': 
    originalFunc=modifiedFunc 
    main() 
```
 
python中所有的东西都是object，包括基本类型。查看一个object的所有属性的方法是：dir(obj)
函数在python中可以像使用变量一样对它进行赋值等操作。
查看属性的方法：

```python
print locals() 
print globals() 
```

当我们import一个module时，python会做以下几件事情

  * 导入一个module
  * 将module对象加入到sys.modules，后续对该module的导入将直接从该dict中获得
  * 将module对象加入到globals dict中




当我们引用一个模块时，将会从globals中查找。这里如果要替换掉一个标准模块，我们得做以下两件事情

+ 将我们自己的module加入到sys.modules中，替换掉原有的模块。如果被替换模块还没加载，那么我们得先对其进行加载，否则第一次加载时，还会加载标准模块。（这里有一个import hook可以用，不过这需要我们自己实现该hook，可能也可以使用该方法hook module import）

+ 如果被替换模块引用了其他模块，那么我们也需要进行替换，但是这里我们可以修改globals dict，将我们的module加入到globals以hook这些被引用的模块。

Monkey patch就是在运行时对已有的代码进行修改，达到hot patch的目的。Eventlet中大量使用了该技巧，以替换标准库中的组件，比如socket。首先来看一下最简单的monkey patch的实现。

```python
class Foo(object):  
    def bar(self):  
        print 'Foo.bar'  
  
def bar(self):  
    print 'Modified bar'  
  
Foo().bar()  
Foo.bar = bar  
Foo().bar()
```
由于Python中的名字空间是开放，通过dict来实现，所以很容易就可以达到patch的目的。


6 . lambda/filter/map/reduce函数
======

#### lambda匿名函数

冒号前是参数，可以有多个，用逗号隔开，冒号右边的返回值;lambda 函数可以接收任意多个参数 (包括可选参数) 并且返回单个表达式的值。lambda 函数不能包含命令，包含的表达式不能超过一个

```python
def f(x):
    return x**2
f(5)
```

等价于

```python
f = lambda x : x**2
f(5)
```

<b>例子2</b>

```python
#!/usr/bin/env python
num =[1, 4, -5, 10, -7, 2, 3, -1]
filtered_and_squared =[]
for number in num:
        if number < 0:
                filtered_and_squared.append(number**2)
print filtered_and_squared
```

等价于

```python
#!/usr/bin/env python
num =[1, 4, -5, 10, -7, 2, 3, -1]
filtered_and_squared = map(lambda x:x**2,filter(lambda x: x<0,num))
print filtered_and_squared
```
更高级的写法就是列表解析器了

```python
#!/usr/bin/env python
num =[1, 4, -5, 10, -7, 2, 3, -1]
filtered_and_squared =[ x**2 for x in num if x < 0 ]
printfiltered_and_squared
```


#### filter()函数
filter函数用法 filter(func,seq) func函数必须是返回值为bool型的函数，并且filter过滤bool值为真的元素
##### 过滤1-100参数的随机数中的奇数

```python
# 第一种写法
from random import randint


def odd(n):
        return n % 2
list = []
for line in range(9):
        list.append(randint(1,100))
print filter(odd,list)


# 第二种写法（使用lambda）
list = []
for line in range(9):
        list.append(randint(1,100))
print filter(lambda n:n%2,list)


# 第三种写法 （使用列表解析）
list = []
for line in range(9):
        list.append(randint(1,100))
print [n for n in list if n%2]


# 第四种写法（列表解析嵌套）
print [n for n in [randint(1,100) for line in range(9)] if n%2 ]
```

#### map()函数
map的内部实现

```python
def map(func,seq):
     mapped_seq = []
     for eachItem in seq:
         mapped_seq.append(func(eachItem))
     return mapped_seq
map(func,seq)
```

例子：

```python
def sum(x):
    return x+2
print map(sum,range(1,9))


##用lambda
print map(lambda x:x+2, range(1,9))
print map(lambda x,y:(x+1,y+2),[1,2,3],[1,2,3])


##用列表解析
[(x+2) for x in range(1,9)]
```


#### reduce()函数

reduce的实现

```python
def reduce(bin_func,seq,initial=None):
 lseq = list(seq)
 if initial is None:
      res = lseq.pop(0)
 else:
      res = initial
 for eachItem in lseq:
      res = bin_func(res,eachItem)
 return res
```

一个例子来解释reduce原理

```python
def sum(x,y):
        return x+y
print sum(4,sum(3,sum(1,2)))
#等价于
print reduce(sum,[1,2,3,4])
```



7 . 显示有限的接口到外部
================
当发布python第三方package时, 并不希望代码中所有的函数或者class可以被外部import, 在__init__.py中添加__all__属性,
该list中填写可以import的类或者函数名, 可以起到限制的import的作用, 防止外部import其他函数或者类

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
from base import APIBase
from client import Client
from decorator import interface, export, stream
from server import Server
from storage import Storage
from util import (LogFormatter,disable_logging_to_stderr,enable_logging_to_kids, info)


__all__ = ['APIBase', 'Client', 'LogFormatter', 'Server', 'Storage', 'disable_logging_to_stderr','enable_logging_to_kids','export', 'info', 'interface', 'stream']
```


 8 . staticmethod/classmethod装饰器
=============================
类中两种常用的装饰, 首先区分一下他们
普通成员函数, 其中第一个隐式参数为对象
- classmethod装饰器,类方法(给人感觉非常类似于OC中的类方法), 其中第一个隐式参数为类- staticmethod装饰器, 没有任何隐式参数. 就是用于那些不需要实例化的方法，python中的静态方法类似与C++中的静态方法

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
class A(object):
    # 普通成员函数
    def foo(self, x):
        print "executing foo(%s, %s)" % (self, x)
        
    # 使用classmethod进行装饰
    @classmethod   
    def class_foo(cls, x):
        print "executing class_foo(%s, %s)" % (cls, x)
        
    # 使用staticmethod进行装饰
    @staticmethod  
    def static_foo(x):
        print "executing static_foo(%s)" % x
        
def test_three_method():
    obj = A()
    # 直接调用成员方法
    obj.foo("para")         # 此处obj对象作为成员函数的隐式参数, 就是self
    obj.class_foo("para")   # 此处类作为隐式参数被传入, 就是cls
    A.class_foo("para")     # 更直接的类方法调用
    obj.static_foo("para")  # 静态方法并没有任何隐式参数, 但是要通过对象或者类进行调用
    A.static_foo("para")
    
if __name__ == '__main__':
    test_three_method()
```
函数输出

```text
executing foo(<__main__.A object at 0x100ba4e10>, para)
executing class_foo(<class '__main__.A'>, para)
executing class_foo(<class '__main__.A'>, para)
executing static_foo(para)
executing static_foo(para)
```


9 . property装饰器
=========================
定义私有类属性将property与装饰器结合实现属性私有化(更简单安全的实现get和set方法)

```python
# python内建函数
property(fget=None, fset=None, fdel=None, doc=None)
```

* fget是获取属性的值的函数;
* fset是设置属性值的函数;
* fdel是删除属性的函数;
* doc是一个字符串(like a comment).

从实现来看，这些参数都是可选的


property有三个方法getter(), setter()和delete() 来指定fget, fset和fdel。 这表示以下这行

```python
class Student(object):


    def __init__(self):
        self._score = 0


    def get_score(self):
        return self._score


    def set_score(self, value):
        if not isinstance(value, int):
            raise ValueError('score must be an integer!')
        if value < 0 or value > 100:
            raise ValueError('score must between 0 ~ 100!')
        self._score = value


s = Student()
s.set_score(1000)

print s.get_score()
```
但是，上面的调用方法又略显复杂，没有直接用属性这么直接简单。
有没有既能检查参数，又可以用类似属性这样简单的方式来访问类的变量呢？对于追求完美的Python程序员来说，这是必须要做到的！
还记得装饰器（decorator）可以给函数动态加上功能吗？对于类的方法，装饰器一样起作用。Python内置的@property装饰器就是负责把一个方法变成属性调用的：


```python
class Student(object):
    def __init__(self):
        self._score = 0


    @property
    def score(self):
        return self._score


    @score.setter
    def score(self, value):
        if not isinstance(value, int):
            raise ValueError('score must be an integer!')
        if value < 0 or value > 100:
            raise ValueError('score must between 0 ~ 100!')
        self._score = value


    @score.deleter
    def score(self):
        del self._score




s = Student()
print "set value score ..."
s.score = 70
print "value is %d" % s.score
print "delete value score ..."
del s.score
print s.score
```

10 . iter魔法
==============
通过yield和__iter__的结合, 我们可以把一个对象变成可迭代的
通过__str__的重写, 可以直接通过想要的形式打印对象


```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-


class TestIter(object):

    def __init__(self):
        self.lst = [1, 2, 3, 4, 5]

    def read(self):
        for ele in xrange(len(self.lst)):
            yield ele

    def __iter__(self):
        return self.read()

    def __str__(self):
        return ','.join(map(str, self.lst))
 
    __repr__ = __str__


def test_iter():
    obj = TestIter()
    for num in obj:
        print num
    print obj


if __name__ == '__main__':
    test_iter()
```


11 . 神奇partial
========================
partial使用上很像C++中仿函数(函数对象).
在stackoverflow给出了类似与partial的运行方式

```python
def partial(func, *part_args):
    def wrapper(*extra_args):
        args = list(part_args)
        args.extend(extra_args)
        return func(*args)
    return wrapper
```

利用用闭包的特性绑定预先绑定一些函数参数, 返回一个可调用的变量, 直到真正的调用执行

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
from functools import partial


def sum(a, b):
    return a + b
    
def test_partial():
    fun = partial(sum, 2)  
    # 事先绑定一个参数, fun成为一个只需要一个参数的可调用变量
    print fun(3)  
    # 实现执行的即是sum(2, 3)
    
if __name__ == '__main__':
    test_partial()
``` 

运行结果

```text
5
```

12 . 神秘eval
============================
eval我理解为一种内嵌的python解释器(这种解释可能会有偏差), 会解释字符串为对应的代码并执行, 并且将执行结果返回

看一下下面这个例子

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-


def test_first():
    return 3


def test_second(num):
    return num


action = {  # 可以看做是一个sandbox
        "para": 5,
        "test_first" : test_first,
        "test_second": test_second
        }


def test_eavl():
    condition = "para == 5 and test_second(test_first) > 5"
    res = eval(condition, action)  # 解释condition并根据action对应的动作执行
    print res


if __name__ == '__main__':
    test_eavl()
```

13 . exec
===============================================
exec在Python中会忽略返回值, 总是返回None, eval会返回执行代码或语句的返回值
exec和eval在执行代码时, 除了返回值其他行为都相同
在传入字符串时, 会使用compile(source, '<string>', mode)编译字节码. mode的取值为exec和eval

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-


def test_first():
    print "hello"
    
def test_second():
    test_first()
    print "second"
    
def test_third():
    print "third"
    
action = {
        "test_second": test_second,
        "test_third": test_third
        }
        
def test_exec():
    exec "test_second" in action
    
if __name__ == '__main__':
    test_exec()  # 无法看到执行结果
```


14 . getattr
=============================

```text

getattr(object, name[, default])Return the value of the named attribute of object. 
name must be a string. If the string is the name of one of the object’s attributes, the result is the value of that attribute. 
For example, getattr(x, ‘foobar’) is equivalent to x.foobar. 
If the named attribute does not exist, default is returned if provided, otherwise AttributeError is raised.
```



通过string类型的name, 返回对象的name属性(方法)对应的值, 如果属性不存在, 则返回默认值, 相当于object.name

```python
# 使用范例
# coding:utf-8
class TestGetAttr(object):
    test = "test attribute"


    def say(self):
        print "test method"




def test_getattr():
    my_test = TestGetAttr()


    try:
        print getattr(my_test, "test")
    except AttributeError:
        print "Attribute Error!"
    try:
        getattr(my_test, "say")()
    except AttributeError: # 没有该属性, 且没有指定返回值的情况下
        print "Method Error!"


if __name__ == '__main__':
    test_getattr()
```

输出结果

```text
test attribute
test method
```

15 . 命令行处理
=====================

```python
import sys, optparse

def process_command_line(argv):
    """
    Return a 2-tuple: (settings object, args list).
    `argv` is a list of arguments, or `None` for ``sys.argv[1:]``.
    """
    if argv is None:
        argv = sys.argv[1:]


    # initialize the parser object:
    parser = optparse.OptionParser(
        formatter=optparse.TitledHelpFormatter(width=78),
        add_help_option=None)


    # define options here:
    parser.add_option(
        # customized description; put --help last
        '-h', '--help', action='help',
        help='Show this help message and exit.')
    settings, args = parser.parse_args(argv)
    # check number of arguments, verify values, etc.:
    if args:
        parser.error('program takes no command-line arguments; '
                     '"%s" ignored.' % (args,))
    # further process settings & args if necessary
    return settings, args

def main(argv=None):
    settings, args = process_command_line(argv)
    # application code here, like:
    # run(settings, args)
    return 0        # success

if __name__ == '__main__':
    status = main()
    sys.exit(status)
```


16 . 读写csv文件
==========================

```python
# 从csv中读取文件, 基本和传统文件读取类似
# coding:utf-8


import csv
with open('data.csv', 'rb') as f:
    reader = csv.reader(f)
    for row in reader:
        print row


# 向csv文件写入
import csv
with open( 'data.csv', 'wb') as f:
    writer = csv.writer(f)
    writer.writerow(['name', 'address', 'age'])  # 单行写入
    data = [
        ('xiaoming ', 'china', '10'),
        ('Lily', 'USA', '12')
    ]
    writer.writerows(data)  # 多行写入
```


17 .  一个非常好用, 很多人又不知道的功能
==============================

```text
>>> name = "andrew"
>>> "my name is {name}".format(name=name)
'my name is andrew'
>>> print "my name is: {0}, my sex is: {1}".format("chenlin","male")
my name is: chenlin, my sex is: male
```

18 . python的普通方法、类方法、静态方法
==========

* staticmethod 基本上和一个全局函数差不多，只不过可以通过类或类的实例对象（python里光说对象总是容易产生混淆， 因为什么都是对象，包括类，而实际上类实例对象才是对应静态语言中所谓对象的东西）来调用而已， 不会隐式地传入任何参数。这个和静态语言中的静态方法比较像。**就是不需要self,可以不实例化，这就是最明显的区别**。
 
* classmethod 是和一个class相关的方法，可以通过类或类实例调用，并将该class对象（不是class的实例对象）隐式地 当作第一个参数传入。就这种方法可能会比较奇怪一点，不过只要你搞清楚了python里class也是个真实地 存在于内存中的对象，而不是静态语言中只存在于编译期间的类型。
 
正常的方法就是和一个类的实例对象相关的方法，通过类实例对象进行调用，并将该实例对象隐式地作为第一 个参数传入，这个也和其它语言比较像。


```python
#!/usr/bin/python
# coding:utf-8

class Person:

    def __init__(self):
        print "init"


    @staticmethod
    def sayHello(word):
        if not word:
            word = 'hello'
        print "[static_method: sayHello] --> I will say %s" % word


    @classmethod
    def introduce(yourclass, hello):  # 这里的yourclass = Person
        yourclass.sayHello(hello)
        print "[class_method: introduce] --> class method introduce() be used."


    def hello(self, hello):
        self.sayHello(hello)
        print "[general_method: hello] --> general method hello() be used."


def main():
    Person.sayHello("haha")  # 静态方法使用类直接调用
    Person.introduce("hello world!")
    # Person.hello("self.hello")
    # TypeError: unbound method hello() must be called with Person instance as first argument (got str instance instead)


    print "#" * 40
    p = Person()
    p.sayHello("haha")
    p.introduce("hello world!")
    p.hello("self.hello")


if __name__ == '__main__':
    main()
```

运行结果

```text

[static_method: sayHello] --> I will say haha
[static_method: sayHello] --> I will say hello world!
[class_method: introduce] --> class method introduce() be used.
########################################
init
[static_method: sayHello] --> I will say haha
[static_method: sayHello] --> I will say hello world!
[class_method: introduce] --> class method introduce() be used.
[static_method: sayHello] --> I will say self.hello
[general_method: hello] --> general method hello() be used.
```

20.import this
================================

彩蛋

```text

The Zen of Python, by Tim Peters
Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!

```


21.去重List
================================

```python
# 方法一
a = [1, 2, 3, 4, 3]
b = []
for index, item in enumerate(a):
    if item not in b:
        b.append(item)
print(b)


# 方法2
L1 = [1, 2, 3, 4, 3]
L2 = []
[L2.append(item) for item in L1 if item not in L2]
print(L2)


# 方法3
a1 = [1, 2, 3, 4, 3]
print(list(set(a1)))
```


21.闭包(closure)
================================
简单说,闭包就是根据不同的配置信息得到不同的结果


再来看看专业的解释:闭包（Closure）是词法闭包（Lexical Closure）的简称，是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。


### 例1

```python
def make_adder(addend):
    def adder(augend):
        return addend+augend
    return adder


p = make_adder(33)
q = make_adder(44)
print(p(100))
print(q(100))
```


**分析一下:**

我们发现,`make_adder`是一个函数,包括一个参数addend,比较特殊的地方是这个函数里面又定义了一个新函数,这个新函数里面的一个变量正好是外部make_adder的参数.也就是说,外部传递过来的addend参数已经和adder函数绑定到一起了,形成了一个新函数,我们可以把addend看做新函数的一个配置信息,配置信息不同,函数的功能就不一样了,也就是能得到定制之后的函数.


再看看运行结果,我们发现,虽然p和q都是make_adder生成的,但是因为配置参数不同,后面再执行相同参数的函数后得到了不同的结果.这就是闭包.


### 例2

```python
def hellocounter(name):
    count = [0]
    def counter():
        count[0] += 1
        print 'Hello,', name, ',', str(count[0]) + ' access!'
    return counter
hello = hellocounter('snake')
print hello()
print hello()
print hello()
```
执行结果

```text

Hello, snake , 1 access!
Hello, snake , 2 access!
Hello, snake , 3 access!
```

**分析一下**


这个程序比较有趣,我们可以把这个程序看做统计一个函数调用次数的函数.count[0]可以看做一个计数器,没执行一次hello函数,count[0]的值就加1。也许你会有疑问: **为什么不直接写count而用一个列表?** 这是python2的一个bug,如果不用列表的话,会报这样一个错误:
`UnboundLocalError: local variable 'count' referenced before assignment.`

什么意思?就是说conut这个变量你没有定义就直接引用了,我不知道这是个什么东西,程序就崩溃了.于是,再python3里面,引入了一个关键字:**nonlocal**,这个关键字是干什么的?就是告诉python程序,我的这个count变量是再外部定义的,你去外面找吧.然后python就去外层函数找,然后就找到了count=0这个定义和赋值,程序就能正常执行了.


**python3**

```python
def hellocounter (name):
    count=0 
    def counter():
        count+=1
        print 'Hello,',name,',',str(count[0])+' access!'
    return counter


hello = hellocounter('snake')
hello()
hello()
hello() 
```


###  例3

```python
def makebold(fn):
    def wrapped():
        return "<b>" + fn() + "</b>"
    return wrapped


def makeitalic(fn):
    def wrapped():
        return "<i>" + fn() + "</i>"
    return wrapped


@makebold
@makeitalic
def hello():
    return "hello world"


print hello() 
```


怎么样?这个程序熟悉吗?这不是传说的的装饰器吗?对,这就是装饰器,其实,装饰器就是一种闭包,我们再回想一下装饰器的概念:对函数(参数,返回值等)进行加工处理,生成一个功能增强版的一个函数。再看看闭包的概念,这个增强版的函数不就是我们配置之后的函数吗?区别在于,装饰器的参数是一个函数或类,专门对类或函数进行加工处理。


22.is和==的区别
==================================

```python
a1 = 'Hi'
b1 = 'Hi'


a2 = "I am using long string for testing"
b2 = "I am using long string for testing"


a3 = "string"
b3 = ''.join(['s', 't', 'r', 'i', 'n', 'g'])


print("a1:{}\nb1:{}".format(id(a1), id(b1)))
print("a2:{}\nb2:{}".format(id(a2), id(b2)))
print("a3:{}\nb3:{}".format(id(a3), id(b3)))
```

结果

```text
a1:140144374621568
b1:140144374621568
a2:140144375064016
b2:140144375064016
a3:140144374574656
b3:140144374574752

```

is的作用是用来检查对象的标示符是否一致，也就是比较两个对象在内存中是否拥有同一块存储空间，它不适合用来判断两个字符串是否相等。