---
layout: post
title: Shell脚本进阶
categories: [shell]
description: shell
keywords: shell
---

最近写了不少的 shell 脚本，也收集了一些提升 shell 编程的文章，感觉有一些确实不错的内容可以进行分享。

首先，[编写可靠 bash 脚本的一些技巧](https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649745834&idx=1&sn=a87c1abec3395327436cc812c8579188&chksm=bed37ad189a4f3c727a9133beaed00d98b78c9dbf7def2f1c55615183eb26eda0fc1be61e93d&mpshare=1&scene=23&srcid&sharer_sharetime=1586314041974&sharer_shareid=0d25aaa0141cb845ff5dc57c13b23352%23rd)中列出了几条通用的建议，非常值得借鉴。

其次，[Bash Pitfalls](http://mywiki.wooledge.org/BashPitfalls)中列出了诸多 shell 中容易踩坑的地方，需要多加注意这些细节。这里有其翻译版本：

- [Bash Pitfalls: 编程易犯的错误（一）](https://kodango.com/bash-pitfalls-part-1)
- [Bash Pitfalls: 编程易犯的错误（二）](https://kodango.com/bash-pitfalls-part-2)
- [Bash Pitfalls: 编程易犯的错误（三）](https://kodango.com/bash-pitfalls-part-3)
- [Bash Pitfalls: 编程易犯的错误（四）](https://kodango.com/bash-pitfalls-part-4)

再次，就是静态分析工具[shellcheck](https://github.com/koalaman/shellcheck)，可以发现一些 shell 脚本中场景的易犯错误，并且还会给出链接，详细说明错误的详情。
