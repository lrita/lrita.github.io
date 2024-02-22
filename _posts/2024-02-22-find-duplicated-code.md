---
layout: post
title: 发现项目中的重复代码
categories: [tool]
description: tool
keywords: duplicated code copy paste detector
---

## 发现项目中的重复代码

当代码项目变大、参与的人增加时，就容易出现代码频繁复制，复用率降低的情况，倾向保守是人之常情。当问题积累到一定程度时，还是需要优化解决一下的。为了解决项目中的重复代码问题，我调研了多个工具，最终发现有两个工具还不错，简单易行，容易继承到CI中，特此介绍一下：

## simian

[simian](https://simian.quandarypeak.com/) 是一个专门用来检测重复代码的静态分析工具，基本上支持常见的各种编程语言，其使用起来也是非常简单。

第一步在[下载页面](https://simian.quandarypeak.com/download/)下载工具包，备份[simian-4.0.0.jar](/images/posts/tools/simian-academic.zip)，其依赖JDK17以上的Java环境，需要提前安装。

第二步解压压缩包，其中只有一个jar包。

第三步执行检测命令：

```sh
# 参数 :
# -threshold=10 设置检测重复代码行数阈值，超过指定行数时会被检测
# -defaultLanguage=LANG 可选值：java, c#, cs, csharp, c, c++, cpp, cplusplus, js, \
#                              javascript, cobol, abap, rb, ruby, vb, jsp, html, xml, \
#                              groovy, asm390，检测不出源码对应语言时，假设为该种语言类型
# -language=LANG 假设全部文件都为该种语言类型
# -reportDuplicateText 显示重复的代码片段
java -jar simian-4.0.0.jar -language=cpp -reportDuplicateText -threshold=10 ../../folly-v2023.03.13.00/**/*.h ../../folly-v2023.03.13.00/**/*.cpp
```

其会输出以下内容，可以从中看到重复的文件行数范围，以及重复的内容，可以选择合适的`-threshold`参数来进行扫描：
```
=====================================================================
Found 15 duplicate lines with fingerprint dfa3c0ad5fd93c0e3154c813486d85fa in the following files:
 Between lines 323 and 337 in /data1/neal/y/folly-v2023.03.13.00/folly/AtomicHashMap-inl.h
 Between lines 363 and 377 in /data1/neal/y/folly-v2023.03.13.00/folly/AtomicHashMap-inl.h
template <
    typename KeyT,
    typename ValueT,
    typename HashFcn,
    typename EqualFcn,
    typename Allocator,
    typename ProbeFcn,
    typename KeyConvertFcn>
typename AtomicHashMap<
    KeyT,
    ValueT,
    HashFcn,
    EqualFcn,
    Allocator,
    ProbeFcn,
=====================================================================
```

其比较明显的缺点是，文件路径只能支持到固定格式层级的目录，比如上面"**/*.h"这种，当源码文件夹较多，深度各自不同时，传递文件路径比较费事。

## pmd

[pmd](https://pmd.github.io/pmd/index.html) 是一个开源静态分析工具，支持常见的16中变成语言，其除了检测重复代码外还有其他功能，此处只介绍其检测重复代码的CPD功能。

第一步，在[github](https://github.com/pmd/pmd/releases)上下载最新发布二进制包`pmd-bin-xxx.zip`。

第二步解压压缩包，其中是一个完成的目录结构，不要破坏其原本的目录结构。

第三步执行检测命令：
```sh
# --minimum-tokens 200 判断重复的最小token数，跟simian的-threshold比较类似，单位不是行，而是词组
# --dir <path> / -d <path> 指定源码目录，比simian就灵活许多
# --file-list <filepath>
# --skip-duplicate-files 跳过重复的文件，这种在c的大规模项目比较常见
# --skip-lexical-errors 这个比较需要，不要因为解析失败而中断检查
./bin/pmd cpd --language cpp --minimum-tokens 200 --skip-lexical-errors --dir ../../folly-v2023.03.13.00
```

其会输出以下内容，可以从中看到重复的文件行数范围，以及重复的内容，可以选择合适的`--minimum-tokens`参数来进行扫描：
```
=====================================================================
Found 16 duplicate lines with fingerprint 916b4dfabbca319a79ecd278b6556e52 in the following files:
 Between lines 273 and 288 in /data1/neal/y/folly-v2023.03.13.00/folly/AtomicHashMap-inl.h
 Between lines 243 and 258 in /data1/neal/y/folly-v2023.03.13.00/folly/AtomicHashMap-inl.h
 Between lines 210 and 225 in /data1/neal/y/folly-v2023.03.13.00/folly/AtomicHashMap-inl.h
template <
    typename KeyT,
    typename ValueT,
    typename HashFcn,
    typename EqualFcn,
    typename Allocator,
    typename ProbeFcn,
    typename KeyConvertFcn>
template <class LookupKeyT, class LookupHashFcn, class LookupEqualFcn>
typename AtomicHashMap<
    KeyT,
    ValueT,
    HashFcn,
    EqualFcn,
    Allocator,
    ProbeFcn,
=====================================================================
```

其输出的格式与`simian`比较类似，两个工具，从使用体感上，`pmd`略胜一筹，扫描发现的问题比较多一丢丢，比较推荐。