---
layout: post
title: 修改docker -v 挂载的文件遇到的问题
categories: [docker]
description: docker
keywords: docker
---

在启动`docker`容器时，为了保证一些基础配置与宿主机保持同步，通常需要将这些配置文件挂载进
`docker`容器，例如`/etc/resolv.conf`/`/etc/hosts`/`/etc/localtime`等。

当这些配置变化时，我们通常会修改这些文件。但是此时遇到了一个问题：

当在宿主机上修改这些文件后，`docker`容器内查看时，这些文件并未发生对应的修改。

然后通过查阅相关资料，发现该问题是由`docker -v`挂载文件和某些编辑器存储文件的行为共同导致
的。

* docker 挂载文件时，并不是挂载了某个文件的路径，而是实打实的挂载了对应的文件，即挂载了某
个指定的`inode`文件。
* 某些编辑器(vi)在编辑保存文件时，采用了`备份、替换`的策略，即编辑过程中，将变更写入新文件，
保存时，再将备份文件替换原文件，此时会导致文件的`inode`发生变化。
* 原`inode`对应的文件其实并没有发生修改。

因此，我们从宿主机上修改这些文件时，应该采用`echo`重定向等操作，避免文件的`inode`发生变化。

# 参考

* [File mount does not update with changes from host](https://github.com/moby/moby/issues/15793)
