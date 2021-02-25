---
title: Braced Initialization
date: 2020-10-14
categories: CS
tags:
- C++
---
Many of the following are excerpts from *Scott Meyers - Effective Modern C++*

### Benefits

Initializing variables with {} has the following benefits:

1. It can be used to initialize non-static members in a class(while parentheses cannot);
2. It would prevent/warn narrowing conversions among built-in types. For example, you cannot initialize an integer by summing doubles(while parenthess and = operator do not check that);
3. It is immune to C++'s most vexing parse.

### Most Vexing Parse

What is most vexing parse? It is really vexing in the look, and you'll shout at how stupid the compiler is:

```c++
struct A
{
    void doSomething(){}
};
 
int main()
{    
    A a();  // A a{} would compile
    a.doSomething();
}
```

Such a seemingly simple program would not compile. This is because `A a()` is treated as a function rather than a struct instance, and that function does not have `doSomething` method of course. 

Why the compiler seems in this way? This might because C does not support constructor calls, and writing with parantheses will let the language think this is a function call. 

To give another example from wikipedia:

```c++
void f(double adouble) {
  int i(int(adouble));  // treat as: int i(int adouble)
  cout << i;
}
```

The intent is to construct a local variable by converting from a double to an int type. However, this will be treated in compiler as "a function named `i` whose return type is `int`, which takes a parameter of type `int` whose name is `adouble`. 

We have some ways to deal with this:

1. `int i((int(adouble)))` Wrap parantheses;
2. `int i((int) adouble)` or `int i(static_cast<int>(adouble))` cast;
3. `int i{int{adouble}}` braced initialization. However braced initialization does not allow, or will warn about the narrow conversions in this case.

### Unexpected Behaviors

Some points to watch out for when using the braced initialization: In [this post](/cs/2020/10/12/type-deduction/), it is mentioned that when you use `auto` with the brace initializer, the type would be deduced as `std::initializer_list`, which may give you behavior you don't want. 

Also in the function overloading case, if one or more constructors declare a parameter of type `std::initializer_list`, if you use brace initialization to initialize an object, you would in have a huge chance using the `std::initializer_list` constructor over others, even when other constructors seem far more suitable. For example:

```c++
class Example {
  public:
  	Example() {
      cout << "default constructor called";
    }
  	Example(bool b) {
      cout << "bool constructor called";
    }
  	Example(int i) {
      cout << "int constructor called";
    }
    Example(std::initializer_list<long double> ld) {
      cout << "initializer constructor called";
    }
};

Example e1{true};  // initializer constructor called, true converts to long double
Example e2{1};  // initializer constructor called, 1 converts to long double
Example e3(); // most vexing parse to declare a function
Example e4{}; // default constructor called
Example e5{ {} }; // initializer constructor called
Example e6({}); // initializer constructor called
```

As you can see, the `std::initializer_list` constructor overshadows other constructors in many cases. This will requires class writers and clients(callers) to take extra attention in making sure the added constructor guarantees backward compatibility, and the objects are using the correct constructor. This can sometimes be confusing for `std::vector`, as we have two constructors for it:

```c++
// fill vector with copy of val n times
vector (size_type n, const value_type& val, const allocator_type& alloc = allocator_type());
vector v(5,2); // {2,2,2,2,2}

// Constructs a container with a copy of each of the elements in il, in the same order
vector (initializer_list<value_type> il, const allocator_type& alloc = allocator_type()); 
vector v{5,2};  // {5,2}
```

