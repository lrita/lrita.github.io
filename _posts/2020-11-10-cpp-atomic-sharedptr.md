---
layout: post
title: C++ 原子智能指针
categories: [c++]
description: c++
keywords: c++ atomic shared_ptr
---

`std::atomic<std::shared_ptr>`是 C++20 进入到标准中的一个提案。但是目前看来实现的方案主要有：

- [libstdc++/std::atomic<std::shared_prt<T>>](https://github.com/gcc-mirror/gcc/blob/41d6b10e96a1de98e90a7c0378437c3255814b16/libstdc%2B%2B-v3/include/bits/shared_ptr_atomic.h#L137) 内部使用`mutex`来保护`std::shared_prt<T>`的更新。
- [anthonyw/atomic_shared_ptr](https://github.com/lrita/atomic_shared_ptr) 内部使用了一个`std::atomic<struct counted_ptr>`，其中`struct counted_ptr`是一个 96bit 的`POD`，其中记录了原始指针和指针的版本号，相当于使用`atomic128`来实现。
- [folly/atomic_shared_ptr](https://github.com/facebook/folly/blob/0deef031cb8aab76dc7e736f8b7c22d701d5f36b/folly/concurrency/AtomicSharedPtr.h) 采用了 linux 在 64bit 系统上，指针只使用了 48bit 的特性，将元数据藏在了指针的剩余 16bit 中。
