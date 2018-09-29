---
layout: post
title: 使用SystemTap给程序注入系统调用setsockopt
categories: [linux, tool, SystemTap]
description: linux SystemTap
keywords: linux, SystemTap
---

最近有个项目开始使用弹性部署，简单来说就是根据业务流量来自动扩容/缩容服务实例数量，但是在运行一段时间后发现该服务依赖的另一个服务的连接数增长的很高，消耗了很大系统资源，导致每隔几天就需要重启一下。

根据分析调查，发现被依赖服务的连接数的增高是因为弹性部署的服务在缩容时，直接删除了VM，而不是先停止进程，导致其并没有关闭连接，同时在被依赖的服务上，也没有也没有开启TCP的`KeepAlive`机制，对端的VM都被删除了，但是之前建立的连接仍然存留。

那原因分析清楚了，这个问题就很好解决了，在修改源码，在建立连接后通过`setsockopt`开启`SO_KEEPALIVE`。但是难题是该服务的源码已经不见了，只存留了发布的二进制文件。

细想之后，只能通过`SystemTap`，给进程注入一个`setsockopt`调用，使其开启`SO_KEEPALIVE`。

因此我们选择在`accept`调用返回的实际注入这个调用，脚本源码为：

```c
%{
#include <net/sock.h>
%}

function set_sock_keepalive:long(fd) %{
  int err = -1;
  int keepalive = 1;
  struct socket *sock = sockfd_lookup(STAP_ARG_fd, &err);
  if (sock != NULL) {
    /*
     * sock_setsockopt 的参数在内核中声明为来自用户空间，
     * 因此其内部会对该值的来源进行校验，该脚本注入的这段C
     * 代码运行在内核空间，因此我们需要临时跳过这层校验。
     * 下面三行就是跳过的方法。
     */
    mm_segment_t oldfs;
    oldfs = get_fs();
    set_fs(KERNEL_DS);
    err = sock_setsockopt(sock, SOL_SOCKET,
            SO_KEEPALIVE, (char __user*)&keepalive, sizeof(keepalive));
    set_fs(oldfs);
    sockfd_put(sock);
  }
  STAP_RETURN(err);
%}

probe begin {
  printf("inject begin... \n")
}

/*
 * 注入点选择accept系统调用返回时，accept的返回值就是新建连接的文件描述符
 * 当触发的进程pid是给定进程时，进行注入操作
 * 在生产环境中，可以删除ok之后的打印以性能
 */
probe syscall.accept.return, syscall.accept4.return {
  fd = $return
  if ((pid() == $1) && (fd != -1)) {
    ok = set_sock_keepalive(fd)
    if (ok)
      printf("set_sock_keepalive %d\n", ok)
  }
}

probe end {
  printf("inject end... \n")
}
```

执行的方式是，`$pid`为指定的进程pid：
```sh
> stap -g inject_keepalive.stp $pid
```