---
layout: post
title: c++使用std::initializer_list构建常见多个连续或表达式的优雅写法
categories: [c++]
description: c++
keywords: c++ std::initializer_list
---

## 问题一

当代码逻辑中需要经常判断多个连续`||`表达式时，经常采用的模式都是：
```cpp
if (get_level() == "L1" || get_level() == "L2" || ... || get_level() == "L10") {
  ...
}
```
当取值逻辑代价较高时，就需要减少取值的次数：
```cpp
auto level = get_level();
if (level == "L1" || level == "L2" || ... || level == "L10") {
  ...
}
```
即使有C++17的加持，这种前置取值的写法经常也不是很优雅：
```cpp
auto level = get_level();
auto type  = get_type();
if ( (level == "L1" || level == "L2" || ... || level == "L10") || (type == TypeFirst || type == TypeSecond)) {
  ...
}
```

## 问题二

当需要写多个值`||`判断时，按上面的常规写法，还是显得啰嗦，因此有时，人们使用`std::unordered_set`来构建或值判断的集合：
```cpp
static std::unordered_set<std::string> levels = {
  "L1",
  "L2",
  ...
  "L10",
};
static std::unordered_set<Type> types = {
  TypeFirst,
  TypeSecond,
};
if (levels.contains(get_level()) || types.contains(get_type())) {
  ...
}
```
这样虽然解决了问题一，但是不满足零代价抽象，其实新增了额外的性能开销，[例如](https://godbolt.org/z/fr1qno5fq)上面这个简单例子编译得到：
```asm
        call    get_level[abi:cxx11]()    # 先获取get_level()
        mov     rsi, QWORD PTR [rsp+8]
        mov     rdi, QWORD PTR [rsp]
        mov     edx, 3339675911
        call    std::_Hash_bytes(void const*, unsigned long, unsigned long) # 计算返回值hash
        xor     edx, edx
        mov     edi, OFFSET FLAT:_ZL6levels
        mov     rcx, rax
        div     QWORD PTR _ZL6levels[rip+8]
        mov     rsi, rdx
        mov     rdx, rsp
        call    std::_Hashtable<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >, std::__detail::_Identity, std::equal_to<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >, std::hash<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >, std::__detail::_Mod_range_hashing, std::__detail::_Default_ranged_hash, std::__detail::_Prime_rehash_policy, std::__detail::_Hashtable_traits<true, true, true> >::_M_find_before_node(unsigned long, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, unsigned long) const
        test    rax, rax # hash find
        je      .L58
        cmp     QWORD PTR [rax], 0
        je      .L58
        ...
```
可见其需要最字符串计算hash，然后在进行hashmap的匹配，在数值集合不大的情况下，这种开销远高于几次简单的字符串比较。

## 解决方案

基于上面的两种问题，现在总结出一个既满足零代价抽象，有写起来比较优雅的方式，使用一个方法`c_contains`：
```cpp
#include <string_view>
#include <type_traits>
#include <initializer_list>
#include <algorithm>

template <typename InputIt, typename T>
bool c_contains(InputIt begin, InputIt end, T &&value) { // (1)
  return std::find(begin, end, std::forward<T>(value)) != end;
}

template <typename C = std::initializer_list<std::string_view>, typename T>
bool c_contains(const C &c, T &&value) {                                 // (2)
  return c_contains(std::begin(c), std::end(c), std::forward<T>(value)); // 调用(1)
}

template <typename U, typename T,
          std::enable_if_t<!std::is_same_v<U, const char *>, int *> = nullptr>
bool c_contains(std::initializer_list<U> c, T &&value) {
  return c_contains(c.begin(), c.end(), std::forward<T>(value)); // 调用(2)
}
```
这里主要就是使用`std::initializer_list`和`std::find`来构成连续`||`值的判断：
```cpp
int foo() {
  if (c_contains({"L1", "L2", ... , "L10"}, get_level()) ||
      c_contains({TypeFirst, TypeSecond}, get_type())) {
    return 0;
  }
  return 1;
}
```
其[编译生成](https://godbolt.org/z/r4ePxYTdW)的：
```asm
        call    get_level[abi:cxx11]()
        mov     rbp, QWORD PTR [rsp+8]
        mov     r12, QWORD PTR [rsp]
        mov     eax, 2
        mov     edi, OFFSET FLAT:.LC0
        mov     ebx, OFFSET FLAT:._79
        jmp     .L15
.L2:
        cmp     QWORD PTR [rbx+16], rbp
        je      .L75
.L5:
        cmp     rbp, QWORD PTR [rbx+32]
        je      .L76
.L8:
        cmp     rbp, QWORD PTR [rbx+48]
        je      .L77
.L11:
        add     rbx, 64
        cmp     rbx, OFFSET FLAT:._79+128
        je      .L14
        mov     rdi, QWORD PTR [rbx+8]
        mov     rax, QWORD PTR [rbx]
.L15:
        cmp     rax, rbp
        jne     .L2
        test    rbp, rbp
        je      .L39
        mov     rdx, rbp
        mov     rsi, r12
        call    memcmp
        test    eax, eax
        jne     .L2
.L39:
        cmp     rbx, OFFSET FLAT:._79+160
        je      .L37
.L24:
        mov     rdi, r12
.L35:
        lea     rax, [rsp+16]
        xor     ebx, ebx
        cmp     rdi, rax
        je      .L1
.L33:
        call    operator delete(void*)
.L1:
        add     rsp, 32
        mov     eax, ebx
        pop     rbx
        pop     rbp
        pop     r12
        ret
.L77:
        test    rbp, rbp
        je      .L12
        mov     rdi, QWORD PTR [rbx+56]
        mov     rdx, rbp
        mov     rsi, r12
        call    memcmp
        test    eax, eax
        jne     .L11
.L12:
        add     rbx, 48
        jmp     .L39
.L76:
        test    rbp, rbp
        je      .L9
        mov     rdi, QWORD PTR [rbx+40]
        mov     rdx, rbp
        mov     rsi, r12
        call    memcmp
        test    eax, eax
        jne     .L8
.L9:
        add     rbx, 32
        jmp     .L39
.L75:
        test    rbp, rbp
        je      .L6
        mov     rdi, QWORD PTR [rbx+24]
        mov     rdx, rbp
        mov     rsi, r12
        call    memcmp
        test    eax, eax
        jne     .L5
.L6:
        add     rbx, 16
        jmp     .L39
.L14:
        cmp     rbp, 2
        je      .L78
        cmp     rbp, 3
        je      .L79
```
可以看出其直接转换成几句依次的`memcmp`，没有额外的抽象代价。

其中`c_contains(2)`是额外增加优化，发现在部分编译器上`{"L1", "L2", ... , "L10"}`被推断为`std::initializer_list<const char *>`，然后在后面`std::find`中，比较`const char *`和`std::string`存在抽象代价。因此使用`c_contains(2)`和`c_contains(3)`共同作用将`{"L1", "L2", ... , "L10"}`推断为`std::initializer_list<std::string_view>`，具体案例可以参考[quick-bench](https://quick-bench.com/q/jVTCpwUHmtFhtD-gEKw4WP99kuQ)。
