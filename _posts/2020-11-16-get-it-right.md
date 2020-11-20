---
layout: post
title: "__get__ it right: controlling attribute access in Python"
---

In Python, you use the dot (`.`) operator to access attributes of an object.
Normally, that isn't something you have to give much thought to.
However, when you want to customize what happens during attribute access,
things can get complicated.


In this post, we'll see how to control what happens when you use the dot operator.
Before we can talk about customizing attribute access,
we need to discuss two related topics:
**class and instance attributes**, and **descriptors**.
If both of those are familiar to you,
feel free to skip ahead.

- TOC
{:toc}

## Class and instance attributes

There are two kinds of attributes in Python:
_class_ and _instance_ attributes.
In the following class,
`volume` is a class attribute,
and `line` is an instance attribute.

```python
class Speaker:
    volume = "low"
    
    def __init__(self, line):
        self.line = line
    
    def speak(self):
        return line if self.volume == "low" else line.upper()
```

Each instance of the class has its own `line` attribute.
It's easy to see why calling `a.speak()` below returns `"hello"`
while `b.speak()` returns `"goodbye"`:

```python
>>> a = Speaker("hello")
>>> b = Speaker("goodbye")
>>> a.speak()
"hello"
>>> b.speak()
"goodbye"
```

The class attribute `volume` is shared by all instances.
If you change its value, that change is visible to both `a` and `b`:

```python
>>> Speaker.volume = "high"
>>> a.speak()
'HELLO'
>>> b.speak()
'GOODBYE'
```

Each instance has an _instance dictionary_ where instance
attributes are stored:

```python
>>> a.__dict__
{'line': 'hello'}
>>> b.__dict__
{'line': 'goodbye'}
```

Class attributes are stored in a _class dictionary_:

```python
>>> Speaker.__dict__
mappingproxy({'__module__': '__main__',
              'volume': 'low',
              '__init__': <function __main__.Speaker.__init__(self, line)>,
              'speak': <function __main__.Speaker.speak(self)>,
              '__dict__': <attribute '__dict__' of 'Speaker' objects>,
              '__weakref__': <attribute '__weakref__' of 'Speaker' objects>,
              '__doc__': None})
```

It's important to remember this distinction between
instance and class attributes (and where they are stored).

## Descriptors

Descriptors are an important concept related to attribute access.
A descriptor is a class that defines one or more of the following methods:

1. `__get__()`,
2. `__set__()`,
3. or `__delete__()`

Below is a simple descriptor class. Its `__get__()` method always returns 0.

```python
class ZeroAttribute:
    """
    Attribute that is always 0
    """
    def __get__(self, obj, owner=None):
        return 0
```

Descriptors are only useful as class variables:

```python
class Foo:
    x = ZeroAttribute()
```

Accessing `Foo.` will run its `__get__()` method:

```python
>>> Foo.x  # calls ZeroAttribute.__get__()
0
```

Even though `x` was defined like a class attribute,
you can also access `x` as an _instance_ attribute,
which _also_ invokes the `ZeroAttribute.__get__()` method:

```python
>>> a = Foo()
>>> a.x  # calls ZeroAttribute.__get__()
0
```

### Arguments to the `__get__()` method
{: .no_toc}

The `__get__()` method accepts two arguments, `obj` and `owner`.

* If the `__get__()` method is called by accessing a _class_ attribute,
  `obj` is set to `None`, and `owner` is set to the class.

* If the `__get__()` method is called by accessing an _instance_ attribute,
  `obj` is set to the instance, and `owner` is set to the type of the instance.

This lets you specify different behaviour
for class attribute access and instance attribute access.
For example, if you explicitly don't want to allow class attribute access,
you can do something like this:

```python
class ZeroAttribute:
    """
    Attribute that is always 0
    """
    def __get__(self, obj, owner=None):
        if obj is None:
            raise AttributeError()  # don't allow accessing as a class attribute
        return 0

class Foo:
    x = ZeroAttribute()
```

