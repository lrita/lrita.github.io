---
layout: post
title: minio/dsync 分布式锁
categories: [distributed-programming]
description: distributed system
keywords: distributed system
---

[minio/dsync](https://github.com/minio/dsync) 是一个 go 语言实现的，分布式锁工具库，使用在[Minio Object Storage](https://min.io)，其设计宗旨是追求简单，因此横向扩展能力比较局限，通常小于 32 个分布式节点。任一节点回向全部节点广播锁请求消息，然后获得`n/2 + 1`赞同的节点会成功获取到锁。释放锁时，还会向全部节点广播请求。

# 设计目标

- 简单设计：容易实现与控制
- 没有主节点：每个节点相互对等，没有主节点概念，因此在宕机容错的过程中可以更简单。
- 弹性容错：允许有`n/2 - 1`个节点宕机
- 自动化重组：宕机节点可以随时重新加入

# 性能

作为一个分布式锁，其性能也至关重要，因为其很可能是一个高频操作。官方给出的数据是：在 16 个节点的集群内，可以达到 `7500 locks/sec`。更详细的信息可以参考[官方文档](https://github.com/minio/dsync#performance)

# 缺陷

## 动态配置

`dsync`的实现并不能想以往的`raft`、`gossip`那样动态添加、删除节点、更新节点信息。如果需要变更集群的配置，需要修改、重启集群内全部节点才能生效。

## `stale lock`

`stale lock`指，持有锁的实例已经宕机，或者由于网络故障造成锁释放的消息无法被送达。在分布式系统中，`stale lock`不是那么容易被检测到的，其会大大影响整个系统的效率。因此`dsync`中加入了`stale lock`[检测机制](https://github.com/minio/dsync/pull/22#issue-176751755)：其首先假设每个节点本地不存在网络故障，因此每个节点本地记录着正确的自身节点的锁持有情况，然后其他节点，周期性调用锁拥有者节点的检验接口，查询对应的锁是否过期。

## 故障恢复问题

还有一个潜在的问题，虽然一个节点需要得到`n/2 + 1`个节点的同意才能获得锁，但是在这期间，有些节点可能会宕机重启，它们可能会在宕机前、宕机后分别同意不同节点的请求，造成它们都获得了`n/2 + 1`个同意回复，造成它们同时获得排他锁。

# 实现

其设计是一个分布式锁客户端框架。首先需要用户实现服务端的逻辑，比如如何验证锁存在、锁获取成功/失败、认证等，API 用户自行设计，客户端也需要用户实现与服务端对应的调用逻辑，适配要求的接口[NetLocker](https://github.com/minio/dsync/blob/fedfb5c974fa2ab238e45a6e6b19d38774e0326f/rpc-client-interface.go#L39-L70)。锁内部的具体实现都由用户自行设计实现，用户可以根据自己的需求来进行实现，比如基于`lease`的锁等。在项目中，也给出了参考实现[dsync/chaos](https://github.com/minio/dsync/tree/fedfb5c974fa2ab238e45a6e6b19d38774e0326f/chaos)

由于没有动态成员本更的设计，其实现就非常的简单，基本上就是一个`FOR-EACH`框架：

## 获取锁

- 向全部节点广播获取锁的请求消息
- 在超时时间内收集每个节点的回复信息
- 如果获得`n/2 + 1`个节点的赞同，则获得锁
- 否则，广播释放消息，然后在等待一个随机的延时后，再次尝试

## 释放锁

- 向全部节点广播释放锁的消息
- 如果某一节点通信失败，再次尝试

# 评价

真的是非常简单的一个实现，也比较粗糙，对于要求不太高的场景可以试试。正确性有一定缺陷，无法避免锁丢失的场景。满足基本可用的要求。
