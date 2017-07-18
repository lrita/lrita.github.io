---
layout: post
title: Intel(R) QuickAssist (QAT) 技术
categories: [linux, intel, crypto]
description: linux intel crypto
keywords: linux intel crypto
---

`Intel(R) QuickAssist (QAT)`是Intel最新CPU上提供的一项`offload`技术，它使用硬件加速帮助
应用程序提高一些通用算法的效率，可以通过访问其驱动与应用程序连同起来。同时`dpdk/spdk`也
有这方面的支持。

最近在看一些资料时，很多方面都提及了该技术的应用。在此先mark一下，一会可能会用得到，特别
是在加密通信/存储方面。

`QuickAssist`为用户提供算法有(基本上主流算法都有):

Cipher algorithms:

* `RTE_CRYPTO_CIPHER_3DES_CBC`
* `RTE_CRYPTO_CIPHER_3DES_CTR`
* `RTE_CRYPTO_CIPHER_AES128_CBC`
* `RTE_CRYPTO_CIPHER_AES192_CBC`
* `RTE_CRYPTO_CIPHER_AES256_CBC`
* `RTE_CRYPTO_CIPHER_AES128_CTR`
* `RTE_CRYPTO_CIPHER_AES192_CTR`
* `RTE_CRYPTO_CIPHER_AES256_CTR`
* `RTE_CRYPTO_CIPHER_SNOW3G_UEA2`
* `RTE_CRYPTO_CIPHER_NULL`
* `RTE_CRYPTO_CIPHER_KASUMI_F8`
* `RTE_CRYPTO_CIPHER_DES_CBC`
* `RTE_CRYPTO_CIPHER_AES_DOCSISBPI`
* `RTE_CRYPTO_CIPHER_DES_DOCSISBPI`
* `RTE_CRYPTO_CIPHER_ZUC_EEA3`

Hash algorithms:

* `RTE_CRYPTO_AUTH_SHA1_HMAC`
* `RTE_CRYPTO_AUTH_SHA224_HMAC`
* `RTE_CRYPTO_AUTH_SHA256_HMAC`
* `RTE_CRYPTO_AUTH_SHA384_HMAC`
* `RTE_CRYPTO_AUTH_SHA512_HMAC`
* `RTE_CRYPTO_AUTH_AES_XCBC_MAC`
* `RTE_CRYPTO_AUTH_SNOW3G_UIA2`
* `RTE_CRYPTO_AUTH_MD5_HMAC`
* `RTE_CRYPTO_AUTH_NULL`
* `RTE_CRYPTO_AUTH_KASUMI_F9`
* `RTE_CRYPTO_AUTH_AES_GMAC`
* `RTE_CRYPTO_AUTH_ZUC_EIA3`

参考文档:
* [Intel® QuickAssist Technology](https://01.org/zh/intel-quickassist-technology)
* [Intel(R) QuickAssist (QAT) Crypto Poll Mode Driver](http://dpdk.org/doc/guides/cryptodevs/qat.html)
* [DPDK-China2017-LiuZeng-Accelerate-VM-IO-via-SPDK.pdf](/images/blog/DPDK-China2017-LiuZeng-Accelerate-VM-IO-via-SPDK.pdf)
