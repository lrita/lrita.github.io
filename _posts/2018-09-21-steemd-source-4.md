---
layout: post
title: steemd 源码分析4 witness
categories: [c++, steem, blockchain]
description: c++ steem blockchain
keywords: c++ steemd steem blockchain
---

# Witness是什么
在steem区块链的网络中，节点分为两种角色，`Witness`和`非Witness`。

`非Witness`是steem区块链网络的普通参与者，可以从其他节点接受数据、传播数据。

`Witness`则是steem区块链网络的维持者，他们通常要保持24小时在线，是steem区块链的基石，负责区块链的收集交易、打包交易和出块工作，同时还运行着steem的经济学相关的工作。正应为他们要不断的在线、工作，会消耗一定的资金来维持服务，因此，在他们出块后会得到一定的[`STEEM Power`](https://www.steem.center/index.php?title=STEEM_Power_(SP))作为回报。

在steem区块链的网络中，对`Witness`进行不停的调度，在每一调度轮次中，会抽获取投票最高的前20个`Witness`节点和一个随机抽取的`Witness`节点组成一个21节点的集合（有些地方讲的是19个投票最高的节点和1个矿工节点和1个随机节点，这是比较老的算法，从第17次硬分叉后，删除了矿工节点，投票最高的节点变为20个），在洗牌后依次负责出块，每3秒钟出一个块。如果负责出块的`Witness`节点在自己负责的时间范围内没有进行出块，不影响负责下一个时间区间的`Witness`节点进行出块，但是没有及时出块的`Witness`节点不会获得报酬并且可能被投票出局[^1]。

# 关于选举

每个steem用户可以给30个`Witness`进行投票。会使用steemd客户端的用户，可以通过steemd客户端进行投票。其他用户还可以在[steemit.com](https://steemit.com/~witnesses)网站上进行投票。

另外用户还可以设置代理投票，然后将自己投票权重转义给代理人，代理人负责投票。

# 出块
当用户自身成为`Witness`后，在自己节点启动时，可以开启`witness_plugin`，在启动后，`witness_plugin`会启动一个[定时器](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/plugins/witness/witness_plugin.cpp#L469-L478)来定期调用[`block_production_loop`](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/plugins/witness/witness_plugin.cpp#L480-L547)函数来判断，是否轮到自己出块。（注意：其中判断时，会调用[`get_slot_time(1)`](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/database.cpp#L1145-L1168)，其意义为获取下一次该出块的时间）。如果[当前时间对应的`Witness`是自己，则进行出块](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/plugins/witness/witness_plugin.cpp#L549-L623)。

# 调度
当网络中第一个节点启动后，会创建出一个`创世块`进行冷启动，此后在每一次接受一个新块后，都会调用[`update_witness_schedule`](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/database.cpp#L3234)进行一次调度的更新。[当当前块高度能被21整除时](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/witness_schedule.cpp#L369)，意味着，一个调度周期的结束。其会重新挑选出一批`Witness`，进行下一周期的出块。这里同时意味着，如果在一个周期内有`Witness`没能成功出块时，会有一些`Witness`在一个调度周期内能出块多次。_我认为这是一个BUG，因为在同一个周期内，`Witness`之间收益是不相等的_。

当一个周期开始是，[首先按投票数进行排序，取出投票数最高的20个节点](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/witness_schedule.cpp#L144-L154)，然后[根据虚拟调度时间来选取一个备份节点](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/witness_schedule.cpp#L193-L214)。选取好足够节点后，然后进行[洗牌](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/witness_schedule.cpp#L337-L352)。

同时再调度的过程中，还会计算一些基本数据，比如：
* 选中的`Witness`中，[主流版本](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/witness_schedule.cpp#L249-L282)是什么，因为steem是通过硬分叉进程升级的，不同版本的各种逻辑会有不同，当某些策略可能需要等到网络中的多数`Witness`都升级到某个版本后才能开始生效，因此在每一轮调度中，都会刷新当前的主流版本。
* 更新一些与`Witness`相关的[经济学数据](https://github.com/steemit/steem/blob/de88134b69e50d5d4cd56484a56f00fa16faef78/libraries/chain/witness_schedule.cpp#L36-L132)

# 总结
可以看出与`Witness`相关逻辑都比较简单，加起来一共可能只有几百行代码而已，整体上跟最前面算法描述一致，没有特别多的细节需要注意。

_注意_:在调度与出块之间存在数据竞态，出块时可能访问的已经释放的内存，从而崩溃。

# 附录
[^1]: [当前steemd的witness列表和调度算法](https://steemd.com/witnesses)
