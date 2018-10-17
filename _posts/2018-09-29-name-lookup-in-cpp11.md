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
> 当遇到未知的`qualified identifier(限定标识符)`时，会去其对应的限定符区域内进行查找，比如命名空间、类作用域、枚举空间等。

```c++
  std::cout << 1; // 解析"cout"时，去命名空间std中进行查找
  struct A {
    typedef int type;
  };
  A::type a;      // 解析"type"时，去A类作用域中查找
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

> `qualified name`用于[声明](https://en.cppreference.com/w/cpp/language/declarations)时，当同在一个声明中的`unqualified name`需要进行`unqualified lookup`时，在`qualified name`对应的类作用域、命名空间进行查找。不在同一个声明中的`unqualified name`，不会受到这个影响。同时也不受声明类型的影响。

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
                                  // 的影响，回去类作用域`C`中查找，得到`C::number`，也就是50。而`brr[number]`
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
`unqualified name`只要指那些左侧没有`::`域符号限定的`name`。在搜索时，在相关命名空间、`using`引入的命名空间等域进行，直到找到一个匹配的类型则停止。

当遇到一个未知的`unqualified name`时，会从当前文件域、命名空间开始查找，具体情形比较简单。下面列出一些需要特殊注意的点：

> 一个变量的定义在其命名空间"X"外时且定义语句中引用了`unqualified name`，解析这个`unqualified name`时，首先从变量命名空间“X”开始查找。有点类似ADL，但是注意区别。

```c++
namespace X {
extern int x;  // 声明，不是定义
int n = 1;     // 在解析x的定义时，其引用了unqualified name “n”，先从其定义域"X"开始搜索，首先发现 n=1
};

int n = 2;     // 如果X::n不存在时，则匹配::n，否则被遮盖。
int X::x = n;  // X::x的定义式，X::x的定义在其命名空间"X"外，且引用了unqualified name “n”，
               // 在解析"n"时，匹配X::n，则X::x为1
```

> 当在命名空间外定义一个`非成员函数`时，如果函数内引用了`unqualified name`时，解析这个`unqualified name`时，依次搜索其函数定义之前的：`本地代码块`、`代码块外层定义变量`、函数定义之前的`命名空间`、函数定义之前的`外层命名空间`、函数定义之前的`全局域`，直到找到一个匹配的`name`，在函数定义之后的相关域不在搜索范围内。

```c++
namespace A {
    namespace N {
       void f();
       int i=3;   // ③: 查找"命名空间"，如果将①②注释掉，这句定义存在，则"i"的定义匹配此处，则i=3
    }
    int i=4;      // ④: 查找"外层命名空间"，如果将①②③注释掉，这句定义存在，则"i"的定义匹配此处，则i=4
}

int i=5;          // ⑤: 查找"全局域"，如果将①②③④注释掉，这句定义存在，则"i"的定义匹配此处，则i=5

// 在命名空间外定义了一个非成员函数f()，这个例子的注释需要按照序号来看。
void A::N::f() {
    int i = 2;         // ②: 其次查找"外层代码块"，如果将①注释掉，这句定义存在，则"i"的定义匹配此处，则i=2
    {
       int i = 1;      // ①: 首先查找“本地代码块”，如果这句定义存在，则"i"的定义匹配此处，则i=1
       std::cout << i; // ⓪: 在代码块中引用了unqualified name "i"，则需要查找"i"的定义
    }
}

// int i; // 如果将⑤移动到f()定义之后则不参与"i"的查找

namespace A {
  namespace N {
    // int i; // 如果将③移动到f()定义之后则不参与"i"的查找
  }
}
```

> 当在类作用域外定义一个`成员函数`时，如果函数内引用了`unqualified name`时，解析这个`unqualified name`时，依次搜索其函数定义之前的：`本地代码块`、`代码块外层定义变量`、`类作用域`、`基类作用域`、函数定义之前的`命名空间`、函数定义之前的`外层命名空间`、函数定义之前的`全局域`，直到找到一个匹配的`name`，在函数定义之后的相关域不在搜索范围内。注意跟上面的区别。

```c++
class B {
    int i; // ④: 如果①②③不存在，”i“匹配此处
};
namespace M {
    int i; // ⑥: 如果①②③④⑤不存在，”i“匹配此处
    namespace N {
        int i; // ⑤: 如果①②③④不存在，”i“匹配此处
        class X : public B {
            int i;    // ③: 如果①②不存在，”i“匹配此处
            void f(); // ⓪: class X中声明了一个函数f()
            // int i; // 将③移动到这里也OK，类成员的可见性受声明的位置的影响。
        };
        // int i; // ⑤移动到这里也可以，跟f()实现代码的位置有关，与f()声明的位置无关
    }
}

