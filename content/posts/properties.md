---
title: Why you should use property factories in Python
date: 2023-09-25T00:32:39+05:00
draft: false
---
If you often use the @property decorator in your classes, do you find yourself repeating the same logic over and over again?

### If this problem is familiar to you, use factories

```Python
from functools import cache  
  
def frozen(value):  
    def gty_get(instance):  
        return value  
  
    return property(gty_get)  
  
  
def default_from(factory):  
  
    def qty_get(instance):  
        return factory()  
  
    return property(cache(qty_get))  
  
  
def constraint(name, key_func):  
    def qty_get(instance):  
        return instance.__dict__[name]  
  
    def qty_set(instance, value):  
        if key_func(value):  
            instance.__dict__[name] = value  
        else:  
            raise ValueError('Value does not meet constraint')  
  
    return property(qty_get, qty_set)
```

```Python
class Foo:  
    a = frozen(5)  
    b = default_from(list)  
    c = constraint('c', lambda x: x % 2 == 0)
```

```Python
Doctest  
  
>>> foo = Foo()  
>>> foo.a
5

>>> foo.a = 10
Traceback (most recent call last):  
    ...
AttributeError: property 'a' of 'Foo' object has no setter  
  
>>> bar = Foo()  
>>> bar.b, foo.b  
([], [])

>>> bar.b is foo.b  
False

>>> foo.b.append(True)  
>>> foo.b  
[True]

>>> foo.c  
Traceback (most recent call last):  
    ...
KeyError: 'c'

>>> foo.c = 6  
>>> foo.c  
6

>>> foo.c = 7  
Traceback (most recent call last):  
    ...
ValueError: Value does not meet constraint
```