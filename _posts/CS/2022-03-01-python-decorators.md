---
title: Python decorators
date: 2022-03-01
categories: CS
tags:
- python
- language
---

（5年前写的，贴在这里方便回看）

Python代码里经常看到一个符号@摆在函数的上面一行，这叫decorator，但它到底是什么，表示什么意思，有什么好处呢？decorator是用来修饰函数的，等于将原函数（比如`foo`）经过了一定加工，达成的效果是`foo = our_decorator(foo)`这个样子，可以简写成`@our_decorator`在`foo`上面。

举个简单的例子：首先我们定义一个函数
```
def foo(x):
      print("Hi, foo has been called with " + str(x))
```
调用此函数
```
foo("Hi")
# Hi, foo has been called with Hi
```
加上decorator：
```
def our_decorator(func):
      def function_wrapper(x):
            print("Before calling " + func.__name__)
            func(x)
            print("After calling " + func.__name__)
      return function_wrapper

foo = our_decorator(foo)
```
我们将foo作为一个function object传输给了our_decorator，our_decorator被调用并返回了function wrapper这个函数。这等于说foo经过了our_decorator的修饰，多了一些功能。但是这时候foo还没有被调用。接下来调用被修饰过的foo函数：
```
foo(42)
"""
Before calling foo
Hi, foo has been called with 42
After calling foo
"""
```
这就是decorator做的事情，我们可以用一个@符号来取代`foo = our_decorator(foo)`这一行，更加简单和pythonic：
```
@our_decorator
def foo(x):
    print("Hi, foo has been called with " + str(x))
```
其它的装饰函数定义和调用foo函数的方法不变。

要理解decorator，主要是要理解callback function的含义，即给现有函数传入一个函数，返回的也是一个函数，而传入函数将作为一个值暂居在被返回的函数里，等到现有函数被调用的时候，实际上调用的是被返回的那个函数，其中也包括了我们传入的函数。

稍微复杂一点，如果我们想给decorator也加parameter的话：
```
def greeting(expr):
    def greeting_decorator(func):
        def function_wrapper(x):
            print(expr + ", " + func.__name__ + " returns:")
            func(x)
        return function_wrapper
    return greeting_decorator

@greeting("ÎºÎ±Î»Î·Î¼ÎµÏÎ±")
def foo(x):
    print(42)

foo("Hi")
```

在装饰函数的时候，尽管我们最后调用的还是原来那个函数的名字`foo`，但是它的元信息已经发生了变化，比如：
```
def greeting(func):
    def function_wrapper(x):
        """ function_wrapper of greeting """
        print("Hi, " + func.__name__ + " returns:")
        return func(x)
    return function_wrapper

@greeting
def f(x):
    """ just some silly function """
    return x + 4

f(10)
print("function name: " + f.__name__)
print("docstring: " + f.__doc__)
print("module name: " + f.__module__)

'''
Hi, f returns:
function name: function_wrapper
docstring:  function_wrapper of greeting 
module name: greeting_decorator
'''
```
我们看到，原函数被传入装饰函数时，还具有原来的信息，但是当我们调用被装饰过的函数时，函数名字、docstring、被调用的模块都发生了变化。如果我们想保留`foo`原来的元函数信息的话，可以在装饰函数中进行元函数信息的保留，即将返回函数的元信息设置成原函数的元信息：
```
def greeting(func):
    def function_wrapper(x):
        """ function_wrapper of greeting """
        print("Hi, " + func.__name__ + " returns:")
        return func(x)
    function_wrapper.__name__ = func.__name__
    function_wrapper.__doc__ = func.__doc__
    function_wrapper.__module__ = func.__module__
    return function_wrapper

f(10)
print("function name: " + f.__name__)
print("docstring: " + f.__doc__)
print("module name: " + f.__module__)

'''
Hi, f returns:
function name: f
docstring:  just some silly function 
module name: __main__
'''
```
哎哟，这么多行累死了是不是？我们可以给 `function_wrapper`这个函数也加一个装饰函数，达到一样的效果：
```
from functools import wraps

def greeting(func):
    @wraps(func)   # 使function_wrapper具有跟func一样的元信息
    def function_wrapper(x):
        """ function_wrapper of greeting """
        print("Hi, " + func.__name__ + " returns:")
        return func(x)
    return function_wrapper
```
举一个现实生活中的例子。用户登录页面的时候，我们常要先检查其是否有访问权限，因为这种操作非常多，可以将之refactor成一个装饰函数：
```
from flask import Flask, request, abort
from functools import wraps

app = Flask(__name__)


def validate_json(*expected_args):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            json_object = request.get_json()
            for expected_arg in expected_args:
                if expected_arg not in json_object:
                    abort(400)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@app.route('/grade', methods=['POST'])
@validate_json('student_id')
def update_grade():
    json_data = request.get_json()
    print(json_data)
    # update database
    return "success!"
```
我们也可以用一连串的decorator，比如以下用了两个：
```
def makebold(fn):
    def wrapped():
        return "<b>" + fn() + "</b>"
    return wrapped

def makeitalic(fn):
    def wrapped():
        return "<i>" + fn() + "</i>"
    return wrapped

@makebold  # hello = makebold(makeitalic(hello)), returns <b><i>hello()<i><b>
@makeitalic  # hello = makeitalic(hello), returns <i>hello()</i>
def hello():
    return "hello world"

print hello() ## returns "<b><i>hello world</i></b>"
```