int i; // ⑦: 如果①②③④⑤⑥不存在，”i“匹配此处

// 定义class X中声明的函数f()
void M::N::X::f()
{
    int i;   // ②: 如果①不存在，”i“匹配此处
    {
      int i; // ①: 如果①存在，”i“首先匹配此处
      i = 16;
    }

    // int i; // 如果这句存在，①②都不存在，也不会匹配此处，因为其出现在调用i的代码块之后
}
namespace M {
  namespace N {
    // int i; // ⑤移动到这里不匹配，其出现在f()实现代码之后
  }
}
```

> 类的静态成员定义时，如果引用了`unqualified name`，其查找过程与类成员函数中`unqualified name`的查找顺序相同

```c++
struct X {
    static int x;
    static const int n = 1;
};
int n = 2;
int X::x = n; // 找到了 X::n，将 X::x 设置为 1 而不是 2
```

> 当一个类的友元函数被定义在类作用域内部时，该友元函数中引用的`unqualified name`的查找顺序与该类的成员函数中的`unqualified name`的查找顺序相同。当类的友元函数被定义在类作用域外部时，该友元函数中引用的`unqualified name`的查找顺序与其所在命名空间的其他函数中的`unqualified name`的查找顺序相同。

```c++
int i = 3;  // ③: 当①②不存在时，f1中的"i"匹配此处
            // 3️⃣: 当1️⃣2️⃣不存在时，f2中的"i"也不会匹配此处
struct X {
    static const int i = 2; // ②: 当这句定义存在时，①不存在时，f1中的"i"匹配此处
                            // 2️⃣: 当1️⃣不存在时，f2中的"i"也不会匹配此处
    friend void f1(int x)
    {
        int i; // ①: 当这句定义存在时，f1中的"i"匹配此处
        i = x; // ⓪: 友元函数f1(int)被定义在类X作用域内部时，"i"的查找顺序与X的成员函数查找顺序相同
    }
    friend int f2();
    // static const int i = 2; // ②移动到此处也OK
};
void f2(int x) {
    int i; // 1️⃣: 当这句定义存在时，f2中的"i"匹配此处
    i = x; // 0️⃣: 友元函数f2(int)被定义在类X作用域外部时，"i"的查找顺序f2(int)所在命名空间的普通函数的查找顺序相同
}
```

> 当一个类A的成员函数被声明为类B的友元，且该声明中包含`unqualified name`，则对`unqualified name`进行查找时：
> * 如果`unqualified name`不是任何模板的参数，则首先去类作用域A中进行查找
> * 如果在类作用域A中未匹配或者其实模板参数，则去类作用域B中进行查找

```c++
// 这个类的成员函数被作为友元
struct A {
    typedef int AT;
    void f1(AT);
    void f2(float);
    template <class T> void f3();
};

// 这个类授予友元关系
struct B {
    typedef char AT;
    typedef float BT;
    friend void A::f1(AT);   // 对 "AT" 的查找时，先到类作用域A中进行查找，匹配"A::AT"
    friend void A::f2(BT);   // 对 "BT" 的查找时，先到类作用域A中进行查找，未匹配，
                             // 再去类作用域B中查找，匹配"B::BT"
    friend void A::f3<AT>(); // 对 "AT" 的查找时，"AT"是模板参数，则直接去类作用域B中查找，匹配"B::AT"
};
```

> `unqualified name`被当做函数默认参数时，查找其定义时，首先考虑同一函数声明中的形参:

```c++
class X {
    int a, b, i, j;
public:
    const int& r;
    X(int i): r(a), // 将 X::r 初始化为指代 X::a
              b(i), // 将 X::b 初始化为形参 i 的值
              i(i), // 将 X::i 初始化为形参 i 的值
              j(this->i) // 将 X::j 初始化为 X::i 的值
    { }
}

int a;
int f(int a, int b = a); // 错误：对 a 的查找找到了形参 a，而不是 ::a
                         // 但在默认实参中不允许使用形参
