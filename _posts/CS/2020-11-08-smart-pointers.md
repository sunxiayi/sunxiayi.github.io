---
title: Smart pointers
date: 2020-11-08
categories: CS
tags:
- C++
---
Many of the following are excerpts from *Scott Meyers - Effective Modern C++*

## Compare unique_ptr & shared_ptr

| Comparison     | unique_ptr                                                   | shared_ptr                                                   |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Custom deleter | Encoded in the type. <br>`unique_ptr<Widget, decltype(deleteFunc)> w(new Widget, deleteFunc)`<br>(Implication)Smaller runtime data structures and faster runtime code, but the pointed-to types must be complete when compiler-generated special functions | Not in the type<br>`shared_ptr<Widget> w(new Widget, deleteFunc)`<br>(Implication)Larger runtime data structures and slower code, but pointed-to types need not be complete when compiler-generated special functions are employed |
| Copy           | Cannot be copied                                             | Can be copied                                                |
| Conversion     | Can convert to shared_ptr                                    | Cannot convert to unique_ptr                                 |
| Size           | Size will be increased by the customer deleter               | Twice the size of a raw pointer: a raw pointer to the resource, a raw pointer to the resource's reference count |
| Control Black  | None                                                         | Control block contains reference count, weak count, custom deleter, allocator etc |

## std::enable_shared_from_this

If an object has two control blocks, it will lead to undefined behavior. Since constructing a shared_pointer from a raw pointer will create a control block, if we accidently create two shared_pointers that point to the same object, they will have different reference counts and will have undefined destruction logic... This will be even worse when we are doing that obliviously, for example:

```c++
vector<shared_ptr<Examples>> transformedExamples;

class Example {
public:
  void addToContainer();
}

void Example::addToContainer() {
  transformedExamples.emplace_back(this);  // it will construct a new shared_ptr to *this
}

shared_ptr<Example> e(new Example);  // we already have a shared_ptr that points to e
e.addToContainer();
```

This can be fixed by inheriting from a template: `std::enable_shared_from_this<T>`:

```c++
class Example: public std::enable_shared_from_this<Example> {
public:
  void addToContainer();
}

void Example::addToContainer() {
  // it will construct a shared_ptr to without duplicating control blocks
  // if this object does not have a control block already, will throw an exeption
  transformedExamples.emplace_back(shared_from_this()); 
}
```

## weak_ptr

What is a weak pointer? It is a smart pointer that does not increase the reference count of the object and thus does not share ownership with the ponited-to resource, it usually points to an object that a shared_ptr points to.

Why do we need a weak pointer? It is useful when we want to detect when the pointer will be dangled, or when we want to break a cycle of reference between two objects.

How do we convert from a weak pointer to a shared pointer? 

```c++
auto sharedpointer = make_shared<Example>();  // create a shared pointer
weak_ptr<Example> weakpointer(sharedpointer);  // create a weak pointer that points to the object that holds by sharedpointer
shared_ptr<Example> sharedpointerConvert = weakpointer.lock();  // convert to a shared pointer if the weak pointer does not dangle
shared_ptr<Example> sharedpointerConvert2(weakpointer);  // another way to convert, will throw bad_weak_ptr if weakpointer expires
if (sharedpointerConvert) {
	// weakpointer does not dangle
}
```

## make constructor

Let's name the way to use `make_unique` and `make_shared` to create pointer "method A", the way to use unique_ptr, shared_ptr + new operator to construct the pointer "method B".

It is preferred to use method A, for the following reasons:

1. Less typing. `auto ptr = make_unique<T>()` vs `unique_ptr<T> ptr(new T)`;
2. More efficient. Method A allocates memory of object pointer + control block pointer at once, producing smaller static code, while unique_ptr allocates them separately;
3. Exception safety. For example, in a function like this: `f(unique_ptr<T>(new T), throw_function())`, the compiler is allowed to call: `new T`, `throw_function()`, `unique_ptr<T>` in order, which causes T to be leaked after an error is thrown in `throw_function()`. Using `make_unique<T>()` avoids this. If you want to achieve exception safety in method B, make sure after `new`, assign that to a smart pointer immediately, such as `shared_ptr<T> assignImmediately(new T)`. 

Some rare but possible reasons to use method B:

1. It enables customized deleter;
2. Method A will use the parentheses to perfect forward the parameters, rather than braces. So when a class(such as in `vector`) has initializer_list parameters, method A will not use that ctor. You need to use the `new` method to use the brace initialization or use some workaround;
3. When object type is quite large and the time between destruction of the last shared_ptr and weak_ptr is huge, what happens is: shared_ptr is `deleted`, the object the shared pointer refers to get deleted, however the control block is alive until no weak_ptr is pointing to this object. This delay of freeing up space won't happen in method B case because space for the pointer to object and control block is allocated separately.