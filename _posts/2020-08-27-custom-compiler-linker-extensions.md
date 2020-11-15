---
layout: post
title: Customizing the compiler and linker used by setuptools
---

This post is about how to customize the compiler and linker used
when building a C/C++ extension for Python using setuptools.


I recently ran into the problem of configuring setuptools to use a custom
compiler and linker, and these are my notes on how I did that.

As an example, here is a simple `setup.py` for building the extension `spam` from
`spammodule.c`:

```python
from setuptools import setup

setup(
    name="spam",
    ext_modules=[Extension("spam", sources=["spammodule.c"])], 
    zip_safe=False
)
```

Invoking `setup.py`, we can see the compiler and flags used for building the extension.
On my Mac:

```bash
$ python setup.py build_ext --inplace

running build_ext
building '*' extension
creating build
creating build/temp.macosx-10.9-x86_64-3.7
gcc -Wno-unused-result -Wsign-compare -Wunreachable-code -DNDEBUG -g -fwrapv -O3 ...
```

Using a different compiler than `gcc` is easy -- just set the value
of the `CC` environment variable:

```bash
$ CC=g++ python setup.py build_ext --inplace

running build_ext
building 'spam' extension
creating build
creating build/temp.macosx-10.9-x86_64-3.7
g++ -Wno-unused-result -Wsign-compare -Wunreachable-code -DNDEBUG -g -fwrapv -O3 ...
```

The compiler has changed from `gcc` to `g++`, but we notice that the
arguments passed to the compiler remain the same.
This can be a problem if you want to use a compiler that doesn't support those argument.

---

**Where do the compiler and linker flags come from?**

If you're wondering where the compiler flags being used
(`-Wno-unsed-result -Wsign-compare ...`) come from,
they are set in the configuration used to compile CPython -- see
[here](https://github.com/python/cpython/blob/master/configure.ac).

You can use `distutils.sysconfig` to inspect their values:

```python
>>> import distutils.sysconfig
>>> distutils.sysconfig.get_config_var("CFLAGS")
```

Also see `distutils.sysconfig.get_config_vars()` to get the values of _all_
configuration options.

---

You can override these flags entirely and pass your own flags using
the `extra_compile_args` and `extra_link_args` options, as follows:


```python
from setuptools.command.build_ext import build_ext
from setuptools import Extension, setup


class custom_build_ext(build_ext):
    def build_extensions(self):
        # Override the compiler executables. Importantly, this
        # removes the "default" compiler flags that would
        # otherwise get passed on to to the compiler, i.e.,
        # distutils.sysconfig.get_var("CFLAGS").
        self.compiler.set_executable("compiler_so", "g++")
        self.compiler.set_executable("compiler_cxx", "g++")
        self.compiler.set_executable("linker_so", "g++")
        build_ext.build_extensions(self)

[
setup(
    name="spam",
    ext_modules=[
        Extension(
            "spam", 
            sources=["spammodule.c"],
            extra_compile_args=["-arch", "x86_64"],
            extra_link_args=["-undefined", "dynamic_lookup"]
        )
    ],
    zip_safe=False,
    cmdclass={"build_ext": custom_build_ext}
)
```

Now we see that we have full control over the flags passed
to the compiler and linker commands:

```bash
$ python setup.py build_ext --inplace
running build_ext
building 'spam' extension
creating build
creating build/temp.macosx-10.9-x86_64-3.7
g++ -I/Users/ashwin/miniconda3/envs/sci37/include/python3.7m -c spammodule.c -o build/temp.macosx-10.9-x86_64-3.7/spammodule.o -arch x86_64
clang: warning: treating 'c' input as 'c++' when in C++ mode, this behavior is deprecated [-Wdeprecated]
creating build/lib.macosx-10.9-x86_64-3.7
g++ -arch x86_64 build/temp.macosx-10.9-x86_64-3.7/spammodule.o -o build/lib.macosx-10.9-x86_64-3.7/spam.cpython-37m-darwin.so -undefined dynamic_lookup
copying build/lib.macosx-10.9-x86_64-3.7/spam.cpython-37m-darwin.so -> ~
```
