《Effective Modern C++》笔记。  
仅供个人学习  
请lyk坚持记录
[![996.icu](https://img.shields.io/badge/link-996.icu-red.svg)](https://996.icu)

* [一，Deducing Types (类型推断)](#一deducing-types-类型推断)  
	-[Item 1: 理解模板类型推断](#item-1理解模板类型推断)  
	-[Item 2: auto 语句中的类型推断](#item-2auto语句中的类型推断)  
	-[Item 3: 理解 decltype](#item-3理解decltype)  
	-[Item 4: 如何观测实际的类型推断](#Item-4如何观测实际的类型推断)
* [二，Auto](#二auto)  
	-[Item 5: 多用 auto 替换显示类型声明](#Item-5多用auto替换显式类型声明)  
	-[Item 6:用显示类型声明来避免 auto 的不合适推断](#Item-6用显示类型声明来避免-auto-的不合适推断)  
* [三，Moving to Modern C++](#三moving-to-modern-c)  
	-[Item 7:辨别生成对象时 () 和 {} 的不同](#item-7辨别生成对象时和的不同)  
	-[Item 8:多用 nullptr 代替 0 和 null](#item-8多用-nullptr-代替-0-和-null)  
	-[Item 9:多用别名定义而不是 typedefs](#item-9多用-alias-declarations-而不是-typedefs)

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
f(x); // T is int, param's type is int&
f(cx); // T is const int,
       // param's type is const int&
f(rx); // T is const int,
       // param's type is const int&
```

ParamType 为 const T&
```c++
f(x);  // T is int, param's type is const int&
f(cx); // T is int, param's type is const int&
f(rx); // T is int, param's type is const int&
```

ParamType 为右值引用 T&&，
```C++
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
x 是一个类型为int 的变量名，所以f1是int类型的。但(x)是比名字更复杂的左值表达式，所以f2是int &类型的。

**Things to Remember**
* decltype almost always yields the type of a variable or expression without
any modifications.
* For lvalue expressions of type T other than names, decltype always reports a
type of T&.
* C++14 supports decltype(auto), which, like auto, deduces a type from its
initializer, but it performs the type deduction using the decltype rules.

## Item 4：如何观测实际的类型推断
书中给出了三种方法：IDE自动推断（getting type deduction information as you edit your code），编译中获取（getting it during compilation）和在运行中获取（getting it at runtime）。

Visual Studio 2015可以对相对简单的类型进行推断:
```c++
const int theAnswer = 42;
auto x = theAnswer;  //把鼠标放在auto上就能看到推断的类型
auto y = &theAnswer;
```

在代码编译时，若出现error，会给出相关的信息，这也能得到推断的类型，如：
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

