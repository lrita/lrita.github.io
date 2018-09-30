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
`qualified identifier(限定标识符)`是由域解析符`::`标识与`class`名、枚举类名、`namespace`或者`decltype`表达式限定的一类`identifier`。

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
* 因为`std`左侧没有`::`进行限定，则对`std`进行`unqualified name lookup`，发现其是一个声明在头文件`<iostream>`中的`namespace`
* 因为`cout`左侧有`::`进行限定且其限定的域为一个`namespace`，则对`cout`进行`qualified name lookup`，发现其是一个声明在`namespace std`中的变量
* 因为`endl`左侧有`::`进行限定且其限定的域为一个`namespace`，则对`endl`进行`qualified name lookup`，发现其是一个声明在`namespace std`中的函数模板
* `<<`没有限定且为一个函数，对`<<`进行`argument-dependent lookup`，发现其是一个声明在`namespace std`中的函数模板声明

**其主要规则是，如果目标是一个`qualified identifier(限定标识符)`，进行`Qualified name lookup`[^3]，否则进行`Unqualified name lookup`[^4]，对于函数还可能进行`Argument-dependent lookup`[^5]**

## Qualified name lookup
> 当遇到未知的`qualified identifier(限定标识符)`时，会去其对应的限定符区域内进行查找，比如命名空间、类空间、枚举空间等。

```c++
  std::cout << 1; // 解析"cout"时，去命名空间std中进行查找
  struct A {
    typedef int type;
  };
  A::type a;      // 解析"type"时，去A类空间中查找
```

> 如果`::`左侧没有限定符，则去全局命名空间进行查找，这样就避免了被本地声明遮盖的情况：

```c++
#include <iostream>
int main() {
  struct std{};
  std::cout << "fail\n"; // Error: unqualified lookup for 'std' finds the struct
  ::std::cout << "ok\n"; // OK: ::std finds the namespace std
}
```

> 当然，在解析`::`右手侧的`name`之前需要先解析`::`左手侧的`name`（除非使用了`decltype`表达式或者`::`左侧为空）。**至于带`::`的`name`查找时`qualified name lookup`还是`unqualified name lookup`，取决于`::`左侧的`name`。当`::`左侧`name`为`namespace`、`class`、`枚举`、`实例化的模板`时，为`qualified name lookup`。**

```c++
struct A {
  static int n;
};
int A::n = 0;
int main() {
  int A;
  A::n = 42;    // OK: 对"A"进行"unqualified lookup"时，忽略了本地变量A
  A b;          // 错误: 对"A"进行"unqualified lookup"会指向本地变量A
  struct A b;   // A 前面需要增加tag "struct" 进行限定，以避免被本地变量A遮盖
}
```

