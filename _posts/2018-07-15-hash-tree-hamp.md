---
layout: post
title: HASH ARRAY MAPPED TRIE-HAMT
categories: [datastructure]
description: datastructure
keywords: datastructure
---

# 介绍

`HASH ARRAY MAPPED TRIE(HAMT)`是一种哈希数据结构[^1]，速度高效而且更小的空间开销。而且避免了传统的哈希结构的问题：

* 创建一个经验值的哈希表
* 哈希表扩缩容

HAMT的树型结构如下图所示：

![](/images/posts/datastructure/hamt-level.png)

`HAMT`主要根据hash值长度(`uint32`/`uint64`)的不同，主要实现有`HAMT-32`和`HAMT-64`，下面的叙述以`HAMT-64`为模板。

对于任意Key，可以计算一个64bit(8byte)的哈希值。然后我们使用这个64bit的哈希值来构建`TRIE`。其中`TRIE`的每一层使用64bit哈希值中的`t`bit，从高位还是低位截取都可以，图中表示的为从低位开始截取。与这`t`bit对应的，在`TRIE`的每一层，我们提供一个2<sup>t</sup>大小的bitmap和array存储下一级指针，则生成的`TRIE`最大深度为`floor(64/t)`（通常`t`至少为5，这样bitmap为32bit，即一个`uint32`，对应各种位运算来说比较方便，而且其对应的32长度的指针数组也不会造成太大的空间浪费，这样对应的`TRIE`的最大深度为12）。

## 查询
当查询时，从`TRIE`的根节点开始，每层取哈希值对应的`level * t`到 `(leve+1) * t`位的bit值为该层的索引`index`。然后在该层的bitmap的第`index`位进行判断：

* 如果是0则表示没有后继，说明不存在目标Key；
* 如果是1，则从该层指针数组中取得指针对应的节点：
  1. 如果指针对应的节点为`K-V`节点，如果`K-V`节点中的Key与目标Key相同，则查找到了对应的Key；
  2. 如果`K-V`节点中的Key与目标Key不相同，则说明不存在目标Key

## 插入
当插入新值时，与查询相似，根据插入Key对应的哈希值来逐层查询`TRIE`，
* 当对应层的bitmap对应的`index`为0时，则创建一个`K-V`节点，将插入值的哈希值、Key、Value存储在`K-V`节点中，然后将`K-V`节点指针存储在该层指针数组的第`index`位置，然后插入成功。
* 当对应层的bitmap对应的`index`为1时，则从指针数组取出对应的指针进行判断：
  1. 如果是`TRIE`节点指针，说明还有下一层，则递归进行插入。
  2. 如果是`K-V`节点则先对哈希值进行判断，如果哈希值不同且为到达最大深度时，则创建一个`TRIE`节点，将原来的`K-V`节点和待插入值的`K-V`节点插入到新建的`TRIE`节点中，然后把新建的`TRIE`节点取代原来指针数组的第`index`位置，这样就使`TRIE`增长了一层，如下图所示。
  3. 如果是`K-V`节点且达到了最大深度、或者对应的哈希值相同，则说明发生了哈希碰撞，则此时使用链表法，将发生碰撞的`K-V`节点使用链表级联。

  ![](/images/posts/datastructure/hamp-insert.png)

## 删除

删除其实是插入的反操作，当查找到对应的Key时，删除其对应的`K-V`节点，清除bitmap的对应位，然后还需一步：`TRIE`的收缩。此时判断对应level的`TRIE`节点下bitmap中1的个数，如果bitmap只有1位置为1，且是一个`K-V`节点时，则可以删除本level的`TRIE`。这个过程是上图的反向操作，从右向左看即是。

# `HAMT`的优点

## 哈希利用率
从`HAMT`的结构看出，其比较一般的哈希表来说，其更加充分利用了哈希值的各个bit，一般的哈希表大小有限，通常采用哈希值对表长度取余的方式（`Index = HashValue % Length`），只能使用哈希值的很少几位，因此发生哈希碰撞的概率更高。如果取上述`t`为5的话，`HAMT-32`可以使用哈希值的30bit，`HAMT-64`可以使用哈希值的60bit。也可以看出，`HAMT-64`的哈希碰撞概率比`HAMT-32`更低。

## 空间利用率
当哈希表的填充率很高时，`HAMT`的空间利用率比一个的开放地址法的哈希表更高。

但是当哈希表很稀疏时，每个`TRIE`节点的指针数组的填充率就会下降，导致空间利用率下降，因此`TRIE`节点还有一个变种实现，稀疏节点：每个`TRIE`节点下的指针数组不再是固定2<sup>t</sup>大小的，因为在哈希表很稀疏的清下，会浪费很多空间。则我们事每个`TRIE`节点的指针数组为变长数组，当`TRIE`节点的bitmap只有一位为1时，则，指针数组的长度为1，当`TRIE`节点的bitmap只有两位为1时，则指针数组的长度为2，如下图所示：

![](/images/posts/datastructure/hamt-sparse.png)

我们可以很容易得出指针数组位置与bitmap的映射关系为：
* 指针数组的长度为bitmap中置1的bit个数
* bitmap中`i`bit为1时，其对应指针数组中的位置为`x`，则`x`等于`(bitmap & ( 1<<i - 1))`中1的个数

这里可以看到，2条对应关系都需要用到计算bitmap中1的个数，这个方法有一个高效算法，在golang的[`math/bits`](https://github.com/golang/go/blob/07b81912d4f7e7faaa0e2367ae834b92f4867819/src/math/bits/bits.go#L128-L162)中就有实现。

当然使用这种实现，效率上比直接分配固定大小的`TRIE`节点略低一些，但是有起高效的算法支持，效率上的损失不会特别大。当然还有一种方法就是采用自省地混合实现。自省是指，`HAMT`内部自己可以通过观察`TRIE`节点的填充率来决定使用哪个实现，在`go-hamt`[^2]就采用了这种实现。

## 不可变性
根据`HAMT`的层次结构，很容易实现不可变的数据结构，这样非常利于实现一种持久化存储结构，比如`BTREE`。

![](/images/posts/datastructure/hamt-imm.png)

从上图可以看出，`HAMT`在一次更新操作后，只需要修改沿途的几个`TRIE`节点，未触及的路径可以直接引用，这样我们可以生成沿途节点的副本，很容易就实现一个`COW`的实现。

[^1]: [Ideal Hash Trees](/images/posts/datastructure/idealhashtrees.pdf)
[^2]: [go-hamt](https://github.com/lleo/go-hamt)