---
title: "不同上下文中的并发问题"
date: 2021-11-29T15:37:20+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["code", "golang"]
tags: ["code", "golang"]
---

问这个问题的起因是我在进行code review时，对一个读map前加读锁的代码留下了如下comment：

`单独给一个读操作加锁没有必要。`

但随后自己隐约觉得哪里不对，查了一下，原来golang的map就是不允许并发读写的，其实对map加读写锁就跟 `if err != nil` 一样常见。自己犯了一个小白错误。

回过头来也庆幸自己会犯这种错误，不然我可能永远只记得 **map是非线程安全的，使用时要加锁** 这个结论，以及使用前map加锁的肌肉记忆，却并不会停下来想一想经常挂在嘴边的线程安全、并发到底在说什么，以及为什么要加锁。

之所以会犯这个初学者错误，原因在于我搞混了几种并发问题发生的维度。**虽然传统的锁（或者golang更推荐的CSP）都是为了解决并发问题，但是在不同的上下文中，要解决的并发问题也各不相同**。

## 业务逻辑层

比如数据库的Serializable隔离级别，读加共享（读）锁，写加排他（写）锁，读写互斥。在业务逻辑层，通常最简单有效的解决并发问题的方法就是**通过加锁把并发变成串行**。我在code review中犯的错误，是因为我以为给map加锁是为了解决业务逻辑上的并发问题，而从业务逻辑上看，没有必要单独给一个读操作加锁，因为并不是一连串操作。

## 语言运行时层

1. golang的slice在append时，如果没有超出cap大小，那么是不会重新分配内存的，这时数据是直接append到底层的数组中的。如果两个线程并发的append同一个slice，那么就可能写同一片内存，这样可能会导致append后的总数不符合预期（变少）。

2. golang的map被设计为非并发安全的（[原因](https://go.dev/doc/faq#atomic_maps)），在应用层如果并发时不加读锁或者写锁，就可能会报错，`fatal error: concurrent map read and map write`。我们可以通过[golang race detector](https://go.dev/doc/articles/race_detector)进行检查。那为什么map的读写需要加锁呢（不管是加在业务逻辑里还是加在语言运行时里）？原因在于map类型的一次读写不是原子性的（需要进行哈希计算、解决哈希冲突等）。

## 编译器/操作系统层

这里又分为四种情形分析：

1. 第一种是并发执行i++，由于i++在编译后在底层的实现是先读i，再+1，再写i，分为三步，操作系统并不保证这三步是原子性的，所以并发时可能有问题。

2. 第二种仍然考虑并发执行i++，即使三步操作是原子性的，但是由于现代CPU是多核心的，每个核心都有自己的缓存，写操作实际是写入各自的缓存中再写到L3 cache和内存中，所以并发执行i++是否线程安全还依赖写操作是写到cpu自己的缓存中还是写到内存中。

3. 第三种是读写超过一个计算机字长的数据（比如struct，或者复合类型，比如string，golang的复数类型等），和第一条类似，这时也无法通过一条机器指令进行读写。

4. 第四种，编译器/处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的，这就是指令重排序（Instruction Reorder）。**指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性。**
   > 所以[go内存模型](https://go.dev/ref/mem)建议我们`Don't be clever.`，绝大多数情况下，请不要自作聪明。`To serialize access, protect the data with channel operations or other synchronization primitives such as those in the sync and sync/atomic packages.`**为了保证读写顺序，使用channel或者其他同步原语比如sync和sync/atomic包中提供的方法来保护你的数据。**

### 拓展：golang中的COW

虽然加锁很快，并且在大多数场景下我们都不会碰到锁性能问题。但是在某些极端场景下，如果对map频繁的加读写锁，还是会带来一些性能损失。我们可以采取copy-on-write（**只有在需要对内存进行写入的时候才进行拷贝，读时直接读原内存**）的方式避免对map加锁，从而提高性能。

方法就是**把对map的写入操作变成新map的替换**，由于map替换本身可以看成是进行一次指针赋值，而指针赋值在golang中是原子性的，所以就不会存在map并发问题。

典型代码可以查看[毛剑](https://github.com/Terry-Mao)的[这段代码](https://github.com/Terry-Mao/gopush-cluster/blob/master/rpc/rand_lb.go#L221-L232)

> 但是这段代码也引起了一些[争议](https://github.com/Terry-Mao/gopush-cluster/issues/44)，主要争议在于golang的指针赋值到底是不是原子的。关于这个问题，可以看看[这个问答](https://stackoverflow.com/questions/21447463/is-assigning-a-pointer-atomic-in-go)，简单来说就是，除了 `sync.atomic` 中的操作以外，其他任何操作都不建议看作是原子性的。因为即使当前版本golang中的实现是原子的，不代表以后某一天不会被改成非原子的。
>
> 这可能也是[go内存模型](https://go.dev/ref/mem)里让我们`Don't be clever.`的原因吧。**我个人的理解是，如果你写golang，可能需要适当的“笨”一点，让运行时帮你做底层的事情；如果你想追求极致的性能，是一个“聪明”的程序员，你可以选择更加底层的语言 :)**
