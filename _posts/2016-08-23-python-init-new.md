---
layout: post
title: python __init__方法和__new__方法的区别
tags:
    - python
    - 编程
categories: python
description: python __init__方法和__new__方法的区别
---

[引言] 使用Python写过面向对象的代码的同学，可能对 _init_ 方法已经非常熟悉了，但是还有一个__new__方法，这两者之间有什么区别，下面做个简单的解释

#### \__init__ 方法是什么？


使用Python写过面向对象的代码的同学，可能对 \__init__ 方法已经非常熟悉了，\__init__ 方法通常用在初始化一个类实例的时候。例如：

```
# -*- coding: utf-8 -*-
class Person(object):
  """Silly Person"""
  def __init__(self, name, age):
    self.name = name
    self.age = age


  def __str__(self):
    return '<Person: %s(%s)>' % (self.name, self.age)


if __name__ == '__main__':
  piglei = Person('piglei', 24)
  print piglei
```


这样便是\__init__最普通的用法了。但\__init__其实不是实例化一个类的时候第一个被调用 的方法。当使用 Persion(name, age) 这样的表达式来实例化一个类时，最先被调用的方法 其实是 \__new__ 方法。


#### \__new__ 方法是什么？


\__new__方法接受的参数虽然也是和\__init__一样，但\__init__是在类实例创建之后调用，而 \__new__方法正是创建这个类实例的方法。new必须以cls作为第一个参数，而init则以self作为其第一个参数

```
# -*- coding: utf-8 -*-
class Person(object):
  """Silly Person"""
  def __new__(cls, name, age):
    print '__new__ called.'
    return super(Person, cls).__new__(cls, name, age)


  def __init__(self, name, age):
    print '__init__ called.'
    self.name = name
    self.age = age


  def __str__(self):
    return '<Person: %s(%s)>' % (self.name, self.age)


if __name__ == '__main__':
    piglei = Person('piglei', 24)
    print piglei
```
执行结果：
```
\__new__ called.
\__init__ called.
< Person: piglei(24)>
```



通过运行这段代码，我们可以看到，\__new__方法的调用是发生在\__init__之前的。其实当 你实例化一个类的时候，具体的执行逻辑是这样的：


* 1.p = Person(name, age)


* 2.首先执行使用name和age参数来执行Person类的\__new__方法，这个\__new__方法会 返回Person类的一个实例（通常情况下是使用 super(Persion, cls).\__new__(cls, ... ...) 这样的方式）


* 3.然后利用这个实例来调用类的\__init__方法，上一步里面\__new__产生的实例也就是 \__init__里面的的 self


所以，\__init__ 和 \__new__ 最主要的区别在于：


* 1.\__init__ 通常用于初始化一个新实例，控制这个初始化的过程，比如添加一些属性， 做一些额外的操作，发生在类实例被创建完以后。它是实例级别的方法。


* 2.\__new__ 通常用于控制生成一个新实例的过程。它是类级别的方法。


但是说了这么多，\__new__最通常的用法是什么呢，我们什么时候需要\__new__？


#### \__new__ 的作用


依照Python官方文档的说法，\__new__方法主要是当你继承一些不可变的class时(比如int, str, tuple)， 提供给你一个自定义这些类的实例化过程的途径。还有就是实现自定义的metaclass。


首先我们来看一下第一个功能，具体我们可以用int来作为一个例子：


假如我们需要一个永远都是正数的整数类型，通过集成int，我们可能会写出这样的代码。


```
class PositiveInteger(int):
    def __init__(self, value):
        super(PositiveInteger, self).__init__(self, abs(value))


i = PositiveInteger(-3)
print i
```


但运行后会发现，结果根本不是我们想的那样，我们任然得到了-3。这是因为对于int这种 不可变的对象，我们只有重载它的\__new__方法才能起到自定义的作用。


这是修改后的代码：

```
class PositiveInteger(int):
    def __new__(cls, value):
        return super(PositiveInteger, cls).__new__(cls, abs(value))


i = PositiveInteger(-3)
print i
```


通过重载\__new__方法，我们实现了需要的功能。


另外一个作用，关于自定义metaclass。其实我最早接触\__new__的时候，就是因为需要自定义 metaclass，但鉴于篇幅原因，我们下次再来讲python中的metaclass和\__new__的关系。


#### 用\__new__来实现单例


事实上，当我们理解了\__new__方法后，我们还可以利用它来做一些其他有趣的事情，比如实现 设计模式中的 单例模式(singleton) 。


因为类每一次实例化后产生的过程都是通过\__new__来控制的，所以通过重载\__new__方法，我们 可以很简单的实现单例模式。


```

class Singleton(object):
    def __new__(cls):
    # 关键在于这，每一次实例化的时候，我们都只会返回这同一个instance对象
        if not hasattr(cls, 'instance'):
            cls.instance = super(Singleton, cls).__new__(cls)
            return cls.instance


obj1 = Singleton()
obj2 = Singleton()


obj1.attr1 = 'value1'
print obj1.attr1, obj2.attr1
print obj1 is obj2
```
输出结果：

```
value1 value1
True
```

可以看到obj1和obj2是同一个实例。