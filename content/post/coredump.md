---
title: "【问题定位】异步回调函数造成踩内存"
date: 2019-06-13T03:54:36+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: []
---

## 问题现象

进程概率性coredump

## 分析过程

1. 分析core文件，堆栈栈顶函数为`strncmp`，coredump的原因是给strncmp传递的字符串指针（char*）为`0x01`，访问非法内存地址。
2. 找到该字符串指针原始定义处，是某函数中的局部变量，内存地址正常。
3. 该字符串生命周期内没有被改写过，但是在某一时刻突变为了`0x01`。
4. 因此最大可能性是其所在的栈空间内存被其他线程踩到了。
5. 通过两个手段来排查这个问题：
   1. 重点关注被踩的内存空间之前被哪些函数使用过。
   2. 一一排查core文件中的所有执行线程，重点关注可能引起踩内存的函数。

## 原因解析

这个字符串的内存空间曾经被用作某个函数的局部变量，而这个函数中有一段逻辑是循环调用一个异步查询接口，并给这个异步查询接口提供一个回调函数，而这个回调函数的作用就是去修改这个局部变量。

问题发生的原因是这个异步回调接口返回的太慢，调用方函数已经运行完毕，此时栈空间已经被操作系统回收并分配给其他函数，这时再执行回调函数修改原先的局部变量，就造成了踩内存。

这里原先的局部变量是个`int`类型，回调函数想将其修改为1，结果就成了将一个`char*`类型的变量值修改为了`0x01`。