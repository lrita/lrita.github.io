---
layout: post
title: C++11中的name lookup
categories: [c++]
description: c++
keywords: c++
---

# `identifier`
C++编译器将文件代码源文件解析后，将代码分解为`identifier`[^1]、数值、运算符等，其中`identifier`是由非数字开头、任意字符数字和下划线组成的部分，其用来组成`声明`、`表达式`、`name`和`qualified identifier`。

在`声明`中`identifier`:
* 不能时语法关键字
* 不要以双下划线（`__`）或者下划线（`_`）开头，以免和编译器或者标准库的内部声明冲突，可以参见`17.6.4.3 [reserved.names]`

`identifier`在表达式中除了表示一些简单的函数和对象外，还可以是：
* the name of an operator function, such as `operator+` or `operator new`;
* the name of a user-defined conversion function, such as `operator bool`;
* the name of a user-defined literal operator function, such as `operator "" _km`;
* the character `~` followed by class name, such as `~MyClass`;
* the character `~` followed by decltype specifier, such as `~decltype(str)`;
* a `template identifier`, such as `MyTemplate<int>`;
* `qualified identifier`, such as `std::string` or `::tolower`.

## `qualified identifier`
`qualified identifier`(限定标识符)是由域解析符`::`标识与`class`名、枚举类名、`namespace`或者`decltype`表达式限定的一类`identifier`。

比如:

* `std::string::npos`
* `::tolower`
* `::std::cout`
* `boost::signals2::connection`

# `name`
`name`是指下面的一个实体或标签：
* an `identifier`
* 操作符函数 (`operator+`, `operator new`);
* [用户定义的转换函数](http://doc.bccnsoft.com/docs/cppreference2015/en/cpp/language/cast_operator.html) (`operator bool`);
* [用户定义的字面值转换符](http://doc.bccnsoft.com/docs/cppreference2015/en/cpp/language/user_literal.html) (`operator "" _km`);
* 模板 id (`name<arg, arg>`).
* `goto`语句指向的`label`

当编译器遇到一个未知的`name`时，会进行`name lookup`[^2]，例如，当编译`std::cout << std::endl;`时：
* 对`std`进行`unqualified name lookup`，发现其是一个声明在头文件`<iostream>`中的`namespace`
* 对`cout`进行`qualified name lookup`，发现其是一个声明在`namespace std`中的变量
* 对`endl`进行`qualified name lookup`，发现其是一个声明在`namespace std`中的函数模板
* 对`<<`进行`argument-dependent lookup`，发现其是一个声明在`namespace std`中的函数模板声明

**其主要规则是，如果目标是一个`qualified identifier`，进行`Qualified name lookup`[^3]，否则进行`Unqualified name lookup`[^4]，对于函数还可能进行`Argument-dependent lookup`[^5]**

## Qualified name lookup

## Unqualified name lookup

## Argument-dependent lookup

# 参考

[^1]: [Identifiers](http://doc.bccnsoft.com/docs/cppreference2015/en/cpp/language/name.html)
[^2]: [Name lookup](http://doc.bccnsoft.com/docs/cppreference2015/en/cpp/language/lookup.html)
[^3]: [Qualified name lookup](http://doc.bccnsoft.com/docs/cppreference2015/en/cpp/language/qualified_lookup.html)
[^4]: [Unqualified name lookup](http://doc.bccnsoft.com/docs/cppreference2015/en/cpp/language/unqualified_lookup.html)
[^5]: [Argument-dependent lookup](http://doc.bccnsoft.com/docs/cppreference2015/en/cpp/language/adl.html)