```

> 在枚举类定义时，`unqualified name`的查找首先考虑当前枚举类的作用域

```c++
const int RED = 7;
enum class color {
    RED,
    GREEN = RED+2, // RED 找到了 color::RED ，而不是 ::RED ，因此 GREEN = 2
    BLUE = ::RED+4 // 通过 qualified name lookup 找到 ::RED ， BLUE = 11
};
```

> 在"try-catch"语句中，`unqualified name`的查找

```c++
int n = 3;           // ③
int f(int n = 2) {   // ②
  try {
     int n = -1;     // 不会匹配到该处
  } catch(...) {
     // int n = 1;   // ①: 加入此处存在，则匹配该处
     assert(n == 2); // ⓪: n 按①②③的顺序进行依次查找，匹配f的参数n，即②
     throw;
  }
  return 0;
}
```

> 对于在表达式中所使用的重载运算符（比如在 a+b 中使用的 operator+），其查找规则和对在如 operator+(a,b) 这样的显式函数调用表达式中所使用的运算符是有所不同的：当处理表达式时要分别进行两次查找：对非成员的运算符重载，也对成员运算符重载（对于同时允许两种形式的运算符）。然后将这两个集合和在重载解析所述内建的运算符重载以平等的方式合并到一起。而当使用显式函数调用语法时，则进行常规的`unqualified name lookup`：

```c++
struct A {};
void operator+(A, A); // 用户定义的非成员 operator+

struct B {
    void operator+(B); // 用户定义的成员 operator+
    void f ();
};

A a;

void B::f() // B 的成员函数定义
{
    operator+(a,a); // 错误：在成员函数中的常规名字查找
                    // 找到了 B 的作用域中的 operator+ 的声明
                    // 并于此停下，而不会达到全局作用域
    a + a; // OK：成员查找找到了 B::operator+，非成员查找
           // 找到了 ::operator+(A,A)，重载决议选中了 ::operator+(A,A)
}
```

### 注入类名[^6]
> 在一个类作用域中，当前类的名称被当做公开的成员名一样对待：

```c++
int X;
struct X {
    void f() {
        X* p;   // OK ： X 指代注入类名，在"X"的作用域中，"X"被当做公开成员，注意体会这个"公开成员的含义"
        ::X* q; // 错误：在全局作用域"::"中，"struct X"被"int X"遮盖
    }
};
```

> 在类继承的过程中，受到继承限制的控制

```c++
namespace detail {
struct A {};
struct B {};
};

struct C : public detail::B, private detail::A {};

struct D : public C {
  A* a0;          // 错误：注入类名 A 受到“private”的修饰，变为非公开成员，不可访问
  detail::A* a1;  // OK：不使用注入类名
  B* b0;          // OK：通过注入类名
  detail::B* b1;  // OK：不使用注入类名
};
```

> 在类模板中，类似普通类情形一样，可以被注入，当做模板名或者类型名。有下列3个情形之一时，注入的名称被当做当前模板名：
> * 它后面紧跟随 `<` 符号（模板实例化标识）
> * 它被当做一个模板模板参数(`template template parameter`)
> * 它是友元类模板声明的详细类型指定符中的最后标识符。
> 此外其他情形，会被当做一个实际类类型被注入，其类型为该类模板实例化后的类型。

```c++
// 注意体会这个例子
template <template <class, class> class> struct A;