```python
>>> Foo.x  # accessing as class attribute, will raise
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 7, in __get__
AttributeError
>>>
>>> a = Foo()
>>> a.x  # accessing as instance attribute, OK
0
```

### Data descriptors and non-data descriptors
{: .no_toc}

A descriptor that only defines `__get__()` is called a _non-data descriptor_.
A descriptor that also defines `__set__()` or `__delete__()` is called a _data_ descriptor.
As you'll see in the next section, this is an important difference to remember.

## Customizing attribute access

Now that you know about class and instance attributes,
and a little bit about descriptors,
you're ready to understand how attribute access _really_ works in Python,
and how to customize it.

When you write `x.y`, the `__getattribute__()` method of `x` is invoked.
The default implementation of `__getattribute__()` does the following:

  - First, it checks if `y` is a _data_ descriptor.
    If so, it returns the result of its `__get__()` method.
  - Next, it tries to find `'y'` in the instance dictionary of `x` and return it.
  - Next, it checks if `y` is a _non_data_ descriptor.
    If so, it returns the result of its `__get__()` method.
  - Next, it tries to find `'y'` in the class dictionary of the type of `x` and return it.
  - Finally, if none of the above worked, it raises an `AttributeError`.
  
If  `__getattribute__()` raises an `AttributeError`,
`x.__getattr__()` is called _if it is defined_.

So you can control the way attribute access works in a few different ways:

* Override `__getattribute__`
* Write a `__getattr__`
* Make the attribute a descriptor object

Let's look at a some examples that will help understand when to use which.

## Example 1: returning `None` when an attribute isn't found

By default, accessing an attribute that doesn't exist gives you an `AttributeError`:

```python
>>> class MyClass: 
...     def __init__(self, x):
...         self.x = x
...
>>> obj = MyClass(42)
>>> obj.x
42
>>> obj.y
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'MyClass' object has no attribute 'y'
```

Let's say we want to get `None` (or some other custom behaviour) when we access
non-existent attributes.

### Overriding `__getattribute__()` - the wrong way

The first approach we might consider is overriding `__getattribute__()`.
Try to spot the problem with the code below:

```python
class MyClass:
    def __init__(self, x):
        self.x = x
        
    def __getattribute__(self, x):
        try:
            return self.__dict__[x]
        except KeyError:
            return None
```

If we try to access a non-existent attribute of a `MyClass` instance,
we get a `RecursionError`!

```python
>>> obj = MyClass(42)
>>> obj.y
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 6, in __getattribute__
  File "<stdin>", line 6, in __getattribute__
  File "<stdin>", line 6, in __getattribute__
  [Previous line repeated 996 more times]
RecursionError: maximum recursion depth exceeded
```

The problem is this line in `MyClass.__getattribute__()`,
which itself calls `MyClass.__getattribute__()`:

```python
return self.__dict__[x]  # calls self.__getattr__("__dict__") - infinite recursion!
```

### Overriding `__getattribute__()` - the less wrong, but still wrong, way

To prevent `MyClass.__getattribute__()` from calling itself recursively,
we could use the base class', i.e., `object`'s implementation of
`__getattribute__()` instead. If that fails, we return `None`:

```python
class MyClass:
    def __init__(self, x):
        self.x = x
        
    def __getattribute__(self, name):
        try:
            return object.__getattribute__(self, name)
        except AttributeError:
            return None
```

This gives us the behaviour we want:

```python
>>> obj = MyClass(42)
>>> obj.x
42
>>> obj.y  # no error, returns None
>>>
```

### Overriding `__getattr__` - the right approach

The more elegant solution is simply
to override `__getattr__`.
Recall that the default implementation of
`__getattribute__` will raise `AttributeError`
when it doesn't find an attribute.
When that happens, `__getattr__` is called:

```python
class MyClass:
    def __init__(self, x):
        self.x =x
        
    def __getattr__(self, name):
        return None
```

