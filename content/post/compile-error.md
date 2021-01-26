---
title: "记一次编译错误的解决过程"
date: 2018-11-01T20:31:15+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: []
---

最近开发新的需求需要使用某个外部模块的库文件，该模块在文档中提供了一个demo，但makefile文件编译报错。通过摸索和学习最终把demo编译成功并运行起来，下面记录一下过程中碰到的问题并进行总结。

## so not found

使用gcc编译c/c++程序时，编译时用`-I`指定头文件查找路径，`-L`指定库文件查找路径，`-l`具体指定依赖的库。

如果指定了`-L`，也使用`-l`链接了该库，但是报如下告警：

```shell
warning: libxxx.so, needed by ./libyyy.so, not found (try using -rpath or -rpath-link)
```

说明该so依赖的其他so无法找到。使用`ldd`命令查看该so依赖的所有其他so，会有类似于

```shell
/usr/bin/ld: cannot find -lxxx
```

的打印，这时就需要找到被依赖的so所在的绝对路径，添加到`/etc/ld.so.conf`文件中，并执行`ldconfig`。

另一种办法是向环境变量`LD_LIBRARY_PATH`中添加路径，指定动态库加载路径；对应的静态库加载路径的环境变量名`LIBRARY_PATH`。
> 注意: 使用`env`命令查看系统中若无环境变量`LD_LIBRARY_PATH`和`LIBRARY_PATH`，则需要使用`export`命令将变量变成环境变量，即该变量在子shell进程中也可见。但重新登录时该环境变量会消失。要想环境变量每次登录都存在，可以向`/etc/profile`文件尾用export添加环境变量。这是因为**每次登录时系统会自动执行/etc/profile脚本**。

## file format not recognized

编译时提示错误如：

```shell
/usr/bin/ld:./libxxx.so: file format not recognized; treating as linker script
/usr/bin/ld:./libxxx.so:2: syntax error
```

意思是这个so文件格式不识别，ld试图将它当作链接文件来看待，但仍然出错。查看发现该so文件大小只有几字节，且附近有一个带后缀.1的文件libxxx.so.1。**原因是该so文件实际是一个软链接文件，链接对象就是libxxx.so.1**；但由于该模块提供的lib压缩包是在windows下解压后通过远程文件系统挂载到linux系统上的，软连接文件被当成普通文件解压了。解决办法是重新创建软连接或直接在linux下解压。

> so后面带的.1是版本号为1的意思。这是linux下动态库版本控制的一种方法。具体可以看[动态链接库的版本控制](https://cvvz.github.io/post/version-control-of-shared-object)一文。

## undefined reference to

`undefined reference to` 即未定义的引用，表示某函数被声明了但是却没有找到对应的实现，这种情况是可以编译成功的，但是链接会失败。类似的还有`Undeclared references`，即未声明的引用，表示找不到函数声明。

出现这种情况只能考虑是编译时还有必要的库没有链接，对库文件夹中所有的库执行`nm`命令，过滤该函数名，找到该函数定义(`T类`)所在的库文件，并将其加入到编译链接库中。

> `nm`用于打印库或可执行文件中的符号名：
>
> * T类：是在库中定义的函数，用T表示；
> * U类：是在库中被调用，但并没有在库中定义(表明需要其他库支持)，用U表示；
