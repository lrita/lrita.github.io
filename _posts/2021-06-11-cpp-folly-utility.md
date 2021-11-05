---
layout: post
title: folly utility 简明摘要
categories: [c++]
description: c++
keywords: c++ utility
---

# 辅助类

## functional

```cpp
// 所在头文件：
#include <folly/functional/Partial.h>

template <typename F, typename... Args>
auto partial(F&& f, Args&&... args);

// 跟 std::bind() 相似，但是不需要使用placeholders来进行参数绑定。
// e.g.
//
// auto p = folly::partial(&Foo::method, foo_pointer);
// p();
//
// folly::partial(Foo, 1, 2)(3); // is equivalent to `Foo(1, 2, 3);`
```

## 编译期数学方法

```cpp
// 所在头文件：
#include <folly/ConstexprMath.h>

// 其中包含一些模板类，可以帮助在编译期推导 max、min、abs、pow、ceil、log2_ceil、log2等常用数学方法。
```

# 线程同步

## spin 帮助方法

```cpp
#include <folly/synchronization/detail/Spin.h>

// 非常简单易用的 spin 循环实现，需要用到的之后可以直接引用
folly::detail::spin_pause_until();
folly::detail::spin_yield_until();
```

## folly::MicroLock

```cpp
#include <folly/MicroLock.h>
// 或者
#include <folly/synchronization/SmallLocks.h>

// 通常从性能角度出发，应该使用 std::mutex（对锁竞争处理的更好）， 如果为了节省内存，可以使用 folly::MicroLock ，
// 其只使用 4B 空间。
// 其具有常规的lock()/try_lock()/unlock()方法。
```

## folly::MicroSpinLock、folly::PicoSpinLock、folly::RWSpinLock
```cpp
#include <folly/MicroSpinLock.h>
// 或者
#include <folly/synchronization/MicroSpinLock.h>

// folly::MicroSpinLock 是一个极简单的 spinlock 实现，采用最简单的while-cas模式。通常不应该被使用。

#include <folly/synchronization/PicoSpinLock.h>

// folly::PicoSpinLock 也是一个最简单的while-cas模式实现的spinlock，通常不应该被使用。但是其使用一个整形数
// 作为spin的载体，其大部分bit可以用来存储数据，剩余bit用来记录lock状态，如果非常需要内存紧凑，可以考虑使用。

#include <folly/synchronization/RWSpinLock.h>

// folly::RWSpinLock 是一个简单的读写spinlock实现，并且支持锁升级、降级逻辑，其升降级遵循boost的 https://www.boost.org/doc/libs/1_47_0/doc/html/thread/synchronization.html#thread.synchronization.mutex_concepts.upgrade_lockable 语义，其可以通过 lock_upgrade() + unlock_upgrade_and_lock() 完成读锁到写锁的升级。
```

