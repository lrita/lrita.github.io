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

# 块数据的传播