> `qualified name`用于[声明](https://en.cppreference.com/w/cpp/language/declarations)时，当同在一个声明中的`unqualified name`需要进行`unqualified lookup`时，在`qualified name`对应的类空间、命名空间进行查找。不在同一个声明中的`unqualified name`，不会受到这个影响。同时也不受声明类型的影响。

上面这一段话比较难描述，需要结合一个例子来理解：
```c++
class X { };
constexpr int number = 100;
struct C {
  class X { };
  static const int number = 50;
  static X arr[number];
};
X C::arr[number], brr[number];    // 错误: "X"经过`unqualified name`解析为`::X`与`C::arr`对应的`C::X`不同。
C::X C::arr[number], brr[number]; // OK: `C::arr`的长度为50，`brr`的长度为100
                                  // `C::arr[number]`中的`number`受同一个声明中的"qualified name"`C::arr`
                                  // 的影响，回去类空间`C`中查找，得到`C::number`，也就是50。而`brr[number]`
                                  // 与`C::arr[number]`不是同一个声明，则不受这种影响，得到`::number`，也就是// 100
```

> 当`::`右侧是`~`加一个`name`时（其实就是析构函数或伪析构函数），则解析这个`name`时，使用`::`左侧的域

```c++
struct C { typedef int I; };
typedef int I1, I2;
extern int *p, *q;
struct A { ~A(); };
typedef A AB;
int main() {
  p->C::I::~I(); // 解析`~I()`时，使用`C::I`这个`name`相同的域`C::`，则找到了`C::I`
  q->I1::~I2();  // 解析`~I2()`时，使用`::I1`这个`name`相同的域，则找到了`::I2`
}
```

> 当`::`左侧为枚举类时，则`::`右侧的`name`必须属于这个枚举类，否则为`ill-formed`

> `qualified name lookup`还可以被用来调用被隐藏的方法，这种调用方式不会调用虚函数：

```c++
struct B { virtual void foo(); };
struct D : B { void foo() override; };
int main()
{
    D x;
    B& b = x;
    b.foo();    // calls D::foo (virtual dispatch)
    b.B::foo(); // calls B::foo (static dispatch)
}
```

> 模板参数进行解析时，从当前域进行解析

```c++
namespace N {
   template<typename T> struct foo {};
   struct X {};
}
N::foo<X> x; // 错误："X"被解析为"::X"，而不是"N::X"
```

> 对一个`namespace N`进行`qualified name lookup`时，首先考虑`namespace N`内的声明和[`inline namespace members`](https://en.cppreference.com/w/cpp/language/namespace#Inline_namespaces)。如果没有匹配的，其次考虑[`using-directives`](https://en.cppreference.com/w/cpp/language/namespace#Using-directives)导入到`namespace N`中的声明

```c++
int x;
namespace Y {
  void f(float);
  void h(int);
}
namespace Z {
  void h(double);
}
namespace A {
  using namespace Y;
  void f(int);
  void g(int);
  int i;
}
namespace B {
  using namespace Z;
  void f(char);
  int i;
}
namespace AB {
  using namespace A;
  using namespace B;
  void g();
}
void h()
{
  AB::g();  // 首先考虑"AB::g"，则不再对namespace A、B进行查找
  AB::f(1); // 首先在namespace AB中查找，未匹配，然后到namespace A、B中查找，查找到"A::f"和"B::f"，
            // 有匹配，则不再对namespace Y进行查找。然后在"A::f"和"B::f"中间选择了"A::f(int)"。
  AB::x++;    // 首先在namespace AB中查找，未匹配。然后到namespace A、B中查找，未匹配。
              // 然后到namespace Y、Z中查找，未匹配。然后报错。
  AB::i++;  // 首先在namespace AB中查找，未匹配。然后到namespace A、B中查找，匹配到"A::i"和"B::i"
            // 存在冲突，报错。
  AB::h(16.8);  // 首先在namespace AB中查找，未匹配。然后到namespace A、B中查找，未匹配。
                // 然后到namespace Y、Z中查找，匹配到"Y::h"和"Z::h"，选择"Z::h(double)"
}
```

> 在`namespace`中进行`qualified name lookup`时，允许通过不同途径匹配到相同的类型

```c++
namespace A { int a; }
namespace B { using namespace A; }
namespace D { using A::a; }
namespace BD {
  using namespace B;
  using namespace D;
}
void g()
{
  BD::a++; // 首先在 namespace BD 中查找，未匹配。然后到 namespace B、D中查找，未匹配。
           // 再下面，namespace B、D都同时指向了 namespace A，则在 namespace A 中匹
           // 配到 A::a，选择 "A::a"
}
```

## Unqualified name lookup

## Argument-dependent lookup

# 参考

[^1]: [Identifiers](https://en.cppreference.com/w/cpp/language/identifiers)
[^2]: [Name lookup](https://en.cppreference.com/w/cpp/language/lookup)
[^3]: [Qualified name lookup](https://en.cppreference.com/w/cpp/language/qualified_lookup)
[^4]: [Unqualified name lookup](https://en.cppreference.com/w/cpp/language/unqualified_lookup)
[^5]: [Argument-dependent lookup](https://en.cppreference.com/w/cpp/language/adl)