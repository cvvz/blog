---
title: "读锁有什么用？"
date: 2021-11-29T15:37:20+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["code", "golang"]
tags: ["code", "golang"]
---

问这个问题的起因是我在进行code review时，对一个读map前加读锁的代码留下了如下comment：

1. 这里从map中读出脏数据不会有什么问题。
2. 即使加了读锁还是不能避免读脏数据的问题。所以这里没必要加读锁

但随后自己隐约觉得哪里不对，查了一下在golang里map就是不允许并发读写的，其实对map加读写锁就跟 `if err != nil` 一样常见。自己犯了一个小白错误。

回过头来也庆幸自己会犯这种错误，不然我可能永远只记得 **map是非线程安全的，使用时要加锁** 这个结论，以及使用前map加锁的肌肉记忆，却并不会停下来想一想为什么要这么做。

之所以会犯这个“初学者”错误，原因在于我**搞混了几种线程安全/数据竞争发生的维度**。虽然锁都是为了解决线程安全/数据竞争的问题的，但是线程安全问题可以出现在如下几个维度，造成的问题也不尽相同，所以引入锁要解决的问题也就不相同。

### 应用维度

最常见的就是数据库的隔离级别，RC就是为了防止脏读，通过加写锁，避免执行事务写数据期间被读到脏数据；RR就是在读之前加读锁，这样就能保证一次事务中，多次读读到的数据是相同的。

**我在code review中犯的错误，是因为我以为给map加读锁就是为了避免数据不一致。然而又因为业务逻辑中并不涉及事务操作，所以我认为没必要加读锁，而且仅仅在读数据的时候加一下锁也避免不了脏读，因为不是一把事务锁。**

### 语言运行时维度

1. golang的slice在append时，如果没有超出cap大小，那么是不会重新分配内存的，这时数据是直接append到底层的数组中的。如果两个线程并发的append同一个slice，那么就可能写同一片内存，这样可能会导致append后的总数不符合预期（变少）。这种情况我们也要用锁，这是语言运行时实现层面对程序员的约束。

2. golang的map被设计为非并发安全的（[原因](https://go.dev/doc/faq#atomic_maps)），在应用层如果并发时不加读锁或者写锁，就可能会报错，`fatal error: concurrent map read and map write`。我们可以通过[golang race detector](https://go.dev/doc/articles/race_detector)进行检查。那为什么map的读写需要加锁呢（不管是由应用程序加，还是由语言特性加）？原因在于map类型的一次读写不是原子性的（需要进行哈希计算、解决哈希冲突等等）。**注意这里所说的原子性，不是指上面说的应用层的原子性；而是语言运行时实现层面对程序员的约束。**

3. 还有一种是在语言实现时就设计为并非原子性赋值的，可以看看这篇文章中的[讨论](https://cloud.tencent.com/developer/article/1810536)，典型的有复数类型。

### 编译器/操作系统维度

**这个原子指的是编译器/操作系统层面的原子性，而不是上述两种原子性**！

这里又分为三种情形：

1. 一种是并发执行i++，由于i++在编译后在底层的实现是先读i，再写i，分为两步，所以并发时可能有问题。

2. 另一种是读写超过一个计算机字长的数据（比如struct，或者复合类型，比如string），这时肯定无法通过一条机器指令进行读写的

3. 第三种是**编译器或处理器可能会对单线程中的读写操作进行重新排序**，只要保证重新排序后不影响单线程的执行即可。但是由于进行了重排序，所以对于多线程而言，读写操作的顺序就可能变化，就不能想当然的认为不用加锁！！
   > 所以[go内存模型](https://go.dev/ref/mem)建议我们`Don't be clever.`，绝大多数情况下，请不要自作聪明。`To serialize access, protect the data with channel operations or other synchronization primitives such as those in the sync and sync/atomic packages.`**为了保证读写顺序，使用channel或者其他同步原语比如sync和sync/atomic包中提供的方法来保护你的数据。**

### golang中的COW

虽然加锁很快，并且在大多数场景下我们都不会碰到锁性能问题。但是在某些极端场景下，如果对map频繁的加读写锁，还是会带来一些性能损失。我们可以采取copy-on-write的方式避免对map加锁，从而提高性能。

方法就是对map的写操作变成整个map的替换，由于map替换本身可以看成是进行一次指针赋值，而指针赋值在golang中是原子性的，所以就不会存在map并发问题。

典型代码可以查看[煎鱼](https://github.com/Terry-Mao)的[这段代码](https://github.com/Terry-Mao/gopush-cluster/blob/master/rpc/rand_lb.go#L221-L232)

> 但是这段代码也引起了一些[争议](https://github.com/Terry-Mao/gopush-cluster/issues/44)，主要争议在于golang的指针赋值到底是不是原子的。关于这个问题，可以看看[这个问答](https://stackoverflow.com/questions/21447463/is-assigning-a-pointer-atomic-in-go)，简单来说就是，除了 `sync.atomic` 中的操作以外，其他任何操作都不建议看作是原子性的。因为即使当前版本golang中的实现是原子的，不代表以后某一天不会被改成非原子的。
>
> 这可能也是[go内存模型](https://go.dev/ref/mem)里让我们`Don't be clever.`的原因吧。**我个人的理解是，如果你写golang，那么你需要能够接受一些性能损耗，而不要自作聪明的想一些奇技淫巧；如果你要追求极致的性能，你可以用C/C++/Rust实现 :)**
