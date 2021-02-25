---
title: Type Deduction(templates, auto, decltype)
date: 2020-10-12
categories: CS
tags:
- C++
---
## Type Deduction(templates, auto, decltype)

Many of the following are excerpts from *Scott Meyers - Effective Modern C++*

### Templates

```c++
====================== Reference or Pointer, not a Universal Referece ======================
template<typename T1>
void f1(T1& param);   // param is passed by lvalue-reference-to-non-const

template<typename T2>
void f2(const T2& param);  // param is passed by const reference to lvalue

template<typename T3>
void f3(T3* param);  // param is passed by pointer

====================== Universal Reference ============================================
template<typename T4>
void f4(T4&& param);  // param is a universal reference

====================== Pass by Value ============================================
template<typename T5>
void f5(T5 param);   // param is passed by value
```

It's clear that `f1` takes an lvalue reference of `T1`, `f2` takes an lvalue reference of `const T2`.

```c++
int x = 27;
const int cx = x;
const int &rx = x;  // rx is a reference to x as a const int, we may modify x's value and rx will change as well, but we cannot change rx's value to change x's value
const int *px = &x; // px is a ptr to x as a const int, we may modify x's value and *px will change as well, but we cannot change *px's value to change x's value. We can change where px is pointed to. The original object px is pointed to should not be changed.
const char* const ptr = "fun stuff";  // ptr is a const ptr*(right to the asterisk) to a const object(left to the asterisk)


f1(x);   // void f1(int& param), T1 is int
f1(cx);  // void f1(const int& param), T1 is const int
f1(rx);  // void f1(cosnt int& param), T1 is const int. **Note, reference-ness of rx is ignored**

f2(x);   // void f2(const int& param), T2 is int
f2(cx);  // void f2(const int& param), T2 is int
f2(rx);  // void f2(const int& param), T2 is int. **Note, reference-ness of rx is ignored**

f3(&x);  // void f3(int* param), T3 is int
f3(px);  // void f3(const int* param), T3 is const int

f4(x);   // void f4(int& param), T4 is int&. Note x is an lvalue, thus T4 would be lvalue. It may seem confusing for the first time, because T4&& should be interpreted as the universal reference, rather than rvalue reference
f4(cx);  // void f4(const int& param), T4 is const int&
f4(rx);  // void f4(const int& param), T4 is const int&
f4(27);  // void f4(int&& param), since 27 is rvalue, T4 is rvalue thus int&&

f5(x);   // void f5(int param), T5 is int

// Here come the surprising ones!
f5(cx);  // void f5(int param), T5 is int.
f5(rx);  // void f5(int param), T5 is int. **Note, reference-ness of rx is ignored**
f5(ptr); // void f5(const char* param), T5 is const char*. Param is passed by value, so const-ness of const ptr is changed. const-ness of the object is however, preserved. 
```

For the last 3 examples, since param is passed by value, reserving const-ness of cx or rx does not make sense, since param is independent from cx or rx. Changing param has no effect on cx or rx.



### Auto

auto, in most cases is an algorithmic transformation from template. For example `auto x = 27`, fits into `T5`, `const auto cx = x` fits into `T5`, `const auto& rx = x` fits into `T2`, `auto&& rv = x` and `auto&& rv = 7` fits into `T4`, `auto x = &lvalue` fits into `T1`. You can view resolving auto as doing an equivalent template type deduction.

There is a case they differ.

```c++
auto x = {2};
auto x{2};
auto x = {1,2,3.0}; // error! can't deduce T for std::initializer_list<T>
```

When constructing an item with the brace, auto is a type of `std::initializer_list<int>`. This type is used to access the values in the initialization list, which is a list of elements of type `const T`. Thus values in the brace should be of the same type.

If you pass a brace initialized item to a function, it will be treated as `std::initializer_list`, and normal template type deduction cannot solve it. For example:

```c++
template<typename T1>
void f1(T1 params);

template<typename T2>
void f2(std::initializer_list<T2> initList);

f1({1,2,3});  // error! can't deduce type for T
f2({1,2,3});  // T2 deduced as int, initList's type is std::initializer_list<int>
```

In C++14, another point to notice that, if you use `auto` as the return type, or use `auto` in lambda parameter declarations, it will be treated as template type deductions, thus returning a braced initializer or passing that into lambdas won't compile.

```c++
// won't compile, can't deduce type
auto example1() {
  return {1,2};
}

auto example2 = [](auto& param) { cout << "error"; };
example2({1,2});
```

Using auto has many virtues, especially when the type is very complicated or not useful to be written out in the full form. However there is one case that auto may not do what you want, that is to use auto in declaring a proxy class, which often is not designed to live longer than a single statement. The book gives an example:

```c++
vector<bool> features();

// operator[] returns T& except for bool, it returns vector<bool>::reference, which is a proxy class. bool declaration implicit converts it to a boolean.
bool highPriority = features()[5];  

// auto deduces type. vector<bool>::reference may be a pointer. features() is a temporary object that does not exist after the following line. so highPriority is a dangling pointer. following code may have undefined behavior
auto highPriority = features()[5];  
```



### decltype

It reminds me of `type()` in Python. At the end they do very similar things and in most cases do not give you surprises. One case it gives you surprise is when you pass a lvalue more complicated than a name, it will reports that type as `T&`.  For example,

```c++
// decltype((x)) is int& rather than int, and x is a local variable! boom!
decltype(auto) f() {
  int x = 0;
  return (x);
}
```

Some usage of `decltype` include: use `decltype` to show return type of the function in the trailing return type syntax(`->`) in C++11, and guard the real return/initialization type in C++14. For example,

```c++
Example e;
Widget& eCopy1 = e;
const Widget& eCopy2 = e;
auto eAuto1 = eCopy1;  // uses auto type deduction, pass by value. So we strip reference-ness off.
auto eAuto2 = eCopy2;  // uses auto type deduction, pass by value. So we strip reference-ness off, then strip const off.
// Therefore, eAuto1 and eAuto2 are of type Widget
decltype(auto) eAutoWithDecltype = eCopy2;  // eAutoWithDecltype type is const Widget& --> guard the real type!
```

Another example to guard the return value:

```c++
template<typename Container, typename Index>
decltype(auto) returnReferenceToObjectInContainer(Container& c, Index i) {
  return c[i];
}
```

Why we might want to do that? Difference containers, such as `std::string`, `std::deque`, `std::vector`, may return elements by value of by reference. In the case a reference is returned, since `auto` uses the template type reduction, if we do not use decltype, the return value will strip the reference off.