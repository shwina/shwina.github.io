---
layout: post
title: Customizing the compiler and linker when building C/C++ extensions for Python
---

This post is about how to customize the compiler and linker used
when building a C/C++ extension.


Sometimes, you want to build an extension module using a different compiler than
`gcc` or whatever compiler is used by setuptools to build extensions.
Additionally, you may want to pass custom flags to the compiler/linker.
Importantly, you may want to prevent the "default" flags from being passed to
your custom compiler,
perhaps because it doesn't support them. This post shows one way of achieving just that.


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
gcc -Wno-unused-result -Wsign-compare -Wunreachable-code -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -I/Users/ashwin/miniconda3/envs/sci37/include -arch x86_64 -I/Users/ashwin/miniconda3/envs/sci37/include -arch x86_64 -I/Users/ashwin/miniconda3/envs/sci37/include/python3.7m -c spammodule.c -o build/temp.macosx-10.9-x86_64-3.7/spammodule.o
creating build/lib.macosx-10.9-x86_64-3.7
gcc -bundle -undefined dynamic_lookup -L/Users/ashwin/miniconda3/envs/sci37/lib -arch x86_64 -L/Users/ashwin/miniconda3/envs/sci37/lib -arch x86_64 -arch x86_64 build/temp.macosx-10.9-x86_64-3.7/spammodule.o -o build/lib.macosx-10.9-x86_64-3.7/*.cpython-37m-darwin.so
```

Using a different compiler than `gcc` is easy -- just set the value
of the `CC` environment variable.
Just for illustration, we'll consider changing the compiler from `gcc` to `g++`:


```bash
running build_ext
building 'spam' extension
creating build
creating build/temp.macosx-10.9-x86_64-3.7
g++ -Wno-unused-result -Wsign-compare -Wunreachable-code -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -I/Users/ashwin/miniconda3/envs/sci37/include -arch x86_64 -I/Users/ashwin/miniconda3/envs/sci37/include -arch x86_64 -I/Users/ashwin/miniconda3/envs/sci37/include/python3.7m -c spammodule.c -o build/temp.macosx-10.9-x86_64-3.7/spammodule.o
clang: warning: treating 'c' input as 'c++' when in C++ mode, this behavior is deprecated [-Wdeprecated]
creating build/lib.macosx-10.9-x86_64-3.7
g++ -bundle -undefined dynamic_lookup -L/Users/ashwin/miniconda3/envs/sci37/lib -arch x86_64 -L/Users/ashwin/miniconda3/envs/sci37/lib -arch x86_64 -arch x86_64 build/temp.macosx-10.9-x86_64-3.7/spammodule.o -o build/lib.macosx-10.9-x86_64-3.7/spam.cpython-37m-darwin.so
copying build/lib.macosx-10.9-x86_64-3.7/spam.cpython-37m-darwin.so ->
```

The compiler has indeed changed from `gcc` to `g++`, but we notice that the
arguments to the compiler remain the same as before.
This can be a problem if the compiler you want to use doesn't support one or more 
of these arguments.

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

One way to override these flags entirely is to set the compiler/linker executables
within distitutils as follows:

```python
from setuptools.command.build_ext import build_ext
from setuptools import Extension, setup


class custom_build_ext(build_ext):
    def build_extensions(self):
        # Override the compiler executables. Importantly, this
        # removes the "default" compiler flags that would
        # otherwise get passed on to to the compiler, i.e.,
        # distutils.sysconfig.get_var("CFLAGS").
        breakpoint()
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

Doing this erases *all* the compiler flags passed to the compiler and linker,
so any flags you wish to pass must be done so via the
`extra_compiler_args` and `extra_link_args` options.

Now we see that we have full control over the flags passed to the compiler and linker commands:

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
