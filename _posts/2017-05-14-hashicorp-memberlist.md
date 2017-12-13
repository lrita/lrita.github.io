---
layout: post
title: hashicorp memberlist 分析
categories: [distributed-programming]
description: distributed systems hashicorp memberlist
keywords: distributed-systems, memberlist
---

# hashicorp memberlist
[memberlist](https://github.com/hashicorp/memberlist)用来管理分布式集群内节点发现、
节点失效探测、节点列表的软件包。

`memberlist`采用[`gossip`](https://en.wikipedia.org/wiki/Gossip_protocol)协议，来传播
消息，该`gossip`构建在[`SWIM`](https://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf)之上。

`memberlist`将相关实现都封装在package内部，对外提供变更节点的相关操作。然后将协议相关的
参数都变成配置项`Config`供用户调优。由于`gossip`协议的收敛速度跟消耗的带宽成正比。因此
用户可以根据自己的期望来进行调节。

## protocol
定义：
* 集群内的广播：集群的内的广播通过udp协议向每个节点发送消息，不确保消息一定能发送到，广播先
进入广播队列，广播队列里的消息可能被其他更优先的消息覆盖掉。广播队列里的消息发送失败超过一定
次数后，就会被抛弃。发送次数参考`Config`里`RetransmitMult`的注释。
* Push/Pull：每隔一个随机浮动的间隔，会随机选取一个节点，跟它建立tcp连接，然后将本地的全部节点
状态、用户数据发送过去，然后对端将其掌握的全部节点状态、用户数据发送回来，然后完成2份数据的合并。
此动作可以加速集群内信息的收敛速度。
* 通讯协议：使用udp协议传输PING消息、间接PING消息、ACK消息、NACK消息、Suspect消息、
Alive消息、Dead消息、用户消息；采用tcp协议传输用户数据、PING消息和PUSH-PULL消息。

#### alive
当新节点启动时，会向集群内广播Alive消息。

#### probe
当节点启动后，每隔一定时间间隔，会选取一个节点对其发送PING消息，当PING消息失败后，会随机选取
`IndirectChecks`个节点发起间接PING的请求和直接更其再发起一个tcp PING消息。
收到间接PING请求的节点会根据请求中的地址发起一个PING消息，将PING的结果返回给间接请求的源节点。

如果探测超时之间内，本节点没有收到任何一个要探测节点的ACK消息，则标记要探测的节点状态为suspect。

#### suspect
当探测一些节点失败时，或者suspect某个节点的信息时，会将本地对应的信息标记为suspect，然后启动一个
定时器，并发出一个suspect广播，此期间内如果收到其他节点发来的相同的suspect信息时，将本地suspect的
确认数+1，当定时器超时后，该节点信息仍然不是alive的，且确认数达到要求，会将该节点标记为dead。

当本节点收到别的节点发来的suspect消息时，会发送alive广播，从而清除其他节点上的suspect标记。

#### dead
当本节点离开集群时或者本地探测的其他节点超时被标记死亡，会向集群发送本节点dead广播。收到dead广播
消息的节点会跟本地的记录比较，当本地记录也是dead时会忽略消息，当本地的记录不是dead时，会删除本地
的记录再将dead消息再次广播出去，形成再次传播。

如果从其他节点收到自身的dead广播消息时，说明本节点相对于其他节点网络分区，此时会发起一个alive广播
以修正其他节点上存储的本节点数据。

#### 广播风暴
为了避免循环广播造成广播风暴，每个节点用一个uint32 `Incarnation`来标识每次广播的序号，当收到小于
本地记录的该节点Incarnation的广播消息时，忽略之。`Incarnation`只有再每一次节点重新alive时，会增长
一次。

## Config
该框架内部有很多实现细节是需要用户自己进行控制的，因此`memberlist`将这些都在`Config`中让用户配置。
同时`Config`还有许多回调接口，用户可以实现这些回调接口以接受通知。

其中`Name`和`AdvertiseAddr`/`AdvertisePort`进行绑定的，对应关系不能变更，否则会造成内部逻辑的混乱。
```go
type Config struct {
    // 节点名称，应该在集群内是唯一的
    Name string

    // 通信层面的抽象，下面有介绍，如果用户不配置，使用内部默认的
    Transport Transport

    // 本地监听的IP和端口，端口同时使用udp和tcp
    BindAddr string
    BindPort int

    // Configuration related to what address to advertise to other
    // cluster members. Used for nat traversal.
    AdvertiseAddr string
    AdvertisePort int

    // ProtocolVersion is the configured protocol version that we
    // will _speak_. This must be between ProtocolVersionMin and
    // ProtocolVersionMax.
    ProtocolVersion uint8

    // TCPTimeout is the timeout for establishing a stream connection with
    // a remote node for a full state sync, and for stream read and write
    // operations. This is a legacy name for backwards compatibility, but
    // should really be called StreamTimeout now that we have generalized
    // the transport.
    TCPTimeout time.Duration

    // IndirectChecks is the number of nodes that will be asked to perform
    // an indirect probe of a node in the case a direct probe fails. Memberlist
    // waits for an ack from any single indirect node, so increasing this
    // number will increase the likelihood that an indirect probe will succeed
    // at the expense of bandwidth.
    IndirectChecks int

    // RetransmitMult is the multiplier for the number of retransmissions
    // that are attempted for messages broadcasted over gossip. The actual
    // count of retransmissions is calculated using the formula:
    //
    //   Retransmits = RetransmitMult * log(N+1)
    //
    // This allows the retransmits to scale properly with cluster size. The
    // higher the multiplier, the more likely a failed broadcast is to converge
    // at the expense of increased bandwidth.
    RetransmitMult int

    // SuspicionMult is the multiplier for determining the time an
    // inaccessible node is considered suspect before declaring it dead.
    // The actual timeout is calculated using the formula:
    //
    //   SuspicionTimeout = SuspicionMult * log(N+1) * ProbeInterval
    //
    // This allows the timeout to scale properly with expected propagation
    // delay with a larger cluster size. The higher the multiplier, the longer
    // an inaccessible node is considered part of the cluster before declaring
    // it dead, giving that suspect node more time to refute if it is indeed
    // still alive.
    SuspicionMult int

    // SuspicionMaxTimeoutMult is the multiplier applied to the
    // SuspicionTimeout used as an upper bound on detection time. This max
    // timeout is calculated using the formula:
    //
    // SuspicionMaxTimeout = SuspicionMaxTimeoutMult * SuspicionTimeout
    //
    // If everything is working properly, confirmations from other nodes will
    // accelerate suspicion timers in a manner which will cause the timeout
    // to reach the base SuspicionTimeout before that elapses, so this value
    // will typically only come into play if a node is experiencing issues
    // communicating with other nodes. It should be set to a something fairly
    // large so that a node having problems will have a lot of chances to
    // recover before falsely declaring other nodes as failed, but short
    // enough for a legitimately isolated node to still make progress marking
    // nodes failed in a reasonable amount of time.
    SuspicionMaxTimeoutMult int

    // PushPullInterval is the interval between complete state syncs.
    // Complete state syncs are done with a single node over TCP and are
    // quite expensive relative to standard gossiped messages. Setting this
    // to zero will disable state push/pull syncs completely.
    //
    // Setting this interval lower (more frequent) will increase convergence
    // speeds across larger clusters at the expense of increased bandwidth
    // usage.
    PushPullInterval time.Duration

    // ProbeInterval and ProbeTimeout are used to configure probing
    // behavior for memberlist.
    //
    // ProbeInterval is the interval between random node probes. Setting
    // this lower (more frequent) will cause the memberlist cluster to detect
    // failed nodes more quickly at the expense of increased bandwidth usage.
    //
    // ProbeTimeout is the timeout to wait for an ack from a probed node
    // before assuming it is unhealthy. This should be set to 99-percentile
    // of RTT (round-trip time) on your network.
    ProbeInterval time.Duration
    ProbeTimeout  time.Duration

    // DisableTcpPings will turn off the fallback TCP pings that are attempted
    // if the direct UDP ping fails. These get pipelined along with the
    // indirect UDP pings.
    DisableTcpPings bool

    // AwarenessMaxMultiplier will increase the probe interval if the node
    // becomes aware that it might be degraded and not meeting the soft real
    // time requirements to reliably probe other nodes.
    AwarenessMaxMultiplier int

    // GossipInterval and GossipNodes are used to configure the gossip
    // behavior of memberlist.
    //
    // GossipInterval is the interval between sending messages that need
    // to be gossiped that haven't been able to piggyback on probing messages.
    // If this is set to zero, non-piggyback gossip is disabled. By lowering
    // this value (more frequent) gossip messages are propagated across
    // the cluster more quickly at the expense of increased bandwidth.
    //
    // GossipNodes is the number of random nodes to send gossip messages to
    // per GossipInterval. Increasing this number causes the gossip messages
    // to propagate across the cluster more quickly at the expense of
    // increased bandwidth.
    //
    // GossipToTheDeadTime is the interval after which a node has died that
    // we will still try to gossip to it. This gives it a chance to refute.
    GossipInterval      time.Duration
    GossipNodes         int
    GossipToTheDeadTime time.Duration

    // EnableCompression is used to control message compression. This can
    // be used to reduce bandwidth usage at the cost of slightly more CPU
    // utilization. This is only available starting at protocol version 1.
    EnableCompression bool

    // SecretKey is used to initialize the primary encryption key in a keyring.
    // The primary encryption key is the only key used to encrypt messages and
    // the first key used while attempting to decrypt messages. Providing a
    // value for this primary key will enable message-level encryption and
    // verification, and automatically install the key onto the keyring.
    // The value should be either 16, 24, or 32 bytes to select AES-128,
    // AES-192, or AES-256.
    SecretKey []byte

    // The keyring holds all of the encryption keys used internally. It is
    // automatically initialized using the SecretKey and SecretKeys values.
    Keyring *Keyring

    // Delegate and Events are delegates for receiving and providing
    // data to memberlist via callback mechanisms. For Delegate, see
    // the Delegate interface. For Events, see the EventDelegate interface.
    //
    // The DelegateProtocolMin/Max are used to guarantee protocol-compatibility
    // for any custom messages that the delegate might do (broadcasts,
    // local/remote state, etc.). If you don't set these, then the protocol
    // versions will just be zero, and version compliance won't be done.
    Delegate                Delegate
    DelegateProtocolVersion uint8
    DelegateProtocolMin     uint8
    DelegateProtocolMax     uint8
    // 当集群内有节点加入、离开、元数据变更都会触发EventDelegate回调
    Events                  EventDelegate
    // 当`Name`和`AdvertiseAddr`/`AdvertisePort`不对应时，会触发ConflictDelegate
    // 回调
    Conflict                ConflictDelegate
    // 当某节点Join该集群时，集群内的每个节点会触发MergeDelegate的接口一次，用于
    // 接受新节点的数据。该接口可以是nil。
    Merge                   MergeDelegate
    // 当探测其他节点时，会触发相关ping接口
    Ping                    PingDelegate
    Alive                   AliveDelegate

    // DNSConfigPath points to the system's DNS config file, usually located
    // at /etc/resolv.conf. It can be overridden via config for easier testing.
    DNSConfigPath string

    // LogOutput is the writer where logs should be sent. If this is not
    // set, logging will go to stderr by default. You cannot specify both LogOutput
    // and Logger at the same time.
    LogOutput io.Writer

    // Logger is a custom logger which you provide. If Logger is set, it will use
    // this for the internal logger. If Logger is not set, it will fall back to the
    // behavior for using LogOutput. You cannot specify both LogOutput and Logger
    // at the same time.
    Logger *log.Logger

    // Size of Memberlist's internal channel which handles UDP messages. The
    // size of this determines the size of the queue which Memberlist will keep
    // while UDP messages are handled.
    HandoffQueueDepth int

    // Maximum number of bytes that memberlist will put in a packet (this
    // will be for UDP packets by default with a NetTransport). A safe value
    // for this is typically 1400 bytes (which is the default). However,
    // depending on your network's MTU (Maximum Transmission Unit) you may
    // be able to increase this to get more content into each gossip packet.
    // This is a legacy name for backward compatibility but should really be
    // called PacketBufferSize now that we have generalized the transport.
    UDPBufferSize int
}
```

#### Delegate
该接口提供一些用户注册的回调函数，当有指定的事件发生时，会触发该回调函数。
```go
type Delegate interface {
    // 当节点上线或更新时，框架层通过该接口获取本节点的元数据，limit为元数据上限
    NodeMeta(limit int) []byte

    // 当节点收到一个其他节点发送的user-data的数据，会触发该回调函数，该函数不能时阻塞调用，
    // 否则会阻塞内部逻辑环节。其他节点调用SendTo/SendBestEffort/SendReliable发送的和Delegate的
    // GetBroadcasts返回的数据的数据会触发该回调
    NotifyMsg([]byte)

    // 当内部状态可以广播数据时，会调用该接口，返回的数据会广播给每个节点，
    // 触发节点的Delegate的NotifyMsg回调。
    // overhead是buffer内已经被占用的字节数，limit是buffer大小，limit-overhead
    // 是用户可以发送的数据大小
    GetBroadcasts(overhead, limit int) [][]byte

    // Push/Pull通过该接口获取用户数据，随着Push/Pull请求进行交换, 当调用memberlist.Join
    // 加入到某一个存在的集群是，会向指定列表内的全部节点发起一次Push/Pull请求。因此，使用
    // join=true来标识是是Jion操作，join=false为定期的Push/Pull请求。
    // 该接口返回的数据发送到对端时，会触发对端的MergeRemoteState接口。
    LocalState(join bool) []byte
    MergeRemoteState(buf []byte, join bool)
}
```
#### Transport
`memberlist`的内部实现包含了全部的通信过程，因此需要一个`Transport`实现来底层的通信api。
`memberlist`内部提供了一个默认实现，当用户在`Config`中不提供`Transport`实现时，就采用默
认实现。

`Transport`实现应该具有以下功能.
```go
type Transport interface {
    // 传入Config中的AdvertiseAddr和AdvertisePort，返回给上层一个用于告知
    // 集群内其他成员的本节点ip和端口
    FinalAdvertiseAddr(ip string, port int) (net.IP, int, error)

    // 向指定addr发送一段数据，返回完成通信的当前时刻，该时刻可以用来判断RTT
    WriteTo(b []byte, addr string) (time.Time, error)

    // PacketCh 返回一个chan，当每从其他成员收到一个udp报文时，将报文封装成
    // *Packet写入该chan，上层逻辑通过该chan读取udp信息。
    PacketCh() <-chan *Packet

    // 创建一个TCP连接给上层使用
    DialTimeout(addr string, timeout time.Duration) (net.Conn, error)

    // 返回一个chan，每当本地tcp端口收到一个连接，通过该chan交由上层处理.
    StreamCh() <-chan net.Conn

    // 退出时，调用该函数，释放相关通信资源。
    Shutdown() error
}
```

