---
title: Lambda in C++
date: 2021-02-25
categories: CS
tags:
- C++
---

*Excerpt from Effective Modern C++ Chapter 6*

### Example

```c++
std::find_if(
	container.begin(), 
	container.end(), 
	[](int val) { return 0 < val && val < 10; }  // lambda expression/closure
)
```



### Use Init Capture

Default capture modes are not recommended in lambda. 

For problems with by-reference capture(&), if lambda captures data in a local function, exiting the local function would make the data captured undefined. Long-term, it’s simply better software engineering to explicitly list the local variables and parameters that a lambda depends on.

For problems with by-value capture(=), if you use a raw pointer, and capture a pointer by value, but the pointer is later deleted outside of lambda, it is very dangerous. Another scenario is that captures only apply to non-static local variables, so you cannot capture a data member of a class by value. For example:

```c++
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters1;
FilterContainer filters2;
FilterContainer filters3;

class Example {
public:
  void addExample() const;
private:
  int divisor;
}

void Example::addExample() const 
{
  filters1.emplace_back([=](int value) { return value % divisor == 0; });  // A
  filters2.emplace_back([](int value) { return value % divisor == 0; });  // B
  filters3.emplace_back([divisor](int value) {return value % divisor == 0; });  // C
}
```

In the above example, line A actually captures the *this* pointer of the Example class. This means the Example object must exist when lambda is invoked. If not, a danling pointer is held in filters1. Line B and C simply won't compile, since no local divisor can be captured. 

Also, since by value capture does not capture static variables, the captured value may seem self-contained but actually is global, which may result in unexpected behaviors. Example:

```c++
void filter() 
{
  static auto divisor = 1;
  filters.emplace_back([=](int value){return value % divisor == 0;});
  // divisor is a static variable which is modified,
  // any lambdas that have been added to filters via this function will exhibit new behavior (corresponding to the new value of divisor).
  ++divisor;  
}
```

The recommended way is to use init capture.

```c++
auto pw = std::make_unique<Widget>();
auto func = [pw = std::move(pw)] { return pw->isTrue(); }
```

For `[pw = std::move(pw)]`, `pw` in the left refers to the scope inside the closure class, `pw ` in `std::move(pw)` refers to the object declared above the lambda in the local scope.

### Comparison with std::bind

In C++, objects can't be move constructed using lambda, but can be emulated by using std::bind. std::bind produces function objects, which contain copies of all the arguments(copy or move constructed) passed to std::bind. Since the bind objects store copies of arguments, the lifetime of the bind object is the same as that of the closure.

Generally speaking, lambdas are preferable over std::bind, for the following reasons:

- Lambdas are much more readable
- Arguments passed into bind will be evaluated when bind object is created, not when the function is invoked, which may result in unexpected behaviors
- When overloading a function with the same name, the function needs to be cast to a proper function pointer type in order to compile, which makes compilers less likely to inline function calls

### Perfect forwarding rules

To use universal reference with lambda, use the following:

```c++
auto f = [](auto&& param) {
  return func(std::forward<decltype(param)>(param));
}
```

When param is an lvalue reference, decltype(param) produces lvalue type. When param is an rvalue reference, decltype(param) produces rvalue type. In C++ implementation, std::forward looks likes this:

```c++
template<typename T>
T&& forward(remove_reference_t<T>& param)
{
  return static_cast<T&&>(param);
}
```

According to the reference collapsing rule: If either reference is an lvalue reference, the result is an lvalue reference. Otherwise (i.e., if both are rvalue references) the result is an rvalue reference. Therefore, whatever type we passed in for param in the lambda, it will produce the correct type.

