---
category: Python
---

使用Python内置的`issubclass`方法很方便的检测一个类是否是另一个类的子类。

这个是`issubclass`的文档：

**issubclass(class, classinfo)**

Return true if class is a subclass (direct, indirect or virtual) of classinfo. A class is considered a subclass of itself. classinfo may be a tuple of class objects, in which case every entry in classinfo will be checked. In any other case, a TypeError exception is raised.

一个类的子类可以是直接的、间接的、或者是**虚拟的**。

issubclass的第二个参数*classinfo*可以是一个类对象或者包含类对象的tuple（只要其中一个检测成功即返回True）。

一些使用示例：

```python
>>> class A(object):
...     pass
...
>>> class B(A):
...     pass
...
>>> class C(B, A):
...     pass
...
>>> class D(C):
...     pass
...
>>> issubclass(D, D), issubclass(D, C), issubclass(D, B), issubclass(D, A), issubclass(D, object)
(True, True, True, True, True)
>>> D.__bases__
(<class '__main__.C'>,)
>>> D.__mro__
(<class '__main__.D'>, <class '__main__.C'>, <class '__main__.B'>, <class '__main__.A'>, <class 'object'>)
```
D是D的子类，D定义时的基类是C,所以D是C的子类，而且D是B，A，object的间接子类。

`__mro__`是类属性， 在类定义完毕Python解析器便通过一种C3算法将所有的父类以method resolution order的顺序保存到一个元组里， 成为类的属性。

所以`issubclass`可以这样简单的实现：

```Python
def issubclass(cls, classinfo):
    if classinfo in cls.__mro__:
        return True
    return False
```

Python的issubclass是内置函数(一般是C实现)，实际上要复杂很多，要检测参数类型，如第一个参数必须是type类型，第二个参数是type类型或者tuple类型。还要考虑该类是否是虚拟的子类，以及子类的子类。

例如：

```python
>>> from collections import abc
>>> class E:
...     def __len__(self):
...         return 1
...
>>> issubclass(E, abc.Sized)
True
>>> E.__mro__
(<class '__main__.E'>, <class 'object'>)
>>> class F:
...     pass
...
>>> issubclass(F, abc.Sized)
False
>>> abc.Sized.register(F)
<class '__main__.F'>
>>> issubclass(F, abc.Sized)
True
```

Python是动态类型语言，长久以来使用Duck type(鸭子类型)形式编程，不管对象是什么类型，只要实现了所需要的方法。

现在有了ABCs, 可以用于判断某个类或者某个对象是不是ABCs的子类或者实例，但这个类并不需要显示的继承于ABCs， 因为python内置的ABCs有一种注册机制可将一个类注册为它的子类。如上例子的`register`方法。

还有一种机制是可以定制一个`__subclasshook__`方法，将某种类型的类认定为子类。

如`abc.Sized`的`__subclasshook__`是这样子的：

```python
@classmethod
def __subclasshook__(cls, C):
    if cls is Sized:
        if any("__len__" in B.__dict__ for B in C.__mro__):
            return True
    return NotImplemented
```

所以有`__len__`方法的E类是abc.Sized的子类， 这个`__subclasshook__`方法是通过`__subclasscheck__`方法调用的，这个`__subclasscheck__`是每一个ABC类都有的方法，在ABCMeta类（其他ABC类都继承于它）实现。

现在的`issubclass`函数的实现，会先判断classinfo是否有`__subclasscheck__`方法，如果有此方法，则判断子类的逻辑由该方法返回，即覆盖`issubclass`的实现（CPython）。

`__subclasscheck__`会分几个步骤进行判断：
1. 调用`__subclasshook__`方法，如果有方法定义
2. 检查自己是否在待检测类的__mro__列表里
3. 递归检查待检测类是否是在注册子类（内置_abc_registry列表属性)
4. 递归检查待检测类是否是自己子类的子类

具体源码在： https://github.com/python/cpython/blob/3.6/Lib/abc.py#L194-L231

相关的CPython实现在： https://github.com/python/cpython/blob/0ccc0f6c7495be9043300e22d8f38e6d65e8884f/Objects/abstract.c#L2223


而基本上`isinstance(object, classinfo)`方法的实现只需要调用`issubclass(type(object), classinfo)`

参考：

1. 29.7. abc — Abstract Base Classes ： https://docs.python.org/3/library/abc.html
2. PEP 3119 -- Introducing Abstract Base Classes： https://www.python.org/dev/peps/pep-3119/
