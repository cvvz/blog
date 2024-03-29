---
title: "重学数据结构和算法"
date: 2021-07-18T21:14:55+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["coding", "golang"]
tags: ["coding", "golang"]
---

> [我的题库](https://github.com/cvvz/go-algorithm)
>
> [我的leetcode](https://leetcode-cn.com/u/cvz)

## 常见数据结构

### 数组

数组的时间效率很高，但是空间效率很低，而且不安全，比如访问越界造成踩内存。

很多高级语言都基于基础的数组实现了**动态数组**，比如Java中的ArrayList、C++ STL中的vector和golang中的slice，动态数组的优势在于可以动态扩容，使用起来很方便，**在实现算法时更加handy**。但是由于封装了额外的数据迁移等操作，时间效率上不如数组高。

### 链表

### 单链表、双链表、循环链表

🌟**技巧：使用哨兵节点简化插入和删除节点的逻辑。**

> 所有高级数据结构都是在数组和链表的基础上衍生出来的。

### 栈和队列

“操作受限”的线性表，只支持两种基本操作：push, pop。

递归的算法都可以用栈来实现。

> 高性能定时器，除了可以用堆实现（**比如golang的timer就是用最小四叉堆**），还可以用**环形队列**，详见 [时间轮算法 HashedWheelTimer](https://zhuanlan.zhihu.com/p/65835110) 、[层级时间轮的 Golang 实现](http://russellluo.com/2018/10/golang-implementation-of-hierarchical-timing-wheels.html)

### hash表

高级语言内置了hash表，比如Java 中的 HashMap，golang中的map数据类型。

> **Java JDK中自带TreeMap**，可以按 key 进行排序。
>
> 但是在 Go 语言的“简约设计”面前，这些都是不存在的——Go 只提供了最基础的 hash map。并且，在借助 range 关键字对 Go 的 map 进行遍历访问的时候，会对 map 的 key 的顺序做随机化处理，也就是说即使是同一个 map 在同一个程序里进行两次相同的遍历，前后两轮访问 key 的顺序也是随机化的。(可以在[这里](https://go.dev/play/p/LYJSbQBjWa6)进行验证)。
>
> 我们可以自己实现，或者借助其他开源解决方案，比如[emirpasic/gods](https://github.com/emirpasic/gods)。

1. hash表来源于**数组**，借助**散列函数**对数组这种数据结构进行扩展，也就是将key映射为数组下标index。

2. 将key转化为数组下标的方法称为**散列函数**，散列函数的计算结果称为**hash值**。数据存储在**hash值**对应的**数组下标**位置。

> 实现hash表所使用的hash算法要求执行**速度快**，值是否能**平均分布**在各个槽中（比如简单的取模算法）。并不是很在乎**安全性**（是否能反向解密出原始数据）和**hash冲突**（哈希值相同）。所以不会使用**加密用**的哈希算法。
>
> hash函数,有加密型和非加密型。加密型的一般用于加密数据、数字摘要等，典型代表就是md5、sha1、sha256、aes256 。非加密型的一般就是查找。

### hash算法

hash算法的应用：

* **hash表中的散列函数。**
* [在分布式系统中的应用](https://time.geekbang.org/column/article/67388)：
  1. 负载均衡（**一致性哈希**）
  2. 数据分片等
* 加密、验证，比如go mod使用go.sum文件来验证依赖库是否发生变化

### 树

#### 二叉树

* 二叉树的遍历：
  * 广度优先搜索：层序遍历
  * 深度优先搜索：前中后序遍历。
   > **“前中后”指的是当前节点和左右子树谁先打印**
   >
   > 深度搜索可以用栈或者递归，递归算法实现起来很简单；广度搜索只能用队列。
* 完全二叉树和满二叉树：
   > **完全二叉树可以用数组实现，而堆是一个完全二叉树，因此堆是用数组实现的**
* 二叉查找树：
   > **中序遍历二叉查找树可以得到一个有序数组，和有序数组的二分查类似**
* 平衡二叉查找树：二叉查找树在频繁的动态更新过程中，可能会出现树的高度远大于 log2n 的情况，从而导致各个操作的效率下降。**极端情况下，会退化为链表**，时间复杂度会退化到 O(n)。所以又发明了**平衡二叉查找树**。
    > “平衡”的意思，其实就是让整棵树左右看起来比较“对称”、比较“平衡”，不要出现左子树很高、右子树很矮的情况。这样就能让整棵树的高度相对来说低一些，相应的插入、删除、查找等操作的效率高一些。
* 红黑树：**红黑树是一种平衡二叉查找树**。它是为了解决普通二叉查找树在数据更新的过程中，复杂度退化的问题而产生的。
    > **Java中的TreeMap可以对哈希表的key进行排序，底层就用到了红黑树**。

#### [堆/优先级队列](https://leetcode-cn.com/tag/heap-priority-queue/problemset/)

> 思考：为啥堆又叫做优先级队列？**看起来堆似乎是一个树，而队列是数组。但是实际上堆是一个完全二叉树，而完全二叉树通常用数组表示，所以堆也是用数组来存储的**。

**堆的核心操作：**

1. 核心中的核心：堆化（heapify）
   1. **从上往下[down](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=101-119)**
   2. **从下往上[up](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=90-99)**
2. 替换堆顶元素，然后从上往下堆化：[Pop](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=57-65)
3. 向堆尾添加元素：[Push](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=50-55)，并进行从下往上堆化
4. 建堆：[Init](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=42-48)

[**堆的应用**](https://time.geekbang.org/column/article/70187):

1. 高性能定时器
   > golang官方库Timer的实现，和informer中的延迟队列的实现都用到了堆
2. [数据流的中位数](https://leetcode-cn.com/problems/find-median-from-data-stream/)、99线问题
3. TopK问题
   1. [静态topK](https://leetcode-cn.com/problems/kth-largest-element-in-an-array/)
   2. [动态topK](https://leetcode-cn.com/problems/kth-largest-element-in-a-stream/)
   > **对于静态topK的问题，如果数据量能够直接加载进内存，那么用快速排序思想求解会更快**
   >
   > **但对于动态topK的问题，就只能使用堆来解决了，因为要通过网络/磁盘IO构造流数据**

### 图

**数据结构（存储方法）**：邻接矩阵（二位数组） 和 邻接表

* **邻接矩阵**
  > **一般图的BFS、DFS，有向无环图（DAG）拓扑排序算法，以邻接矩阵（二维数组）的形式考察的比较的多。而树型结构（链表）一般用二叉树的BFS和DFS来考察。**
* 邻接表

## 常见算法

### 排序

**冒泡排序、插入排序、选择排序**：三种O(n2)的**简单无脑的**算法，数据量不大时可以用一用。

**归并排序**：不断二分，递归下去，然后“**从下往上**”merge。由于这个merge无法原地执行，因此空间复杂度为O(n)。

> 虽然归并排序用到了递归，但是空间复杂度不是O(n2)，因为每次merge时，下面那一层的内存就被释放掉了。
>
> 归并排序是stable的，所以**golang的[sort.Stable](https://github.com/golang/go/blob/master/src/sort/sort.go#L378-L404)使用归并排序实现**。

**快速排序**：选择一个pivot，然后“**从上往下**”不断进行**原地的**partition操作，由于partition是原地的，因此空间复杂度为O(1)。
   > **快速排序的关键是原地partition**
因为快速排序优化了内存使用，所以应用比归并排序要广泛。但是快速排序在最坏情况下的时间复杂度是 O(n2)，**需要解决这个“复杂度恶化”的问题**。这个问题的根因还是我们选择的分区点（pivot）不合理导致的，理想的分区点应该是左右两边的数据量差不多，最好能二分，这样递归的层次就最少。pivot的选择方法最常见的有两种：

1. 三数取中法：选择三个或者更多的数，选择他们的中间值作为pivot。
2. 随机法：从概率上说不会一直选的都是最差的点作为pivot。

**归并排序**和**快速排序**都用到了**分治**思想。

我们可以借鉴快排的思想，来解决非排序的问题，比如用O(n)的时间复杂度解决[静态topK](https://leetcode-cn.com/problems/kth-largest-element-in-an-array/)问题。

**堆排序**：[heapSort](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/sort/sort.go;l=66-81)

1. 建堆：
   1. 方法一：**从第一个非叶子结点开始**，依次执行从上往下堆化，即[Init](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=42-48)
   2. 方法二：不断入队，并对新入队节点执行从下往上堆化，即[Push](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=50-55)
2. 调整：交换堆顶和堆尾元素，堆尾元素出队，从上往下重新堆化，即[Pop](https://cs.opensource.google/go/go/+/refs/tags/go1.17.5:src/container/heap/heap.go;l=57-65)

### 搜索

**广度优先搜索和深度优先搜索**在算法面试中都是非常有用的工具，也就是说**掌握BFS和DFS是基础要求**，很多时候**使用任意一种**搜索算法就能解决某些与图相关的面试题。

如果面试题要求在无权图中找出两个节点之间的最短距离，那么广度优先搜索可能是更合适的算法。
如果面试题要求找出符合条件的路径，那么深度优先搜索可能是更合适的算法。

### 二分查找

比较简单

## 基本算法思想

### [贪心](https://leetcode-cn.com/tag/greedy/problemset/)

> 严格地证明贪心算法的正确性，是非常复杂的，需要涉及比较多的数学推理。而且，从实践的角度来说，大部分能用贪心算法解决的问题，贪心算法的正确性都是显而易见的，也不需要严格的数学推导证明。

### [分治](https://leetcode-cn.com/tag/divide-and-conquer/problemset/)

> 分治经常用在海量数据处理的场景下，内存无法直接装载全部数据，就将数据分批装载进内存处理，再将结果进行合并。（给1TB的订单排序）
>
> **要判断清楚数据规模是不是可以直接装载进内存**，比如获取10亿个整数第k大的数：10亿个整数 = 80亿Byte( int=64bit ) ≈（不足）8GB，这个时候要考虑单机的实际可用内存大小是否可以直接装载8GB。如果能直接装进去，那么可以用快排思想做；如果不能直接装进去，那么构造大小为k的堆，然后从文件读数据进行处理。
> 还有的时候如果文件太大，那么需要进行**数据分片**，分成若干个小文件以后，再分而治之，即逐个处理可以直接载入内存的小文件，最后合并得到结果。

### [回溯](https://leetcode-cn.com/tag/backtracking/problemset/)

> 通常**递归实现**。DFS利用的就是回溯算法思想。

### [动态规划](https://leetcode-cn.com/tag/dynamic-programming/problemset/)

> dp的主要学习难点跟递归类似，那就是，求解问题的过程不太符合人类常规的线性思维方式。
