---
title: "动态链接库的版本控制"
date: 2018-10-20T21:00:51+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: []
---

## DLL Hell

**[DLL Hell](https://en.wikipedia.org/wiki/DLL_Hell)**：同一台机器上，运行着A和B两个程序，他们使用了同一个so；程序A在升级时使用新的so**直接覆盖**老的so，此时可能会造成程序B无法正常运行。

因此需要对动态链接库进行版本控制。

## so name

在介绍版本控制前，需要先了解动态链接库的三种name：`real name`、`soname`、`link name`。

* **link name**：`libxxx.so`称为动态链接库的`link name`。
* **real name**：实际编译出来的动态链接库是具有版本号后缀的，如`libxxx.so.x.y.z`，称为动态链接库的`real name`。
  >其中`x`代表主版本号，`y`代表小版本号，`z`代表duild号。
* **soname**：`link name`+`主版本号`，即`libxxx.so.1`。

## 编译动态库

编译动态链接库时要带上编译选项`-soname`以指定soname。例如编译动态库`libtest.so.1.0.0`时，编译方式如下：

```shell
gcc -fPIC -o test.o -c test.c
gcc -shared -Wl,-soname,libtest.so.1 -o libtest.so.1.0.0 test.o
```

通过`readelf -d`查看动态段，可以发现`soname`信息被记录到了`libtest.so.1.0.0`的文件头中：

```shell
readelf -d libtest.so.1.0.0 | grep soname
 0x0000000e (SONAME) Library soname: [libtest.so.1]
```

**此时执行`ldconfig`命令将自动生成`libtest.so.1`文件，它是一个指向`libtest.so.1.0.0`的软连接**。

不难想到：

* 如果主版本发生变化，新老版本的soname会发生变化。
* 如果小版本发生变化，新老版本的soname应该保持不变。

## 编译程序

以使用上面编译好的`libtest.so.1.0.0`动态库的程序为例，编译的标准步骤如下：

1. 创建一个指向real name文件的link name文件，即 `ln -s libtest.so.1.0.0 libtest.so`
2. 编译程序，通过指定`-ltest`，编译器会去查找`libtest.so`文件，但实际参与编译的是`libtest.so.1.0.0`文件
3. 编译器发现`libtest.so.1.0.0`中记录着soname `libtest.so.1`，告诉程序在运行时应该引用`libtest.so.1`
4. 而`libtest.so.1`文件，则是通过执行`ldconfig`命令生成出来的指向`libtest.so.1.0.0`的软链接，所以程序实际运行过程中使用的是`libtest.so.1.0.0`

## 升级动态库

1. 小版本升级，比如从`libtest.so.1.0.0`升级为`libtest.so.1.1.1`。这个时候，按照约定它的soname`libtest.so.1`是不变的，所以使用者可以直接把新版本so丢到机器上，执行`ldconfig`，新生成的`libtest.so.1`就变成了指向`libtest.so.1.1.1`的软连接。小版本升级是后向兼容的，所以这里直接进行升级是没有问题的。
2. 主版本升级，比如从`libtest.so.1.1.1`升级为`libtest.so.2.0.0`。这个时候，按照约定它的soname变成了`libtest.so.2`，此时`ldconfig`生成的软连接为`libtest.so.2`，指向`libtest.so.2.0.0`。一般主版本升级会有后向兼容性问题，但是由于使用了新的soname，因此对使用老版本so的程序没有影响。