template<class T1, class T2>
struct X {
    X<T1, T2>* p;   // X后跟随"<"，则X被当做模板 template<class, class> struct X 对待
                    // 同理，此处改为 X<int, int> *p 也是成立的，因为X是一个模板
    using a = A<X>; // X被当做模板A的模板模板参数，则X被当做模板 template<class, class> struct X 对待
    template<class U1, class U2>
    friend class X; // X被当友元模板类的标识符，则X被当做模板 template<class, class> struct X 对待，即::X
                    // 此处含义为，当模板X实例化后，X<T1,T2>拥有友元X<U1,U2>，X<U1,U2>可以被实例化为多个类型
    X* q;           // 此处为以上三种情形之外，X被当做了一个实例化的类型名，其类型为X<T1,T2>。
                    // 当X<T1,T2>被实例为X<int,int>时，q的类型为`X<int,int> *`
                    // 当X<T1,T2>被实例为X<double,double>时，q的类型为`X<double,double> *`
};
```

### Argument-dependent lookup

依赖于实参的名字查找是C++程序设计语言中的名字查找机制之一。英文为“argument-dependent lookup”，因此缩写为ADL。ADL依据函数调用中的实参的数据类型查找未限定（unqualified）的函数名（或者函数模板名）。这也被称作“克尼格查找”（Koenig lookup），虽然安德鲁·克尼格并不是它的发明者。[^8]

如果通常的未限定（unqualified）名字查找所产生的候选集包括下述情形，则不会启动依赖于实参的名字查找：
* 类成员声明（此种情形仅指普通的类成员函数，不指类成员运算符函数）
* 块作用域内的函数的声明，不含(using-declaration)
* 任何不是函数或者函数模板的声明（例如函数对象或者另一个变量其名字与被查询的函数名字冲突）

**函数调用表达式**的每个实参的**类型**用于**确定**命名空间与类的相关集合（associated set of namespaces and classes）并用于函数名字查找（这句话的意思简而言之就是**ADL查找的集合范围如何确定**）：
1. 基本类型（fundamental type）实参的命名空间与类的相关集合为空。
```
如果参数的命名空间为空，将其添加到查找集合内，但是查找时，会直接跳过空集。
```
2. 类类型（class type，指struct, class, union类型）, 相关集合包括
   1. 类类型自身；
   2. 该类型的所有的直接或间接基类；
   3. 如果类类型 T 是另一个类 G 的成员（嵌套类型），则那个包含了类类型 T 的类 G；
   4. 该类类型的所有相关类的最内层外围命名空间。
   ```c++
#include <iostream>
namespace ADL {
struct A;
struct Base {
  friend void func2(const A &a) { std::cout << "2.2" << std::endl; }
};
struct A : public Base {
  friend void func1(A a) { std::cout << "2.1" << std::endl; }
  struct B {};
  friend void func3(B b) { std::cout << "2.3" << std::endl; }
};
void func4(A a) { std::cout << "2.4" << std::endl; }
};

int main() {
  ADL::A a;
  ADL::A::B b;
  func1(a);  // 2.1
  func2(a);  // 2.2
  func3(b);  // 2.3
  func4(a);  // 2.4
}
    ```
3. 如果实参是[类模板](https://zh.cppreference.com/w/cpp/language/class_template)特化后得到的类型，在上述规则外，还检验下列规则，并添加其关联类与命名空间到集合：
    1. 类型模板形参（type template parameter）所对应的模板实参的类型，不包括非类型的模板形参、模板模板形参；
    2. 模板模板实参（template template argument）所在的命名空间；
    3. 模板模板实参所在的类（如果这个类包含了这个成员模板）。
    ```c++
    ```
4. 对于枚举类型的实参，添加枚举定义于其中的命名空间到集合。如果枚举类型是类成员，则添加该类到集合。
5. 如果实参是类型 T 的指针或者是类型 T 的数组的指针，则检验类型 T 并添加其类与命名空间的关联集到集合。
6. 如果实参是函数类型，那么检验函数参数类型与函数返回值类型，并添加其类与命名空间的关联集到集合。
7. 如果实参是类 X 的成员函数 F 的指针类型参数，那么该成员函数的形参类型、该成员函数返回值的类型、该成员函数所属类 X 的相关集合都被加入到关联集到集合。
8. 如果实参是类 X 的数据成员 T 的指针类型参数，那么该成员类型、该数据成员所属类 X 的相关集合都被加入到关联集到集合。
9. 若参数是[重载函数集的取址表达式](https://zh.cppreference.com/w/cpp/language/overloaded_address)（或对函数模板）的名称，则检验重载集中的每个元素，并添加其类与命名空间的关联集到集合。
    1. 另外，若重载集为模板 id （带模板实参的模板名），则检验其所有类型模板实参与模板模板实参（但不含非类型模板实参），并添加其类与命名空间的关联集到集合。
10. 如果相关集合中的任何命名空间是[内联命名空间](https://zh.cppreference.com/w/cpp/language/namespace)（inline namespace）, 则添加其外围命名空间到关联集合。
```c++
#include <iostream>
namespace adl {
inline namespace inner {
struct A {};
}

void func(const inner::A &a) {
  std::cout << "enclosed namespace added." << std::endl;
}
}