最后，任何callable object都可以成为装饰主体，因此除了函数，我们还可以用class。如：
```
class decorator2:
    
    def __init__(self, f):
        self.f = f
        
    def __call__(self):
        print("Decorating", self.f.__name__)
        self.f()

@decorator2
def foo():
    print("inside foo()")

foo()
```
当进行装饰的时候，`__init__`被调用并保存了`f`的信息，当`foo`被调用的时候，`__call__`函数被调用。

在现实生活中常用的decorator有：

**@classmethod （俺尚不是很理解它的应用价值）**

在一个类的函数上使用@classmethod，函数返回该类的构造函数。这样的情况下，我们调用了一个类中的函数，同时也创造了一个对象。比如：
```
from datetime import date

# random Person
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    @classmethod
    def fromBirthYear(cls, name, birthYear):
        return cls(name, date.today().year - birthYear)

    def display(self):
        print(self.name + "'s age is: " + str(self.age))

person = Person('Adam', 19)
person.display()

person1 = Person.fromBirthYear('John',  1985)
person1.display()

'''
Adam's age is: 19
John's age is: 31
'''
```

@classmethod比@staticmethod多的一个好处是，它能正确反映继承关系。如：
```
from datetime import date

# random Person
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    @staticmethod
    def fromFathersAge(name, fatherAge, fatherPersonAgeDiff):
        return Person(name, date.today().year - fatherAge + fatherPersonAgeDiff)

    @classmethod
    def fromBirthYear(cls, name, birthYear):
        return cls(name, date.today().year - birthYear)

    def display(self):
        print(self.name + "'s age is: " + str(self.age))

class Man(Person):
    sex = 'Male'

man = Man.fromBirthYear('John', 1985)
print(isinstance(man, Man))      # True

man1 = Man.fromFathersAge('John', 1965, 20)
print(isinstance(man1, Man))      # False
```

**@property**

在类里，我们常用一个getter和setter来对类中的变量进行操作，这是为了data encapsulation的考虑，也可以在set和get的时候进行bound check。@property就提供了这样一个方法。

```
class Person(object):  
    def __init__(self, first_name, last_name):
        self.first_name = first_name
        self.last_name = last_name

    @property
    def full_name(self):
        return self.first_name + ' ' + self.last_name

    @full_name.setter
    def full_name(self, value):
        first_name, last_name = value.split(' ')
        self.first_name = first_name
        self.last_name = last_name

    @full_name.deleter
    def full_name(self):
        del self.first_name
        del self.last_name
```
我们可以看到，full_name函数经过@property装饰，有了getter的功能，而经过@full_name.setter的装饰，有了setter的功能。虽然在类中，函数不可以有相同的名字，但他们经过了装饰，所以这是可行的。要注意的是，getter的装饰函数叫@property，setter的装饰函数是@func.setter。property函数是这样定义的：`property(fget=None, fset=None, fdel=None, doc=None)`。装饰之后，我们得到：
```
full_name.fget is full_name_getter    # True  
full_name.fset is full_name_setter    # True  
```
之后我们在这个类的对象上进行set或者get的操作的话，就会调用相应的函数。比如person.name = 'Bob'，那么就会调用full_name_getter函数。但要注意的是，只有当类从object继承的时候，才会有这样的效果。所以在定义类的时候，一定要从object继承哦。

####Reference
1. [Python tutorial](http://www.python-course.eu/python3_decorators.php)
2. [How to make a chain of function decorators?](https://stackoverflow.com/questions/739654/how-to-make-a-chain-of-function-decorators)
3. [Primer on Python Decorators](https://realpython.com/blog/python/primer-on-python-decorators/)
4. [Python classmethod()](https://www.programiz.com/python-programming/methods/built-in/classmethod)
5. [Python Properties](http://stackabuse.com/python-properties/)
6. [Clear example]([https://stackoverflow.com/questions/308999/what-does-functools-wraps-do](https://stackoverflow.com/questions/308999/what-does-functools-wraps-do))