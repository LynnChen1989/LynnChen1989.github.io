---
layout: post
title: 一些算法
tags:
    - 算法
categories: 算法
description:  一些常用的算法
---

[引言] 一些常用算法科普

### 简介

插入排序的工作原理是，对于每个未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。时间复杂度O(n^2)

### 步骤
<ul>
<li>从第一个元素开始，该元素可以认为已经被排序</li>
<li>取出下一个元素，在已经排序的元素序列中从后向前扫描</li>
<li>如果被扫描的元素（已排序）大于新元素，将该元素后移一位</li>
<li>重复步骤3，直到找到已排序的元素小于或者等于新元素的位置</li>
<li>将新元素插入到该位置后</li>
<li>重复步骤2~5</li>
</ul>


### 实现
~~~python
# coding=utf-8
def insertion_sort(lst):
    lst_len = len(lst)
    if lst_len <=2 :
        return lst

    for i in range(1, lst_len): #range的start=1, 从第二个元素开始
        key = lst[i]
        j = i - 1

        while j >= 0 and lst[j] > key:
            lst[j+1] = lst[j]
            j -= 1
        lst[j+1] = key #这里打个断点，单步调试就可以看到执行的逻辑
    return lst


lst = [8,5,6,3,2,1,4,7]
print(insertion_sort(lst))
~~~


# 冒泡排序

### 介绍

冒泡排序的原理非常简单，它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。

### 步骤

<ul>
<li>比较相邻的元素。如果第一个比第二个大，就交换他们两个。</li>
<li>对第0个到第n-1个数据做同样的工作。这时，最大的数就“浮”到了数组最后的位置上。</li>
<li>针对所有的元素重复以上的步骤，除了最后一个。</li>
<li>持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。</li>
</ul>

### 实现

~~~python
# coding=utf-8
def bubble_sort(lst):
    lst_len = len(lst)
    if lst_len < 2 :
        return lst

    for i in range(lst_len):
        for j in range(1, lst_len-i):
            if lst[j-1] > lst[j]:
                lst[j - 1], lst[j] = lst[j], lst[j - 1]
    return lst


lst = [2,3,4,8,6,5,1,7]
print(bubble_sort(lst))
~~~

**优化1：某一趟遍历如果没有数据交换，则说明已经排好序了，因此不用再进行迭代了。**

~~~python
#用一个标记记录这个状态即可。
def bubble_sort2(ary):
    n = len(ary)
    for i in range(n):
        flag = 1                    #标记
        for j in range(1,n-i):
            if  ary[j-1] > ary[j] :
                ary[j-1],ary[j] = ary[j],ary[j-1]
                flag = 0
        if flag :                   #全排好序了，直接跳出
            break
    return ary
~~~

**优化2：记录某次遍历时最后发生数据交换的位置，这个位置之后的数据显然已经有序了。**

~~~python
# 因此通过记录最后发生数据交换的位置就可以确定下次循环的范围了。
def bubble_sort3(ary):
    n = len(ary)
    k = n                           #k为循环的范围，初始值n
    for i in range(n):
        flag = 1
        for j in range(1,k):        #只遍历到最后交换的位置即可
            if  ary[j-1] > ary[j] :
                ary[j-1],ary[j] = ary[j],ary[j-1]
                k = j               #记录最后交换的位置
                flag = 0
        if flag :
            break
    return ary
~~~


# 选择排序

### 介绍

选择排序无疑是最简单直观的排序。它的工作原理如下。

### 步骤

<ul>
<li>在未排序序列中找到最小（大）元素，存放到排序序列的起始位置。</li>
<li>再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。</li>
<li>以此类推，直到所有元素均排序完毕。</li>
</ul>

### 实现
~~~python
def select_sort(ary):
    n = len(ary)
    for i in range(0,n):
        min = i                             #最小元素下标标记
        for j in range(i+1,n):
            if ary[j] < ary[min] :
                min = j                     #找到最小值的下标
        ary[min],ary[i] = ary[i],ary[min]   #交换两者
    return ary
~~~