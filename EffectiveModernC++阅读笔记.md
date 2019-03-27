请坚持记录

# Deducing Types 类型推断
## Item 1: 理解模板类型推断
### 1)函数声明中的类型
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
f(x); // T is int, param's type is const int&
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
* **当传入的是左值时，类型推断会推向左值引用；若传入的是右值才推为右值引用。**

ParamType 是值，T
```c++
f(x); // T's and param's types are both int
f(cx); // T's and param's types are again both int
f(rx); // T's and param's types are still both int
```
* **会将传入的值去引用，并去const。**

### 当传入数组会怎么样？
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
Things to Remember
* During template type deduction, arguments that are references are treated as
non-references, i.e., their reference-ness is ignored.
* When deducing types for universal reference parameters, lvalue arguments get
special treatment.
* When deducing types for by-value parameters, const and/or volatile arguments
are treated as non-const and non-volatile.
* During template type deduction, arguments that are array or function names
decay to pointers, unless they’re used to initialize references.


### 存在的疑问：
1，什么是Universal reference?
2，什么样的参数是volatile？

### 2) auto语句中的类型推断
一般情况下，可以用