int main() {
  adl::inner::A a;
  func(a); // 因为 adl::inner 为 inline namespace，则将其最内存外围 namespace adl
           // 到查找关联集合中，则可以查找到 adl::func
}
```
11. 如果相关集合中的一个命名空间直接包含了内联命名空间，则内联命名空间被增加到相关集合中。
```c++
#include <iostream>
namespace adl {
struct A {};
inline namespace inner {
void func(const A &a) { std::cout << "inline namespace added." << std::endl; }
}
}

int main() {
  adl::A a;
  func(a); // namespace adl中包含 inline namespace adl::inner，
           // 则将 adl::inner 添加到查找关联集合中，则可以查找到 adl::inner::func
}
```
12. 在确定命名空间与类的关联集后，为了进一步的 ADL 处理，忽略此集中所有于类中找到的声明，除了命名空间作用域的友元函数及函数模板，陈述于后述点2。以下列特殊规则，合并普通[无限定查找](https://zh.cppreference.com/w/cpp/language/lookup)找到的声明集合，与在 ADL 所生成关联集的所有元素中找到的声明集合:
    1. 忽略关联命名空间中的 [using 指令](https://zh.cppreference.com/w/cpp/language/namespace#using_.E6.8C.87.E4.BB.A4)
    2. 声明于关联类中的命名空间作用域友元函数（及函数模板）通过 ADL 可见，即使它们通过普通查找不可见。
    3. 忽略函数与函数模板外的所有名称（与变量不冲突）

## 模板的lookup

> 对于在模板的定义中所使用的非依赖名[^7]，当检查该模板的定义时将进行无限定的名字查找。在这个位置与声明之间的绑定并不会受到在实例化点可见的声明的影响。而对于在模板定义中所使用的依赖名，其查找则推迟到得知其模板实参之时。此时，ADL[^5] 将同时在模板的定义语境和在模板的实例化语境中检查可见的具有外部连接的 (C++11 前)函数声明，而非 ADL 的查找则只检查在模板的定义语境中可见的具有外部连接的 (C++11 前)函数声明。（换句话说，在模板定义之后添加新的函数声明，除非通过 ADL 否则仍时不可见的。）如果在 ADL 查找所检查的命名空间中，在某个别的翻译单元中声明了一个具有外部连接的更好的匹配声明，或者如果当同样检查这些翻译单元时其查找会导致歧义，则其行为是未定义的。无论哪种情况，如果某个基类取决于某个模板形参，则无限定名字查找不会检查它的作用域（在定义点和实例化点都不会）。

```c++
void f(char); // f 的第一个声明

template<class T>
void g(T t) {
    f(1);    // 非依赖名：名字查找找到了 ::f(char) 并于此时绑定
    f(T(1)); // 依赖名：查找推迟
    f(t);    // 依赖名：查找推迟
//  dd++;    // 非依赖名：名字查找未找到声明
}

enum E { e };
void f(E);   // f 的第二个声明
void f(int); // f 的第三个声明
double dd;

void h() {
    g(e);  // 实例化 g<E>，此处
           // 对 'f' 的第二次和第三次使用
           // 进行查找并找到了 ::f(char)（常规查找）和 ::f(E)（ADL）
           // 然后重载解析选择了 ::f(E)。
           // 这调用了 f(char)，然后两次调用 f(E)
    g(32); // 实例化 g<int>，此处
           // 对 'f' 的第二次和第三次使用
           // 进行了查找仅找到了 ::f(char)
           // 然后重载解析选择了 ::f(char)
           // 这三次调用了 f(char)
}

typedef double A;
template<class T> class B {
   typedef int A;
};
template<class T> struct X : B<T> {
   A a; // 对 A 的查找找到了 ::A (double)，而不是 B<T>::A
};
```

# 参考

[^1]: [Identifiers](https://en.cppreference.com/w/cpp/language/identifiers)
[^2]: [Name lookup](https://en.cppreference.com/w/cpp/language/lookup)
[^3]: [Qualified name lookup](https://en.cppreference.com/w/cpp/language/qualified_lookup)
[^4]: [Unqualified name lookup](https://en.cppreference.com/w/cpp/language/unqualified_lookup)
[^5]: [Argument-dependent lookup](https://en.cppreference.com/w/cpp/language/adl)
[^6]: [Injected Class Name](https://en.cppreference.com/w/cpp/language/injected-class-name)
[^7]: [Dependent Names](https://en.cppreference.com/w/cpp/language/dependent_name)
[^8]: [依赖于实参的名字查找](https://zh.wikipedia.org/wiki/依赖于实参的名字查找)