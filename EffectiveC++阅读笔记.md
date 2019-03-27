《Effective Modern C++》笔记。
仅供个人学习
请lyk坚持记录

# Deducing Types 类型推断
## Item 1: 理解模板类型推断
### 1)模板函数声明中的类型
```c++
template<typename T>
void f(ParamType param);

int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int
```

ParamType为普通引用,即 ParamType 为 T&
```c++
int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int
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
**引用折叠**
* X& &、X& &&和X&& &都折叠成类型X&
* 类型X&& &&折叠成X&&

* **当传入的是左值时，类型推断会推向左值引用；若传入的是右值才推为右值引用。**

ParamType 是值，T
```c++
f(x);  // T's and param's types are both int
f(cx); // T's and param's types are again both int
f(rx); // T's and param's types are still both int
```
* **会将传入的值去引用，并去const。**

**当传入数组会怎么样？**
是的，聪明的你会发现下面的格式是错的
```c++
void myFunc(int param[]);
```
并且在数组名字单独出现时，往往被视作一个指针
```c++
void myFunc(int* param);
```
然而我们可以声明一个参数是对一个数组的引用。设name是一个长度为13的const char[]。
```c++
template<typename T>
void f(T& param); // template with by-reference parameter

f(name); // pass array to f
```
T会被推断为const char [13]，f的参数是const char (&)[13].
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

### 2) auto语句中的类型推断
一般情况下，可以用模板函数中的类型推断来判断auto的结果。如
```c++
auto x = 27;
const auto cx = x;
const auto& rx = x;

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

这样，下面的类型推断就很自然了
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
const char name[] = // name's type is const char[13]
"R. N. Briggs";
auto arr1 = name;   // arr1's type is const char*
auto& arr2 = name;  // arr2's type is const char (&)[13]

void someFunc(int, double); // someFunc is a function;
														// type is void(int, double)
auto func1 = someFunc; // func1's type is void (*)(int, double)
auto& func2 = someFunc; // func2's type is void (&)(int, double)
```

如果用auto的对象是初始化列表，推断的结果也是初始化列表
```c++
int x = {27};
auto y = {27}; // type is std::initializer_list<int>
```
这也是auto和模板类型推断会不同的唯一地方：
```c++
auto x = { 11, 23, 9 }; // x's type is
												// std::initializer_list<int>
template<typename T> // template with parameter
void f(T param);  // declaration equivalent to
								  // x's declaration
f({ 11, 23, 9 }); // error! can't deduce type for T
```
只能像下面这样用模板推断initializer_list:
```c++
template<typename T>
void f(std::initializer_list<T> initList);
f({ 11, 23, 9 }); // T deduced as int, and initList's
									// type is std::initializer_list<int>
```
另外，C++14允许使用auto来推断函数返回类型或lambda函数参数，但这种用法是使用的模板类型推断，而不是auto的。因此，若返回的是初始化列表并不能编译：
```c++
auto createInitList()
{
return { 1, 2, 3 }; // error: can't deduce type
} 								  // for { 1, 2, 3 }

auto resetV = [&v](const auto& newValue) { v = newValue; }; // C++14
resetV({ 1, 2, 3 }); // error! can't deduce type for { 1, 2, 3 }
```
**Things to Remember**
* auto type deduction is usually the same as template type deduction, but auto
type deduction assumes that a braced initializer represents a std::initial
izer_list, and template type deduction doesn’t.
* auto in a function return type or a lambda parameter implies template type
deduction, not auto type deduction.

###理解decltype
先来一些no surprise的例子
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

在C++11中，decltype主要用于函数返回类型(return type)基于参数类型(parameter types)的情况。
```c++
template<typename Container, typename Index> // works, but
auto authAndAccess(Container& c, Index i)    // requires
-> decltype(c[i]) 													 // refinement
{
	authenticateUser();
	return c[i];
}
```
