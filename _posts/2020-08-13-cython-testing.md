---
layout: post
title: "Using pytest with Cython"
---

This post is about using `pytest` for testing `cdef` functions in Cython.


## The problem

Consider the `primes` function below,
which returns the first `n` prime numbers as a C++ vector:

```cython
# file: foo.pxd

cdef vector[int] primes(unsigned int nb_primes)
```

How can we write tests for `primes`
in a way that they are automatically discovered and run
by a test runner such as `pytest`?

---
**Note**: the `.pxd` file above contains only the "definition" of `primes`.
Here is the actual implementation:

```cython
# file: foo.pyx

# distutils: language=c++
# distutils: extra_compile_args=-std=c++11

from libcpp.vector cimport vector

cdef vector[int] primes(unsigned int nb_primes):
    cdef int n, i
    cdef vector[int] p
    p.reserve(nb_primes)  # allocate memory for 'nb_primes' elements.

    n = 2
    while p.size() < nb_primes:  # size() for vectors is similar to len()
        for i in p:
            if n % i == 0:
                break
        else:
            p.push_back(n)  # push_back is similar to append()
        n += 1

    # Vectors are automatically converted to Python
    # lists when converted to Python objects.
    return p
```
---

## Writing the tests

`cdef` functions can only be called from Cython code.
Thus, tests for `cdef` functions must be written in Cython.
Let's write a very simple test for `primes` in a separate file `foo_tests.pyx`:

```cython
# file: foo_tests.pyx

# distutils: language=c++
# distutils: extra_compile_args=-std=c++11

from foo cimport primes

def test_primes(data, expect):
    assert primes(0) == []
```

Compiling both using Cython:

```bash
$ cythonize -i foo.pyx foo_tests.pyx
```

At this point, running `pytest` doesn't do anything:

```bash
$ pytest

====================== no tests ran in 0.02s ======================
```

## Making tests discoverable

To make our Cython tests discoverable, we can make them attributes of a
Python module `test_foo.py` as follows:

```python
# file: test_foo.py

import importlib
import sys

# list of Cython modules containing tests
cython_test_modules = ["foo_tests"]

for mod in cython_test_modules:
    try:
        # For each callable in `mod` with name `test_*`,
        # set the result as an attribute of this module.
        mod = importlib.import_module(mod)
        for name in dir(mod):
            item = getattr(mod, name)
            if callable(item) and name.startswith("test_"):
                setattr(sys.modules[__name__], name, item)
    except ImportError:
        pass
```

Now, our tests are discovered by `pytest` but they are not actually run:

```bash
$ pytest
======================= test session starts =======================
collected 0 items                                                          

===================== warnings summary ============================
foo_tests.pyx:19
  foo_tests.pyx:19: PytestCollectionWarning: cannot collect 'test_primes' because it is not a function.
    def test_primes(data, expect):

-- Docs: https://docs.pytest.org/en/latest/warnings.html
====================== 1 warning in 0.02s =========================
```

## Making tests runnable

A function from a Cython module is not a "regular" Python function,
and `pytest` refuses to collect it:

```python
>>> import inspect
>>> from foo_tests import test_primes   
>>> type(test_primes)                   
_cython_3_0a5.cython_function_or_method
>>> inspect.isfunction(test_primes)
False
```

To get around this, we can wrap `test_primes` in a simple decorator that turns it
into a plain Python function:

```python
def cytest(func):
    """
    Wraps `func` in a plain Python function.
    """

    @functools.wraps(func)
    def wrapped(*args, **kwargs):
        bound = inspect.signature(func).bind(*args, **kwargs)
        return func(*bound.args, **bound.kwargs)

    return wrapped
```

Wrapping `test_primes` with the `cytest` decorator, we see that it now behaves
like a regular Python function, and thus is collected and run by `pytest`:

```python
@cytest
def test_primes(data, expect):
    assert primes(0) == []
```

```
>>> from foo_tests import test_primes
>>> inspect.isfunction(test_primes)
True
```

```bash
$ pytest
======================= test session starts =======================
test_foo.py . 
======================= 1 passed in 0.02s =========================
```

## Parametrizing tests

In case you're wondering: Yes! It is possible to parametrize tests with this
method:

```python
@pytest.mark.parametrize(
    "data,expect",
    [
        (0, []),
        (1, [1]),
        (2, [1, 3]),
        (3, [1, 3, 5])
    ]
)
@cytest
def test_primes(data, expect):
    assert primes(0) == []
```

```bash
$ pytest
======================= test session starts =======================
test_foo.py ....
======================= 4 passed in 0.16s =========================
```

## Summary

To summarize, it is possible to use `pytest` to test Cython code,
using the following approach:

1. Write the tests in a Cython module (e.g., `foo_tests.pyx`),
   wrapping each test function in the `@cytest` decorator shown above.

2. Create a Python module (e.g., `test_foo.py`) that imports the test
   functions in `foo_tests.pyx`, setting them as its attributes.
   If you have several Cython modules containing tests, it might be
   convenient to have a single `test_cython.py` module that imports
   tests from all of them.

3. Run `pytest` as usual.

[tutorial]: https://cython.readthedocs.io/en/latest/src/tutorial/cython_tutorial.html#primes-with-c


