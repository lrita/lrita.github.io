---
layout: post
title: C++中的Digraphs、Trigraphs和Tokens
categories: [c++]
description: c++
keywords: c++
---

偶然在[C++ Quiz](http://cppquiz.org)上看到一道题：
```c++
// 以下代码的输出是什么？
#include<iostream>

int main(){
  int x=0; //What is wrong here??/
  x=1;
  std::cout<<x;
}
```

这个看似简单，实际很容易采坑。

之前也是偶然间了解到C++的`Digraph(双字符组)`，但是当时没有进行扩展了解，没想到C++还有`Trigraph(三字符组)`...，这个概念其实也很简单，维基百科的词条`三字符组与双字符组`[^1]写的也很清楚，就直接搬运过来一下。

# 缘起
C语言的源程序的最低必须的字符集是基于7位ASCII码字符集，是[ISO 646-1983 Invariant Code Set](https://zh.wikipedia.org/wiki/ISO/IEC_646)的一个超集。ISO 646最初是1972年颁布的一项国际化的7位ASCII标准，规定了12个字符所对应的[码位](https://zh.wikipedia.org/wiki/码位)保持对各国标准开放：```# $ @ [ \ ] ^ ` { | } ~``` 。
因此法国标准AFNOR NF Z 62010-1982把码位0x7c（ASCII码的 | ）定义为ù，用法文键盘就难以输入C语言的位或运算符 | ；码位0x7e（ASCII码的 ～）定义为 ¨ （即[分音符](https://zh.wikipedia.org/wiki/分音符)），法文键盘就难以输入C语言的位非运算符 ～ 。
加拿大法语标准CSA Z243.4-1985中把码位0x5e（ASCII码的 ^ ）在定义为É，导致难以输入C语言的异或运算符 ^ 。

# 三字符组
为解决上述的C语言源代码输入问题，C语言标准规定预处理器（C preprocessor）在扫描处理C语言源文件时，替换下述的3字符出现为1个字符

| 三字符组 | 替换为 |
| ---- | ---- |
| `??=` | `#` |
| `??/` | `\` |
| `??'` | `^` |
| `??(` | `[` |
| `??)` | `]` |
| `??!` | `|` |
| `??<` | `{` |
| `??>` | `}` |
| `??-` | `~` |

_注意_: **编译器对`三字符组`的处理是在解析注释、宏的步骤的前面[^2]，可以理解为优先处理`三字符组`**

那我们再回头看上面那个题，其等价于：
```c++
// 以下代码的输出是什么？
#include<iostream>

int main(){
  int x=0; //What is wrong here\   <- ??/被解释为\，使得自动折行
  x=1;                             <- 此行其实是被注释掉的
  std::cout<<x;
}
```

故，如果希望在源程序中有两个连续的问号，且不希望被预处理器替换，这种情况出现在字符常量、字符串字面值或者是程序注释中，可选办法是用字符串的自动连接：`"...?""?..."`或者转义序列：`"...?\?..."`。

_注意_: **`Trigraph(三字符组)`在`C++17`被移除了语法**[^3]

从Microsoft Visual C++ 2010版开始，该编译器默认不再自动替换三字符组。如果需要使用三字符组替换（如为了兼容古老的软件代码），需要设置编译器命令行选项`/Zc:trigraphs`

g++仍默认支持三字符组，但会给出编译警告。

# 双字符组
1994年公布了一项C语言标准的修正案，引入了更具有可读性的5个双字符组。这也包括进了[C99](https://zh.wikipedia.org/wiki/C99)标准。

| 双字符组 | 替换为 |
| ----- | ---- |
| `<:` | `[` |
| `:>` | `]` |
| `<%` | `{` |
| `%>` | `}` |
| `%:` | `#` |

**不同于`三字符组`在源文件的任何出现都会被预处理器替换，`双字符`如果出现在字符串字面值（quoted string）、字符常量、程序注释中将不被替换**。双字符组的替换发生在编译器对源程序的tokenization阶段（即识别出关键字、标识符等，类似于自然语言的“断词”），仅当双字符组作为一个token或者token的组成部分时（如`%:%:`被替换为预处理运算符`##`），双字符组才被替换为单字符。
g++支持上述双字符组替换。但Microsoft Visual C++不支持双字符组替换。

# Token
C++标准支持C语言的三字符组与双字符组（包括C99中的增补）。C++自身还提供了下述内置的关键字：

| 关键字 | 等价于 |
| ----- | ----- |
| `and` | `&&` |
| `bitor` | `|` |
| `or` | `||` |
| `xor` | `^` |
| `compl` | `~` |
| `bitand` | `&` |
| `and_eq` | `&=` |
| `or_eq` | `|=` |
| `xor_eq` | `^=` |
| `not` | `!` |
| `not_eq` | `!=` |

Microsoft Visual C++编译器要求如果使用上述关键字，必须包含头文件[ciso646](https://zh.wikipedia.org/wiki/Ciso646)，否则编译报错。如“ error C2065: 'not' : undeclared identifier”。而g++编译器就不要求包含头文件ciso646。

# 参考
[^1]: [三字符组与双字符组](https://zh.wikipedia.org/wiki/三字符组与双字符组)
[^2]: [Phases of translation](https://en.cppreference.com/w/cpp/language/translation_phases)
[^3]: [Alternative operator representations](https://en.cppreference.com/w/cpp/language/operator_alternative)