```python
>>> obj = MyClass(42)
>>> obj.x  # OK - __getattribute__ will find attribue 'x'
42
>>> obj.y  # OK - __getattr__ is invoked and returns None
```

## Example 2 - an attribute that updates itself when accessed

As another example,
consider writing an attribute that is updated every time you access it:

```python
>>> obj = MyClass()
>>> obj.x
0
>>> obj.x
1
>>> obj.x
2
```

### Using a non-data descriptor

When you want to control the access behaviour of a specific attribute,
a descriptor is generally the right tool for the job.

The `IncrementingAttribute.__get__()` method below returns 0 the first time
it is called for an instance.
Subsequently, it returns 1, 2, 3, etc.
It does this by storing an internal attribute `_value` in the instance.

```python
class IncrementingAttribute:
    def __get__(self, obj, owner=None):
        # rdon't allowaccessing as a class attribute
        if obj is None:
            raise AttributeError()
            
        # if accessing for the first time, return 0,
        # otherwise, increment by 1 and return the result
        if not hasattr(obj, "_value"):
            obj._value = -1
        obj._value += 1
        return obj._value

class MyClass:
    x = IncrementingAttribute()
```

As expected, the `x` is updated each time it is accessed:

```
```python
>>> obj = MyClass()
>>> obj.x
0
>>> obj.x
1
```

What happens if we "reset" the value of `x` and then try to access it?

```python
>>> obj.x = 0
>>> obj.x
0
>>> obj.x
0
>>> obj.x
0
```

Oh no! The `x` has somehow lost the ability to update itself.
To understand why,
it's first important to understand what happens when the following line is executed:

```python
obj.x =  0
```

Because the descriptor object `MyClass.x` does not define a `__set__()` method,
this line will fall back to the default behaviour of setting an attribute.
That is, it will add an entry `'x'` to the instance dictionary of `obj`.
You can see this by inspecting the `__dict__` attribute before and after
setting the attribute `x`:

```python
>>> obj.__dict__
{'_value': 2}
>>> obj.x = 1   # adds entry 'x' to obj.__dict__
{'_value': 2, 'x': 1}
```

Also recall that `x` is a _non-data descriptor_, that is,
the descriptor class `IncrementingAttribute` only defines a `__get__()` method.
Because `__getattribute__` looks in the instance dictionary _before_
checking for non-data descriptors,
it finds `x` in the instance dictionary and returns that.

### Using a data descriptor

The solution to the above problem is to also define a `__set__()` method in our descriptor class:

```python
class IncrementingAttribute:
    def __get__(self, obj, owner=None):
        # don't allowaccessing as a class attribute
        if obj is None:
            raise AttributeError()
        # if accessing for the first time, return 0,
        # otherwise, increment by 1 and return the result
        if not hasattr(obj, "_value"):
            obj._value = -1
        obj._value += 1
        return obj._value
        
    def __set__(self, obj, value):
        obj._value = value - 1

class MyClass:
    x = IncrementingAttribute()
```

Now, we get the expected reset behaviour:

```python
>>> obj = MyClass()
>>> obj.x
0
>>> obj.x
1
>>> obj.x = 0
>>> obj.x
0
>>> obj.x
1
```

### What about `property`?

You could also use a `property` to achieve the same result:

```python
class MyClass:
    @property
    def x(self):
        if not hasattr(self, "_value"):
            self._value = -1
        self._value += 1
        return self._value

    @x.setter
    def x(self, value):
        self._value = value - 1
```

In fact, `@property` is really just a way to define a (data) descriptor!
So if you find yourself writing properties that all look the same
(either in the same class or across different classes),
that's a sign that you should write a descriptor instead.

## Further reading

If you're looking for a detailed "Introduction to descriptors",
or examples of how descriptors can be used,
see the
[Descriptor HowTo](https://docs.python.org/3/howto/descriptor.html)
by Raymond Hettinger.
It's one of my favourite parts of the official Python docs!
