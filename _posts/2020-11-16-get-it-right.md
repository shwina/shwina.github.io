---
layout: post
title: "__get__ it right: controlling attribute access in Python"
---

In this post, I explain the difference between
`__get__`, `__getattr__` and `__getattribute__`.
I also explain descriptors and how they fit into the picture.


## dunder methods

Dunder methods are special methods you can define when writing a class.
What makes them different from "regular" methods,
is that they are typically invoked using some kind of special syntax.

For many Python programmers,
`__init__` is the most common dunder method they encounter.
It is invoked using the name of the class followed by `()`:

```
class Point:
    def __init__(self, x, y):
        self.x =x
        self.y = y

Point(1, 2)  # invokes Point.__init__()
```

Similarly, the `__eq__` method is invoked using `==` or `!=`:

```
class Point:
    ...
    
    def __eq__(self, other):
        return self.x, self.y == other.x, other.y


a = Point(1, 2)
b = Point(1, 2)
c = Point(1, 3)

a == b  # invokes Point.__eq__()
a != c  # invokes Point.__eq__()
```

## Attribute access can be customized

Let's talk about another kind of special syntax:

```
x.y
```

Yes! When you access an attribute, that _is_ special syntax,
and you can customize what happens when an attribute is accessed using dunder methods.
`__get__`, `__getattr__` and `__getattribute__` are all dunder methods that
control attribute access. But it's important to learn when to use which.

## The difference between `__get__`, `__getattr__` and `__getattribute__`

What happens when you write `x.y` in Python?
Well, it's complicated.

### `__getattribute__` and `__getattr__`

When you write `x.y` and `x` is an instance,
something like the following happens:

```python
try:
    return x.__getattribute__("y")
except AttributeError:
    if not hasattr(type(obj), "__getattr__"):
        raise
return type(obj).__getattr__(obj, name)
```











---

Actually `Point(...)` doesn't invoke `Point.__init__()` directly,
but rather
[`Point.__new__()`](https://docs.python.org/3/reference/datamodel.html#object.__new__)
which _in turn_ invokes `Point.__init__()`. But we won't get into that here.

---

## What is a descriptor?

To save you any suspense,
I'll go ahead and define what a descriptor is.
Any class that defines one or more of the following methods is a descriptor:

1. `__get__()`,
2. `__set__()`,
3. or `__delete__()`

If you're looking for a proper "Introduction to descriptors" or
a more technical discussion,
see the
[Descriptor HowTo](https://docs.python.org/3/howto/descriptor.html)
by Raymond Hettinger.
It's one of my favourite parts of the official Python docs!
