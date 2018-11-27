---
layout: post
title: 安装最新版 devtoolset-8
categories: [linux, tool]
description: linux, tool, devtoolset
keywords: linux, tool, devtoolset
---

```
工具就是生产力
```

为了更好的debug，在测试环境中安装了比较新的linux kernel 4.16，但是安装完毕后，很多常用的debug工具都不能很好的适配最新版的内核。没办法，又得研究如何更新这些工具们。

转了一圈，发现从源码安装的难度不低，故放弃了。随后转向了`devtoolset`

> For certain applications, more recent versions of some software components are often needed in order to use their latest new features. Red Hat Software Collections is a Red Hat offering that provides a set of dynamic programming languages, database servers, and various related packages that are either more recent than their equivalent versions included in the base Red Hat Enterprise Linux system, or are available for this system for the first time.

简单说，RedHat 推出`Software Collections`的目的就是为了解决想在`RedHat`系统下能使用新版本的工具，让同一个工具（如gcc）的不同版本能在系统中共存，在需要的时候切换到对应的版本中，类似 pyenv(python)、rvm(ruby) 或者 nvm(node)。`RedHat`与`CentOS`师出同源，一样可以使用。

如何安装和使用`devtoolset`，可以参考[官方指导文档](https://www.softwarecollections.org/en/scls/rhscl/devtoolset-7/)。在看装前，我们可以先到[Information for build](https://cbs.centos.org/koji/buildinfo?buildID=23609)搜索自己需要的package对应的源码版本，看`devtoolset`中的安装包对应的源码版本是多少，是否满足自身的需求。

比如linux kernel 4.16对应的systemtap的源码版本要求systemtap-3.3以上。在`devtoolset-7`的安装包中[devtoolset-7-systemtap-3.1-4s.el7](https://cbs.centos.org/koji/buildinfo?buildID=22765)对应的源码版本只有systemtap-3.1。所以只能寻求更高版本的支持。因此在该网站上找到了[devtoolset-8-systemtap-3.3-1.el6](https://cbs.centos.org/koji/buildinfo?buildID=23609)。但是`devtoolset-8`还没有正式发行，无法从yum上进行安装。其具体的安装方法是：

* 到[`sclo7-devtoolset-8-rh-candidate`](https://cbs.centos.org/repos/sclo7-devtoolset-8-rh-candidate/x86_64/os/Packages/)现在全部RPM包(这里偷懒下载全部包，这样就不用处理他们之间复杂的相互依赖关系)
* 下载完成后，在RPM包的目录中执行`yum install *.rpm`，安装全部
* 然后执行`scl enable devtoolset-8 bash`切换环境。

```sh
> stap -V # 我们可以执行这个命令进行验证，我们得到了Systemtap-3.3，支持linux kernel 2.6.28-4.18
Systemtap translator/driver (version 3.3/0.173, rpm 3.3-1.el7)
Copyright (C) 2005-2018 Red Hat, Inc. and others
This is free software; see the source for copying conditions.
tested kernel versions: 2.6.18 ... 4.18-rc0
enabled features: AVAHI BOOST_STRING_REF DYNINST LIBRPM LIBSQLITE3 NLS NSS READLINE
```