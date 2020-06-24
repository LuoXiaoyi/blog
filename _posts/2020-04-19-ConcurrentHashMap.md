---
layout:    post
title:     理解 ConcurrentHashMap
category:  JDK
tags:  ConcurrentHashMap
---

> 众所周知，```ConcurrentHashMap``` 是在 ```JDK 1.5``` 中由 ```Doug Lea``` 大神引入的一个可以在高并发场景下使用的且线程安全的高性能 KV Map。如果您在此之前还不知道这个数据结构或者说未曾使用过，那这篇文章可能就不太适合您了，您可以先熟悉一下它的使用场景以及使用方法，再来看这篇文章，对您的帮助应该会更大。通过阅读本博客，笔者希望您能清晰的了解到以下几个要点
> 1. ```ConcurrentHashMap``` 的设计目标
> 2. ```ConcurrentHashMap``` 的各个设计目标的实现原理
> 3. ```ConcurrentHashMap``` 的重点方法算法解读

## ConcurrentHashMap 设计目标
终极目标就是实现一个```线程安全```、```高吞吐```、```低时延```的数据结构，为使用者能够在高并发场景下快速、高效的开发出高性能并发应用做铺垫。
* 支持 ```线程安全``` ，在多线程的环境下，使用者可以不需要添加额外的同步操作，安全的对 ConcurrentHashMap 对象进行增、删、改、查等操作；
* 支持 ```高吞吐``` ，即在多线程环境下， ConcurrentHashMap 对象能够尽可能高并发，甚至是并行的响应各个线程的增、删、改、查等操作；
* 支持 ```低时延``` ，即在多线程环境下，能够在尽可能短的 rt（response time）内让各线程完对 ConcurrentHashMap 对象的增、删、改、查等操作。

## ConcurrentHashMap 各个设计目标的实现原理
1. 为了达到```线程安全```、```高吞吐```，```Doug Lea```大神在实现 ConcurrentHashMap 的过程当中巧妙的应用了```同步锁``` 与 ```CAS``` 相结合的方式来达成目的；
2. 为了减少 Map 中内存的使用，对于每个 bucket 的操作并没有单独的使用一个锁对象，而是直接使用了 bucket 节点自带的 ```synchronized monitor``` 锁对象；
3. 为了提升数据的插入速率，在 bucket 的第一个元素插入时，会使用 ```CAS``` 操作，而非使用锁（笔者认为，这里主要的原因可能在于，很大概率下，大部分 bucket 上应该只会有一个元素，当然这是建立在良好的 hash 算法加合适的 bucket 数组大小的前提之下的）；
4. 为了提升查询速率，即```低时延```，作者又做了对同一 bucket 上的对象在某些特定情况下（链表的节点数目大于等于 8）做从```链表```到```红黑树```的存储转化；
5. 从```链表```到```红黑树```阈值设置成 8 是基于进入 Map 的元素在某个 bucket 上出现的次数遵循 ```泊松分布``` 这个假设，并基于平均发生 0.5 次（要么发生要么不发生，故取平均值 0.5）而得到的结果，即 P(X) = exp(-m) * pow(m, X)/X!。其中 m = 0.5，那当 X = 8 时，P(X) 约等于 5.876141839043835E-8，可见这种同一个 bucket 上同时存在 8 个及以上的元素发生的概率已经极低了，但尽管如此，为了防止极低概率事件的发生，并且保证查询效率，当大于 8 时，转红黑树；当然，这里有一点需要注意的是，并不是任何情况下，在链表元素数量大于等于 8 时都会做转化，只有在 table 的总 bucket 数量大于等于 MIN_TREEIFY_CAPACITY（64）才会做转化，否则只会对 Map 做扩容处理；
6. 两个线程基于随机 hash 的前提下，在访问不同元素造成锁竞争的可能性可以粗略的由如下计算公式得出：1 / (8 * #elements)，这里的 8 和 4、5 中的 8 是一致的；
7. 使用了 ```@sun.misc.Contended``` 来避免伪共享，用于高效维护 Map 的 size；

## ```ConcurrentHashMap``` 重点方法算法解读
源码实在太多，一一解读实在太多，而且也没必要，这里只看相关方法算法，即描述执行流程。
### public V put(K key, V value)
插入新的 KV 对，如果 key 已经存在，则返回之前的 value，否则返回 null，具体的算法如下：
* key/value 空值判断，若二者任一为空，则抛出 ```NullPointerException```；
* 对 ```key.hashCode()``` 做 ```spread``` 计算，算法很简单，就是用高 16 位与低 16 位做异或运算，并与 ```0x7fffffff``` 进行位与运算，这样做的目的是为了使减少 hash 冲突，让其变得更分散；
* 执行 key/value 的插入操作；
  * 如果当前 Map 为空，则执行 Map 的初始化工作；
  * 如果当前 key 所要插入的位置为 null，则执行 ```CAS``` 操作，将当前的 KV 对插入到 table 对应的 bucket 上，若 ```CAS``` 执行成功，则完成插入，否则继续；
  * 若当前要插入的位置不为 null，且当前 bucket 上的 hash 值为 MOVED，则表示当前 bucket 上的所有元素已经迁移到了新的 table 上，但是当前还未完成扩容的过程，则尝试帮助扩容，即调用 ```helpTransfer(tab, f)``` 方法，其实内部最终调用的是 ```transfer(tab, nextTab)```，这其中的 ```netxTab``` 即为扩容后的新表；
  * 对插入位置的 bucket 上的 node 加锁，这里的锁是 ```synchronized monitor```
    * 如果当前 node 的 hash 值大于等于 0，则表示是普通的链表节点，从而通过 node 的 next 指针遍历链表，若链表中以及存在相同的 key，则替换之，否则在链表的末尾插入新的 KV 对；
    * 如果当前 node 为 ```TreeBin``` 则在当前红黑树中插入新的 KV 对，同样的，如果事先存在相同 key 的值，则替换之；
  * 若上一步新 KV 对插入的是在普通链表上，且节点的元素个数已经大于等于 8，则调用 ```treeifyBin``` 方法执行 ```链表``` 到 ```红黑树``` 的转换，当然该操作并非一定会执行真正的转换，当且仅当 table 数组的 length 大于等于 MIN_TREEIFY_CAPACITY（64） 才会执行转换操作；
* 调用 ```addCount``` 更新 Map 的 size

### private final void treeifyBin(Node<K,V>[] tab, int index) 
如果有必要，则把 tab 当前的 index 位置的 ```链表``` 转换成 ```红黑树```，具体算法如下：
* 若当前 tab.lengh < MIN_TREEIFY_CAPACITY（16），则执行扩容，从而更好的适配当前的 map 中元素的数量；
* 否则执行转换操作
  * 将原来的普通链表节点统一转换为 ```TreeNode``` 节点组成的链表；
  * 调用 ```TreeBin``` 的构造函数，传入已经构建好的 ```TreeNode``` 链表，创建一颗 ```红黑树```，这个构建过程就不展开了，关于红黑树的构建，又可以展开写一大篇，我之前也写过一篇；
  * 将 ```TreeBin``` 放在 tab 的 index 位置上（这里需要注意的是，TreeBin 也是一个 Node，只是比较特殊，其 hash 值为 TREEBIN = -2）。

### private final void addCount(long x, int check)


### private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)