需要明确的是，spinlock类的锁都不太适合在应用层面随意使用，你必须明显进行测试、并且清楚自己了解内在机理，否则你只应该简单的使用 `std::mutex`或者`folly::SharedMutex`等，可以参考[Spinlocks Considered Harmful](https://matklad.github.io//2020/01/02/spinlocks-considered-harmful.html)等。

## baton

```cpp
// 所在头文件：
#include <folly/synchronization/Baton.h>

// Baton 通常用作线程间同步、等待、通知的标识符号，常用姿势是，一些线程调用 wait() 方法等待另
// 一些线程完成某项工作，其完成以后调用 post() 方法进行通知。 其跟一般PV信号量的区别是，Baton
// 更轻量化、通知策略更简单(没有FILO/FIFO等策略)、仅能够通知一次，在简单场景中更高效。
//
// 声明：MayBlock表示十分会被长时间block，其实就是内部衡量是否需要一直spin的依据，
// 否则会调用 futex 相关 syscall 释放 cpu
// Atom 指示使用什么原子操作的实现，通常使用 std::atomic 即可
template <bool MayBlock = true, template <typename> class Atom = std::atomic>
class Baton;

// 常规用法：
// Baton 用于同步线程间的PV (block/wakeup)，但是不像 semaphores
// 信号量那样可以多次pv，baton 仅支持单次pv操作，folly::Future 中的
// block/wakeup 就是使用 Baton<> 来实现的。

Baton<> baton;
Baton<false> spin_baton;

// 其基本方法有：
bool Baton<>::ready(); // 测试是否已经被标记置位
void Baton<>::post();  // 置位
bool Baton<>::try_wait(); // 等同 ready
// 等待直到被置位，可以传入一个 wait_options 来控制 spin 的最大时间，默认2us
void Baton<>::wait(const WaitOptions& opt = wait_options());
```

## folly::SaturatingSemaphore

```cpp
// 所在头文件：
#include <folly/synchronization/SaturatingSemaphore.h>

// folly::SaturatingSemaphore 是一个经典的信号量实现，可以支持多个poster和多个waiter同时调用PV。
// 而且可以对一个SaturatingSemaphore可以多次设置post状态，而且幂等（但是不累积，设置成post状态后，
// 所有wait()调用都会通过，简而言之就是只有PV状态，而没有对应的计数），然后可以通过reset()方法进行恢复。
//
// 其跟Baton的主要区别就是Baton只支持有且仅有一个poster，SaturatingSemaphore支持多个，且可以并发调用post()

///  方法：
///   bool ready():
///     Returns true if the flag is set by a call to post, otherwise false.
///     Equivalent to try_wait, but available on const receivers.
///   void reset();
///     Clears the flag.
///   void post();
///     Sets the flag and wakes all current waiters, i.e., causes all
///     concurrent calls to wait, try_wait_for, and try_wait_until to
///     return.
///   void wait(
///       WaitOptions opt = wait_options());
///     Waits for the flag to be set by a call to post.
///   bool try_wait();
///     Returns true if the flag is set by a call to post, otherwise false.
///   bool try_wait_until(
///       time_point& deadline,
///       WaitOptions& = wait_options());
///     Returns true if the flag is set by a call to post before the
///     deadline, otherwise false.
///   bool try_wait_for(
///       duration&,
///       WaitOptions& = wait_options());
///     Returns true if the flag is set by a call to post before the
///     expiration of the specified duration, otherwise false.
```

## folly::LifoSem

```cpp
#include <folly/synchronization/LifoSem.h>

// folly::LifoSem 相对于 folly::SaturatingSemaphore 维护了一个通知的顺序，后入先出，主要是尽快通知较活跃的线程。
// 并且post()可以指定通知的个数，不像 folly::SaturatingSemaphore post()会通知全部的wait().
//
// 其内部用一个对象池维护了waiter的相对顺序，然后按顺序进行通知。
```

## Hazard Pointer

```cpp
// 所在头文件:
#include <folly/synchronization/Hazptr.h>

// hazard pointer 支持一写多读
// 其实现依赖thread local机制。
// 其原始的 hazard pointer 并没有直接暴露给用户来进行操作，而是给用户提供一个 hazptr_holder 对象进行持有 hazard pointer
```

# Atomic

## folly::AtomicStruct

```cpp
// AtomicStruct 可以原子地操作一个大小小于等于8byte的对象，其内部原理就是把对象转换成对应大小的int类型，
// 使用std::atmoic<int>来操作。是一个比较有助的封装。对象大小大于8byte的则无法使用该封装。
#include <folly/synchronization/AtomicStruct.h>

struct A {
  int32_t a;
};

folly::AtomicStruct<A> a;
auto xx = a.load();
```

# 对象管理、单例等

## 线程维度的单例对象

folly 的`folly/SingletonThreadLocal.h`文件中提供了宏`FOLLY_DECLARE_REUSED`以创建简便易用的线程维度的单例对象，可以减少逻辑上临时对象的反复创建、销毁的开销，要求该对象有一个`clear()`方法(通常是 STL 容器)即可。可以用这个宏优化线程池中对应函数中反复创建、销毁的临时容器。

其实现，就是创建一个对应对象的`thread_local`的单例，离开作用域的时候，调用对象的`clear()`方法：

```cpp
// in folly/SingletonThreadLocal.h
#define FOLLY_DECLARE_REUSED(name, ...)                                        \
  struct __folly_reused_type_##name {                                          \
    __VA_ARGS__ object;                                                        \
  };                                                                           \
  auto& name =                                                                 \
      ::folly::SingletonThreadLocal<__folly_reused_type_##name>::get().object; \
  auto __folly_reused_g_##name = ::folly::makeGuard([&] { name.clear(); })
```

用法

```cpp
#include <folly/SingletonThreadLocal.h>

void traverse_perform(int root);
template <typename F>
void traverse_each_child_r(int root, F const&);
void traverse_depthwise(int root) {
  // preserves some of the memory backing these per-thread data structures
  FOLLY_DECLARE_REUSED(seen, std::unordered_set<int>);
  FOLLY_DECLARE_REUSED(work, std::vector<int>);
  // example algorithm that uses these per-thread data structures
  work.push_back(root);
  while (!work.empty()) {
    root = work.back();
    work.pop_back();
    seen.insert(root);
    traverse_perform(root);
    traverse_each_child_r(root, [&](int item) {
      if (!seen.count(item)) {
        work.push_back(item);
      }
    });
  }
}
```
