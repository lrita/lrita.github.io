---
layout: post
title: 优化变种CRC算法
categories: [algorithm]
description: crc64 redis algorithm
keywords: crc64 redis algorithm
---

# CRC算法

[循环冗余校验-CRC](https://zh.wikipedia.org/wiki/循環冗餘校驗)是常用的一种校验数据一致性的算法。关于其原理与发展可以参考其维基百科。

关于该算法的优化也经常是大家关心的一个方向，现在最常用的实现基本都是查表法，下图是别人总结的相关算法的优化历史[^1]:

![](/images/posts/tools/fast-crc32.png)

可以看出，当因特尔提出了`slice-by-x`的优化后，其速度得到的飞跃。

然而CRC算法的变种数量非常之多，但是已经被实现的那些优化只是少数的几种，比如`ISO`、`ECMA`等。同时这些优化后的代码已经丢失的大量中间过程，是的其难以被移植到其他变种之上。

# 优化变种算法

经过一番研究考证，CRC64变种采用`slice-by-x`优化的主要步骤为：

* 将原始查询表转化为8重表格
```
// table[0] 是原始查询表， table[1...7] 是生成的8重表格
for (n = 0; n < 256; n++) {
    crc = table[0][n];

    for (k = 1; k < 8; k++) {
        crc = table[0][crc & 0xff] ^ (crc >> 8);
        table[k][n] = crc;
    }
}
```
* 优化CRC算法，每个循环计算8个字节
```
while (len >= 8) {
    crc ^= *(uint64_t *)byte; // 小端计算方式
    crc = table[7][crc & 0xff] ^
          table[6][(crc >> 8) & 0xff] ^
          table[5][(crc >> 16) & 0xff] ^
          table[4][(crc >> 24) & 0xff] ^
          table[3][(crc >> 32) & 0xff] ^
          table[2][(crc >> 40) & 0xff] ^
          table[1][(crc >> 48) & 0xff] ^
          table[0][crc >> 56];
    byte += 8;
    len -= 8;
}
while (len) {
    crc = table[0][(crc ^ *byte++) & 0xff] ^ (crc >> 8);
    len--;
```

# 实现

[github.com/lrita/crc64](https://github.com/lrita/crc64)采用上述方法，优化了`redis`使用的 CRC64 变种算法，使其速度从`381.29 MB/s`提升到`1474.16 MB/s`.

# 参考

[^1]: [Fast CRC32](https://create.stephan-brumme.com/crc32/)