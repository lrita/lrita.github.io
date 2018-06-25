---
layout: post
title: steemd 源码分析3 逻辑主体
categories: [c++, steem, blockchain]
description: c++ steem blockchain
keywords: c++ steemd steem blockchain
---

前面几篇关于`steemd`的文章，简单描述了`steemd`中的一些模块和机制，剩下可以值得写的模块可能还剩`p2p`。再次之前，我们可以先全面了解一下`steemd`，这样能够使我们全面的了解`steemd`。

本篇文章，就串讲一下`steemd`逻辑主体，从用户的写入操作到`steemd`如何处理、传播、出块、同步等等。

# transaction的传播

`steemd`的用户只会有读写2中操作。读操作比较简单，就是通过`json_rpc`调用各种插件中的`get_xxx`方法，将用户关心的数据从本地`database`中读出返回给用户。写操作就比较复杂，因为在区块链中，任何数据的变更都需要让每个节点打成共识，传播出去。在`steemd`中，所有的变更操作都会成为一个[`operation`](https://github.com/steemit/steem/blob/62b48877d9f731c3fe00ef818e3324a0a3de3e63/libraries/protocol/include/steem/protocol/operations.hpp#L15-L97)，在`steemd`中有很多种`operation`：

```
   typedef fc::static_variant<
            vote_operation,
            comment_operation,
            transfer_operation,
            transfer_to_vesting_operation,
            withdraw_vesting_operation,
            limit_order_create_operation,
            limit_order_cancel_operation,
            feed_publish_operation,
            convert_operation,
            account_create_operation,
            account_update_operation,
            witness_update_operation,
            account_witness_vote_operation,
            account_witness_proxy_operation,
            pow_operation,
            custom_operation,
            report_over_production_operation,
            delete_comment_operation,
            custom_json_operation,
            comment_options_operation,
            set_withdraw_vesting_route_operation,
            limit_order_create2_operation,
            placeholder_a_operation,               // A new op can go here
            placeholder_b_operation,               // A new op can go here
            request_account_recovery_operation,
            recover_account_operation,
            change_recovery_account_operation,
            escrow_transfer_operation,
            escrow_dispute_operation,
            escrow_release_operation,
            pow2_operation,
            escrow_approve_operation,
            transfer_to_savings_operation,
            transfer_from_savings_operation,
            cancel_transfer_from_savings_operation,
            custom_binary_operation,
            decline_voting_rights_operation,
            reset_account_operation,
            set_reset_account_operation,
            claim_reward_balance_operation,
            delegate_vesting_shares_operation,
            account_create_with_delegation_operation,
            witness_set_properties_operation,

#ifdef STEEM_ENABLE_SMT
            /// SMT operations
            claim_reward_balance2_operation,
            smt_setup_operation,
            smt_cap_reveal_operation,
            smt_refund_operation,
            smt_setup_emissions_operation,
            smt_set_setup_parameters_operation,
            smt_set_runtime_parameters_operation,
            smt_create_operation,
#endif
            /// virtual operations below this point
            fill_convert_request_operation,
            author_reward_operation,
            curation_reward_operation,
            comment_reward_operation,
            liquidity_reward_operation,
            interest_operation,
            fill_vesting_withdraw_operation,
            fill_order_operation,
            shutdown_witness_operation,
            fill_transfer_from_savings_operation,
            hardfork_operation,
            comment_payout_update_operation,
            return_vesting_delegation_operation,
            comment_benefactor_reward_operation,
            producer_reward_operation
         > operation;
```
然后用户写操作就是创建一个对应的`operation`然后将其封装在一个[`transaction`](https://github.com/steemit/steem/blob/62b48877d9f731c3fe00ef818e3324a0a3de3e63/libraries/protocol/include/steem/protocol/transaction.hpp#L8-L107)中，然后需要签名的操作，用自己的私钥进行签名，然后发送给`steemd`。

在`steemd`收到这个`transaction`后，就会运行其逻辑处理，进行一系列的操作：

![](/images/posts/steem/steemd-api.png)

用户给`steemd`发送`transaction`，调用的是插件`network_broadcast_api`中的[`broadcast_transaction`](https://github.com/steemit/steem/blob/62b48877d9f731c3fe00ef818e3324a0a3de3e63/libraries/plugins/apis/network_broadcast_api/network_broadcast_api.cpp#L30-L37)/`broadcast_transaction_synchronous`这2个api。

①`broadcast_transaction`是进行的异步逻辑，当`steemd`收到该交易在本地节点处理完成后即可返回，`broadcast_transaction_synchronous`是进行同步逻辑，其会阻塞等待用户的这个`transaction`被打包、出块，然后同步应用到各个节点后再返回。`broadcast_transaction_synchronous`这个api在最新的代码中已经被删除。

在上图中蓝色线的路线，就是`steemd`节点通过`broadcast_transaction`api收到用户`transaction`后的主要逻辑。至于什么行为对应什么`operation`，这些都是客户端完成的。

②`steemd`在收到一个`transaction`后，先调用插件[`chain_plugin`](https://github.com/steemit/steem/tree/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/chain)中的[`accept_transaction`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/chain/chain_plugin.cpp#L564-L578)方法。③在`accept_transaction`，会通过一个队列，将全部`transaction`发送到一个处理线程中串行处理，在前面讲过`json-rpc`模块是多线程的。在处理线程中，将队列中的`transaction`依次取出，然后调用[`chain::database`](https://github.com/steemit/steem/tree/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain)的[`push_transaction`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L755-L777)方法。

④此处调用`push_transaction`就是现将用户的变更操作**临时**应用到本地节点、同时进行验证和校验。

1. 先创建一个`database`的写事务，用于时候回滚数据，因为这些变更是**临时**的。
2. 调用该`transaction`中的每个`operation`的`validate`方法，这些`validate`在[这里](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/protocol/steem_operations.cpp#L11-L678)，定义每种`operation`参数要满足哪些要求。理论上一个`transaction`中可以封装多个`operation`。
3. 校验每个`transaction`的签名和时间。
4. 将该`transaction`存储在本地的`database`中。
5. 调用通知`on_pre_apply_transaction`，这里是`steemd`中使用的一种事件通知机制。这个是在`chain::database`中使用的比较多的一个机制⑥，可以简单看做一个回调函数链表，其中的回调函数由其他插件⑧在初始化时注册进去。当调用通知时，会依次触发这些回调函数，这里回调函数的参数是`transaction`。
6. 依次调用通知`pre_apply_operation`，参数是每个`operation`。
7. ⑤依次获取每个`operation`对应的`evaluator`，然后调用其`apply`方法。这里是`steemd`业务逻辑的核心了。这里处理每种`operation`应该如何变更`database`中的数据。比如一个转账的`operation`，该给谁加余额，给谁减余额。`evaluator`也有一套注册机制，总的来说比较简单，这里就不将了，每个`operation`的`evaluator`的定义在[这里](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/steem_evaluator.cpp#L83-L2497)。在这里的这次数据变更在之前创建的写事务中，因此是**临时**。只有当该`transaction`被打包、出块、并同步到多数witness节点（75%）上后，其数据变更才是永久的。
8. 依次调用通知`post_apply_operation`。
9. 然后将该`transaction`存储在`_pending_tx`中。如果轮到本节点打包出块时，该节点就会将`_pending_tx`中的`transaction`打包成为一个新的`block`，这个留在后面讲。

在`steemd`将`transaction`应用到本地后，然后再调用`p2p::p2p_plugin`插件的[`broadcast_transaction`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/p2p/p2p_plugin.cpp#L750-L754)api⑨将其通过p2p网络广播出去⑩，p2p网络的实现是继承自开源项目`graphene`，这个模块值得将来再开一个专题来讲。

当其他节点通过p2p网络收到一个`transaction`后⑪，会触发本地节点`graphene::net::node`的回调函数[`node_impl::on_message`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/net/node.cpp#L1783-L1859)，（在上图中绿色线连接的逻辑都是在p2p网络收到消息后的调用逻辑），在这里，其会根据消息的类型再调用不同的回调函数，当消息是`transaction`时，其调用[`node_impl::process_ordinary_message`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/net/node.cpp#L3943-L4002),然后在触发`p2p::p2p_plugin`中的回调函数[`impl::handle_transaction`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/p2p/p2p_plugin.cpp#L241-L261)⑫。然后再调用插件[`chain_plugin`](https://github.com/steemit/steem/tree/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/chain)中的[`accept_transaction`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/chain/chain_plugin.cpp#L564-L578)方法⑭。此后运行的逻辑跟③后的都一样的。

此时，用户发送给`steemd`的一个`transaction`，在本地节点和其他节点上都会进行一般相同的逻辑处理。我们将这些变更称为**临时**数据。

# 打包出块

`steemd`使用[`DPOS`](https://steemit.com/dpos/@legendx/dpos)共识机制，简单来说，只有“超级节点”(witness)才有权收集传播中的`transaction`，对其进行打包，变成一个`block`，然后再把这个`block`传播出去。

当本节点为“超级节点”(witness)时，可以启动`witness`插件，在配置`config.ini`的`plugin`项加入`witness`，例如`plugin = witness`，同时配置好文件中的`witness`项和与`witness`项对应的`private-key`。

在`witness`插件启动后，会启动一个定时器，定时触发调度逻辑[`maybe_produce_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/witness/witness_plugin.cpp#L541-L615)⑮，该函数会执行`witness`的调度逻辑（该处可以开个模块另行讲解），如果不该本节点轮次时，就返回等待下一次调度，如果轮到本节点的轮次，则会调用[`chain::database`](https://github.com/steemit/steem/tree/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain)的[`generate_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L800-L817)⑯，该打包的过程中：

1. 回滚前面传播`transaction`时的**临时**数据变更，因为这些数据可能是[不可靠的](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L843-L854)。因为只有被变成`block`并传播到各个节点上的数据才是大家公认可靠的数据。
2. 然后将本节点收集到的被广播的`transaction`中包含的`operation`再进行一次校验，该处使用的校验方式是将这些`operation`在本地**应用**一次⑰（逻辑跟④中的那些步骤是一样的，可以看上面的描述），这种校验方式时非常重的一个操作，**应用**完后，还需要再**回滚**一次数据，这样极大的消耗的性能，极大地限制了该系统的TPS。
3. 将本地收集的`transaction`，放入一个`block`中，然后更新该`block`的元数据，比如序号、前一个`block`的id等等，建立该`block`跟以前`block`的关系。
4. 对新生成的`block`进行签名。
5. 调用[`push_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L610-L640)⑱，将新生成的`block`**应用㉒**到本地`database`，这个**应用㉒**操作会涉及到很多逻辑，留在后面再讲。
6. 调用`p2p`插件的[`broadcast_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/p2p/p2p_plugin.cpp#L744-L748)⑲方法，将这个签名后的`block`广播出去。

# 块数据的传播

当新生成的`block`被广播出去后，收到该信息的节点会触发其会触发本地节点`graphene::net::node`的回调函数[`node_impl::on_message`]⑪，然后根据消息内`block`的标识，再触发回调函数[`node_impl::process_block_message`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/net/node.cpp#L3602-L3682)，其进行一些简单的处理。然后触发`p2p`插件的[`p2p_plugin_impl::handle_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/p2p/p2p_plugin.cpp#L172-L239)⑬方法。该方法很简单，直接去调用插件[`chain_plugin`](https://github.com/steemit/steem/tree/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/chain)中的[`accept_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/plugins/chain/chain_plugin.cpp#L538-L562)方法⑳。在`accept_block`中将这个新`block`放入写入队列中㉑，跟③处是同一个队列。然后该队列的消费者，从队列中取出该`block`数据，然后调用[`push_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L610-L640)将`block`中的数据**应用㉒**到本地`database`，此处与⑱处相同，即块生成的节点，和接收块的节点都运行相同的**应用㉒**逻辑。

根据状态机原理，初始状态的相同的两份数据，经过相同的状态迁移，最终的状态也相同，这样就保证了各个`steemd`节点的数据最终一致性。

# 区块链的生长

在每个`steemd`节点收到新`block`后㉒，会首先判断是否发生了分叉，如果发生了分叉，则进行分叉修复㉓。

## 分叉处理
`steemd`采用的是长链原则，即发生分叉时，无条件信任长链的数据。如果事先本节点**应用**短链的数据，则会回滚掉短链涉及的全部数据变更，然后再重新**应用**长链上的数据修改。当一条链被`75%`的的“超级节点”(witness)**应用**后，该条链才会变成不可变更的，在此之前本地`steemd`节点上的区块链数据可能会发生多次回滚、变更。至于一条链需要多久才能被`75%`的的“超级节点”(witness)**应用**变得稳定，这个得视各种情况而定，如果“超级节点”(witness)的数量且相互之间延时很低，则发生分叉的概况就低，则达成`75%`共识则更快，如果从中有人破坏，故意分叉或者网络情况差，无意造成分叉，则达成共识更慢。

目前`steemd`能够处理1024个块内的分叉，即假设的分叉时长不长于`1024*3s`(51分钟)。

`steemd`本地采用了`fork_db`来缓存每个收到的`block`，如果新`block`与本地的链的最后一个`block`相连，则直接使本地的链生长，**应用**该`block`中的数据。如果新`block`与本地的链的最后一个`block`不相连，则先将其缓存在`fork_db`中，然后比较哪条链长，如果`fork_db`缓存中的链比本地链长时，则回滚本地链上的`block`，使得`database`恢复到长短2条链共同的分叉点`block`处，然后将长链的后续`block`**应用**到本地`database`。这段逻辑在[`这里`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L669-L727)。

## 生长
当处理完分叉情况后，则会调用[`apply_block`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L2729-L2798)㉔方法，将`block`中的数据**应用**到本地`database`。

此时先验证该`block`的签名、merkle根等信息。感觉校验的逻辑应该移动到分叉处理之前，这样能极大的减少伪造块带来的分叉，从而避免了被攻击的可能，因为分叉回滚会极大的消耗`database`仅有的处理能力。

当该`block`被校验完成后，依次调用[`apply_transaction`](https://github.com/steemit/steem/blob/807fb40ec137a987dc53cee6d8455c7b6c47aeed/libraries/chain/database.cpp#L3128-L3131)方法，把`block`中的`transaction`**应用**到本地`database`，此处的**应用**逻辑与④处相同，就不再赘述了。

当`block`中的`transaction`**应用**完成后，更新本地节点的全局元数据（当前block号、时间、witness状态等），再然后处理一些收尾工作，比如清理过期的`transaction`、`orders`等，奖励对应`block`的witness（此处与`steemd`的激励机制有关）。

# 总结

至此，就完成了一个用户更新操作的全部流程：从用户的一个变更操作，变成一个`transaction`，再变成一个`block`，最后同步到`database`的变更。

从中可见，区块链并没有什么技术上的创新，使用的都是现成技术方案：
* 数字签名、hash
* 批处理，多个`transaction`打包成一个`block`
* `p2p`网络
* [`RSM`](https://lrita.github.io/2018/03/27/blockchain-and-rsm/)，从`transaction`的收集、打包、出块，块的传播，链的生长，就是`RSM`的逻辑。