《Effective Modern C++》笔记。  
仅供个人学习  
请lyk坚持记录

[TOC]


* [一，Deducing Types (类型推断)](#一deducing-types-类型推断)  
  -[Item 1: 理解模板类型推断](#item-1理解模板类型推断)    
  -[Item 2: auto 语句中的类型推断](#item-2auto语句中的类型推断)   
  -[Item 3: 理解 decltype](#item-3理解decltype)  
  -[Item 4: 如何观测实际的类型推断](#Item-4如何观测实际的类型推断)  
* [二，Auto](#二auto)  
  -[Item 5: 多用 auto 替换显示类型声明](#Item-5多用auto替换显式类型声明)  
  -[Item 6: 用显示类型声明来避免 auto 的不合适推断](#Item-6用显示类型声明来避免-auto-的不合适推断)  
* [三，Moving to Modern C++](#三moving-to-modern-c)    
  -[Item 7: 辨别生成对象时 () 和 {} 的不同](#item-7辨别生成对象时和的不同)    
  -[Item 8: 多用 nullptr 代替 0 和 null](#item-8多用-nullptr-代替-0-和-null)    
  -[Item 9: 多用别名定义而不是 typedefs](#item-9多用-alias-declarations-而不是-typedefs)    
  -[Item 10: 多用 scoped enums 而不是 unscoped enums](#item-10多用-scoped-enums-而不是-unscoped-enums)    
  -[Item 11: 使用 deleted 函数而不是 private undefined 函数](#item-11使用-deleted-函数而不是-private-undefined-函数)    
  -[Item 12: 将重写的函数声明为 override](#item-12将重写的函数声明为-override)

  
# 一，Deducing Types (类型推断)
## Item 1:理解模板类型推断

**模板函数声明中的类型**

```c++
template<typename T>
void f(ParamType param);

int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int
```

ParamType为普通引用,即 ParamType 为 T&
```c++
template<typename T>
void f(T& param);
```
```c++
void f(T& param);
f(x); // T is int, param's type is int&
f(cx); // T is const int,
       // param's type is const int&
f(rx); // T is const int,
       // param's type is const int&
```

ParamType 为 const T&
```c++
void f(const T& param);
f(x);  // T is int, param's type is const int&
f(cx); // T is int, param's type is const int&
f(rx); // T is int, param's type is const int&
```

ParamType 为右值引用 T&&，
```C++
void f(T&& param);
f(x); // x is lvalue, so T is int&,
      // param's type is also int&
f(cx); // cx is lvalue, so T is const int&,
       // param's type is also const int&
f(rx); // rx is lvalue, so T is const int&,
       // param's type is also const int&
f(27); // 27 is rvalue, so T is int,
       // param's type is therefore int&&
```

在《C++ Primer》16.2.5中介绍了这种情况：“当我们将一个左值传递给函数的优质引用参数，且此右值引用指向模板类型参数（如T&&）时，编译器推断模板类型参数为实参的左值引用类型。”因此f(x)中T为int&， f(cx)中T为const int&。同时，“我们不能（直接）定义一个引用的引用”，这时有另外的一个规则，叫**引用折叠**
```
引用折叠:
- X& &、X& &&和X&& &都折叠成类型X&
- 类型X&& &&折叠成X&&
- 当传入的是左值时，类型推断会推向左值引用；若传入的是右值才推为右值引用。
```

ParamType 是值，T
```c++
void f(T param);
f(x);  // T's and param's types are both int
f(cx); // T's and param's types are again both int
f(rx); // T's and param's types are still both int
```
会将传入的值去引用，并去const。

**当传入数组会怎么样？** 是的，聪明的你会发现下面的格式是错的
```c++
void myFunc(int param[]);
```
在数组名字单独出现时，往往被视作一个指针
```c++
void myFunc(int* param);
```
然而我们可以声明一个参数是对一个数组的引用。设name是一个长度为13的const char[]。
```c++
template<typename T>
void f(T& param); // template with by-reference parameter

f(name); // pass array to f
```
T会被推断为const char [13]，f的参数是const char (&)[13]。  
进一步的，如果想知道一个数组的大小，可以像下面这样
```c++
template<typename T, std::size_t N> 
constexpr std::size_t arraySize(T (&)[N]) noexcept 
{
	return N; 
} // 注意以后会提到的constexpr & noexcept
```

最后给出书中的tips。

**Things to Remember**
* During template type deduction, arguments that are references are treated as
non-references, i.e., their reference-ness is ignored.
* When deducing types for universal reference parameters, lvalue arguments get
special treatment.
* When deducing types for by-value parameters, const and/or volatile arguments
are treated as non-const and non-volatile.
* During template type deduction, arguments that are array or function names
decay to pointers, unless they’re used to initialize references.


**存在的疑问：**
1，什么是Universal reference?
2，什么样的参数为volatile？

## Item 2:auto语句中的类型推断
如何推断下面的代码中 auto 分别代表了什么类型呢？
```c++
auto x = 27;
const auto cx = x;
const auto& rx = x;
```
一般情况下，可以用模板函数中的类型推断来判断auto的结果。如：
```c++
template<typename T> 			// conceptual template for
void func_for_x(T param); // deducing x's type
func_for_x(27); 

template<typename T> 						 // conceptual template for
void func_for_cx(const T param); // deducing cx's type
func_for_cx(x);

template<typename T> 							// conceptual template for
void func_for_rx(const T& param); // deducing rx's type
func_for_rx(x);
```
根据 Item 1，我们可以将模板的类型推断分为三种：  
* Case 1: The type specifier is a pointer or reference, but not a universal reference.  
* Case 2: The type specifier is a universal reference.  
* Case 3: The type specifier is neither a pointer nor a reference.  

这样，我们可以很自然地推断出下面的例子 auto 具体指代的内容：
```c++
auto x = 27; // case 3 (x is neither ptr nor reference)
const auto cx = x; // case 3 (cx isn't either)
const auto& rx = x; // case 1 (rx is a non-universal ref.)

auto&& uref1 = x; // x is int and lvalue,
                  // so uref1's type is int&
auto&& uref2 = cx; // cx is const int and lvalue,
                   // so uref2's type is const int&
auto&& uref3 = 27; // 27 is int and rvalue,
                   // so uref3's type is int&&
```

同样的，对于数组、函数，也有引用和非引用的区别
```c++
const char name[] = "R. N. Briggs";  // name's type is const char[13]

auto arr1 = name;   // arr1's type is const char*
auto& arr2 = name;  // arr2's type is const char (&)[13]

void someFunc(int, double); // someFunc is a function;
                            // type is void(int, double)
auto func1 = someFunc; // func1's type is void (*)(int, double)
auto& func2 = someFunc; // func2's type is void (&)(int, double)
```

如果用 auto 的对象是初始化列表，推断的结果也是初始化列表
```c++
int x = {27};
auto y = {27}; // type is std::initializer_list<int>
```
这也是 auto 和模板类型推断会不同的唯一地方：
```c++
auto x = { 11, 23, 9 }; // x's type is
                        // std::initializer_list<int>
template<typename T> // template with parameter
void f(T param);  // declaration equivalent to
                  // x's declaration
f({ 11, 23, 9 }); // error! can't deduce type for T
```
只能像下面这样用模板推断 initializer_list:
```c++
template<typename T>
void f(std::initializer_list<T> initList);
f({ 11, 23, 9 }); // T deduced as int, and initList's
                  // type is std::initializer_list<int>
```
另外，虽然C++14允许使用 'auto' 来推断函数返回类型或 lambda 函数参数，但这种用法是使用的模板类型推断。因此，若返回的是初始化列表并不能编译：
```c++
auto createInitList()
{
return { 1, 2, 3 }; // error: can't deduce type
}                   // for { 1, 2, 3 }

auto resetV = [&v](const auto& newValue) { v = newValue; }; // C++14
resetV({ 1, 2, 3 }); // error! can't deduce type for { 1, 2, 3 }
```
**Things to Remember**
* auto type deduction is usually the same as template type deduction, but auto
type deduction assumes that a braced initializer represents a std::initial
izer_list, and template type deduction doesn’t.
* auto in a function return type or a lambda parameter implies template type
deduction, not auto type deduction.

## Item 3:理解decltype

先来一些 no surprise 的例子
```c++
const int i = 0;         // decltype(i) is const int
bool f(const Widget& w); // decltype(w) is const Widget&
                         // decltype(f) is bool(const Widget&)
struct Point {
  int x, y;              // decltype(Point::x) is int
};                       // decltype(Point::y) is int
Widget w;                // decltype(w) is Widget
if (f(w)) …              // decltype(f(w)) is bool

template<typename T>     // simplified version of std::vector
class vector {
public:
  …
  T& operator[](std::size_t index);
  …
};
vector<int> v;           // decltype(v) is vector<int>
…
if (v[0] == 0) …         // decltype(v[0]) is int&
```

在C++11中，decltype 主要用于函数返回类型(return type)基于参数类型(parameter types)的情况：
```c++
template<typename Container, typename Index> // works, but
auto authAndAccess(Container& c, Index i)    // requires
-> decltype(c[i])                            // refinement
{
  authenticateUser();
  return c[i];
}
```
在C++14中可以直接使用 auto 作为返回类型：
```c++
template<typename Container, typename Index> // C++14;
auto authAndAccess(Container& c, Index i)    // not quite
{                                            // correct
  authenticateUser();
  return c[i]; // return type deduced from c[i]
}
```

值得注意的是，尽管 c[i] 一般返回的是引用类型，但由于模板类型推断是去引用的，所以下面的代码是错误的。
```c++
std::deque<int> d;
…
authAndAccess(d, 5) = 10; // authenticate user, return d[5],
                          // then assign 10 to it;
                          // this won't compile!
```
这里，函数 authAndAccess 返回的是一个右值(rvalue)，C++ 中给右值赋值是被禁止的。  

C++14可以 decltype(auto) 来实现返回与c[i]一致的类型：
```c++
template<typename Container, typename Index> // C++14; works,
decltype(auto)                               // but still
authAndAccess(Container& c, Index i)         // requires
{                                            // refinement
authenticateUser();
return c[i];
}
```
decltype(auto)也可以用在变量的声明中：
```c++
Widget w;
const Widget& cw = w;
auto myWidget1 = cw; // auto type deduction:
                     // myWidget1's type is Widget
decltype(auto) myWidget2 = cw; // decltype type deduction:
                               // myWidget2's type is
                               // const Widget&
```

decltype 的 surprise 出现在右值容器上。用户可能只想得到临时容器中的拷贝，如：
```c++
std::deque<std::string> makeStringDeque(); // factory function
// make copy of 5th element of deque returned
// from makeStringDeque
auto s = authAndAccess(makeStringDeque(), 5);
```
这时用decltype(auto)就会得到对容器元素的引用，这是并不安全的。为了让 authAndAccess 能同时接受左值和右值，需要用上Universal reference：
```c++
template<typename Container, typename Index> // c is now a
decltype(auto) authAndAccess(Container&& c, // universal
                             Index i); // reference
```
书上还提到了forward，虽然不太了解还是放上来吧：
```c++
template<typename Container, typename Index> // final
decltype(auto)                               // C++14
authAndAccess(Container&& c, Index i)        // version
{
authenticateUser();
return std::forward<Container>(c)[i];
}

template<typename Container, typename Index> // final
auto                                         // C++11
authAndAccess(Container&& c, Index i)        // version
-> decltype(std::forward<Container>(c)[i])
{
authenticateUser();
return std::forward<Container>(c)[i];
}
```

变量名是个左值，对一个变量名字使用 decltype，会得到该变量的 declared type。但对于比变量名字更复杂的**左值表达式**，decltype 总是会得到一个左值引用。比如下面的这个**奇怪**的例子：
```c++
decltype(auto) f1()
{
int x = 0;
…
return x; // decltype(x) is int, so f1 returns int
}

decltype(auto) f2()
{
int x = 0;
…
return (x); // decltype((x)) is int&, so f2 returns int&
}
```
x 是一个类型为 int 的变量名，所以 f1 是 int 类型的。但 (x) 是比名字更复杂的左值表达式，所以 f2 是 int & 类型的。

**Things to Remember**
* decltype almost always yields the type of a variable or expression without
any modifications.
* For lvalue expressions of type T other than names, decltype always reports a
type of T&.
* C++14 supports decltype(auto), which, like auto, deduces a type from its
initializer, but it performs the type deduction using the decltype rules.

## Item 4：如何观测实际的类型推断
书中给出了三种方法：IDE 自动推断（getting type deduction information as you edit your code），编译中获取（getting it during compilation）和在运行中获取（getting it at runtime）。

Visual Studio 2015可以对相对简单的类型进行推断:
```c++
const int theAnswer = 42;
auto x = theAnswer;  //把鼠标放在auto上就能看到推断的类型
auto y = &theAnswer;
```

在代码编译时，若出现 error，会给出相关的信息，这也能得到推断的类型，如：
```c++
template<typename T> // declaration only for TD;
class TD;            // TD == "Type Displayer"
TD<decltype(x)> xType; // elicit errors containing
TD<decltype(y)> yType; // x's and y's types
```
由于TD并没有定义，会出现如下的错误：  
error: aggregate 'TD<**int**> xType' has incomplete type and
       cannot be defined
error: aggregate 'TD<**const int \***> yType' has incomplete type
       and cannot be defineds

在函数运行时，你也可以使用函数typeid：
```C++
std::cout << typeid(x).name() << '\n'; 
std::cout << typeid(*x).name() << '\n'; 
```

然而，这些方法都或多或少有些问题，如VS中的类型推断可能不是很直白：
```C++
const std::_Simple_types<std::_Wrap_alloc<std::_Vec_base_types<Widget,
std::allocator<Widget> >::_Alloc>::value_type>::value_type *
```
有个Boost TypeIndex library（http://boost.com )，可以在std::type_info::name 和 IDEs没那么好用的情况下使用。

**Things to Remember**
* Deduced types can often be seen using IDE editors, compiler error messages,
and the Boost TypeIndex library.  
* The results of some tools may be neither helpful nor accurate, so an understanding
of C++’s type deduction rules remains essential.  

# 二，auto
## Item 5:多用auto替换显式类型声明
auto有很多好处：
### 可以避免未定义的情况：
```c++
int x1; // potentially uninitialized
auto x2; // error! initializer required
auto x3 = 0; // fine, x's value is well-defined
```
### 可以避免繁琐的类型声明，且可以随参数改变：
```c++
template<typename It> // 设It是一个迭代器类型
void dwim(It b, It e)
{
  while (b != e) {
    auto currValue = *b;
    …
  }
}
```
在C++14中，还可以在lambda表达式中使用**auto**：
```c++
auto derefUPLess =                      // comparison func.
  [](const std::unique_ptr<Widget>& p1, // for Widgets
  const std::unique_ptr<Widget>& p2)    // pointed to by
{ return *p1 < *p2; };                  // std::unique_ptrs

auto derefLess =       // C++14 comparison
  [](const auto& p1,   // function for
  const auto& p2)      // values pointed
{ return *p1 < *p2; }; // to by anything pointer-like
```

书中提到了std::function 对象：在C++11标准库中，std::function 是一个产生类似于函数指针的模板。与函数指针不同的是，函数指针只能指向函数，而std::function 可以指向所有的可调用对象。如：
```c++
std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)> func;

std::function<bool(const std::unique_ptr<Widget>&,
                   const std::unique_ptr<Widget>&)>
derefUPLess = [](const std::unique_ptr<Widget>& p1,
                 const std::unique_ptr<Widget>& p2)
{ return *p1 < *p2; };
```

由于 std::function 是一个模板，所以它会比 auto 使用更多的空间。  

### auto 可以避免不合适的类型声明

比如下面的代码：
```c++
std::vector<int> v;
…
unsigned sz = v.size();
```

一般情况下这没有问题，然而 **unsigned** 和 **std::vector&lt;int&gt;::size_type** 并不完全一致。  

在32位的Windows系统中，两者一致；在64位的 Windows 系统中，**unsigned** 是32位的，而 **std::vector&lt;int&gt;::size_type** 是64位的,从而可能导致错误。  

再看下面的例子：
```c++
std::unordered_map<std::string, int> m;
…
for (const std::pair<std::string, int>& p : m)
{
… // do something with p
}
```
是不是一眼看上去并没有错误？然而在 std::unordered_map 中的 key 是 **const** 类型的，也就是说 pair 应该为 **std::pair&lt;const std::string, int&gt;**。使用 auto 就能避免这种错误的类型声明。  

**Things to Remember**
* auto variables must be initialized, are generally immune to type mismatches
that can lead to portability or efficiency problems, can ease the process of
refactoring, and typically require less typing than variables with explicitly
specified types.
* auto-typed variables are subject to the pitfalls described in Items 2 and 6.

## Item 6:用显示类型声明来避免 auto 的不合适推断  

有意思的是，瞎用auto也会导致错误，**std::vector&lt;bool&gt;** 就是一个典型的例子：

```c++
std::vector<bool> features(const Widget& w);
Widget w;
…
bool highPriority = features(w)[5]; // is w high priority?
processWidget(w, highPriority); // process w in accord with its priority

auto highPriority_auto = features(w)[5];
processWidget(w, highPriority_auto); // undefined behavior!
```

一般来说，std::vector::operator[] 会返回相应的引用类型，唯独除了 bool 。这是由于C++在实现 vector&lt;bool&gt; 时是根据二进制数某位上的值来得到各个value的，而C++禁止对位的引用，所以 operator[] 会返回 std::vector&lt;bool&gt;::reference 作为替代，它使用起来就和 bool& 类似。然而，在上面的代码中，我们想要的是**bool**类型而不是 reference ，所以这里应该使用显式的 bool 类型声明。

更进一步的，仔细看上面这段代码，feature(w) 产生了一个临时变量，而 feature(w)[5] 是对这个临时变量的 vector&lt;bool&gt;::reference 。因此，highPriority_auto 实际包含了对这个临时变量的元素的指针，这导致在下一句中， highPriority_auto 包含了一个空指针，从而导致严重错误。

**Proxy Class**  
Proxy class: a class that exists for the purpose of emulating and augmenting the behavior of some other type.
Proxy 类的目的模仿或改进某个类型的行为。如 std::vector&lt;bool&gt;::reference 就是一个**proxy class**，用来模仿 vector&lt;bool&gt;的operator[]，让其看起来像是返回了一个bool&。  
智能指针 std::shared_ptr 和 std::unique_ptr 等也是 Proxy class，为了改进 生指针(Raw pointer) 在资源管理上的行为。  
说白了，proxy class 的作用就是产生一个代理(proxy)，若对含有 proxy class 的代码使用 auto ，很可能会得到 proxy class的类型而不是**看上去**直接的类型。

如果非要用 auto 也不是不行：
```c++
auto highPriority = static_cast<bool>(features(w)[5]);
```

**Things to Remember**
* “Invisible” proxy types can cause auto to deduce the “wrong” type for an initializing
expression.
* The explicitly typed initializer idiom forces auto to deduce the type you want
it to have.

# 三，Moving to Modern C++
## Item 7:辨别生成对象时()和{}的不同

当一个类没有以 initializer_list 为参数的构造函数时，使用()和{}并无二致：
```c++
class Widget {
public:
  Widget(int i, bool b); // ctors not declaring
  Widget(int i, double d); // std::initializer_list params
…
};
Widget w1(10, true); // calls first ctor
Widget w2{10, true}; // also calls first ctor
Widget w3(10, 5.0); // calls second ctor
Widget w4{10, 5.0}; // also calls second ctor
```
但若有了以 initializer_list 为参数的构造函数，使用{}会优先匹配对应的构造函数：
```c++
class Widget {
public:
  Widget(int i, bool b); // as before
  Widget(int i, double d); // as before
  Widget(std::initializer_list<long double> il); // added
…
};

Widget w1(10, true); // uses parens and, as before,
                     // calls first ctor
Widget w2{10, true}; // uses braces, but now calls
                     // std::initializer_list ctor
                     // (10 and true convert to long double)
Widget w3(10, 5.0); // uses parens and, as before,
                    // calls second ctor
Widget w4{10, 5.0}; // uses braces, but now calls
                    // std::initializer_list ctor
                    // (10 and 5.0 convert to long double)
```
这里可以看到，true 和 10 等都被隐式类型转换成了 long double，可见 initializer_list 的构造函数的优先级非常高。  
它的优先级甚至高于了移动构造函数：
```C++
class Widget {
public:
	Widget(int i, bool b); // as before
	Widget(int i, double d); // as before
	Widget(std::initializer_list<long double> il); // as before
	operator float() const; // convert
	… // to float
};

Widget w5(w4); // uses parens, calls copy ctor
Widget w6{w4}; // uses braces, calls
               // std::initializer_list ctor
               // (w4 converts to float, and float
               // converts to long double)
Widget w7(std::move(w4)); // uses parens, calls move ctor
Widget w8{std::move(w4)}; // uses braces, calls
                          // std::initializer_list ctor
                          // (for same reason as w6)
```
**待编辑：说实话，这里并不太懂，w4会直接使用operator()?**

```c++
class Widget {
public:
  Widget(int i, bool b); // as before
  Widget(int i, double d); // as before
  Widget(std::initializer_list<bool> il); // element type is
                                          // now bool
  … // no implicit
}; // conversion funcs

Widget w{10, 5.0}; // error! requires narrowing conversions
```
从上面这例可以看出，编译器非常倾向于使用 std::initializer_list 的构造函数，从而使得正确匹配参数的构造函数并不会被使用。另外还有个小知识点：{}初始器中，缩小范围的隐式类型转换会报错。  
只有在无法进行类型转换的情况下，编译器才会考虑其他的构造函数：
```c++
class Widget {
public:
  Widget(int i, bool b); // as before
  Widget(int i, double d); // as before
// std::initializer_list element type is now std::string
  Widget(std::initializer_list<std::string> il);
  … // no implicit
}; // conversion funcs

Widget w1(10, true); // uses parens, still calls first ctor
Widget w2{10, true}; // uses braces, now calls first ctor
Widget w3(10, 5.0); // uses parens, still calls second ctor
Widget w4{10, 5.0}; // uses braces, now calls second ctor
```

那么，当初始化是使用空的 {} 会怎么样呢？是空的 initializer_list 还是调用默认构造函数？结论是**调用默认构造函数**。当然了，如果你想使用空的 ()，那会变成调用一个对应名字的函数了：
```c++
class Widget {
public:
  Widget(); // default ctor
  Widget(std::initializer_list<int> il); // std::initializer_list ctor
  … // no implicit
}; // conversion funcs

Widget w1; // calls default ctor
Widget w2{}; // also calls default ctor
Widget w3(); // most vexing parse! declares a function!
```

如果真的想调用空的 initializer_list，需要两重括号：
```c++
Widget w4({}); // calls std::initializer_list ctor
               // with empty list
Widget w5{{}}; // ditto
```

对于模板编写者来说，若想自己定义对于 () 中参数的反应，可以像如下代码这样：
```c++
template<typename T, // type of object to create
	typename... Ts> // types of arguments to use
void doSomeWork(Ts&&... params)
{
  T localObject1(std::forward<Ts>(params)...); // using parens  
  // 或者像下面这样
  T localObject2{std::forward<Ts>(params)...}; // using braces
…
}
```
这样，对于下面的代码，localObject1 会是一个由10个元素组成的 vector，而 localObject2 是由2个元素组成的 vector。
```c++
doSomeWork<std::vector<int>>(10, 20);
```

**Things to Remember**
* Braced initialization is the most widely usable initialization syntax, it prevents
narrowing conversions, and it’s immune to C++’s most vexing parse.
* During constructor overload resolution, braced initializers are matched to
std::initializer_list parameters if at all possible, even if other constructors
offer seemingly better matches.
* An example of where the choice between parentheses and braces can make a
significant difference is creating a std::vector&lt;numeric type&gt; with two
arguments.
* Choosing between parentheses and braces for object creation inside templates
can be challenging.

## Item 8:多用 nullptr 代替 0 和 null
当 C++ 发现一个需要用指针的地方出现了0时，它通常会把0看作是一个空指针。然而，C++主要的原则还是将0看作是一个int。NULL 也类似，编译允许在一些情况下将 NULL 当成一个整数（如 long）。
在 C++98 中，使用0和 NULL 的重载可能会有一些意外：
```c++
void f(int); // three overloads of f
void f(bool);
void f(void*);

f(0); // calls f(int), not f(void*)
f(NULL); // might not compile, but typically calls
         // f(int). Never calls f(void*)
```
如果 NULL 的定义是 0L，那么编译的结果可能是模糊的，因为从 long 到 int 和从 0L 到 void* 是同样优先的。  
所以，之所以推荐使用 nullptr，就是因为 nullptr 没有 int 类型，从而可以把它看成是一个可以代表所有类型的空指针。
```c++
f(nullptr); // calls f(void*) overload
```
在 cppreference 网站上找到个有意思的例子：
```c++
template<class F, class A>
void Fwd(F f, A a)
{
    f(a);
}
 
void g(int* i)
{
    std::cout << "Function g called\n";
}
 
int main()
{
    g(NULL);           // Fine
    g(0);              // Fine
 
    Fwd(g, nullptr);   // Fine
//  Fwd(g, NULL);  // ERROR: No function g(int)
}
```
这个例子有意思在哪呢？可以知道，如果单纯的调用函数 g:
```c++
g(0);
g(NULL);
```
是不会出错的，0 和 NULL 都被转换成了空指针；然而模板中 0 和 NULL 都被推断为 int 类型，从而导致了错误。这种错误排查起来可是很令人头疼的。

**Things to Remember**
* Prefer nullptr to 0 and NULL.
* Avoid overloading on integral and pointer types.

## Item 9:多用 alias declarations 而不是 typedefs
当定义较长且常用时，C++11 以前一般用 typedef 来简化代码：
```C++
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS;
```
C++11 提供了 alias declarations：
```C++
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>;
```
他们俩提供了相同的作用。那么为什么还提倡使用 alias declarations 呢？首先 alias declarations 在涉及到函数指针时更直观：
```c++
// FP is a synonym for a pointer to a function taking an int and
// a const std::string& and returning nothing
typedef void (*FP)(int, const std::string&); // typedef

// same meaning as above
using FP = void (*)(int, const std::string&); // alias declaration
```

其次， alias declarations 可以模板化，而 typedef 不能：
```c++
template<typename T> // MyAllocList<T>
using MyAllocList = std::list<T, MyAlloc<T>>; // is synonym for std::list<T, MyAlloc<T>>
MyAllocList<Widget> lw; // client code
```
```c++
template<typename T> // MyAllocList<T>::type
struct MyAllocList {                     // is synonym for
  typedef std::list<T, MyAlloc<T>> type; // std::list<T,
};                                       // MyAlloc<T>>
MyAllocList<Widget>::type lw; // client code
```
当使用 typedef 时，如果我们想用 MyAllocList 在模板里创建一个对象，需要使用 typename:
```c++
template<typename T>
class Widget { // Widget<T> contains
private: // a MyAllocList<T>
  typename MyAllocList<T>::type list; // as a data member
…
};
```
在这里，MyAllocList&lt;T&gt;::type 是一个依赖于参数T的依赖类型(dependent type)，而C++要求依赖类型必须使用 typename。  
而若是使用 alias template，代码会明快很多：
```c++
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>; // as before

template<typename T>
class Widget {
private:
  MyAllocList<T> list; // no "typename",
… // no "::type"
};
```

之所以会这样，是因为使用 alias template 的 MyAllocList<T> 必定是一个类型，而使用 typedef 的 MyAllocList<T>::type 并不一定是类型，所以后者需要使用 typename 来保证编译不会出错。作者在这顺带吐槽了一下：“That sounds
crazy, but don’t blame compilers for this possibility. It’s the humans who have been
known to produce such code.”

在 C++11 中，有个叫 type_traits 的标准库，提供了允许用户得到不同的参数类型：
```c++
std::remove_const<T>::type // yields T from const T
std::remove_reference<T>::type // yields T from T& and T&&
std::add_lvalue_reference<T>::type // yields T& from T
```
注意这里都使用了 ::type。没错，这是 C++11 的，之所以不用 alias template 是由于一些历史原因。在 C++14 中，alias template 的 type_traits 也被加上了:
```c++
std::remove_const<T>::type // C++11: const T → T
std::remove_const_t<T> // C++14 equivalent
std::remove_reference<T>::type // C++11: T&/T&& → T
std::remove_reference_t<T> // C++14 equivalent
std::add_lvalue_reference<T>::type // C++11: T → T&
std::add_lvalue_reference_t<T> // C++14 equivalent
```
“The C++11 constructs remain valid in C++14, but I don’t know why you’d want to use them.”

当然了，如果你想自己实现 C++14 类似的 alias templates，可以像下面这样做：
```C++
template <class T>
using remove_const_t = typename remove_const<T>::type;

template <class T>
using remove_reference_t = typename remove_reference<T>::type;

template <class T>
using add_lvalue_reference_t = typename add_lvalue_reference<T>::type;
```

**Things to Remember**
* typedefs don’t support templatization, but alias declarations do.
* Alias templates avoid the “::type” suffix and, in templates, the “typename”
prefix often required to refer to typedefs.
* C++14 offers alias templates for all the C++11 type traits transformations.

## Item 10:多用 scoped enums 而不是 unscoped enums

在 C++98 有 enum 类型，它的用法是这样的：
```c++
enum Color { black, white, red };
```
然而这样使用会有隐患，比如若在这个定义的下面加上这样一句，就会出错：
```c++
auto white = false;
```
这是由于 enum 里面的变量是包括在整个和 Color 一样的参数空间里的，即 Color 在哪有效，这些 black 等就在哪有效。 这样的范围没有被限制的 enums 被称作 unscoped enums。 为了避免这种隐患，C++11 引入了 scoped enums:
```c++
enum class Color { black, white, red }; // black, white, red
                                        // are scoped to Color
auto white = false; // fine, no other "white" in scope
Color c = white; // error! no enumerator named
                 // "white" is in this scope
Color c = Color::white; // fine
auto c = Color::white; // also fine (and in accord
                       // with Item 5's advice)
```
Scoped enums 也被称为 enum classes。  
除了作用范围以外，unscoped enums 和 scoped enums 还有类型转换上的差距：
```c++
enum Color { black, white, red }; // unscoped enum
vector<size_t> primeFactors(size_t x);
Color c = red;
...
if (c < 14.5) { // compare Color to double (!)
  auto factors = primeFactors(c);// compute prime factors of a Color (!)
…
}
```
在 unscoped enums 里面的 enumerators 是被隐式转换为整数类型的，所以上面的代码不会出错。但是 scoped enums 是没有这种隐式类型转换的：
```c++
enum class Color { black, white, red }; // enum is now scoped
Color c = Color::red; // as before, but
…                     // with scope qualifier
if (c < 14.5) { // error! can't compare
                // Color and double
  auto factors = primeFactors(c); // error! can't pass Color to
                                  // function expecting std::size_t
…
}
```
若真的想用，得用显示类型转换：
```c++
if (static_cast<double>(c) < 14.5) { // odd code, but
                                     // it's valid
  auto factors = primeFactors(static_cast<std::size_t>(c)); // suspect, but it compiles
…
}
```
在 C++98，是不允许 enum 提前声明(forward-declared)的，因为编译器要根据 enum 里面 enumerators 的数量来决定用什么类型来表示这个 enum，从而提高程序的**空间使用效率/运行速度**。在 C++11 中允许了 forward-declared，这是因为 C++11 默认了 enumerators 的类型。想自己修改也行：
```c++
enum class Status; // underlying type is int
enum class Status: std::uint32_t; // underlying type for
                                  // Status is std::uint32_t
                                  // (from <cstdint>)
enum class Status: std::uint32_t { 
	good = 0,
  failed = 1,
  incomplete = 100,
  corrupt = 200,
  audited = 500,
  indeterminate = 0xFFFFFFFF
};

enum Color: std::uint8_t;
```

unscoped enums 也不是一无是处，它 space leaking 的特点也具有一定的价值：
```c++
using UserInfo =          // type alias; see Item 9
  std::tuple<
    std::string, // name
    std::string,            // email
    std::size_t> ;          // reputation
UserInfo uInfo; // object of tuple type
…

auto val = std::get<1>(uInfo); // get value of field 1
// 显然这里使用 get<1> 是想要得到 email 地址，然而这并不直观并且不易维护
// 利用 unscoped enums 可以改变这点


enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo; // as before
…
auto val = std::get<uiEmail>(uInfo); // ah, get value of
                                     // email field
```
注意上面的代码使用了 unscoped enums 的隐式类型转换。  
如果在这里想用使用 scoped enums 也行：
```c++
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo; // as before
…
auto val =
  std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);
```
但这显得有点繁杂，那是不是可以用一个返回 size_t 的函数代替 static_cast 呢？这是一个 tricky 的地方：std::get< > 是一个模板，尖括号里面的值的类型需要在编译时就能知道，否则编译器不能将模板实例化，从而导致编译失败。因此我们需要一个 constexpr 函数模板，它可以返回任意类型的 enum：
```c++
template<typename E>
constexpr typename std::underlying_type<E>::type
  toUType(E enumerator) noexcept
{
  return static_cast<typename std::underlying_type<E>::type>(enumerator);
}
```
C++14 可以改成：
```c++
template<typename E> // C++14
constexpr std::underlying_type_t<E>
  toUType(E enumerator) noexcept
{
  return static_cast<std::underlying_type_t<E>>(enumerator);
}
```
或用上 auto：
```c++
template<typename E> // C++14
constexpr auto
  toUType(E enumerator) noexcept
{
  return static_cast<std::underlying_type_t<E>>(enumerator);
}
```
这样，获取 email 的信息可以这样用了：
```c++
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

**Things to remember**
* C++98-style enums are now known as unscoped enums.
* Enumerators of scoped enums are visible only within the enum. They convert
to other types only with a cast.
* Both scoped and unscoped enums support specification of the underlying type.
The default underlying type for scoped enums is int. Unscoped enums have no
default underlying type.
* Scoped enums may always be forward-declared. Unscoped enums may be
forward-declared only if their declaration specifies an underlying type.

## Item 11:使用 deleted 函数而不是 private undefined 函数

如果你想避免其他开发者使用某些函数，一般来说不定义这个函数就行了（这不显然么）。然而其实并没那么显然，在定义一个类时，C++ 会自己定义一些函数（在 item 17(这里到时加个链接) 中有详细介绍）。比如流的拷贝问题。在C++ 标准库中有个 basic_ios 的模板类，所有的输出、输入流都继承自这个模板类。对流的拷贝的作用并不清楚，所以要避免用户对流进行拷贝。最简单的做法是做一个空的定义，下面的代码来自 C++98（包括注释）：
```c++
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
…
private:
  basic_ios(const basic_ios& );           // not defined
  basic_ios& operator=(const basic_ios&); // not defined
};
```
但是这样的话成员函数或是友元都有权限使用 private 里面的函数。C++11 有更好的做法—— deleted functions:
```c++
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
…
  basic_ios(const basic_ios& ) = delete;
  basic_ios& operator=(const basic_ios&) = delete;
…
};
```
使用 delete 会让你在编译前就知道对这些函数的调用是不合法的，而若是友元等访问未定义的 private 函数，会在编译时才能知道。  
有个小细节，deleted functions 是定义在 public 里的，原因是 C++ 会先检查函数的可访问性（accessibility），然后才检查函数的删除属性（deleted status）。如果放在 private 里面，C++ 可能只会说函数在 private 不能访问，而不是说该函数不能使用。因此，将代码的 private-and-not-defined 成员改成 public deleted 成员有益于改进 error 信息。  

另外，deleted functions 可以用在其他的非成员函数上，如我们定义一个函数：
```c++
bool isLucky(int number);
```
有这么一些对它的调用：
```c++
if (isLucky('a')) … // is 'a' a lucky number?
if (isLucky(true)) … // is "true"?
if (isLucky(3.5)) … // should we truncate to 3
                    // before checking for luckiness?
```
如果我们想保证函数的输入只有整数的，可以使用 deleted function 来避免使用类型转换的合法类型：
```c++
bool isLucky(int number); // original function
bool isLucky(char) = delete; // reject chars
bool isLucky(bool) = delete; // reject bools
bool isLucky(double) = delete; // reject doubles and floats
```

deleted functions 还能避免模板函数的不合适的实例化。比如我们有一个作用于内置指针的模板：
```c++
template<typename T>
void processPointer(T* ptr);
```
在 C++ 中，有两类特殊的指针。一类是 void* 指针，因为它们不能被解引用或自增、自减；另一类是 char* 指针，因为它们指向的是 C 风格的字符串，而不是单个的 char。想避免使用这些指针类型的调用，可以用 deleted functions：
```C++
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;
```
另外，使用 const void* 和 const char* 的调用也应该被避免：
```c++
template<>
void processPointer<const void>(const void*) = delete;

template<>
void processPointer<const char>(const char*) = delete;
```
如果还想彻底点，还有 const volatile void* 和 const volatile char* ，然而暂时不懂 volatile（划掉）。  

如果是在类里使用模板，像下面的代码是不会编译的：
```c++
class Widget {
public:
…
  template<typename T>
  void processPointer(T* ptr)
  { … }
private:
  template<> // error!
  void processPointer<void>(void*);
};
```
其原因是对模板的特殊规定只能在命名空间（namespace scope）里写，而不能在类空间(class scope)。下面的代码可以实现上面代码的效果：
```c++
class Widget {
public:
  …
  template<typename T>
  void processPointer(T* ptr)
  { … }
  …
};
template<> 
void Widget::processPointer<void>(void*) = delete; // still public but deleted
```

总结一下，C++98 的 private-and-not-defined 成员实现的就是 C++11 中 deleted functions 实现的效果，但它在类外不能用，在类内不一定能用，就算能用了也要等到链接（link-time）时才能起作用。所以坚持使用 deleted functions 吧。  

**Things to Remember**
* Prefer deleted functions to private undefined ones.
* Any function may be deleted, including non-member functions and template instantiations.

## Item 12:将重写的函数声明为 override

虚函数是 C++ 的类的一个工具。你可以在基类（base class）中定义或声明虚函数，并在派生类（derived classed）中重写（override）这个虚函数。然而何时该调用 overrided 的虚函数并不显然。先看下面这个例子：

```C++
class Base {
public:
	virtual void doWork(); // base class virtual function
…
};
class Derived: public Base {
public:
	virtual void doWork(); // overrides Base::doWork ,"virtual" is optional here
  … 
};
// create base class pointer to derived class object;
// see Item 21 for info on std::make_unique
std::unique_ptr<Base> upb = std::make_unique<Derived>(); 
… 
upb->doWork(); // call doWork through base class ptr; derived class function is invoked
```
上面这个例子，就是通过了一个基类的接口，调用了派生类的函数。  
重写（Overring）的发生有以下几个必要要求：
* 基类函数必须是**虚函数（virtual）**
* 基类与派生类的该函数（下称基派函）**同名**（除了析构函数）
* 基派函**参数一致**
* 基派函**const 性（constness）一致**
* 基派函**返回类型与 exception specifications 兼容（compatible)**  

C++11 还加了一点：  
* 基派函**引用限定符（reference qualifiers）一致**

成员函数的引用限定符可以用来保证函数只被用于左值或右值，如：
```C++
class Widget {
public:
…
 void doWork() &; // this version of doWork applies only when *this is an lvalue
 void doWork() &&; // this version of doWork applies only when *this is an rvalue
};                 
…
Widget makeWidget(); // factory function (returns rvalue)
Widget w; // normal object (an lvalue)
…
w.doWork(); // calls Widget::doWork for lvalues (i.e., Widget::doWork &)
makeWidget().doWork(); // calls Widget::doWork for rvalues (i.e., Widget::doWork &&)
```

由重写引起的错误并不会被编译器发现，因为它是往往合法的，只是没有按你的原目的进行。如下面的代码，就完全没有任何重写：
```c++
class Base {
public:
  virtual void mf1() const;
  virtual void mf2(int x);
  virtual void mf3() &;
  void mf4() const;
};
class Derived: public Base {
public:
  virtual void mf1();               // 没 const
  virtual void mf2(unsigned int x); // 多了 unsigned
  virtual void mf3() &&;            // & &&
  void mf4() const;                 // base 不是 virtual
};
```
C++11 给出了避免上面这种情况的方案：把要重写的函数声明为 override：
```C++
class Derived: public Base {
public:
	virtual void mf1() override;
	virtual void mf2(unsigned int x) override;
	virtual void mf3() && override;
	virtual void mf4() const override;
};
```
这样，上面的代码就会因为重写失败而产生错误。

在 C++98 中也有 override：
```C++
class Warning { // potential legacy class from C++98
public:
…
void override(); // legal in both C++98 and C++11
…                // (with the same meaning)
};
```
所以如果看见老代码用了 override 函数并不用改。  
接下来补充一下引用限定符（reference qualifiers）的内容。假设有个Widget：
```c++
class Widget {
public:
	using DataType = std::vector<double>; // see Item 9 for
	…                                     // info on "using"
	DataType& data() & { return values; }
	DataType data() && { return std::move(values); }// for rvalue Widgets, return rvalue
	…
	private:
	DataType values;
};

// 给出两种调用
Widget w;
…
auto vals1 = w.data(); // copy w.values into vals1

Widget makeWidget();
auto vals2 = makeWidget().data();
```
这样，vals1 和 vals2 调用的是不同的 data() 函数。这就是 接受右值的函数定义 的作用。

**Things to Remember**
* Declare overriding functions override.
* Member function reference qualifiers make it possible to treat lvalue and rvalue objects (\***this**) differently.
